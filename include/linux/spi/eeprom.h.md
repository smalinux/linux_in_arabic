## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ EEPROM؟

تخيل إنك عندك جهاز صغير — راوتر، كاميرا، ترموستات — وعايز تحفظ فيه إعدادات بسيطة زي: الـ MAC address، أو calibration data لحساس درجة حرارة، أو رقم سيريال. محتاج ذاكرة صغيرة بتفضل محفوظة حتى لو الكهرباء اتقطعت. ده بالظبط دور الـ **EEPROM** — *Electrically Erasable Programmable Read-Only Memory*.

الـ EEPROM ده زي دفتر ملاحظات صغير: تقدر تكتب فيه، تمسح، وتقرأ، والمعلومات مبتتمسحش لو النور اتقطع. حجمه صغير جداً (من بضع bytes لـ 2 MB)، لكنه كفيل لحفظ الإعدادات الضرورية.

---

### الـ SPI إيه دخله؟

الـ **SPI** هي *Serial Peripheral Interface* — بروتوكول تواصل سلكي بسيط. بدل ما الـ CPU يكلم الـ EEPROM بـ 8 أسلاك (parallel)، بيكلمه بـ 4 أسلاك بس:
- `MOSI` — data من الـ CPU للـ EEPROM
- `MISO` — data من الـ EEPROM للـ CPU
- `SCLK` — clock signal
- `CS` — chip select (عشان تختار الـ chip دي بالذات)

الفكرة زي ما بتبعت رسالة واحدة حرف حرف على التليفون بدل ما تصرخ كل الكلمة دفعة واحدة.

---

### الـ AT25 — الـ Standard الأشهر

معظم الـ SPI EEPROMs بتفهم نفس مجموعة أوامر أساسية. **Atmel** عملت الـ **AT25** كـ standard chip، وبعدين كل الشركات التانية (Microchip, ST, Cypress...) اتبعت نفس الـ command set. يعني driver واحد يشغّل chips من vendor مختلفين.

الأوامر الأساسية:
| الأمر | الكود | المعنى |
|-------|--------|---------|
| `READ` | `0x03` | اقرأ bytes |
| `WRITE` | `0x02` | اكتب bytes |
| `WREN` | `0x06` | فعّل الكتابة |
| `RDSR` | `0x05` | اقرأ Status Register |

---

### المشكلة اللي الـ file بيحلها

عندنا chips SPI EEPROM من vendors مختلفة — بأحجام مختلفة، وبـ address bits مختلفة. الـ kernel driver (`at25.c`) محتاج يعرف:

1. **حجم الـ chip** — كم byte في المجموع؟
2. **حجم الـ page** — الكتابة بتتم في pages، لازم يعرف الحجم عشان ميكتبش أكتر من page في write واحد.
3. **عدد address bytes** — chip صغيرة (أقل من 256 bytes) محتاجة بس byte واحد للعنوان، أما chip كبيرة (> 64KB) محتاجة 3 bytes.
4. **خصائص خاصة** — زي الـ read-only mode أو الـ `EE_INSTR_BIT3_IS_ADDR` اللي بنشرحه بعد كده.

**الـ `struct spi_eeprom` في الـ file ده هو الـ contract** — بتملاه في الـ board setup code، وبتديه للـ driver عشان يعرف هو شغال مع chip عاملة إزاي.

---

### الـ `EE_INSTR_BIT3_IS_ADDR` — لماذا موجودة؟

في مشكلة طريفة: الـ chip **M95040** من ST مثلاً حجمها 512 bytes — يعني محتاجة 9 bits للعنوان (0–511). لكن الـ address field في الأمر بيحمل 8 bits بس. حل هذه المشكلة؟ الـ bit الإضافي (A8) بيتحط في الـ bit 3 من الـ instruction byte نفسه.

```
Normal READ:  [0x03][ADDR_HIGH][ADDR_LOW]   <-- 16-bit address
M95040 READ:  [0x0B][ADDR_LOW]              <-- bit3 في الأمر = A8
              (0x03 | 0x08 = 0x0B)
```

---

### قصة الـ Platform Data

زمان قبل Device Tree، الـ board code كانت بيتعلّم الـ kernel بـ chips المتصلة. الطريقة كانت:

```c
/* Board setup code (e.g. arch/arm/mach-xxx/board-xxx.c) */
static struct spi_eeprom at25_chip = {
    .name      = "at25256",   /* chip name */
    .byte_len  = 32768,       /* 256 kbit = 32 KB */
    .page_size = 64,          /* write in 64-byte pages */
    .flags     = EE_ADDR2,    /* 2-byte (16-bit) addresses */
};

static struct spi_board_info board_spi_devices[] = {
    {
        .modalias        = "at25",
        .platform_data   = &at25_chip,  /* هنا بيتمرر الـ struct */
        .chip_select     = 0,
        .max_speed_hz    = 20000000,
    },
};
```

الـ driver بياخد `platform_data` ويعرف كل حاجة يحتاجها.

---

### الـ Subsystem

الـ file ده جزء من **SPI EEPROM / NVMEM subsystem**. اللي يشمل:

- الـ **SPI core** — اللي بيوفر infrastructure التواصل
- الـ **NVMEM subsystem** — Non-Volatile Memory، بيوفر unified API للوصول للذاكرة غير المتطايرة
- الـ **at25 driver** — الـ driver الفعلي

---

### الملفات المكوّنة لهذا الـ Subsystem

#### Core Headers

| الملف | الدور |
|-------|-------|
| `include/linux/spi/eeprom.h` | **الملف ده** — الـ platform data struct لـ SPI EEPROMs |
| `include/linux/spi/spi.h` | الـ SPI core API |
| `include/linux/spi/spi-mem.h` | الـ SPI memory interface (أبسط API للـ memory-like devices) |
| `include/linux/nvmem-provider.h` | NVMEM subsystem API للـ providers |

#### الـ Driver الرئيسي

| الملف | الدور |
|-------|-------|
| `drivers/misc/eeprom/at25.c` | الـ driver الوحيد اللي بيستخدم `spi_eeprom` struct — بيدعم Atmel AT25، Cypress FM25 FRAMs، وأغلب SPI EEPROMs المتوافقة |

#### ملفات ذات صلة في نفس الـ subsystem

| الملف | الدور |
|-------|-------|
| `drivers/misc/eeprom/Kconfig` | إعدادات build للـ EEPROM drivers (`EEPROM_AT25`) |
| `drivers/misc/eeprom/at24.c` | نظير الـ at25 لكن على I2C بدل SPI |
| `drivers/misc/eeprom/eeprom_93xx46.c` | Microwire EEPROM driver (بروتوكول تاني) |
| `drivers/spi/spi.c` | SPI core implementation |

---

### الصورة الكاملة في سطر واحد

**الـ `include/linux/spi/eeprom.h`** هو الـ "بطاقة تعريف" لأي SPI EEPROM chip — بتحدد حجمها، طريقة عنونتها، وخصائصها الخاصة، عشان الـ `at25` driver يقدر يتعامل معاها بشكل صحيح بغض النظر عن الـ vendor.
## Phase 2: شرح الـ SPI EEPROM (at25) Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

الـ **EEPROM** (Electrically Erasable Programmable Read-Only Memory) هو نوع من الذاكرة غير المتطايرة (non-volatile) بتُستخدم في الـ embedded systems لتخزين البيانات الصغيرة زي: الـ calibration data، الـ MAC addresses، الـ configuration parameters، والـ bootloader environment variables.

معظم الـ SPI EEPROMs — من شركات زي Atmel (AT25xxx)، ST (M95xxx)، Microchip (25AAxxx) — بتشترك في نفس الـ command set:

| Command | Opcode | وظيفته |
|---------|--------|--------|
| `READ`  | `0x03` | قراءة بيانات |
| `WRITE` | `0x02` | كتابة بيانات |
| `WREN`  | `0x06` | تفعيل الكتابة |
| `RDSR`  | `0x05` | قراءة الـ status register |

**المشكلة الحقيقية:** الـ EEPROMs دي بتختلف في تفصيلة واحدة محورية — **عدد بايتات الـ address**. بعضها يعنوّن بـ 8-bit، وبعضها 16-bit، وبعضها 24-bit. كمان بعضها عنده حيلة غريبة: حجمه أكبر من اللي عدد بايتات الـ address بتاعه يستوعبه — فبيحتاج يحشر البت الإضافي في الـ instruction byte نفسه.

لو كل driver كتب نفس الـ SPI transaction logic من الصفر ده هيبقى **code duplication** صريح — ومشكلة صيانة ضخمة.

---

### الحل — الـ kernel بيعمل إيه؟

الـ kernel بيحل المشكلة بـ driver واحد اسمه `at25` (في `drivers/misc/eeprom/at25.c`) بيتعامل مع كل الـ SPI EEPROMs اللي بتفهم الـ AT25-compatible command set. الـ driver ده محتاج يعرف specs الـ chip بالتحديد — وده بالظبط دور الـ `struct spi_eeprom`.

الـ `struct spi_eeprom` هي **platform data** — معلومات بتتبعت من الـ board code (أو الـ device tree) للـ driver عشان يعرف يتكلم مع الـ chip الصح.

---

### التشبيه الحقيقي — وظيفة الـ driver كأنه موظف استقبال

تخيل إنك بتشغّل فندق (= الـ SPI bus) وعندك غرف مختلفة (= الـ EEPROM chips). الموظف (= الـ `at25` driver) بيتعامل مع كل النزلاء بنفس الأسلوب (نفس الـ SPI commands)، لكن محتاج **ورقة بيانات** عن كل غرفة (= `struct spi_eeprom`) تقوله:

| حقل في الـ struct | المعادل في التشبيه |
|-------------------|--------------------|
| `byte_len`        | حجم الغرفة (كام متر) |
| `page_size`       | كام نزيل تقدر تدخّل دفعة واحدة |
| `name`            | اسم الغرفة |
| `flags` (EE_ADDR1/2/3) | نوع القفل — هل المفتاح 1 رقم أو 2 أو 3 |
| `EE_READONLY`     | الغرفة محجوزة للمشاهدة فقط |
| `EE_INSTR_BIT3_IS_ADDR` | قفل خاص — الرقم السري الإضافي مخبّي في نفس المفتاح الأساسي |
| `context`         | ملاحظة خاصة من مدير الفندق |

---

### الـ Big Picture Architecture

```
+---------------------------------------------------+
|              Board / Platform Code                |
|  (arch/arm/mach-xxx/ أو Device Tree)             |
|                                                   |
|  struct spi_eeprom my_eeprom = {                 |
|      .byte_len  = 512,                           |
|      .page_size = 16,                            |
|      .flags     = EE_ADDR1,                      |
|  };                                               |
+-------------------+-------------------------------+
                    |  platform_data
                    v
+---------------------------------------------------+
|           SPI Core (drivers/spi/spi.c)           |
|                                                   |
|   struct spi_device {                            |
|       struct device dev;       <── bus model     |
|       struct spi_controller *controller;         |
|       void *controller_data;   <── board data    |
|   }                                               |
|                                                   |
|   spi_bus_type (linux/spi/spi.h)                 |
+--------+------------------+------------------------+
         |                  |
         | spi_device       | spi_transfer / spi_message
         v                  v
+------------------+  +----------------------------+
|  at25 Driver     |  |  SPI Controller Driver     |
| (drivers/misc/   |  |  (e.g. drivers/spi/        |
|  eeprom/at25.c)  |  |   spi-pl022.c)             |
|                  |  |                            |
| - يفهم AT25 cmds |  | - يتعامل مع الـ hardware   |
| - يستخدم         |  | - يحوّل spi_message لـ     |
|   spi_eeprom     |  |   actual clock pulses      |
|   struct         |  |                            |
+--------+---------+  +----------------------------+
         |
         v
+---------------------------+
|  MTD / NVMEM Subsystem    |
|  (بيوفّر /dev/mtd0        |
|   أو sysfs NVMEM node)    |
+---------------------------+
         |
         v
+---------------------------+
|  User / Other Drivers     |
|  (يقرؤوا/يكتبوا الـ EEPROM)|
+---------------------------+
```

---

### الـ Core Abstraction — الفكرة المحورية

الفكرة المحورية هي **فصل الـ chip description عن الـ protocol logic**:

- **الـ `struct spi_eeprom`** = وصف ثابت للـ hardware (بيجي من الـ board code أو الـ DT).
- **الـ `at25` driver** = logic متغيرة بتستخدم الوصف ده لتوليد الـ SPI transactions الصح.

ده مبدأ الـ **data-driven driver** — بدل ما تكتب driver لكل chip، بتكتب driver واحد وبتديه data تصف الـ chip.

---

### تشريح الـ `struct spi_eeprom`

```c
struct spi_eeprom {
    u32  byte_len;      /* الحجم الكلي للـ EEPROM بالبايت
                           مثال: 512 لـ M95040، 131072 لـ AT25128 */

    char name[10];      /* اسم الـ chip للـ debugging والـ sysfs
                           مثال: "at25128" */

    u32  page_size;     /* أقصى عدد بايت تكتبهم في عملية write واحدة
                           الـ EEPROM بيشترط إنك تكتب page كاملة
                           أو أجزاء منها بس ما تعبرش الحد */

    u16  flags;
    /* EE_ADDR1 (0x0001): الـ address byte واحد  → max 256 bytes
       EE_ADDR2 (0x0002): الـ address bytes اتنين → max 65536 bytes
       EE_ADDR3 (0x0004): الـ address bytes تلاتة → max 16MB
       EE_READONLY(0x0008): ممنوع الكتابة (write-protect) */

    /* الحالة الخاصة: M95040 عنده 512 byte لكن بيستخدم EE_ADDR1
       فين البت الإضافي (A8)؟ بيتحشر في bit3 من الـ opcode نفسه */
#define EE_INSTR_BIT3_IS_ADDR 0x0010

    void *context;      /* pointer عام لبيانات إضافية
                           الـ driver نفسه ما بيستخدمهوش — بيتركه
                           للـ board code أو الـ caller */
};
```

#### كيف الـ Address Encoding بيشتغل؟

**الحالة العادية (EE_ADDR2):**
```
SPI Transaction لـ READ:
  [WREN opcode=0x03] [ADDR_HIGH] [ADDR_LOW] [data clocked in ...]
  |                  |           |
  1 byte             2 bytes address
```

**الحالة الخاصة (EE_INSTR_BIT3_IS_ADDR مع EE_ADDR1):**
```
M95040 — 512 bytes, address = 9 bits (A0..A8)
  [opcode | (A8 << 3)] [A7..A0] [data ...]
  |                    |
  bit3 في الـ opcode   الـ address العادي
```

ده بيوضح ليه الـ `flags` field ضروري — بدونه الـ driver مش هيعرف يبني الـ SPI transaction الصح.

---

### علاقة الـ `struct spi_eeprom` بالـ `struct spi_device`

**الـ SPI subsystem** (مهم فهمه قبل ما نكمل) بيعرّف `struct spi_device` كـ proxy للـ chip على الـ SPI bus. كل `spi_device` عنده `dev.platform_data` أو بيتعامل معاه من خلال الـ device tree.

```
struct spi_device
│
├── dev.platform_data  ──────────────────► struct spi_eeprom *
│   (الـ board code بيحطها هنا)            (chip-specific params)
│
├── max_speed_hz       ──────────────────► max SPI clock للـ chip
├── mode               ──────────────────► SPI_MODE_0 مثلاً
└── controller         ──────────────────► struct spi_controller *
```

الـ `at25` driver بيعمل:
```c
struct spi_eeprom *chip = spi_get_drvdata(spi)->chip;
/* أو من platform_data */
chip = dev_get_platdata(&spi->dev);
```

---

### رسم العلاقة بين الـ structs

```
Board / DT
    │
    │  تعريف spi_eeprom
    ▼
┌─────────────────────┐
│   struct spi_eeprom │
│  ┌───────────────┐  │
│  │ byte_len: 512 │  │
│  │ page_size: 16 │  │
│  │ flags: ADDR1  │  │
│  │  + INSTR_BIT3 │  │
│  └───────────────┘  │
└──────────┬──────────┘
           │ platform_data
           ▼
┌─────────────────────────────┐
│     struct spi_device        │
│  ┌──────────────────────┐   │
│  │ dev (struct device)  │   │
│  │ controller           │   │
│  │ max_speed_hz         │   │
│  │ mode / bits_per_word │   │
│  └──────────────────────┘   │
└──────────────┬──────────────┘
               │ bيتبعت لـ probe()
               ▼
┌─────────────────────────────┐
│       at25 Driver            │
│  probe():                    │
│  1. اقرأ spi_eeprom من       │
│     platform_data            │
│  2. حدد عدد address bytes    │
│  3. سجّل NVMEM device        │
│                              │
│  read/write():               │
│  1. ابني SPI command         │
│     (opcode + address)       │
│  2. أرسل spi_message         │
└─────────────────────────────┘
```

---

### الـ NVMEM Subsystem — من يستهلك الـ EEPROM؟

**الـ NVMEM subsystem** (بيقدم `/sys/bus/nvmem/devices/`) هو الـ consumer الرئيسي للـ `at25` driver. الـ subsystem ده بيسمح لأي driver تاني يقرأ من الـ EEPROM بطريقة موحدة، مثلاً:

- driver الـ network card بيقرأ الـ MAC address
- driver الـ sensor بيقرأ الـ calibration coefficients
- الـ bootloader بيقرأ الـ environment variables

---

### الـ at25 Driver — إيه اللي بيملكه وإيه اللي بيفوّضه؟

| المسؤولية | الـ at25 Driver | SPI Core / Controller |
|-----------|----------------|----------------------|
| بناء الـ READ/WRITE commands | نعم | لا |
| حساب عدد bytes الـ address | نعم (من flags) | لا |
| إدارة الـ WREN sequence قبل الكتابة | نعم | لا |
| الانتظار بعد الكتابة (polling RDSR) | نعم | لا |
| الـ CS (chip select) toggling | لا | نعم (SPI Core) |
| الـ clock timing والـ mode | لا | نعم (Controller) |
| الـ DMA للنقل | لا | نعم (Controller) |
| تقديم NVMEM interface | نعم | لا |

---

### مثال عملي — board code لـ M95040

```c
/* في board file أو DSDT/DT equivalent */
static struct spi_eeprom m95040_pdata = {
    .byte_len  = 512,           /* 512 bytes total */
    .name      = "m95040",
    .page_size = 16,            /* 16-byte write page */
    .flags     = EE_ADDR1 |     /* single address byte */
                 EE_INSTR_BIT3_IS_ADDR, /* A8 goes into opcode bit3 */
};

static struct spi_board_info my_spi_devices[] = {
    {
        .modalias        = "at25",          /* driver name */
        .platform_data   = &m95040_pdata,
        .max_speed_hz    = 5000000,         /* 5 MHz */
        .chip_select     = 0,
        .mode            = SPI_MODE_0,
    },
};
```

لما الـ `at25` driver بيستلم الـ `probe()`، أول حاجة بيعملها:
```c
/* من at25.c */
at25 = devm_kzalloc(&spi->dev, sizeof(*at25), GFP_KERNEL);
at25->chip = *(struct spi_eeprom *)spi->dev.platform_data;

/* بيحدد عدد address bytes من الـ flags */
if (at25->chip.flags & EE_ADDR1)
    at25->addrlen = 1;
else if (at25->chip.flags & EE_ADDR2)
    at25->addrlen = 2;
else
    at25->addrlen = 3;
```

---

### لماذا الـ `context` Pointer موجود؟

الـ `void *context` هو escape hatch للحالات اللي محتاج فيها تربط الـ `spi_eeprom` بـ data إضافية مش موجودة في الـ struct. مثلاً: لو الـ board عندها logic خاصة بعد الكتابة (زي تشغيل LED أو lock hardware) — بتحطها في `context` وبتجيبها في الـ driver عبر:

```c
struct spi_eeprom *chip = dev_get_platdata(&spi->dev);
struct my_board_data *bdata = chip->context;
```

الـ kernel نفسه ما بيلمسش الـ `context` — ده pure convention بين الـ board code والـ driver.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### ملخص الـ File

الـ file ده بسيط جداً — بيعرّف struct واحدة بس اسمها `spi_eeprom`، وبيتحط في الـ `platform_data` عشان الـ driver اللي اسمه `at25` يعرف خصائص الـ EEPROM chip اللي بيتكلم معاها على الـ SPI bus.

---

### 0. الـ Flags — Cheatsheet

| Flag | القيمة | المعنى |
|------|--------|--------|
| `EE_ADDR1` | `0x0001` | الـ EEPROM بيستخدم **8-bit address** (مساحة أقصاها 256 byte) |
| `EE_ADDR2` | `0x0002` | الـ EEPROM بيستخدم **16-bit address** (مساحة أقصاها 64 KB) |
| `EE_ADDR3` | `0x0004` | الـ EEPROM بيستخدم **24-bit address** (مساحة أقصاها 16 MB) |
| `EE_READONLY` | `0x0008` | الـ chip read-only — الـ driver هيرفض أي write operation |
| `EE_INSTR_BIT3_IS_ADDR` | `0x0010` | الـ bit 3 في الـ instruction byte بيعمل دور extra address bit (A8/A16/A24) — للـ chips زي ST M95040 اللي حجمها أكبر من عدد address bytes |

> ملاحظة: الـ flags دي بتتحط في `spi_eeprom.flags` وبيقرأها الـ `at25` driver وقت الـ probe.

---

### 1. الـ Structs المهمة

#### `struct spi_eeprom`

**الغرض:** بتوصف خصائص EEPROM chip معينة على الـ SPI bus — الحجم والاسم وطريقة العنونة وهل read-only.

```c
struct spi_eeprom {
    u32   byte_len;     /* total size of the chip in bytes */
    char  name[10];     /* human-readable chip name, e.g. "at25256b" */
    u32   page_size;    /* write page size in bytes (for page-write ops) */
    u16   flags;        /* bitmask: EE_ADDR1/2/3, EE_READONLY, etc. */
    void *context;      /* optional: pointer to extra driver-private data */
};
```

**شرح الـ fields:**

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `byte_len` | `u32` | الحجم الكلي للـ chip بالبايت — بيتحسب منه الـ address range |
| `name` | `char[10]` | اسم الـ chip زي `"at25256b"` — للـ debugging والـ sysfs |
| `page_size` | `u32` | أكبر عدد bytes تقدر تكتبهم في operation واحدة — الـ driver بيقسم الـ write requests على حسبه |
| `flags` | `u16` | bitmask بيحدد عدد address bytes ونوع العنونة وحماية الكتابة |
| `context` | `void *` | pointer اختياري لبيانات إضافية — الـ driver بيستخدمه لو محتاج custom logic |

---

### 2. علاقة الـ Structs

الـ `spi_eeprom` بتتحط في الـ `platform_data` اللي جوه الـ `spi_board_info` أو بتيجي عن طريق الـ device tree. الـ `at25` driver بيجيبها من `spi_device`.

```
┌─────────────────────────────────┐
│        spi_board_info           │
│  ┌──────────────────────────┐   │
│  │  platform_data ──────────┼───┼──► struct spi_eeprom
│  └──────────────────────────┘   │         │
└─────────────────────────────────┘         │
           │                                │
           ▼                                ▼
    struct spi_device              ┌────────────────┐
    (kernel runtime object)        │   byte_len     │
           │                       │   name[10]     │
           ▼                       │   page_size    │
    at25 driver probe()            │   flags        │
    بيقرأ platform_data            │   *context     │
    ويبني الـ mtd device           └────────────────┘
```

---

### 3. Lifecycle Diagram

```
Board Init / Device Tree
         │
         │  بيتعرّف struct spi_eeprom في platform data
         ▼
  spi_register_board_info()
  أو OF (device tree) parsing
         │
         ▼
  spi_new_device()  ──►  struct spi_device يتعمل
         │                (بيحمل pointer للـ spi_eeprom في platform_data)
         │
         ▼
  at25_probe()
    │
    ├── يجيب spi_eeprom من dev->platform_data أو device tree
    ├── يتحقق من الـ flags (ADDR1/2/3، READONLY)
    ├── يحسب address size من byte_len + flags
    ├── يعمل mtd_info ويسجله كـ MTD device
    └── الـ chip جاهز للقراءة والكتابة
         │
         ▼
  Runtime Operations
    ├── mtd_read()  → at25_ee_read()  → SPI READ command + address bytes
    └── mtd_write() → at25_ee_write() → SPI WRITE command + page_size chunks
         │
         ▼
  at25_remove()
    └── mtd_device_unregister() + تحرير الـ resources
```

---

### 4. Call Flow Diagrams

#### قراءة بيانات من الـ EEPROM

```
user/kernel calls mtd_read(mtd, offset, len, buf)
  │
  └─► at25_ee_read(mtd, from, len, retlen, buf)
        │
        ├── يحسب عدد address bytes من spi_eeprom.flags
        │     EE_ADDR1 → 1 byte
        │     EE_ADDR2 → 2 bytes
        │     EE_ADDR3 → 3 bytes
        │
        ├── يبني SPI command buffer:
        │     [AT25_READ][addr_high?][addr_mid?][addr_low]
        │
        ├── لو EE_INSTR_BIT3_IS_ADDR:
        │     يحط الـ bit 8/16/24 في bit 3 من الـ instruction byte
        │
        └─► spi_sync(spi, &msg)
              └─► SPI hardware: SCLK, MOSI, MISO
                    └─► EEPROM chip responds with data
```

#### كتابة بيانات للـ EEPROM

```
user/kernel calls mtd_write(mtd, offset, len, buf)
  │
  └─► at25_ee_write(mtd, to, len, retlen, buf)
        │
        ├── يتحقق من EE_READONLY flag → يرجع -EROFS لو موجود
        │
        ├── يقسم الـ write على page_size chunks:
        │     while (remaining > 0):
        │       chunk = min(remaining, page_size - (offset % page_size))
        │
        ├── لكل chunk:
        │   ├── WREN command (Write Enable)
        │   ├── WRITE command + address + data
        │   └── ينتظر WIP bit يخلص (write-in-progress polling)
        │
        └─► كل الـ SPI transactions بتمشي عبر spi_sync()
```

---

### 5. Address Encoding — كيف بيتحسب العنوان

```
مثال: M95040 من ST — حجمه 512 bytes، بيستخدم EE_ADDR1 + EE_INSTR_BIT3_IS_ADDR

address = 0x100  (البايت رقم 256)

بدون EE_INSTR_BIT3_IS_ADDR:
  command = [READ][0x00]  ← address byte واحد → بس يوصل 255 byte!

مع EE_INSTR_BIT3_IS_ADDR:
  bit8 من الـ address = 1
  command = [READ | 0x08][0x00]  ← الـ bit 3 في الـ instruction = الـ bit 8 من العنوان
  ✓ يوصل للـ byte رقم 256 صح
```

---

### 6. استراتيجية الـ Locking

الـ `spi_eeprom` struct نفسه **ما فيهوش أي lock** — ده مجرد configuration object بيتقرأ مرة واحدة في الـ probe ومبيتغيرش.

الـ locking الحقيقي بيحصل في طبقات أعلى:

| الطبقة | الـ Lock | بيحمي إيه |
|--------|---------|-----------|
| **MTD layer** | `mutex` جوه `mtd_info` | تسلسل الـ read/write operations |
| **SPI core** | `spi_master.bus_lock_mutex` | الـ SPI bus — ما يتكلمش عليه إلا device واحد في وقت واحد |
| **at25 driver** | `lock` (mutex داخلي) | حالة الـ chip — لو في write جارية ما يبدأش تاني |

```
mtd_write()
  └─► mutex_lock(&at25->lock)        ← يمنع concurrent access للـ chip
        └─► spi_sync()
              └─► spi_master.bus_lock ← الـ SPI core يتحكم في الـ bus
                    └─► hardware
        └─► mutex_unlock(&at25->lock)
```

> الترتيب دايماً: **at25 lock → SPI bus lock** — عكسه هيعمل deadlock.

---

### ملاحظة على الـ `#include <linux/memory.h>`

الـ include ده في الملف بيجيب تعريفات عامة للـ memory subsystem زي `struct memory_block` وغيرها — لكن في سياق `spi_eeprom.h` الواقعي، الاحتياج منه هو النوع `u32` وغيره اللي بيجي عبر الـ include chain. الـ `spi_eeprom` struct نفسه مش بيستخدم أي struct من `memory.h` مباشرة.
## Phase 4: شرح الـ Functions

> الـ header ده (`include/linux/spi/eeprom.h`) مش بيعرّف functions — هو بيعرّف **data structure** واحدة بس (`struct spi_eeprom`) مع مجموعة **flag macros** بتوصف capabilities وprotocol options للـ SPI EEPROM chips. الـ driver اللي بيستخدم الـ struct ده هو `drivers/misc/eeprom/at25.c`.

---

### ملخص سريع — Cheatsheet

| العنصر | النوع | الغرض |
|--------|-------|--------|
| `struct spi_eeprom` | platform data struct | بتحدد geometry وprotocol للـ chip |
| `byte_len` | `u32` field | الحجم الكلي للـ EEPROM بالبايت |
| `name[10]` | `char[]` field | اسم الـ chip (debug / sysfs) |
| `page_size` | `u32` field | حجم الـ write page بالبايت |
| `flags` | `u16` field | بيجمع الـ flag bits التالية |
| `EE_ADDR1` | flag `0x0001` | عنوان 8-bit (1 byte address) |
| `EE_ADDR2` | flag `0x0002` | عنوان 16-bit (2 byte address) |
| `EE_ADDR3` | flag `0x0004` | عنوان 24-bit (3 byte address) |
| `EE_READONLY` | flag `0x0008` | منع الكتابة |
| `EE_INSTR_BIT3_IS_ADDR` | flag `0x0010` | bit3 من الـ opcode بيحمل MSB من العنوان |
| `context` | `void *` field | pointer خاص بالـ platform |

---

### الـ Category الوحيدة: Platform Data Definition

الـ header ده مش بيوفر API بالمعنى التقليدي — دوره هو تعريف **platform_data contract** بين الـ board-level code (أو device tree) وبين الـ `at25` driver. الـ driver بياخد pointer لـ `struct spi_eeprom` من `spi_device->dev.platform_data` أو من device properties (في حالة DT/ACPI).

---

### شرح `struct spi_eeprom` — تفصيلي

```c
struct spi_eeprom {
    u32  byte_len;        /* total chip capacity in bytes */
    char name[10];        /* chip name string */
    u32  page_size;       /* write page size in bytes */
    u16  flags;           /* protocol & capability flags */
    void *context;        /* opaque platform-specific pointer */
};
```

الـ struct ده بيتحط في `platform_data` للـ SPI device قبل ما الـ driver يـ probe. كل chips اللي بتفهم الـ AT25 command set (READ, WRITE, WREN, RDSR, WRSR) ممكن تتعرّف بيه — هو مش مخصص لـ Atmel بس.

#### الـ Fields بالتفصيل

---

##### `byte_len` — الحجم الكلي

```c
u32 byte_len;
```

- بيحدد الـ total addressable capacity بالبايت، مثلاً `512`, `4096`, `65536`.
- الـ `at25` driver بيستخدمه لتحديد الـ `mtd->size` وللـ bounds checking في كل read/write operation.
- **لازم** يكون consistent مع الـ address width اللي بتحطه في `flags` — لو حطيت `EE_ADDR1` (1 byte = 256 address) مع `byte_len = 512` فالـ `EE_INSTR_BIT3_IS_ADDR` flag بيحل هذا التناقض.

---

##### `name[10]` — اسم الـ Chip

```c
char name[10];
```

- string قصيرة بتظهر في الـ sysfs وفي kernel messages عند الـ probe.
- مش functional — بس مهمة للـ debugging وللـ identification.
- الـ buffer صغير (10 chars) — خليه موجز: `"AT25256B"` مثلاً.

---

##### `page_size` — حجم الـ Write Page

```c
u32 page_size;  /* for writes */
```

- الـ SPI EEPROMs بتتطلب إن الـ write operations تكون within page boundaries — لو تعدّيت حدود الـ page أثناء الكتابة، الـ address pointer بيـ wrap around جوا نفس الـ page وتخسر البيانات.
- الـ `at25` driver بيقسم الـ write requests الكبيرة لـ chunks لا تتعدى الـ `page_size` وكل chunk لازم تبدأ وتخلص في نفس الـ page.
- قيم شائعة: `8`, `16`, `32`, `64`, `128`, `256` bytes.

**pseudocode للـ page-aligned write في `at25`:**

```
write_to_eeprom(buf, offset, len):
    while len > 0:
        /* calculate remaining bytes in current page */
        page_offset = offset % page_size
        segment    = min(len, page_size - page_offset)

        spi_write(WREN_cmd)               /* write enable latch */
        spi_write(WRITE_cmd + addr + segment_data)
        poll_WIP_bit_until_clear()        /* wait for internal write cycle */

        offset += segment
        len    -= segment
```

---

##### `flags` — Protocol Flags

```c
u16 flags;
```

بيتحدد من OR للـ macros التالية:

---

###### `EE_ADDR1` — 8-bit Address

```c
#define EE_ADDR1  0x0001  /*  8 bit addrs */
```

- الـ READ/WRITE opcodes بيتبعهم byte عنوان واحد بس.
- يغطي حتى 256 bytes — مناسب لـ chips صغيرة جداً زي `AT25010` (128B).
- الـ `at25` driver بيبعت الأوامر كـ `[opcode][A7:A0]` (2 bytes total header).

---

###### `EE_ADDR2` — 16-bit Address

```c
#define EE_ADDR2  0x0002  /* 16 bit addrs */
```

- عنوان من 2 bytes، يغطي حتى 64KB.
- الأشيع — بيناسب أغلب chips من 256 bytes لـ 64KB.
- الـ header بيبقى `[opcode][A15:A8][A7:A0]` (3 bytes).

---

###### `EE_ADDR3` — 24-bit Address

```c
#define EE_ADDR3  0x0004  /* 24 bit addrs */
```

- عنوان من 3 bytes، يغطي حتى 16MB.
- للـ chips الكبيرة زي `AT25M02` (256KB) أو أكبر.
- الـ header بيبقى `[opcode][A23:A16][A15:A8][A7:A0]` (4 bytes).

**ملاحظة مهمة:** الـ flags الثلاثة دي mutually exclusive — لازم تختار واحد بس.

---

###### `EE_READONLY` — منع الكتابة

```c
#define EE_READONLY  0x0008  /* disallow writes */
```

- لو set، الـ `at25` driver بيرفض كل write requests بـ `-EROFS`.
- مفيد لـ chips بتحمل calibration data أو MAC addresses مش المفروض تتعدل.
- بيتشيك في `at25_ee_write()` قبل أي SPI transaction.

---

###### `EE_INSTR_BIT3_IS_ADDR` — الـ Opcode bit3 كـ Address MSB

```c
#define EE_INSTR_BIT3_IS_ADDR  0x0010
```

هذا الـ flag بيعالج حالة خاصة موثقة في الـ comment نفسه:

**المشكلة:** chips زي `ST M95040` حجمها 512 bytes، لكن بتستخدم address byte واحد بس (8 bits = 256 address max). العنوان bit8 (A8) مش موجود في الـ address field.

**الحل:** هذه الـ chips بتستخدم **bit3 من الـ opcode byte** كـ A8. يعني:
- `READ` opcode عادةً `0x03` — لو bit3 set بيبقى `0x0B` ويمثل A8=1.
- `WRITE` opcode عادةً `0x02` — لو bit3 set بيبقى `0x0A` ويمثل A8=1.

```
Normal AT25 READ for address 0x00FF:
  [0x03][0x00][0xFF]  (EE_ADDR1: 1-byte address)

M95040 READ for address 0x01FF (A8=1):
  [0x0B][0xFF]        (bit3 of opcode = A8, then A7:A0)
```

الـ driver لما يشوف الـ flag ده بيـ OR الـ bit3 في الـ opcode لو الـ address > 255.

---

##### `context` — Opaque Platform Pointer

```c
void *context;
```

- pointer حر للـ platform code تحطه بأي بيانات إضافية محتاجها.
- الـ `at25` driver core مش بيستخدمه — بس بيوفره للـ platform callbacks لو وجدت.
- في الغالب بيبقى `NULL` في معظم الاستخدامات.

---

### سياق الاستخدام الكامل — من الـ Board Code للـ Driver

#### 1. Static Platform Data (Legacy)

```c
/* board.c */
#include <linux/spi/eeprom.h>

static struct spi_eeprom at25_data = {
    .byte_len  = SZ_32K,          /* 32768 bytes */
    .name      = "at25256",
    .page_size = 64,
    .flags     = EE_ADDR2,        /* 16-bit address */
};

static struct spi_board_info board_spi_devices[] = {
    {
        .modalias        = "at25",
        .max_speed_hz    = 5000000,  /* 5 MHz */
        .chip_select     = 0,
        .platform_data   = &at25_data,
    },
};
```

#### 2. Device Tree (Modern)

في الـ DT، الـ `at25` driver بيبني `struct spi_eeprom` تلقائياً من device properties:

```dts
/* board.dts */
eeprom@0 {
    compatible = "atmel,at25";
    reg = <0>;
    spi-max-frequency = <5000000>;
    size = <32768>;       /* byte_len */
    pagesize = <64>;      /* page_size */
    address-width = <16>; /* → EE_ADDR2 */
    /* read-only;  → EE_READONLY */
};
```

الـ driver في `at25_probe()` بيعمل:
```c
/* drivers/misc/eeprom/at25.c — simplified */
err = device_property_read_u32(dev, "size", &pdata->byte_len);
err = device_property_read_u32(dev, "pagesize", &pdata->page_size);
err = device_property_read_u32(dev, "address-width", &addr_width);
switch (addr_width) {
    case 8:  pdata->flags |= EE_ADDR1; break;
    case 16: pdata->flags |= EE_ADDR2; break;
    case 24: pdata->flags |= EE_ADDR3; break;
}
if (device_property_present(dev, "read-only"))
    pdata->flags |= EE_READONLY;
```

---

### جدول الـ Flag Combinations الشائعة

| الـ Chip | `byte_len` | `page_size` | `flags` |
|----------|-----------|------------|---------|
| AT25010 (128B) | 128 | 8 | `EE_ADDR1` |
| AT25020 (256B) | 256 | 8 | `EE_ADDR1` |
| AT25040 (512B) | 512 | 8 | `EE_ADDR1 \| EE_INSTR_BIT3_IS_ADDR` |
| AT25080 (1KB) | 1024 | 32 | `EE_ADDR2` |
| AT25256 (32KB) | 32768 | 64 | `EE_ADDR2` |
| AT25512 (64KB) | 65536 | 128 | `EE_ADDR2` |
| AT25M02 (256KB) | 262144 | 256 | `EE_ADDR3` |
| Any read-only | any | any | `flags \| EE_READONLY` |

---

### العلاقة مع `linux/memory.h`

الـ `#include <linux/memory.h>` في الـ header ده **غير مباشر الاستخدام** — مش محتاجه فعلياً لتعريف الـ `struct spi_eeprom` نفسها (اللي بتحتاج فقط `u32`, `u16`, `char`). ممكن يكون موروث أو للـ forward declarations في بعض الـ architectures. الـ `linux/types.h` هو اللي بيوفر الـ `u32`/`u16` فعلياً.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem اللي بنتكلم عنه هو **SPI EEPROM** driver (المعروف بـ `at25`) اللي بيستخدم `struct spi_eeprom` كـ platform data. الـ debugging هنا بيتوزع على طبقتين: software (kernel + userspace tools) وهardware (إشارات SPI فعلية).

---

### Software Level

#### 1. debugfs entries

الـ `at25` driver مش بيسجل entries خاصة في debugfs، لكن الـ SPI subsystem نفسه بيوفر:

```bash
# اتأكد إن debugfs متماونت
mount -t debugfs none /sys/kernel/debug

# شوف الـ SPI controllers الموجودة
ls /sys/kernel/debug/spi*/

# على بعض الأنظمة بتلاقي:
cat /sys/kernel/debug/spi0/statistics
```

| Entry | المعنى |
|-------|---------|
| `/sys/kernel/debug/spi<N>/` | إحصائيات الـ controller |
| `/sys/kernel/debug/regmap/spi<N>.<CS>/` | لو الـ driver بيستخدم regmap |

لو محتاج تضيف entries يدويًا أثناء الـ development:

```c
/* في probe() الخاص بيك */
debugfs_create_u32("byte_len", 0444, dentry, &ee->chip.byte_len);
debugfs_create_u32("page_size", 0444, dentry, &ee->chip.page_size);
```

---

#### 2. sysfs entries

```bash
# ابحث عن الـ device في sysfs
ls /sys/bus/spi/devices/
# مثال: spi0.0 → EEPROM على SPI bus 0، chip select 0

# معلومات الـ device
cat /sys/bus/spi/devices/spi0.0/modalias
# → spi:at25

# الـ MTD layer لو الـ EEPROM اتسجل كـ MTD device
ls /sys/class/mtd/
cat /sys/class/mtd/mtd0/name
cat /sys/class/mtd/mtd0/size
cat /sys/class/mtd/mtd0/erasesize

# الـ NVMEM layer (الـ kernel الحديث يسجل at25 كـ nvmem provider)
ls /sys/bus/nvmem/devices/
cat /sys/bus/nvmem/devices/at25-0/nvmem   # قراءة raw bytes
```

| sysfs Path | المحتوى |
|-----------|---------|
| `/sys/bus/spi/devices/spiX.Y/` | الـ SPI device node |
| `/sys/class/mtd/mtdN/size` | حجم الـ EEPROM بالـ bytes |
| `/sys/class/mtd/mtdN/erasesize` | حجم الـ page (page_size) |
| `/sys/bus/nvmem/devices/*/nvmem` | raw binary data |

---

#### 3. ftrace — tracepoints وإيفنتات

```bash
# شوف الـ SPI events المتاحة
ls /sys/kernel/debug/tracing/events/spi/

# فعّل كل SPI events
echo 1 > /sys/kernel/debug/tracing/events/spi/enable

# أو فعّل events بعينها
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_transfer_start/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_transfer_stop/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_message_start/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_message_done/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل read/write للـ EEPROM
dd if=/dev/mtd0 of=/tmp/eeprom_dump.bin bs=256 count=1

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# وقّف
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
```

مثال على output متوقع:
```
at25-probe-0  [000] ....   123.456: spi_message_start: master=spi0 chip_select=0
at25-probe-0  [000] ....   123.457: spi_transfer_start: master=spi0 chip_select=0 len=3 tx=[03 00 00]
at25-probe-0  [000] ....   123.458: spi_transfer_stop:  master=spi0 chip_select=0 len=3
```

الـ `[03 00 00]` يعني: أمر READ (0x03) + address 0x0000 — ده طبيعي لـ AT25 EEPROM.

---

#### 4. printk / dynamic debug

```bash
# فعّل dynamic debug للـ at25 driver بالكامل
echo "file drivers/misc/eeprom/at25.c +p" > /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل الـ SPI framework messages
echo "file drivers/spi/*.c +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق إن التفعيل نجح
grep at25 /sys/kernel/debug/dynamic_debug/control

# شوف الـ output
dmesg -w | grep -E "at25|spi"
```

لو بتبني الـ kernel يدويًا، ضيف في `Kconfig` أو compile:
```bash
# في kernel command line
dyndbg="file at25.c +p; file spi.c +p"
```

---

#### 5. Kernel Config options للـ debugging

```
CONFIG_SPI_DEBUG=y              # يضيف verbose logging في SPI core
CONFIG_DEBUG_FS=y               # لازم يكون enabled عشان debugfs يشتغل
CONFIG_DYNAMIC_DEBUG=y          # يخلي pr_debug() قابلة للتفعيل runtime
CONFIG_MTD_DEBUG=y              # verbose logging للـ MTD layer
CONFIG_MTD_DEBUG_VERBOSE=3      # مستوى التفصيل (0-3)
CONFIG_NVMEM=y                  # الـ NVMEM framework
CONFIG_EEPROM_AT25=m            # الـ at25 driver نفسه (module أحسن للـ debug)
CONFIG_SPI_SPIDEV=y             # يسمح لك تتعامل مع SPI من userspace
CONFIG_FAULT_INJECTION=y        # لو عايز تتست error paths
CONFIG_FAIL_MTD_IO=y            # inject MTD I/O failures
```

---

#### 6. أدوات خاصة بالـ subsystem

**spi-tools / spidev:**
```bash
# لو CONFIG_SPI_SPIDEV=y، تقدر تتكلم مع الـ EEPROM مباشرة من userspace
# تحقق من الـ device
ls /dev/spidev*

# اقرأ أول 4 bytes من الـ EEPROM (AT25 READ command)
# READ = 0x03, address = 0x000000
python3 - <<'EOF'
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)          # bus=0, device=0
spi.max_speed_hz = 1000000
spi.mode = 0
# Read command: 0x03 + 2-byte address + dummy bytes
result = spi.xfer2([0x03, 0x00, 0x00, 0xFF, 0xFF, 0xFF, 0xFF])
print("Status:", hex(result[0]))
print("Data:", [hex(b) for b in result[1:]])
spi.close()
EOF
```

**mtd-utils:**
```bash
# اقرأ الـ EEPROM بالكامل
mtd_debug read /dev/mtd0 0 256 /tmp/eeprom_content.bin
hexdump -C /tmp/eeprom_content.bin

# اكتب بيانات تجريبية
echo "TESTDATA" > /tmp/test.bin
mtd_debug write /dev/mtd0 0 8 /tmp/test.bin

# تحقق من الكتابة
mtd_debug read /dev/mtd0 0 8 /tmp/verify.bin
hexdump -C /tmp/verify.bin
```

**nvmem interface:**
```bash
# قراءة مباشرة من NVMEM
dd if=/sys/bus/nvmem/devices/at25-0/nvmem bs=1 count=16 2>/dev/null | hexdump -C

# كتابة (لو مش READONLY)
printf '\xDE\xAD\xBE\xEF' | dd of=/sys/bus/nvmem/devices/at25-0/nvmem bs=4 count=1
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة في dmesg | المعنى | الحل |
|---------------|--------|------|
| `at25 spi0.0: error -EINVAL, address` | الـ address المطلوب أكبر من `byte_len` | تحقق من `byte_len` في platform data |
| `at25 spi0.0: read 0 bytes` | الـ SPI transfer رجع صفر | تحقق من الاتصال الفيزيائي وclock polarity |
| `at25 spi0.0: timeout` | الـ EEPROM مش بيرد | تحقق من CS (chip select) وVCC |
| `spi_master spi0: failed to transfer one message` | الـ SPI controller فشل | راجع `dmesg` للـ controller errors |
| `at25: probe of spi0.0 failed with error -ENODEV` | الـ driver مش لاقي الـ chip | تحقق من الـ compatible string في Device Tree |
| `at25 spi0.0: page_size must be a power of 2` | قيمة خاطأة في platform_data | صحح `page_size` |
| `at25 spi0.0: write protect is set` | الـ WP pin مفعّل | افصل أو disable الـ WP pin |
| `at25 spi0.0: address bits overflow` | `EE_INSTR_BIT3_IS_ADDR` مش متحدد | ضيف الـ flag ده للـ chips زي M95040 |
| `-EROFS: Read-only file system` | الـ `EE_READONLY` flag متحدد | طبيعي لو الـ EEPROM readonly بالقصد |
| `spi0.0: can't set 5000000 hz on CS0` | الـ clock بيعدي حد الـ chip | قلل `max_speed_hz` في platform data |

---

#### 8. استراتيجية dump_stack() و WARN_ON()

النقاط المهمة اللي تحط فيها checks:

```c
/* في at25_probe() — تحقق من صحة الـ platform data */
static int at25_probe(struct spi_device *spi)
{
    struct spi_eeprom *chip = spi->dev.platform_data;

    /* تحقق إن page_size power of 2 */
    WARN_ON(chip->page_size & (chip->page_size - 1));

    /* تحقق إن byte_len > 0 */
    if (WARN_ON(chip->byte_len == 0)) {
        return -EINVAL;
    }

    /* تحقق إن flag واحد بس من address flags متحدد */
    int addr_flags = !!(chip->flags & EE_ADDR1) +
                     !!(chip->flags & EE_ADDR2) +
                     !!(chip->flags & EE_ADDR3);
    if (WARN_ON(addr_flags != 1)) {
        dev_err(&spi->dev, "exactly one address mode flag required\n");
        return -EINVAL;
    }
}

/* في at25_ee_write() — تحقق من الـ alignment */
static ssize_t at25_ee_write(...)
{
    /* تحقق إن الكتابة مش بتعدي حدود الـ EEPROM */
    WARN_ON_ONCE(off + count > chip->byte_len);

    /* لو SPI transfer رجع error غير متوقع */
    if (status < 0) {
        dump_stack(); /* لتتبع من فين جاء الـ call */
        dev_err(dev, "SPI write failed at offset %lld\n", off);
    }
}
```

---

### Hardware Level

#### 1. التحقق إن الـ hardware state يطابق الـ kernel state

```bash
# تحقق إن الـ driver اتحمل وشاف الـ EEPROM
dmesg | grep -E "at25|eeprom|spi0\.0"
# المتوقع: "at25 spi0.0: 256 byte 25x020 EEPROM, writable, 16 bytes/write"

# الـ byte_len اللي الـ kernel شايفه
cat /sys/class/mtd/mtd0/size
# لازم يساوي byte_len في platform_data

# الـ page_size (erasesize في MTD context)
cat /sys/class/mtd/mtd0/erasesize

# هل الـ EEPROM accessible؟
dd if=/dev/mtd0 of=/dev/null bs=256 count=1 && echo "OK" || echo "FAIL"
```

---

#### 2. Register Dump باستخدام devmem2 / /dev/mem

الـ SPI EEPROM نفسها مفيهاش registers تقدر تقرأها عن طريق `/dev/mem` لأنها SPI device (serial). لكن الـ SPI Controller registers تقدر تقرأها:

```bash
# اول لازم تعرف الـ base address من Device Tree أو datasheet
# مثال لـ Raspberry Pi SPI0:
SPI_BASE=0xFE204000   # لـ RPi4

# قرأ SPI CS register
devmem2 $SPI_BASE 0x4 w   # CS register
devmem2 $((SPI_BASE + 0x4)) w   # CLK register
devmem2 $((SPI_BASE + 0x8)) w   # DLEN register

# بديل: قراءة عبر /sys/kernel/debug/regmap
find /sys/kernel/debug/regmap -name "registers" | head -5
cat /sys/kernel/debug/regmap/spi0/registers 2>/dev/null | head -20
```

**تفسير SPI CS register (BCM2835 مثلاً):**

```
Bit 0-1: CS   → الـ chip select المفعّل (0, 1, أو 2)
Bit 4:   CPHA → clock phase
Bit 5:   CPOL → clock polarity
Bit 7:   TA   → Transfer Active (1 = تراسل جاري)
Bit 16:  DONE → Transfer Done
Bit 17:  RXD  → RX FIFO has data
Bit 18:  TXD  → TX FIFO can accept data
```

---

#### 3. Logic Analyzer / Oscilloscope

**الإعدادات المطلوبة لـ SPI EEPROM (AT25 series):**

| Parameter | القيمة |
|-----------|--------|
| Protocol | SPI Mode 0 (CPOL=0, CPHA=0) |
| Clock | 1–5 MHz للـ debugging |
| Channels | SCLK, MOSI, MISO, CS# |
| Trigger | Falling edge على CS# |
| Logic level | 3.3V أو 1.8V حسب الـ chip |

**ما تتوقعه على الـ logic analyzer لعملية READ:**

```
CS#  ‾‾‾‾\_______________________________/‾‾‾‾
SCLK     _/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_
MOSI ____[  03  ][  addr_hi ][  addr_lo  ]____
MISO ________________________[   DATA    ]____
```

- **MOSI byte 1**: `0x03` = READ command
- **MOSI byte 2-3** (أو 2 للـ `EE_ADDR1`): العنوان
- **MISO**: البيانات المقروءة

**أوامر AT25 المهمة:**
```
0x03 = READ
0x02 = WRITE
0x06 = WREN (Write Enable) — لازم يتبعث قبل كل WRITE
0x04 = WRDI (Write Disable)
0x05 = RDSR (Read Status Register)
0x01 = WRSR (Write Status Register)
```

**تحقق من الـ WREN sequence:**
```
CS# ‾‾\_________/‾‾‾‾‾\_______________________/‾‾
        [  0x06  ]      [0x02][addr][data...]
MOSI ___[  WREN  ]_____[         WRITE          ]___
```
لو مشفتش `0x06` قبل `0x02`، الكتابة هتفشل.

---

#### 4. مشاكل الـ hardware الشائعة وأنماطها في الـ kernel log

| المشكلة | النمط في dmesg | السبب |
|---------|---------------|-------|
| الـ EEPROM مش موجودة على الـ bus | `at25: probe of spi0.0 failed with error -ENODEV` | مش موصّل صح أو VCC مش شغّال |
| CS مش شغّال | `spi_master: message timeout` | CS GPIO مش متحدد صح |
| Mode خاطأ (CPOL/CPHA) | بيانات عشوائية في القراءة | غير CPOL/CPHA في DT أو platform data |
| الـ clock بطيء جداً | وقت طويل في القراءة | عادي مش بيسبب error، بس performance |
| الـ clock سريع جداً | `SPI transfer failed` أو CRC errors | قلل `max_speed_hz` |
| WP pin مفعّل | `-EROFS` على الكتابة | شيل الـ WP connection أو disable بـ GPIO |
| الـ VCC مش stable | intermittent errors | قس الـ VCC بـ oscilloscope أثناء التراسل |
| الـ capacitance عالية على الـ MISO | بيانات shifted | قصّر الـ wires أو ضيف pull-up resistors |

---

#### 5. Device Tree Debugging

```bash
# شوف الـ DT nodes المتعلقة بالـ SPI
dtc -I fs /proc/device-tree | grep -A 20 "eeprom\|at25\|spi@"

# أو استخدم fdtdump
cat /proc/device-tree/soc/spi@7e204000/eeprom@0/compatible 2>/dev/null
# المتوقع: "atmel,at25"

# تحقق من كل properties
find /proc/device-tree -name "compatible" -exec grep -l "at25\|eeprom" {} \;

# اقرأ الـ reg property (chip select)
hexdump /proc/device-tree/soc/spi@7e204000/eeprom@0/reg

# اقرأ الـ spi-max-frequency
hexdump /proc/device-tree/soc/spi@7e204000/eeprom@0/spi-max-frequency
```

**مثال DT صحيح لـ AT25 EEPROM:**

```dts
&spi0 {
    status = "okay";

    eeprom@0 {
        compatible = "atmel,at25";
        reg = <0>;                        /* chip select 0 */
        spi-max-frequency = <5000000>;    /* 5 MHz */

        /* هذه القيم لازم تطابق struct spi_eeprom */
        size = <256>;                     /* byte_len */
        address-width = <8>;              /* EE_ADDR1 = 8-bit addr */
        pagesize = <16>;                  /* page_size */
    };
};
```

**مشاكل DT شائعة:**

```bash
# تحقق إن address-width صحيحة
# 8  → EE_ADDR1 (chips <= 256 bytes)
# 16 → EE_ADDR2 (chips <= 64KB)
# 24 → EE_ADDR3 (chips > 64KB)
hexdump -e '1/4 "%d\n"' /proc/device-tree/.../address-width

# لو address-width خاطأ، الـ driver هيقرأ garbage
# لأن bytes الـ address هتكون أقل أو أكتر من المطلوب
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

```bash
#!/bin/bash
# SPI EEPROM Debug Script

EEPROM_DEV=${1:-"/dev/mtd0"}
SPI_DEV=${2:-"spi0.0"}
NVMEM_DEV=${3:-"at25-0"}

echo "=== 1. Kernel Messages ==="
dmesg | grep -E "at25|${SPI_DEV}|eeprom" | tail -20

echo ""
echo "=== 2. SPI Device Info ==="
ls -la /sys/bus/spi/devices/${SPI_DEV}/ 2>/dev/null || echo "Device not found"
cat /sys/bus/spi/devices/${SPI_DEV}/modalias 2>/dev/null

echo ""
echo "=== 3. MTD Info ==="
cat /proc/mtd 2>/dev/null
mtdinfo ${EEPROM_DEV} 2>/dev/null

echo ""
echo "=== 4. Read First 16 Bytes ==="
dd if=${EEPROM_DEV} bs=16 count=1 2>/dev/null | hexdump -C

echo ""
echo "=== 5. NVMEM Read ==="
ls /sys/bus/nvmem/devices/ 2>/dev/null
dd if=/sys/bus/nvmem/devices/${NVMEM_DEV}/nvmem bs=16 count=1 2>/dev/null | hexdump -C

echo ""
echo "=== 6. Enable SPI Tracing ==="
echo 1 > /sys/kernel/debug/tracing/events/spi/enable 2>/dev/null && \
echo "SPI tracing enabled — run your test, then check:" && \
echo "  cat /sys/kernel/debug/tracing/trace"

echo ""
echo "=== 7. Dynamic Debug ==="
echo "file drivers/misc/eeprom/at25.c +p" > \
    /sys/kernel/debug/dynamic_debug/control 2>/dev/null && \
echo "at25 dynamic debug enabled"
```

#### تفسير الـ output

```
=== MTD Info Output ===
$ mtdinfo /dev/mtd0
Name:                           at25
Type:                           nor
Eraseblock size:                16 bytes    ← هذا page_size
Amount of eraseblocks:          16          ← byte_len / page_size = 256/16
Minimum input/output unit size: 1 byte
Sub-page size:                  1 byte
Character device major/minor:   90:0
Bad blocks are allowed:         false
Device is writable:             true        ← EE_READONLY مش متحدد
```

```
=== SPI Trace Output ===
$ cat /sys/kernel/debug/tracing/trace | grep spi
kworker/0:2-45 [000] d... spi_message_start:  master=spi0 chip_select=0
kworker/0:2-45 [000] d... spi_transfer_start: tx=[06] len=1           ← WREN
kworker/0:2-45 [000] d... spi_transfer_start: tx=[02 00 00 41] len=4  ← WRITE addr=0x0000 data=0x41
kworker/0:2-45 [000] d... spi_message_done:   master=spi0 status=0

# status=0 → نجاح
# tx=[02 00 00 41] → WRITE(0x02) + addr_hi(0x00) + addr_lo(0x00) + data(0x41='A')
```

```bash
# التحقق من write protect status عبر RDSR (Read Status Register)
# Status Register bits: WIP(0), WEL(1), BP0(2), BP1(3), SRWD(7)
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 1000000
r = spi.xfer2([0x05, 0x00])  # RDSR command
status = r[1]
print(f'Status Register: 0x{status:02X}')
print(f'  WIP  (Write In Progress): {(status>>0)&1}')
print(f'  WEL  (Write Enable Latch): {(status>>1)&1}')
print(f'  BP0  (Block Protect 0): {(status>>2)&1}')
print(f'  BP1  (Block Protect 1): {(status>>3)&1}')
print(f'  SRWD (Status Reg Write Disable): {(status>>7)&1}')
spi.close()
"
# WIP=1 → الـ EEPROM بتكتب جوّاها دلوقتي (write cycle جاري)
# WEL=0 → Write Enable مش متحدد → الكتابة مش هتنفع
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على STM32MP1 — EEPROM مش بيتكتب

#### العنوان
فشل كتابة بيانات الـ MAC address على EEPROM خارجي أثناء bring-up

#### السياق
بنشتغل على **industrial gateway** بيشغل Linux على **STM32MP1**، الـ gateway ده بيحتج يخزن MAC address وserial number في **SPI EEPROM** نوعه **M95040** من ST (512 bytes). الـ board جديدة والـ bring-up أولانه.

#### المشكلة
الـ driver بيتحمل تمام، لكن أي write operation بترجع `-EIO`. القراءة شغالة. الـ dmesg بيقول:

```
at25 spi0.1: write failed at byte 256
```

#### التحليل
الـ M95040 فيها مشكلة معروفة — هي 512 bytes بس بتستخدم **بس byte واحد للـ address** (A0–A7، يعني 8 bits = 256 address فقط). لتخطي الـ 256 bytes العليا، الـ bit A8 بيتحط في **bit 3 من الـ instruction byte** مش في الـ address byte التاني.

في `spi_eeprom`:

```c
struct spi_eeprom {
    u32  byte_len;   /* = 512 */
    char name[10];   /* "m95040"  */
    u32  page_size;  /* = 16 bytes */
    u16  flags;
    /* EE_ADDR1 = 8-bit address — بس ده كافي لـ 256 byte مش 512 */
    /* EE_INSTR_BIT3_IS_ADDR = الـ bit الإضافي في instruction */
};
```

المشكلة: الـ platform_data أو الـ DT بيحدد `EE_ADDR1` فقط بدون `EE_INSTR_BIT3_IS_ADDR`، فالـ driver مش بيحسب الـ A8 في الـ opcode، وبالتالي الـ write فوق byte 255 بتروح على address غلط.

الـ flag الناقص هو:
```c
#define EE_INSTR_BIT3_IS_ADDR   0x0010
```

ده بيخلي الـ `at25` driver يعمل OR بين الـ high address bit وبين الـ write opcode (0x02 → 0x0A للـ upper half).

#### الحل

**في الـ Device Tree:**

```dts
&spi0 {
    eeprom@1 {
        compatible = "atmel,at25";
        reg = <1>;
        spi-max-frequency = <5000000>;
        spi-cpha;

        /* تعريف الـ EEPROM parameters */
        size = <512>;
        pagesize = <16>;
        address-width = <8>;   /* EE_ADDR1 */

        /* الـ flag المهم لـ M95040 */
        /* at25 driver بيترجمه لـ EE_INSTR_BIT3_IS_ADDR */
        spi-nor,ignore-high-byte-addr;
    };
};
```

**أو لو بنستخدم platform_data قديم:**

```c
static struct spi_eeprom m95040_data = {
    .byte_len  = 512,
    .name      = "m95040",
    .page_size = 16,
    /* بنحدد EE_ADDR1 لأن الـ address byte واحد */
    /* بنضيف EE_INSTR_BIT3_IS_ADDR لأن A8 في الـ opcode */
    .flags     = EE_ADDR1 | EE_INSTR_BIT3_IS_ADDR,
};
```

#### الدرس المستفاد
الـ `EE_INSTR_BIT3_IS_ADDR` flag موجود بالظبط لحالات زي M95040 — EEPROM أكبر من قدرة الـ address bytes بتاعتها. لازم تقرأ الـ datasheet وتحدد الـ flags الصح من أول يوم.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — EEPROM readonly مش اتحترم

#### العنوان
الـ firmware بيكتب فوق بيانات الـ calibration في EEPROM المفروض تكون read-only

#### السياق
**Android TV box** بيشغل kernel على **Allwinner H616**. في الـ board في **SPI EEPROM** نوعه **AT25256B** (32KB) بيخزن بيانات **Wi-Fi RF calibration** اللي المفروض تتكتب مرة واحدة في المصنع وما تتغيرش أبدًا.

#### المشكلة
بعد OTA update، الـ Wi-Fi calibration اتمسحت وأداء الـ Wi-Fi وقع. الـ log بيظهر إن user-space process كتبت على `/dev/mtd0`.

#### التحليل
الـ `spi_eeprom` struct فيها flag اسمه:

```c
#define EE_READONLY  0x0008  /* disallow writes */
```

الـ `at25` driver بيشوف الـ flag ده ومايسجلش الـ write ops على الـ MTD device. لو الـ flag مش موجود، الـ EEPROM بيتعامل معاه كـ read-write عادي.

المشكلة إن الـ DT entry للـ EEPROM ما عندهاش الـ `read-only` property:

```dts
/* غلط — مفيش حماية */
eeprom@0 {
    compatible = "atmel,at25";
    reg = <0>;
    spi-max-frequency = <10000000>;
    size = <32768>;
    pagesize = <64>;
    address-width = <16>;
    /* ناقص: read-only */
};
```

الـ `at25` driver بيترجم الـ DT property `read-only` لـ `EE_READONLY` flag في الـ `spi_eeprom.flags`، وبعدين في الـ probe:

```c
/* في drivers/misc/eeprom/at25.c — مثال مبسط */
if (at25->chip.flags & EE_READONLY)
    /* ما بيسجلش write methods على MTD */
    mtd->_write = NULL;
```

لأن `mtd->_write` بيبقى NULL، أي write call بترجع `-EROFS`.

#### الحل

**تعديل الـ DT:**

```dts
eeprom@0 {
    compatible = "atmel,at25";
    reg = <0>;
    spi-max-frequency = <10000000>;
    size = <32768>;
    pagesize = <64>;
    address-width = <16>;
    read-only;   /* ده بيضيف EE_READONLY تلقائي */
};
```

**التحقق:**

```bash
# بعد الـ reboot
cat /sys/class/mtd/mtd0/ro
# لازم يرجع 1

# محاولة كتابة لازم تفشل
echo "test" > /dev/mtd0
# bash: /dev/mtd0: Read-only file system
```

#### الدرس المستفاد
الـ `EE_READONLY` flag مش بس documentation — ده بيأثر فعلًا على الـ MTD layer. أي EEPROM بيخزن بيانات factory calibration أو MAC addresses لازم يتعلم بـ `read-only` في الـ DT من أول يوم.

---

### السيناريو الثالث: IoT Sensor Node على RK3562 — page write corruption

#### العنوان
بيانات الـ sensor configuration بتتخرب بعد كتابة أكتر من 8 bytes

#### السياق
**IoT sensor node** بيشغل embedded Linux على **RK3562** مع **SPI EEPROM** نوعه **CAT25C04** (512 bytes من Catalyst/ON Semi). الـ node بيخزن sensor calibration coefficients في الـ EEPROM.

#### المشكلة
كل ما الـ application بيكتب block أكبر من 8 bytes، الـ data بتتخرب. القراءة بترجع بيانات غلط في الـ bytes 9–16.

#### التحليل
الـ **CAT25C04** عندها **page size = 8 bytes**، مش 16. الـ SPI EEPROM write operation بتعمل **page wrap-around** — لو كتبت 16 bytes وبدأت من byte 0، الـ bytes من 8 لـ 15 بيتكتبوا فوق bytes 0 لـ 7 في نفس الـ page.

الـ `spi_eeprom.page_size` field هو اللي بيتحكم في ده:

```c
struct spi_eeprom {
    u32  byte_len;   /* = 512 */
    char name[10];   /* "cat25c04" */
    u32  page_size;  /* لازم = 8، مش 16 */
    u16  flags;
};
```

الـ `at25` driver بيقطع الـ write requests حسب الـ `page_size` علشان يتجنب الـ wrap-around:

```c
/* في at25.c — الـ driver بيحسب الـ chunk size */
/* بيكتب max (page_size - offset_within_page) bytes في كل مرة */
nr_bytes = min_t(unsigned, count,
    at25->chip.page_size - (off % at25->chip.page_size));
```

لو `page_size` اتحددت بـ 16 وهي في الحقيقة 8، الـ driver هيكتب 16 bytes دفعة واحدة والـ EEPROM هيعمل wrap.

#### الحل

**تصحيح الـ DT:**

```dts
eeprom@0 {
    compatible = "atmel,at25";
    reg = <0>;
    spi-max-frequency = <2000000>;  /* CAT25C04 max = 3MHz */
    size = <512>;
    pagesize = <8>;     /* الصح — 8 bytes مش 16 */
    address-width = <8>;  /* EE_ADDR1 */
};
```

**أو في platform_data:**

```c
static struct spi_eeprom cat25c04_config = {
    .byte_len  = 512,
    .name      = "cat25c04",
    .page_size = 8,    /* من الـ datasheet: 8-byte page */
    .flags     = EE_ADDR1,
};
```

**للتحقق من الـ page size المستخدمة:**

```bash
# شوف الـ EEPROM parameters من sysfs
cat /sys/bus/spi/devices/spi0.0/eeprom_pagesize 2>/dev/null
# أو اشتغل بـ write صغير وتأكد
dd if=/dev/zero bs=8 count=1 of=/dev/mtd1 seek=0
hexdump -C /dev/mtd1 | head -4
```

#### الدرس المستفاد
الـ `page_size` في `spi_eeprom` مش مجرد performance hint — غلطته بتخرب الـ data. دايمًا راجع الـ datasheet لـ "Page Write" section وحدد الـ page size الصح.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — اسم الـ EEPROM أطول من 9 حروف

#### العنوان
kernel panic غامض أثناء probe الـ SPI EEPROM في ECU

#### السياق
**Automotive ECU** بيشغل Linux على **i.MX8MP**. الـ engineer بيضيف support لـ **SPI EEPROM** نوعه **AT25HP512B** (برتبط بـ high-endurance EEPROM) في platform_data قديم (pre-DT board).

#### المشكلة
الـ kernel بيعمل panic أثناء الـ `probe`:

```
BUG: stack-out-of-bounds in at25_probe
Unable to handle kernel NULL pointer dereference
```

الـ stack trace بيشير لكود بيستخدم `chip.name`.

#### التحليل
الـ `spi_eeprom` struct عندها:

```c
struct spi_eeprom {
    u32  byte_len;
    char name[10];    /* مصفوفة بـ 10 bytes فقط! */
    u32  page_size;
    u16  flags;
    void *context;
};
```

الـ `name` field **عندها 10 chars فقط** — يعني 9 حروف + null terminator. الـ engineer كتب:

```c
static struct spi_eeprom at25hp512b_config = {
    .byte_len  = 65536,
    .name      = "AT25HP512B",   /* 10 حروف = overflow! */
    .page_size = 128,
    .flags     = EE_ADDR2,
};
```

الاسم `"AT25HP512B"` ده **10 حروف** — بيملا الـ array بالظبط بدون null terminator (`\0`). أي `strlen` أو copy بعد كده بتعدي حدود الـ struct وبتخرب الـ `page_size` أو الـ `flags`.

على **i.MX8** مع stack protection، ده بيطلع `panic` فورًا.

#### الحل

```c
static struct spi_eeprom at25hp512b_config = {
    .byte_len  = 65536,
    .name      = "at25512b",   /* اختصرنا الاسم — 8 حروف + null = 9 bytes OK */
    .page_size = 128,
    .flags     = EE_ADDR2,     /* 16-bit address لـ 64KB */
};
```

أو استخدام الـ DT بدل platform_data (مش محدود بـ 10 chars):

```dts
eeprom@0 {
    compatible = "atmel,at25";
    label = "AT25HP512B-config-store";  /* اسم مش محدود */
    reg = <0>;
    size = <65536>;
    pagesize = <128>;
    address-width = <16>;
};
```

**للتأكد في الكود إنك مش بتتجاوز الـ limit:**

```c
/* فحص وقت الكمبايل */
BUILD_BUG_ON(sizeof("AT25HP512B") > sizeof(((struct spi_eeprom *)0)->name));
```

#### الدرس المستفاد
الـ `name[10]` في `spi_eeprom` هو **hard limit** — 9 حروف بالكتير. في الـ automotive context، الأسماء بتبقى descriptive وطويلة، فاستخدم DT `label` property أو اختصر الاسم.

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — EEPROM موجود بس الـ driver ما بيشتغلش

#### العنوان
`at25` driver ما بيعملش probe للـ EEPROM رغم إن الـ hardware موجود وشغال

#### السياق
**custom industrial board** على **TI AM62x** بيحتوي على **SPI EEPROM** نوعه **25LC256** (Microchip, 32KB). الـ board جديدة كليًا والـ engineer بيعمل bring-up من zero.

#### المشكلة
الـ `at25` driver موجود في الـ kernel config، الـ SPI bus شغال (SPI flash تاني على نفس الـ bus شغال)، لكن الـ EEPROM ما بيظهرش في `/dev/mtd*` وما في أي error في الـ dmesg.

#### التحليل
الـ `at25` driver بيحتاج معلومات الـ EEPROM تيجيله بطريقة من اتنين:
1. عن طريق `spi_eeprom` struct في `spi_board_info.platform_data`
2. عن طريق DT properties محددة (`size`, `pagesize`, `address-width`)

بدون أي منهم، الـ driver مش هيقدر يشتغل لأن ما عندوش `byte_len` ولا `page_size`.

الـ engineer كتب DT entry ناقص:

```dts
/* غلط — ناقص الـ EEPROM parameters */
&spi1 {
    status = "okay";

    eeprom@1 {
        compatible = "atmel,at25";
        reg = <1>;
        spi-max-frequency = <5000000>;
        /* ناقص: size, pagesize, address-width */
    };
};
```

الـ `at25` driver بيعمل `of_property_read_u32` على `size` — لو ما لقيهاش، بيرجع error وما بيكملش الـ probe، وبدون message واضحة في القديم.

الـ `spi_eeprom.byte_len` هو أول field لازم يتحدد — بدونه الـ driver مش هيحجز الـ MTD device خالص.

#### الحل

**DT صح:**

```dts
&spi1 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi1_pins>;

    eeprom@1 {
        compatible = "atmel,at25";
        reg = <1>;
        spi-max-frequency = <5000000>;

        /* الـ 3 properties دول مطلوبين إلزامي */
        size = <32768>;          /* byte_len = 32KB */
        pagesize = <64>;         /* 25LC256 page = 64 bytes */
        address-width = <16>;    /* EE_ADDR2 لـ 32KB */
    };
};
```

**للتحقق بعد الـ reboot:**

```bash
# الـ EEPROM لازم يظهر
ls /dev/mtd*
# mtd0  mtd0ro  ...

# تفاصيل الـ MTD device
cat /proc/mtd
# dev:    size   erasesize  name
# mtd0: 00008000 00000040 "25LC256"

# تجربة قراءة
dd if=/dev/mtd0 of=/tmp/eeprom_backup.bin bs=64
hexdump -C /tmp/eeprom_backup.bin | head
```

**لو حابب تتأكد إن الـ driver فهم الـ parameters صح:**

```bash
# enable dynamic debug
echo "module at25 +p" > /sys/kernel/debug/dynamic_debug/control
dmesg | grep at25
# at25 spi1.1: 25LC256, 32768 bytes, 16-bit addr, 64-byte page, readonly=no
```

#### الدرس المستفاد
الـ `at25` driver **مش بيكمل probe** بدون الـ 3 properties: `size` (→ `byte_len`)، `pagesize` (→ `page_size`)، و`address-width` (→ `flags`). Enable الـ dynamic debug من أول bring-up يوفر ساعات من الـ debugging.
## Phase 7: مصادر ومراجع

### مصادر رسمية في kernel

#### التوثيق الرسمي داخل الـ kernel

| المسار | المحتوى |
|--------|---------|
| `Documentation/driver-api/spi.rst` | الـ SPI subsystem كامل — الـ API، الـ driver model، الـ I/O |
| `Documentation/spi/spi-summary.rst` | ملخص عملي لكتابة الـ SPI drivers |
| `Documentation/spi/spidev.rst` | الـ userspace interface عبر `/dev/spidevN.N` |
| `Documentation/devicetree/bindings/eeprom/at25.yaml` | الـ device tree bindings للـ AT25 EEPROM |
| `drivers/misc/eeprom/at25.c` | الـ driver الرئيسي اللي يستخدم `struct spi_eeprom` من `include/linux/spi/eeprom.h` |
| `include/linux/spi/eeprom.h` | الـ header موضوع البحث — تعريف `struct spi_eeprom` والـ flags |

الـ online version للتوثيق الرسمي:
- [Serial Peripheral Interface (SPI) — The Linux Kernel documentation](https://docs.kernel.org/driver-api/spi.html)
- [spi-summary.rst على kernel.org](https://www.kernel.org/doc/Documentation/spi/spi-summary.rst)

---

### مقالات LWN.net

**الـ** LWN.net هو المرجع التاريخي الأهم لتطور الـ Linux kernel internals.

| المقال | الأهمية |
|--------|---------|
| [SPI core](https://lwn.net/Articles/138037/) | أول مقال يوثّق إضافة الـ SPI subsystem للـ kernel — David Brownell |
| [simple SPI framework](https://lwn.net/Articles/154425/) | الـ framework الأولي قبل ما يتطور لصورته الحالية |
| [SPI redux … driver model support](https://lwn.net/Articles/149698/) | تحديث الـ SPI ليدعم الـ Linux driver model بشكل صحيح |
| [SPI core refresh](https://lwn.net/Articles/162268/) | تحديث جوهري للـ core بعد التجارب الأولى |
| [spi: Add slave mode support](https://lwn.net/Articles/723440/) | إضافة دعم الـ SPI slave mode |
| [eeprom: New ee1004 driver for DDR4 memory](https://lwn.net/Articles/739655/) | driver جديد للـ EEPROM — مقارنة مفيدة مع at25 |
| [Device tree overlays](https://lwn.net/Articles/617800/) | الـ DT overlays اللي بتأثر على كيفية تعريف الـ SPI EEPROMs |

---

### نقاشات الـ mailing list (lore.kernel.org)

الـ patches دي بتوضح تطور الـ `struct spi_eeprom` والـ `at25` driver عبر الزمن:

| الـ patch | الأهمية |
|-----------|---------|
| [eeprom: at25: Convert the driver to the spi-mem interface](https://lore.kernel.org/linux-mtd/20190903170850.6aee808b@collabora.com/T/) | تحويل الـ driver ليستخدم الـ `spi-mem` API الحديث بدل forge الـ SPI messages يدوياً |
| [misc: eeprom: at25: add Cypress FRAM functionality](https://lore.kernel.org/all/c7beacd629a47a0bd5023353378a3eda4d7d0f89.1444139729.git.jiri.prchal@aksignal.cz/) | إضافة دعم الـ FRAMs من Cypress — FM25V05، FM25V10 |
| [eeprom: at25: Remove in kernel API](https://lore.kernel.org/patchwork/patch/625450/) | إزالة الـ `memory_accessor` API القديم واستبداله بـ nvmem |
| [eeprom: at25: Use DMA safe buffers](https://lore.kernel.org/all/230a9486fc68ea0182df46255e42a51099403642.1648032613.git.christophe.leroy@csgroup.eu/) | إصلاح مشكلة الـ DMA باستخدام `kmalloc` بدل الـ stack buffers |
| [nvmem: eeprom: at25: fix FRAM byte_len](https://lore.kernel.org/all/1638537604213144@kroah.com/T/) | إصلاح حساب الـ `byte_len` للـ FRAM |
| [PATCH 0/3: Configure at25 from device tree](https://fa.linux.kernel.narkive.com/qSAYmK4O/patch-0-3-of-spi-eeprom-configure-at25-from-device-tree-and-autoload-its-driver) | تفعيل autoload الـ driver من الـ device tree |
| [PATCH 0/2: SPI EEPROM device tree interaction improvements](https://linux.kernel.narkive.com/expT6dxN/patch-0-2-spi-eeprom-device-tree-interaction-improvements) | تحسينات الـ DT properties للـ AT25 |

---

### الـ source code على GitHub/Kernel.org

```
# الـ driver الرئيسي
https://github.com/torvalds/linux/blob/master/drivers/misc/eeprom/at25.c

# الـ DT bindings
https://kernel.googlesource.com/pub/scm/linux/kernel/git/next/linux-next/+/refs/heads/akpm-base/Documentation/devicetree/bindings/eeprom/at25.yaml

# الـ Kconfig entry
CONFIG_EEPROM_AT25: https://cateee.net/lkddb/web-lkddb/EEPROM_AT25.html
```

---

### موارد eLinux.org

| الصفحة | ما تفيدك فيه |
|--------|-------------|
| [RPi SPI — eLinux.org](https://elinux.org/RPi_SPI) | توثيق عملي لـ SPI على Raspberry Pi مع أمثلة EEPROM |
| [Tests: MSIOF-SPI-CS-GPIO](https://elinux.org/Tests:MSIOF-SPI-CS-GPIO) | اختبارات فعلية مع Microchip 25LC040 SPI EEPROMs |
| [BeagleBoard Zippy](https://elinux.org/BeagleBoard_Zippy) | مثال على ربط SPI EEPROM بـ BeagleBoard |

---

### kernelnewbies.org — تغييرات الـ kernel المتعلقة بـ EEPROM

| الصفحة | ما يهمك فيها |
|--------|-------------|
| [Linux 4.14 Changes](https://kernelnewbies.org/Linux_4.14) | تغييرات الـ EEPROM subsystem في kernel 4.14 |
| [Linux 5.4 Changes](https://kernelnewbies.org/Linux_5.4) | nvmem ودعم الـ SPI memory في kernel 5.4 |
| [Linux 5.14 Changes](https://kernelnewbies.org/Linux_5.14) | تحديثات misc eeprom_93xx46 وغيرها |
| [Linux 3.0 Driver Architecture](https://kernelnewbies.org/Linux_3.0_DriverArch) | نظرة عامة على تغييرات معمارية الـ drivers |

---

### كتب موصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول المهمة:**
  - Chapter 14: The Linux Device Model
  - Chapter 15: Memory Mapping and DMA (مهم لفهم DMA-safe buffers في at25)
- متاح مجاناً: https://lwn.net/Kernel/LDD3/

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصول المهمة:**
  - Chapter 17: Devices and Modules
  - Chapter 13: The Virtual Filesystem (للفهم الكامل لـ sysfs و nvmem)

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصول المهمة:**
  - Chapter 8: Device Driver Basics
  - Chapter 10: MTD Subsystem (SPI Flash و EEPROM في embedded systems)

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- Chapter 8: I2C and SPI — أفضل شرح عملي للـ SPI drivers مع أمثلة EEPROM

---

### search terms للبحث عن معلومات أكثر

```
# للبحث في الـ kernel source
git log --all --oneline -- drivers/misc/eeprom/at25.c
git log --all --oneline -- include/linux/spi/eeprom.h

# كلمات البحث على Google/DuckDuckGo
"struct spi_eeprom" linux kernel
"EE_ADDR1" OR "EE_ADDR2" linux spi eeprom flags
"EE_INSTR_BIT3_IS_ADDR" M95040 ST microelectronics
at25 driver nvmem linux kernel
spi_eeprom platform_data linux

# للبحث في lore.kernel.org
https://lore.kernel.org/linux-spi/?q=at25+eeprom
https://lore.kernel.org/linux-mtd/?q=spi+eeprom

# للبحث في patchwork
https://patchwork.kernel.org/project/spi-devel-general/list/?q=eeprom
```

---

### جدول ملخص المراجع الأهم

| المرجع | النوع | الأولوية |
|--------|-------|----------|
| `Documentation/driver-api/spi.rst` | توثيق رسمي | عالية جداً |
| `drivers/misc/eeprom/at25.c` | source code | عالية جداً |
| [LWN: SPI core](https://lwn.net/Articles/138037/) | مقال تاريخي | عالية |
| [lore: spi-mem conversion](https://lore.kernel.org/linux-mtd/20190903170850.6aee808b@collabora.com/T/) | mailing list | عالية |
| [eLinux RPi SPI](https://elinux.org/RPi_SPI) | مثال عملي | متوسطة |
| LDD3 Chapter 14-15 | كتاب | عالية |
| Embedded Linux Primer Ch. 8 | كتاب | متوسطة |
## Phase 8: Writing simple module

الـ `spi/eeprom.h` بسيط — بيعرّف struct واحد بس هو `spi_eeprom` اللي بيحمل metadata بتاعة الـ EEPROM chip. مفيش function مباشرة نعمل لها kprobe في الـ header نفسه، بس الـ driver الحقيقي `at25` بيستخدم الـ struct ده وبيقرأ/يكتب على الـ SPI bus.

الخيار الأذكى هنا: نعمل **kprobe** على `at25_ee_read` — الـ function الجوهرية في driver الـ `at25` اللي بيتنادى كل ما حاجة طلبت قراءة من الـ EEPROM. ده بيخلينا نشوف إيه الـ offset وإيه الـ size لكل read operation في الـ runtime.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * eeprom_probe.c
 * Hooks at25_ee_read() to log every SPI EEPROM read request.
 * Useful for debugging EEPROM access patterns at runtime.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/spi/eeprom.h>  /* struct spi_eeprom — gives us the chip metadata types */

/* ---------------------------------------------------------------
 * kprobe handler — fires just before at25_ee_read() executes
 * pt_regs holds the CPU registers at the moment of the probe hit;
 * on x86-64: rdi=1st arg, rsi=2nd arg, rdx=3rd arg, rcx=4th arg
 * at25_ee_read signature:
 *   ssize_t at25_ee_read(void *priv, char *buf, loff_t off, size_t count)
 * ---------------------------------------------------------------*/
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Extract call arguments from registers (x86-64 System V ABI) */
    loff_t  offset = (loff_t)regs->si;   /* 2nd arg: offset inside EEPROM  */
    size_t  count  = (size_t)regs->dx;   /* 3rd arg: number of bytes to read */

    pr_info("eeprom_probe: at25_ee_read hit — offset=%lld bytes=%zu\n",
            offset, count);

    return 0; /* 0 = continue normal execution, don't skip the function */
}

/* kprobe descriptor — we attach to the at25 driver's read path */
static struct kprobe kp = {
    .symbol_name = "at25_ee_read",  /* kernel symbol to hook           */
    .pre_handler = handler_pre,     /* callback before function runs   */
};

/* ---------------------------------------------------------------
 * module_init: register the kprobe so the kernel installs the
 * breakpoint at the target symbol address.
 * ---------------------------------------------------------------*/
static int __init eeprom_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        /* Symbol might not exist if at25 driver is not loaded */
        pr_err("eeprom_probe: register_kprobe failed, err=%d\n", ret);
        pr_err("eeprom_probe: is the at25 SPI EEPROM driver loaded?\n");
        return ret;
    }

    pr_info("eeprom_probe: planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ---------------------------------------------------------------
 * module_exit: MUST unregister the kprobe before the module is
 * removed — otherwise the breakpoint stays in kernel text and
 * causes a panic on the next hit.
 * ---------------------------------------------------------------*/
static void __exit eeprom_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("eeprom_probe: kprobe removed from %s\n", kp.symbol_name);
}

module_init(eeprom_probe_init);
module_exit(eeprom_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe on at25_ee_read to trace SPI EEPROM read calls");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | لازم لأي module — بيوفر `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kernel.h` | بيوفر `pr_info`, `pr_err` للـ logging |
| `linux/kprobes.h` | بيعرّف `struct kprobe` و `register_kprobe` / `unregister_kprobe` |
| `linux/spi/eeprom.h` | بيجيب تعريف `struct spi_eeprom` — ضروري لو احتجنا نعمل cast للـ `priv` pointer جوا الـ handler |

---

#### الـ `handler_pre` — قلب الـ module

الـ kprobe بيوقف التنفيذ لحظة ما الـ CPU يوصل لأول instruction في `at25_ee_read` ويشغّل الـ `pre_handler` بدل ما يكمّل. الـ `pt_regs` بيحمل snapshot من الـ registers في اللحظة دي.

على معمارية **x86-64** الـ System V ABI بتحط الـ arguments بالترتيب في: `rdi`, `rsi`, `rdx`, `rcx`. إحنا بنجيب:
- `regs->si` = الـ `off` (offset جوا الـ EEPROM)
- `regs->dx` = الـ `count` (عدد البايتات المطلوبة)

الـ return value صفر بيقول للـ kprobe framework "كمّل تنفيذ الـ function الأصلية عادي".

---

#### الـ `struct kprobe kp`

الـ `.symbol_name` بيقول للـ kernel على أنهي symbol نحط الـ breakpoint. الـ kernel بيبحث في الـ kallsyms ويحوّل الاسم لـ address. الـ `.pre_handler` هو الـ callback اللي بيتشغّل قبل الـ function.

---

#### الـ `module_init`

`register_kprobe` بتحط **software breakpoint** (على x86: بتبدّل أول byte بـ `int3`) في عنوان الـ `at25_ee_read`. لو الـ at25 driver مش موجود في الـ kernel أو مش مـ`load` الـ symbol مش هيتلاقي والـ register هيرجع error.

---

#### الـ `module_exit`

`unregister_kprobe` ضروري لازم يتعمل في الـ exit. لو اتحذف الـ module من غير ما يشيل الـ breakpoint، الـ kernel هيتـ call الـ handler function اللي بقى عنوانها invalid وهيحصل **kernel panic** فوراً. ده مش اختياري.

---

### الـ Makefile

```makefile
obj-m += eeprom_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### تشغيل الـ Module

```bash
# تحميل الـ at25 driver الأول (لو مش موجود)
sudo modprobe at25

# تحميل الـ module بتاعنا
sudo insmod eeprom_probe.ko

# شاهد الـ output في الـ kernel log
sudo dmesg -w

# لما تخلص
sudo rmmod eeprom_probe
```

**مثال على الـ output المتوقع:**

```
[  123.456789] eeprom_probe: planted kprobe at at25_ee_read (ffffffffc0123456)
[  124.001234] eeprom_probe: at25_ee_read hit — offset=0 bytes=256
[  124.002345] eeprom_probe: at25_ee_read hit — offset=256 bytes=256
[  125.789012] eeprom_probe: kprobe removed from at25_ee_read
```

كل سطر بيوضح إن حاجة في الـ system طلبت قراءة 256 byte من الـ EEPROM بدءاً من offset معين — ده مفيد جداً لـ debugging الـ firmware update code أو الـ calibration data اللي بتتخزن في EEPROM.
