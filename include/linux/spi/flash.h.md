## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

الملف `include/linux/spi/flash.h` هو **header صغير جداً** — 32 سطر فقط — بيعرّف struct واحدة اسمها `flash_platform_data`. الـ struct دي هي الطريقة القديمة (pre-Device Tree) اللي كانت بيها الـ board initialization code بتقول للـ kernel: "عندك SPI flash chip على الـ board دي، واسمها كذا، وعايزها تتقسم لـ partitions كذا."

---

### الصورة الكبيرة — القصة من الأول

#### المشكلة اللي بيحلها

تخيل إنك بتبني router أو embedded device. على الـ PCB فيه chip صغيرة — **SPI NOR Flash** أو **DataFlash** — فيها الـ bootloader والـ firmware والـ config. الـ chip دي متوصلة بـ SPI bus.

الـ Linux kernel محتاج يعرف عن الـ chip دي قبل ما يشغلها:
- **اسمها** إيه؟ (عشان تظهر صح في `/proc/mtd`)
- **نوعها بالظبط** إيه؟ (بعض الـ chips مش بتدعم JEDEC auto-detection)
- **هتتقسم إزاي؟** — يعني فين الـ bootloader، فين الـ kernel، فين الـ rootfs؟

زمان — قبل ما الـ Device Tree ينتشر — الإجابة على الأسئلة دي كانت بتيجي من **board file** في `arch/.../mach-xxx/board-yyy.c`. الـ board file ده بيملا `flash_platform_data` ويبعتها للـ driver وقت الـ registration.

#### التشبيه البسيط

فكّر في الـ `flash_platform_data` زي **ورقة التعليمات** اللي بتبعتها مع الـ driver:
- "الـ flash دي اسمها `m25p80`"
- "قسّمها لـ 3 أجزاء: bootloader (256KB) + kernel (2MB) + rootfs (باقي الـ chip)"

الـ driver بياخد الورقة دي، يعمل الـ MTD device، ويسجّل الـ partitions.

---

### الـ Subsystem اللي ينتمي له

الملف ده جزء من **تقاطع subsystemين**:

| Subsystem | الدور |
|---|---|
| **SPI Subsystem** (`drivers/spi/`, `include/linux/spi/`) | الـ transport layer — بيتحكم في إزاي البيانات بتتنقل على الـ SPI bus |
| **MTD Subsystem** (`drivers/mtd/`, `include/linux/mtd/`) | الـ storage layer — بيعرض الـ flash للـ kernel كـ MTD device قابل للـ partition |

الـ `flash.h` نفسه موجود في `include/linux/spi/` لأنه بيوصف SPI flash من ناحية الـ board configuration، لكن بيشتغل مع الـ MTD stack اللي بيعمل الـ partitioning الفعلي.

في MAINTAINERS، الـ file ده بيقع تحت نطاق **SPI NOR SUBSYSTEM** و**MEMORY TECHNOLOGY DEVICES (MTD)** معاً.

---

### محتوى الملف — الـ struct الوحيدة

```c
struct flash_platform_data {
    char         *name;     /* اسم الـ MTD device (e.g., "m25p80") */
    struct mtd_partition *parts;    /* array of partitions */
    unsigned int  nr_parts; /* عدد الـ partitions */

    char         *type;     /* chip type لو مش بيدعم JEDEC auto-detect */
};
```

**الـ `struct mtd_partition`** (من `include/linux/mtd/partitions.h`) بتعرّف كل partition:

```c
struct mtd_partition {
    const char *name;      /* اسم الـ partition (e.g., "bootloader") */
    uint64_t    size;      /* حجمها بالـ bytes */
    uint64_t    offset;    /* بداية الـ partition من أول الـ chip */
    uint32_t    mask_flags;/* flags زي read-only */
    /* ... */
};
```

---

### مثال واقعي — Router Board File

```c
/* arch/arm/mach-xxx/board-router.c */
#include <linux/spi/flash.h>
#include <linux/mtd/partitions.h>

/* تعريف الـ partitions على الـ 8MB SPI flash */
static struct mtd_partition router_flash_parts[] = {
    {
        .name   = "bootloader",
        .offset = 0,
        .size   = 256 * 1024,       /* 256 KB */
        .mask_flags = MTD_WRITEABLE, /* read-only للأمان */
    },
    {
        .name   = "kernel",
        .offset = MTDPART_OFS_APPEND, /* بعد الـ bootloader مباشرة */
        .size   = 2 * 1024 * 1024,   /* 2 MB */
    },
    {
        .name   = "rootfs",
        .offset = MTDPART_OFS_APPEND,
        .size   = MTDPART_SIZ_FULL,  /* باقي الـ chip */
    },
};

/* الـ platform data اللي بتتبعت للـ driver */
static struct flash_platform_data router_flash_data = {
    .name     = "router-flash",
    .parts    = router_flash_parts,
    .nr_parts = ARRAY_SIZE(router_flash_parts),
    .type     = "m25p64",  /* Micronix 64Mbit NOR flash */
};
```

لما الـ `spi_nor_probe()` في `drivers/mtd/spi-nor/core.c` بيشتغل، بيعمل `dev_get_platdata()` يجيب الـ `flash_platform_data` دي، وبيستخدم `name` و`type` لتعريف الـ chip، وبيبعت `parts` و`nr_parts` لـ `mtd_device_register()` عشان يعمل الـ partitions.

---

### الـ DataFlash — الحالة الخاصة

**DataFlash** هي flash chips من Atmel (سلسلة AT45xxx) بتتميز بإن:
- الـ **page sizes** مش powers of two (مثلاً 264 bytes بدل 256)
- الـ partitions لازم تكون **sector-aligned**

الـ driver بتاعها `drivers/mtd/devices/mtd_dataflash.c` بيستخدم نفس الـ `flash_platform_data` بالظبط:

```c
/* من mtd_dataflash.c */
struct flash_platform_data *pdata = dev_get_platdata(&spi->dev);

device->name = (pdata && pdata->name) ? pdata->name : priv->name;

err = mtd_device_register(device,
        pdata ? pdata->parts : NULL,    /* الـ partitions */
        pdata ? pdata->nr_parts : 0);   /* عددها */
```

---

### وضعه الحالي — Legacy Code

الـ `flash_platform_data` ده **legacy mechanism**. الـ boards الحديثة بتستخدم **Device Tree** بدل board files، وبتعرّف الـ partitions جوه الـ DTS مباشرةً:

```dts
/* modern DTS approach - no flash_platform_data needed */
flash@0 {
    compatible = "jedec,spi-nor";
    partitions {
        compatible = "fixed-partitions";
        bootloader: partition@0 {
            label = "bootloader";
            reg = <0x0 0x40000>;
            read-only;
        };
    };
};
```

لكن الـ struct لسه موجودة في الـ kernel عشان:
1. الـ **legacy boards** اللي لسه بتستخدم board files
2. الـ **drivers** اللي بتدعم الاتنين (DT + platform data)

---

### الملفات المرتبطة

| الملف | الدور |
|---|---|
| `include/linux/spi/flash.h` | **الملف ده** — تعريف `flash_platform_data` |
| `include/linux/mtd/partitions.h` | تعريف `struct mtd_partition` |
| `include/linux/mtd/mtd.h` | تعريف `struct mtd_info` — الـ MTD device الأساسي |
| `include/linux/mtd/spi-nor.h` | الـ API الحديث لـ SPI NOR flash |
| `drivers/mtd/spi-nor/core.c` | الـ SPI NOR driver الرئيسي — بيستخدم `flash_platform_data` في `spi_nor_probe()` |
| `drivers/mtd/devices/mtd_dataflash.c` | driver لـ Atmel AT45 DataFlash — بيستخدم `flash_platform_data` |
| `drivers/mtd/devices/sst25l.c` | driver لـ SST SPI flash — بيستخدم `flash_platform_data` |
| `drivers/mtd/devices/mchp23k256.c` | driver لـ Microchip SRAM — بيستخدم `flash_platform_data` |
| `drivers/spi/spi-butterfly.c` | AVR Butterfly board — مثال على استخدام `flash_platform_data` في board code |

---

### ملخص سريع

الـ `include/linux/spi/flash.h` هو **glue header بسيط** بيربط بين الـ board initialization code والـ SPI flash drivers. بيسمح للـ board code إنه يقول للـ driver: "الـ chip دي اسمها كذا، نوعها كذا، وعايزاها تتقسم كذا." ده الأساس اللي شغّل كذا مليون embedded device قبل ما الـ Device Tree يبقى standard.
## Phase 2: شرح الـ SPI Flash Platform Data Framework

### المشكلة اللي بتحلها

في الأنظمة المدمجة، عندك flash chip متوصلة على SPI bus — زي `m25p80` أو `at25df641` أو DataFlash من Atmel. الـ kernel لما يشوف الـ SPI device، بيحتاج يعرف:

1. **اسم** الـ flash device عشان الـ MTD subsystem يعرفه صح.
2. **نوع الـ chip** لو مش ممكن auto-detect بـ JEDEC ID — بعض الـ chips القديمة مش بتدعمها.
3. **partition layout** — إيه المناطق في الـ flash؟ bootloader فين، kernel فين، rootfs فين؟

المشكلة الأساسية: **الـ SPI flash driver نفسه ما يعرفش** هذه المعلومات — هي board-specific خالص. كل board ليها layout مختلف. لو حطيت الـ layout جوا الـ driver، هيبقى مش reusable.

**الحل التقليدي (pre-DT era):** نمرر البيانات من **board init code** للـ driver عن طريق `platform_data` — وده بالظبط الـ `struct flash_platform_data`.

---

### الحل اللي بيقدمه الـ Kernel

الـ kernel بيفصل بين ثلاث طبقات:

| الطبقة | المسؤولية |
|--------|-----------|
| **Board code** (`arch/.../board-yyy.c`) | بيعرف الـ hardware — يحدد الأسماء والـ partitions |
| **SPI flash driver** (e.g. `m25p80.c`) | بيتعامل مع الـ chip protocol نفسها |
| **MTD subsystem** | بيوفر abstraction موحدة فوق أي flash |

الـ `struct flash_platform_data` هو الـ **contract** بين الـ board code والـ SPI flash driver.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────┐
  │                   User Space / MTD Utils                │
  │              (mtdinfo, flash_erase, nandwrite)          │
  └───────────────────────────┬─────────────────────────────┘
                              │ ioctl / read / write
  ┌───────────────────────────▼─────────────────────────────┐
  │                     MTD Subsystem                       │
  │         (drivers/mtd/mtdcore.c, mtdpart.c)              │
  │   - يسجل كل flash device كـ /dev/mtdX                  │
  │   - يدير الـ partitions                                 │
  └───────────────────────────┬─────────────────────────────┘
                              │ struct mtd_info ops
  ┌───────────────────────────▼─────────────────────────────┐
  │               SPI NOR Flash Driver                      │
  │         (drivers/mtd/devices/m25p80.c  أو              │
  │          drivers/mtd/spi-nor/core.c)                    │
  │   - يقرأ الـ flash_platform_data                       │
  │   - يعمل JEDEC detection لو مش موجود type              │
  │   - يسجل الـ partitions في MTD                         │
  └───────────────────────────┬─────────────────────────────┘
                              │ SPI transfers
  ┌───────────────────────────▼─────────────────────────────┐
  │                    SPI Subsystem                        │
  │         (drivers/spi/spi.c)                             │
  │   - يدير الـ spi_master و spi_device                   │
  └───────────────────────────┬─────────────────────────────┘
                              │ hardware registers
  ┌───────────────────────────▼─────────────────────────────┐
  │              SPI Controller Driver                      │
  │   (e.g. drivers/spi/spi-omap2-mcspi.c)                 │
  └─────────────────────────────────────────────────────────┘

  Board Init Code (arch/arm/mach-omap/board-overo.c):
  ┌─────────────────────────────────────────────────────────┐
  │  flash_platform_data → spi_board_info → spi_register_   │
  │  board_info()  →  SPI core  →  driver probe()           │
  └─────────────────────────────────────────────────────────┘
```

---

### الـ Struct بالتفصيل

```c
/* include/linux/spi/flash.h */
struct flash_platform_data {
    char         *name;     /* اسم الـ MTD device — يظهر في /proc/mtd */
    struct mtd_partition *parts;    /* مصفوفة الـ partitions */
    unsigned int  nr_parts; /* عدد الـ partitions */
    char         *type;     /* نوع الـ chip لو مش auto-detectable */
};
```

**الـ `struct mtd_partition`** (من `include/linux/mtd/partitions.h`):

```c
struct mtd_partition {
    const char   *name;        /* اسم الـ partition — /dev/mtdX */
    const char *const *types;  /* parsers لو فيه subpartitions */
    uint64_t      size;        /* حجم الـ partition بالـ bytes */
    uint64_t      offset;      /* بداية الـ partition من أول الـ flash */
    uint32_t      mask_flags;  /* flags تتشال من الـ master MTD */
    uint32_t      add_flags;   /* flags تتضاف */
    struct device_node *of_node;
};
```

---

### العلاقة بين الـ Structs

```
  flash_platform_data
  ┌─────────────────────────────┐
  │ name: "spi-flash"           │
  │ type: "m25p80"              │
  │ nr_parts: 3                 │
  │ parts ──────────────────────┼──► mtd_partition[0]
  └─────────────────────────────┘     ┌──────────────────────────┐
                                      │ name:   "bootloader"     │
                                      │ offset: 0x00000000       │
                                      │ size:   0x00040000 (256K)│
                                      └──────────────────────────┘
                                     mtd_partition[1]
                                      ┌──────────────────────────┐
                                      │ name:   "kernel"         │
                                      │ offset: MTDPART_OFS_APPEND│
                                      │ size:   0x00200000 (2M)  │
                                      └──────────────────────────┘
                                     mtd_partition[2]
                                      ┌──────────────────────────┐
                                      │ name:   "rootfs"         │
                                      │ offset: MTDPART_OFS_APPEND│
                                      │ size:   MTDPART_SIZ_FULL │
                                      └──────────────────────────┘
```

---

### مثال حقيقي — Board Code

```c
/* arch/arm/mach-omap1/board-htcherald.c (مثال مبسط) */

static struct mtd_partition htc_flash_partitions[] = {
    {
        .name   = "bootloader",
        .offset = 0,
        .size   = SZ_256K,          /* 256 KB */
    },
    {
        .name   = "kernel",
        .offset = MTDPART_OFS_APPEND,   /* يبدأ بعد السابق مباشرة */
        .size   = SZ_2M,            /* 2 MB */
    },
    {
        .name   = "rootfs",
        .offset = MTDPART_OFS_APPEND,
        .size   = MTDPART_SIZ_FULL, /* كل الباقي */
    },
};

static struct flash_platform_data htc_flash_data = {
    .name     = "spi-flash",
    .parts    = htc_flash_partitions,
    .nr_parts = ARRAY_SIZE(htc_flash_partitions),
    /* .type = NULL  →  سيتم auto-detect بـ JEDEC */
};

static struct spi_board_info htc_spi_board_info[] = {
    {
        .modalias        = "m25p80",        /* اسم الـ driver */
        .max_speed_hz    = 6000000,         /* 6 MHz */
        .bus_num         = 1,
        .chip_select     = 0,
        .platform_data   = &htc_flash_data, /* هنا بنمرر البيانات */
    },
};

static void __init htc_init(void) {
    spi_register_board_info(htc_spi_board_info,
                            ARRAY_SIZE(htc_spi_board_info));
}
```

---

### كيف الـ Driver بيستخدم البيانات دي

```c
/* drivers/mtd/devices/m25p80.c — مبسط */
static int m25p_probe(struct spi_device *spi)
{
    struct flash_platform_data *data;
    const char *flash_name = NULL;

    /* استخراج الـ platform_data اللي حطها board code */
    data = dev_get_platdata(&spi->dev);

    if (data && data->type)
        flash_name = data->type;   /* استخدم النوع الصريح */
    else
        flash_name = NULL;          /* اعتمد على JEDEC auto-detect */

    /* ... probe الـ chip, بناء struct mtd_info ... */

    if (data && data->nr_parts) {
        /* سجل الـ partitions في MTD subsystem */
        mtd_device_register(mtd, data->parts, data->nr_parts);
    } else {
        mtd_device_register(mtd, NULL, 0);
    }

    return 0;
}
```

---

### التشابه الحقيقي — قياس من الواقع

تخيل إنك **مقاول بناء** (SPI flash driver). بتبني بيوت، بس مش بتقرر ازاي تقسم الأوضة — ده شغل **المالك** (board code).

| المقاول = SPI Flash Driver | المالك = Board Code |
|---------------------------|---------------------|
| عارف يبني أي نوع غرف | يحدد عدد الغرف واستخدامها |
| بيعرف يشتغل مع أي سيراميك | يختار نوع التشطيب |
| ما بيعرفش إيه اللي هيتخزن | بيقرر: غرفة نوم، مطبخ، مكتب |
| **الـ `struct flash_platform_data`** = **عقد التشطيب** بين المالك والمقاول |

الـ **`name`** = لافتة البيت (عنوانه في `/proc/mtd`).
الـ **`parts`** = مخطط تقسيم الأوضة (partition layout).
الـ **`nr_parts`** = عدد الأوضة.
الـ **`type`** = المواصفات الخاصة لو المقاول مش هيعرف يكتشفها لوحده (مثلاً chips بدون JEDEC ID).

---

### الـ Core Abstraction

الـ `flash_platform_data` هو **وسيط تمرير المعلومات من الـ board layer للـ driver layer** — بيحل مشكلة الـ **separation of concerns**:

- الـ driver يعرف **كيف** يتكلم مع الـ chip.
- الـ board code يعرف **ما هو** موجود على الـ board وكيف مقسّم.

---

### ما بيملكه هذا الـ Header مقابل ما بيفوّضه

| يملك | يفوّض |
|------|--------|
| تعريف `struct flash_platform_data` | تسجيل الـ device في MTD — ده شغل `mtd_device_register()` |
| تعريف الـ fields الضرورية (name, parts, type) | الـ partition parsing من DT أو cmdline — ده شغل MTD partition parsers |
| forward declaration لـ `struct mtd_partition` | تفاصيل الـ SPI protocol — ده شغل `spi.h` و driver |
| | الـ JEDEC chip detection — ده شغل الـ flash driver نفسه |

---

### ملاحظة مهمة — الـ DT Era

الـ `flash_platform_data` هو **legacy mechanism** من قبل Device Tree. في الـ kernel الحديث، نفس المعلومات بتيجي من الـ `.dts` file:

```dts
/* مثال DTS حديث */
flash@0 {
    compatible = "jedec,spi-nor";
    reg = <0>;
    spi-max-frequency = <50000000>;

    partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        bootloader@0 {
            label = "bootloader";
            reg = <0x0 0x40000>;
        };
        kernel@40000 {
            label = "kernel";
            reg = <0x40000 0x200000>;
        };
    };
};
```

لكن الـ `struct flash_platform_data` لازال موجود لدعم الـ boards القديمة والـ out-of-tree BSPs اللي ما اتحولتش لـ DT لحد دلوقتي. الـ **MTD subsystem** هو اللي بيوحد الاتنين — سواء الـ data جت من `platform_data` أو من DT، النتيجة النهائية هي نفس الـ `mtd_info` objects المسجلة.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### ملاحظة أولية على الـ File

الـ `include/linux/spi/flash.h` ملف صغير جداً — 32 سطر بس — لكنه **نقطة التقاء** بين ثلاث طبقات مهمة:
- الـ **board code** (arch/…/board-xxx.c) اللي بيحدد الـ flash hardware
- الـ **SPI bus layer** اللي بينقل البيانات
- الـ **MTD subsystem** اللي بيدير الـ flash partitions

الـ file ده بيعرّف struct واحدة بس: `flash_platform_data`.

---

### 0. Flags, Macros & Constants المستخدمة مع الـ File

الـ `flash.h` نفسه مفيش فيه flags، لكن الـ structs اللي بيشير ليها (`mtd_partition`) بتستخدم الـ constants دي:

#### Partition Offset Constants (من `mtd/partitions.h`)

| Constant | Value | المعنى |
|---|---|---|
| `MTDPART_OFS_RETAIN` | `-3` | الـ partition بياخد كل المساحة المتبقية ماعدا `size` bytes في الآخر |
| `MTDPART_OFS_NXTBLK` | `-2` | الـ partition يبدأ من أول erase block بعد السابق |
| `MTDPART_OFS_APPEND` | `-1` | الـ partition يبدأ بعد اللي قبله مباشرةً |
| `MTDPART_SIZ_FULL` | `0` | الـ partition يمتد لآخر الـ flash |

#### Partition Flag Masks (من `mtd/mtd.h`)

| Flag | المعنى |
|---|---|
| `MTD_WRITEABLE` | الـ partition قابل للكتابة |
| `MTD_NO_ERASE` | مفيش erase (RAM-based) |
| `MTD_POWERUP_LOCK` | locked عند الـ power-up |

---

### 1. الـ Structs المهمة

#### 1.1 `struct flash_platform_data` — البطل الرئيسي

**الغرض:** بتحمل الـ board-specific metadata عن شريحة الـ SPI flash، وبتتمرر للـ driver عبر `spi_board_info.platform_data`.

```c
/* include/linux/spi/flash.h */
struct flash_platform_data {
    char             *name;     /* اسم اختياري للـ MTD device (للـ cmdline partitions) */
    struct mtd_partition *parts;  /* مصفوفة static partitions */
    unsigned int      nr_parts;  /* عدد الـ partitions */
    char             *type;     /* نوع الشريحة لو مش بتدعم JEDEC auto-detect */
    /* يمكن يتضاف فيها JEDEC IDs مستقبلاً */
};
```

| الحقل | النوع | إلزامي؟ | الوظيفة |
|---|---|---|---|
| `name` | `char *` | لا | اسم الـ MTD device في `/proc/mtd` ولـ `mtdparts=` kernel cmdline |
| `parts` | `struct mtd_partition *` | لا | مصفوفة الـ partitions الـ static |
| `nr_parts` | `unsigned int` | لا | حجم المصفوفة `parts` |
| `type` | `char *` | لا | اسم الـ chip model (مثلاً `"m25p80"`, `"w25q32bv"`) لو الـ JEDEC ID مش كافي |

**ازاي الـ driver بيستخدمها:**

```c
/* من drivers/mtd/spi-nor/core.c — spi_nor_probe() */
struct flash_platform_data *data = dev_get_platdata(dev);

/* استخدام name */
if (data && data->name)
    nor->mtd.name = data->name;

/* استخدام type — override الـ auto-detection */
if (data && data->type)
    flash_name = data->type;

/* تسجيل الـ partitions */
return mtd_device_register(&nor->mtd,
                            data ? data->parts : NULL,
                            data ? data->nr_parts : 0);
```

---

#### 1.2 `struct mtd_partition` — تعريف الـ Partition

**الغرض:** بتصف partition واحد جوه الـ flash: اسمه، حجمه، مكانه.

```c
/* include/linux/mtd/partitions.h */
struct mtd_partition {
    const char *name;         /* اسم الـ partition (مثلاً "bootloader", "rootfs") */
    const char *const *types; /* parsers للـ sub-partitions (مثلاً FIT, RedBoot) */
    uint64_t   size;          /* الحجم بالبايت، أو MTDPART_SIZ_FULL */
    uint64_t   offset;        /* الـ offset من بداية الـ flash، أو MTDPART_OFS_* */
    uint32_t   mask_flags;    /* flags تتمسح من الـ parent MTD */
    uint32_t   add_flags;     /* flags تتضاف للـ partition */
    struct device_node *of_node; /* ربط بالـ Device Tree node */
};
```

| الحقل | الاستخدام الشائع |
|---|---|
| `name` | `"u-boot"`, `"kernel"`, `"rootfs"`, `"data"` |
| `size` | `SZ_256K`, `SZ_4M`, أو `MTDPART_SIZ_FULL` للآخر |
| `offset` | `0` للأول، `MTDPART_OFS_APPEND` للباقي |
| `mask_flags` | `MTD_WRITEABLE` لجعل الـ partition read-only |
| `add_flags` | نادر الاستخدام |

---

#### 1.3 `struct spi_board_info` — الحلقة الرابطة

ده مش في الـ `flash.h` لكنه اللي بيحمل الـ `flash_platform_data` ويوصلها للـ driver:

```c
/* include/linux/spi/spi.h */
struct spi_board_info {
    char        modalias[SPI_NAME_SIZE]; /* اسم الـ driver */
    const void  *platform_data;          /* ← هنا بيتحط flash_platform_data */
    const struct software_node *swnode;
    void        *controller_data;
    int         irq;
    u32         max_speed_hz;
    u16         bus_num;
    u16         chip_select;
    u32         mode;
};
```

**مثال حقيقي من `arch/arm/mach-dove/cm-a510.c`:**

```c
/* تعريف flash data */
static const struct flash_platform_data cm_a510_spi_flash_data = {
    .type = "w25q32bv",  /* Winbond 32Mbit NOR flash */
};

/* ربطها بالـ SPI device */
static struct spi_board_info __initdata cm_a510_spi_flash_info[] = {
    {
        .modalias      = "m25p80",                   /* اسم الـ driver */
        .platform_data = &cm_a510_spi_flash_data,    /* ← platform data */
        .irq           = -1,
        .max_speed_hz  = 20000000,                   /* 20 MHz */
        .bus_num       = 0,
        .chip_select   = 0,
    },
};
```

---

#### 1.4 `struct mtd_info` — الـ MTD Device Object

الـ driver بيملّي الـ struct ده ويسجله في الـ MTD layer. الـ `flash_platform_data` بتغذيه بالـ name والـ partitions.

```c
/* مختصر من include/linux/mtd/mtd.h */
struct mtd_info {
    u_char   type;          /* MTD_NORFLASH, MTD_DATAFLASH, إلخ */
    uint32_t flags;         /* MTD_WRITEABLE, إلخ */
    uint64_t size;          /* الحجم الكلي بالبايت */
    uint32_t erasesize;
    uint32_t writesize;
    const char *name;       /* ← بييجي من flash_platform_data.name */
    /* ... */
};
```

---

### 2. مخطط العلاقات بين الـ Structs (ASCII)

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    Board Init Code                          │
  │              (arch/.../board-xxx.c)                         │
  │                                                             │
  │   flash_platform_data                                       │
  │  ┌───────────────────┐                                      │
  │  │ name: "my-flash"  │                                      │
  │  │ type: "w25q32bv"  │                                      │
  │  │ parts ────────────┼──→ mtd_partition[]                   │
  │  │ nr_parts: 3       │    ┌──────────────────┐              │
  │  └───────────────────┘    │ name: "uboot"    │              │
  │           │               │ size: 0x40000    │              │
  │           │               │ offset: 0        │              │
  │           ↓               ├──────────────────┤              │
  │   spi_board_info          │ name: "kernel"   │              │
  │  ┌───────────────────┐    │ size: 0x300000   │              │
  │  │ modalias:"m25p80" │    │ offset: APPEND   │              │
  │  │ platform_data ────┼─┐  ├──────────────────┤              │
  │  │ max_speed_hz      │ │  │ name: "rootfs"   │              │
  │  │ bus_num: 0        │ │  │ size:  FULL      │              │
  │  │ chip_select: 0    │ │  │ offset: APPEND   │              │
  │  └───────────────────┘ │  └──────────────────┘              │
  └────────────────────────┼────────────────────────────────────┘
                           │
  spi_register_board_info()│
                           ↓
  ┌─────────────────────────────────────────────────────────────┐
  │                    SPI Core / Driver                        │
  │                                                             │
  │   spi_device                                                │
  │  ┌───────────────────┐                                      │
  │  │ dev.platform_data ┼──→ flash_platform_data (same ptr)   │
  │  │ modalias          │                                      │
  │  │ max_speed_hz      │                                      │
  │  └───────────────────┘                                      │
  │           │                                                 │
  │           │  dev_get_platdata(&spi->dev)                    │
  │           ↓                                                 │
  │   spi_nor / dataflash driver                                │
  │  ┌───────────────────┐                                      │
  │  │ mtd_info          │                                      │
  │  │  ├ name ←─────────┼── pdata->name                       │
  │  │  ├ size            │                                     │
  │  │  └ ...             │                                     │
  │  └───────────────────┘                                      │
  │           │                                                 │
  │           │  mtd_device_register(mtd, pdata->parts,        │
  │           │                      pdata->nr_parts)           │
  │           ↓                                                 │
  │   MTD Subsystem (/dev/mtdX, /dev/mtdblockX)                │
  └─────────────────────────────────────────────────────────────┘
```

---

### 3. مخطط الـ Lifecycle

```
  [Board Init]
      │
      │  static struct flash_platform_data myflash = { ... };
      │  static struct mtd_partition parts[] = { ... };
      │  static struct spi_board_info info[] = { .platform_data = &myflash };
      │
      ↓
  [Kernel Boot — arch_initcall / machine_init]
      │
      │  spi_register_board_info(info, ARRAY_SIZE(info));
      │    └→ kernel يحفظ الـ info في قائمة داخلية
      │
      ↓
  [SPI Controller Driver Probe]
      │
      │  spi_register_controller(ctlr);
      │    └→ kernel يبدأ يـ probe كل device في الـ board_info list
      │    └→ بييجي على flash info → بيعمل spi_device جديد
      │         spi_device->dev.platform_data = myflash  ←─────────┐
      │                                                             │
      ↓                                                             │
  [Flash Driver Probe — مثلاً spi_nor_probe()]                     │
      │                                                             │
      │  data = dev_get_platdata(&spi->dev);  ─────────────────────┘
      │  // data هي نفس flash_platform_data اللي البورد حددها
      │
      │  1. لو data->type موجود → استخدمه بدل JEDEC auto-detect
      │  2. لو data->name موجود → سمّي الـ MTD device بيه
      │  3. اعمل spi_nor_scan() لتهيئة الـ chip
      │  4. mtd_device_register(&nor->mtd, data->parts, data->nr_parts)
      │       └→ MTD layer بيسجل الـ partitions
      │       └→ /dev/mtd0, /dev/mtd1, ... بيظهروا
      │
      ↓
  [Runtime]
      │
      │  userspace يقرأ/يكتب على /dev/mtdX أو /dev/mtdblockX
      │  الـ driver بيتعامل مع الـ SPI flash مباشرةً
      │
      ↓
  [Teardown — Driver Remove]
      │
      │  mtd_device_unregister(&nor->mtd)
      │    └→ كل الـ partitions بتتشال من /dev/
      │  spi_mem_put_drvdata() / devm cleanup
      │  الـ flash_platform_data نفسها بقت static → مش بتتحرر
      └→ (بتفضل موجودة لحد reboot)
```

---

### 4. مخطط الـ Call Flow

#### 4.1 من الـ Board Code للـ MTD Registration

```
board_init()
  └→ spi_register_board_info(cm_a510_spi_flash_info, 1)
       └→ [kernel يحفظ board info]

spi_register_controller(controller)
  └→ spi_match_master_to_boardinfo()
       └→ spi_new_device(master, chip_info)
            └→ device_add(&spi->dev)
                 └→ spi_drv_probe()  [driver: m25p80 / spi-nor]
                      └→ spi_nor_probe(spimem)
                           ├→ dev_get_platdata()        ← يجيب flash_platform_data
                           ├→ spi_nor_scan(nor, "w25q32bv", &hwcaps)
                           │    ├→ spi_nor_detect()     ← بيقرأ JEDEC ID
                           │    ├→ spi_nor_init_params()
                           │    └→ spi_nor_init()
                           └→ mtd_device_register(&nor->mtd, parts, nr_parts)
                                ├→ mtd_add_device()
                                ├→ add_mtd_partitions()
                                │    └→ لكل partition: allocate_partition()
                                │         └→ device_add() → /dev/mtdN
                                └→ [MTD notifiers fired]
```

#### 4.2 استخدام `type` لتجاوز الـ Auto-Detection

```
spi_nor_probe()
  ├→ data->type = "w25q32bv"  ← موجود
  └→ spi_nor_scan(nor, "w25q32bv", hwcaps)
       └→ spi_nor_match_name("w25q32bv")
            ├→ يدور في جدول الـ manufacturers[]
            │    └→ يلاقي Winbond W25Q32BV
            └→ يـ skip الـ JEDEC read
                 └→ يستخدم الـ params المسجّلة مباشرةً
```

#### 4.3 استخدام `parts` لتعريف الـ Partitions

```
mtd_device_register(&nor->mtd, parts, nr_parts)
  └→ parse_mtd_partitions()
       ├→ لو nr_parts > 0 → استخدم الـ static parts مباشرةً
       │    (بدل ما يحاول يـ parse من OF/cmdline)
       └→ add_mtd_partitions(mtd, parts, nr_parts)
            └→ for each part:
                 allocate_partition(mtd, part, i, offset)
                   ├→ child = kzalloc(mtd_info)
                   ├→ child->name = part->name
                   ├→ child->offset = part->offset
                   ├→ child->size = part->size
                   ├→ child->flags = (master->flags & ~mask_flags) | add_flags
                   └→ add_mtd_device(child)
                        └→ /dev/mtdN created
```

---

### 5. استراتيجية الـ Locking

الـ `flash_platform_data` نفسها **مفيش فيها locking** — ده by design لأنها:

1. **Read-only بعد الـ init**: البورد بتحددها في compile time أو boot time كـ static data. مفيش حد بيكتب فيها بعد الـ probe.
2. **بتتمرر مرة واحدة**: الـ SPI core بياخد pointer ليها وقت الـ probe، وبعدين الـ driver بيقرأ منها بس مرة واحدة.

**اللي فيه locking هو الـ MTD layer وأعمق:**

| الـ Lock | المكان | بيحمي إيه |
|---|---|---|
| `mtd_table_mutex` | `drivers/mtd/mtdcore.c` | قائمة الـ MTD devices عند الـ register/unregister |
| `mutex` في `struct dataflash` | `mtd_dataflash.c` | الـ SPI transactions (read/write/erase) |
| `nor->lock` (rwsem) | `spi-nor/core.c` | الـ NOR flash state عند العمليات المتزامنة |
| `spi->controller->lock` | `spi/spi.c` | الـ SPI bus نفسه |

**ترتيب الـ Locking (lock ordering):**

```
spi->controller->lock       (أعلى مستوى — bus lock)
  └→ nor->lock / dataflash->lock   (device lock)
       └→ mtd_table_mutex          (registry lock — فقط عند probe/remove)
```

> الـ `flash_platform_data` بتتقرأ **قبل** أي lock يتأخد، في أول الـ probe function، وده آمن لأنها static ومش بتتغير.

---

### 6. الفرق بين `name` و `type` في `flash_platform_data`

ده confusion شائع — الـ kernel نفسه بيوضحه في comment:

```
/*
 * For some (historical?) reason many platforms provide two different
 * names in flash_platform_data: "name" and "type". Quite often name is
 * set to "m25p80" and then "type" provides a real chip name.
 * If that's the case, respect "type" and ignore a "name".
 */
```

| الحقل | يُستخدم في | الغرض الفعلي |
|---|---|---|
| `name` | `nor->mtd.name` | الاسم اللي بيظهر في `/proc/mtd` |
| `type` | `spi_nor_scan(flash_name, ...)` | تحديد نوع الـ chip للـ driver |

**مثال توضيحي:**

```c
/* خطأ شائع قديم */
static struct flash_platform_data old_style = {
    .name = "m25p80",     /* ده اسم الـ driver مش اسم الـ flash! */
    .type = "m25p32",     /* ده النوع الحقيقي */
};

/* الاستخدام الصح */
static struct flash_platform_data correct = {
    .name = "main-flash", /* اسم وصفي للـ MTD device */
    .type = "w25q32bv",   /* نوع الـ chip الحقيقي */
};
```

---

### 7. صورة شاملة — من الـ Board للـ /dev

```
  ┌──────────────────────────────────────────────────────┐
  │  Board Code (compile-time static data)               │
  │                                                      │
  │  flash_platform_data:                                │
  │    name="my-nor"  type="w25q32bv"                    │
  │    parts=[uboot|kernel|rootfs]  nr_parts=3           │
  └──────────────────┬───────────────────────────────────┘
                     │ platform_data pointer
                     ↓
  ┌──────────────────────────────────────────────────────┐
  │  SPI Layer                                           │
  │  spi_device → dev.platform_data → flash_platform_data│
  └──────────────────┬───────────────────────────────────┘
                     │ dev_get_platdata()
                     ↓
  ┌──────────────────────────────────────────────────────┐
  │  SPI NOR Driver (spi-nor/core.c)                     │
  │                                                      │
  │  1. pdata->type → chip identification                │
  │  2. pdata->name → mtd_info.name                      │
  │  3. pdata->parts + nr_parts → mtd_device_register() │
  └──────────────────┬───────────────────────────────────┘
                     │
                     ↓
  ┌──────────────────────────────────────────────────────┐
  │  MTD Subsystem                                       │
  │                                                      │
  │  /dev/mtd0  → full flash ("my-nor")                  │
  │  /dev/mtd1  → "uboot"   (offset=0,      size=256K)  │
  │  /dev/mtd2  → "kernel"  (offset=256K,   size=3M)    │
  │  /dev/mtd3  → "rootfs"  (offset=3.25M,  size=rest)  │
  │                                                      │
  │  /dev/mtdblock0..3  (block device interface)         │
  └──────────────────────────────────────────────────────┘
```
## Phase 4: شرح الـ Functions

> الـ `include/linux/spi/flash.h` هو header بسيط جداً — ما فيهوش functions خالص. كل اللي فيه تعريف struct واحد بس: `flash_platform_data`. لكن الـ struct ده بيلعب دور محوري في ربط الـ board-level platform data بالـ SPI flash driver، وبالتالي الشرح هنا هيتمحور حول الـ data structures والـ APIs اللي بتتعامل معاها في سياق أوسع.

---

### ملخص سريع — Cheatsheet

| العنصر | النوع | الغرض |
|--------|-------|--------|
| `flash_platform_data` | `struct` | بيانات الـ board اللي بتتبعت للـ SPI flash driver وقت الـ probe |
| `flash_platform_data.name` | `char *` | اسم الـ MTD device (بيُستخدم في `mtdparts=` kernel cmdline) |
| `flash_platform_data.parts` | `struct mtd_partition *` | array الـ static partitions |
| `flash_platform_data.nr_parts` | `unsigned int` | عدد الـ partitions في الـ array |
| `flash_platform_data.type` | `char *` | نوع الـ flash chip يدوياً لو مش بيدعم JEDEC ID |

---

### الـ Data Structure: `flash_platform_data`

#### تعريف الـ Struct

```c
struct flash_platform_data {
    char             *name;     /* optional: MTD device name */
    struct mtd_partition *parts; /* optional: static partition table */
    unsigned int      nr_parts; /* number of partitions */
    char             *type;     /* optional: chip type string */
    /* future: JEDEC IDs, etc. */
};
```

#### شرح تفصيلي لكل field

---

##### `name` — اسم الـ MTD device

```c
char *name;
```

- **الـ** `name` ده بيتبعت للـ MTD layer عشان يُسمّي الـ MTD device المُنشأ — بيظهر في `/proc/mtd` وبيتعرف به في kernel cmdline بـ `mtdparts=`.
- لو `NULL`، الـ driver بيختار اسم افتراضي (عادةً اسم الـ SPI device نفسه زي `"m25p80"`).
- **الـ** field ده optional تماماً — الـ driver مش بيفشل لو مش موجود.

**مثال board code:**
```c
static struct flash_platform_data my_flash_data = {
    .name = "nor-flash",  /* سيظهر في /proc/mtd */
};
```

---

##### `parts` — جدول الـ Static Partitions

```c
struct mtd_partition *parts;
```

- **الـ** `parts` بيشير لـ array من `struct mtd_partition` بيحدد الـ static partition layout للـ flash.
- الـ driver بيمرر الـ array دي مع عدد الـ partitions لـ `mtd_device_register()` أو ما يعادلها.
- لو `NULL`، مفيش static partitions — الـ kernel ممكن يلجأ لـ dynamic parsers (RedBoot, OF, etc.) بناءً على الـ driver config.

**مثال:**
```c
static struct mtd_partition my_partitions[] = {
    {
        .name   = "bootloader",
        .offset = 0,
        .size   = SZ_256K,
    },
    {
        .name   = "kernel",
        .offset = MTDPART_OFS_APPEND,
        .size   = SZ_4M,
    },
    {
        .name   = "rootfs",
        .offset = MTDPART_OFS_APPEND,
        .size   = MTDPART_SIZ_FULL,  /* باقي الـ flash */
    },
};

static struct flash_platform_data my_flash = {
    .parts    = my_partitions,
    .nr_parts = ARRAY_SIZE(my_partitions),
};
```

**تفاصيل `struct mtd_partition`:**

| Field | النوع | الغرض |
|-------|-------|--------|
| `name` | `const char *` | اسم الـ partition (يظهر في `/proc/mtd`) |
| `types` | `const char * const *` | parsers للـ sub-partitions (RedBoot, etc.) |
| `size` | `uint64_t` | حجم الـ partition — `MTDPART_SIZ_FULL` = باقي الـ flash |
| `offset` | `uint64_t` | بداية الـ partition — `MTDPART_OFS_APPEND` = بعد السابق مباشرة |
| `mask_flags` | `uint32_t` | flags تُزال من الـ master (مثلاً `MTD_WRITEABLE` لـ read-only partition) |
| `add_flags` | `uint32_t` | flags تُضاف للـ partition |
| `of_node` | `struct device_node *` | ربط بـ Device Tree node |

**الـ Magic values للـ offset:**

```c
#define MTDPART_OFS_RETAIN  (-3)  /* ابدأ عشان يفضل size بايت في الآخر */
#define MTDPART_OFS_NXTBLK  (-2)  /* ابدأ من أول erase block تالي */
#define MTDPART_OFS_APPEND  (-1)  /* ابدأ بعد الـ partition اللي قبله */
#define MTDPART_SIZ_FULL    (0)   /* امتد لآخر الـ flash */
```

---

##### `nr_parts` — عدد الـ Partitions

```c
unsigned int nr_parts;
```

- **الـ** `nr_parts` لازم يتطابق بالظبط مع عدد العناصر في الـ `parts` array.
- لو `parts != NULL` و `nr_parts == 0`، مفيش حاجة هتتعمل — الـ driver بيتجاهل الـ array.
- الـ macro `ARRAY_SIZE()` هو الأكثر أماناً لتعبئة الـ field ده.

---

##### `type` — نوع الـ Flash Chip اليدوي

```c
char *type;
```

- **الـ** `type` بيُستخدم لـ chips اللي مش بتدعم JEDEC ID query أو بتحتاج identification يدوية.
- الـ driver بيقارن الـ string ده بجدول داخلي من الـ chip names المدعومة.
- مثال قيمة: `"m25p80"`, `"at45db161d"` (DataFlash).
- لو `NULL`، الـ driver بيحاول يعمل auto-detection عن طريق JEDEC/SFDP read.

**مثال DataFlash (مش بيدعم JEDEC):**
```c
static struct flash_platform_data df_flash = {
    .type = "at45db161d",  /* Atmel DataFlash — manual ID */
};
```

> ملاحظة مهمة على DataFlash: الـ page/block/sector sizes مش powers of two — لازم الـ partitions تبدأ على حدود sector.

---

### سياق الاستخدام — كيف بيتعامل معاها الـ SPI flash driver

#### الـ Flow من الـ Board Code للـ Driver

```
Board Init Code (arch/.../mach-xxx/board-yyy.c)
        |
        | .platform_data = &flash_platform_data
        v
spi_board_info / spi_register_board_info()
        |
        v
SPI core → spi_add_device()
        |
        v
SPI flash driver .probe() (e.g., m25p80_probe / spi-nor probe)
        |
        | spi_get_device_id() أو JEDEC query
        | dev_get_platdata(dev) ← هنا بياخد الـ flash_platform_data
        v
mtd_device_register(mtd, data->parts, data->nr_parts)
```

#### Pseudocode لـ Driver Probe

```c
static int spi_flash_probe(struct spi_device *spi)
{
    struct flash_platform_data *data = dev_get_platdata(&spi->dev);

    /* 1. حدد نوع الـ chip */
    if (data && data->type)
        /* استخدم الـ type string للـ manual lookup */
        chip = find_chip_by_name(data->type);
    else
        /* حاول JEDEC auto-detection */
        chip = jedec_probe(spi);

    /* 2. سمّي الـ MTD device */
    if (data && data->name)
        mtd->name = data->name;
    else
        mtd->name = dev_name(&spi->dev);

    /* 3. سجّل الـ MTD مع الـ partitions */
    return mtd_device_register(mtd,
                               data ? data->parts    : NULL,
                               data ? data->nr_parts : 0);
}
```

---

### ملاحظات مهمة على الـ Locking والـ Lifetime

| الجانب | التفاصيل |
|--------|----------|
| **Ownership** | الـ `flash_platform_data` بتُعرَّف static في الـ board file — الـ driver مش بيعمل لها `kfree()` |
| **Thread safety** | الـ struct بيُقرأ مرة واحدة في الـ `probe()` — مفيش locking مطلوب |
| **Lifetime** | لازم تفضل valid طول عمر الـ device — خليها `static` دايماً في الـ board code |
| **DT migration** | في الأنظمة الحديثة المعتمدة على Device Tree، الـ `flash_platform_data` deprecated ومش بتُستخدم — الـ partitions بتيجي من `fixed-partitions` DT binding |

---

### مقارنة: Platform Data vs Device Tree Partitions

| الجانب | `flash_platform_data` | Device Tree |
|--------|----------------------|-------------|
| الاستخدام | legacy board files | الأنظمة الحديثة |
| الـ partitions | `parts` array في C | `fixed-partitions` node |
| الـ chip type | `type` string | `compatible` property |
| الاسم | `name` field | `label` property |
| المرونة | compile-time only | runtime، قابل للتغيير |

**مثال DT مقابل:**
```dts
/* Device Tree equivalent */
flash@0 {
    compatible = "jedec,spi-nor";
    label = "nor-flash";          /* يقابل .name */

    partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        bootloader@0 {
            label = "bootloader";
            reg = <0x0 0x40000>;  /* 256K */
        };
        kernel@40000 {
            label = "kernel";
            reg = <0x40000 0x400000>;  /* 4M */
        };
    };
};
```
## Phase 5: دليل الـ Debugging الشامل

السياق هنا: **`flash_platform_data`** هي struct بسيطة جداً — بتعرّف اسم الـ SPI flash، نوعه، والـ MTD partitions بتاعته. الـ debugging هنا بيشمل ثلاث طبقات: الـ **SPI bus** نفسه، الـ **MTD/flash** layer، والـ **board/platform data** الـ misconfiguration.

---

### Software Level

#### 1. debugfs Entries

الـ SPI subsystem والـ MTD بيكشفوا entries مفيدة في `/sys/kernel/debug/`:

```bash
# شوف كل الـ SPI controllers الموجودة
ls /sys/kernel/debug/spi*/

# إحصائيات الـ SPI device (messages, errors, bytes)
cat /sys/kernel/debug/spi0/spi0.0/statistics

# MTD partitions المتعرّفة
cat /sys/kernel/debug/mtd/mtd0/bad_blocks   # لو NAND
cat /sys/kernel/debug/mtd/mtd0/eccstats

# SPI NOR flash specific (لو بيستخدم spi-nor driver)
ls /sys/kernel/debug/spi-nor/
cat /sys/kernel/debug/spi-nor/spi0.0/params
```

الـ `statistics` file بيكشف:

| الحقل | المعنى |
|-------|--------|
| `messages` | عدد الـ SPI messages المكتملة |
| `errors` | عدد الـ transfers الفاشلة |
| `timedout` | timeouts حصلت |
| `bytes_tx` / `bytes_rx` | البيانات المتبادلة |
| `transfers_split_maxsize` | كام مرة اتقسم transfer بسبب حد الحجم |

#### 2. sysfs Entries

```bash
# معلومات الـ SPI device
cat /sys/bus/spi/devices/spi0.0/modalias
cat /sys/bus/spi/devices/spi0.0/driver_override

# الـ MTD partitions المتعرّفة من flash_platform_data
cat /proc/mtd
# مثال على output:
# dev:    size   erasesize  name
# mtd0: 00100000 00010000 "bootloader"
# mtd1: 00400000 00010000 "kernel"
# mtd2: 00b00000 00010000 "rootfs"

# معلومات تفصيلية عن كل partition
cat /sys/class/mtd/mtd0/name
cat /sys/class/mtd/mtd0/size
cat /sys/class/mtd/mtd0/erasesize
cat /sys/class/mtd/mtd0/flags
cat /sys/class/mtd/mtd0/type    # "nor" أو "nand" أو "dataflash"

# الـ SPI controller
cat /sys/bus/spi/devices/spi0.0/max_speed_hz
cat /sys/bus/spi/devices/spi0.0/bits_per_word
cat /sys/bus/spi/devices/spi0.0/mode
```

#### 3. ftrace — Tracepoints و Events

```bash
# فعّل الـ tracing
mount -t tracefs nodev /sys/kernel/tracing

# شوف الـ events المتاحة للـ SPI و MTD
grep -r "spi\|mtd" /sys/kernel/tracing/available_events

# فعّل SPI events
echo 1 > /sys/kernel/tracing/events/spi/enable
# أو بشكل انتقائي:
echo 1 > /sys/kernel/tracing/events/spi/spi_transfer_start/enable
echo 1 > /sys/kernel/tracing/events/spi/spi_transfer_stop/enable
echo 1 > /sys/kernel/tracing/events/spi/spi_message_start/enable
echo 1 > /sys/kernel/tracing/events/spi/spi_message_done/enable

# فعّل MTD events
echo 1 > /sys/kernel/tracing/events/mtd/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل العملية المشكوك فيها هنا...

# اقرأ النتيجة
cat /sys/kernel/tracing/trace

# مثال على output:
# spi0-0    [000]   123.456: spi_message_start: spi0.0
# spi0-0    [000]   123.457: spi_transfer_start: spi0.0 len=4
# spi0-0    [000]   123.460: spi_transfer_stop:  spi0.0 len=4

# تتبع function calls في spi-nor driver
echo "spi_nor_*" > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
cat /sys/kernel/tracing/trace
```

#### 4. printk و Dynamic Debug

```bash
# فعّل dynamic debug لـ SPI NOR driver
echo "module spi_nor +p" > /sys/kernel/debug/dynamic_debug/control
echo "module m25p80 +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل dynamic debug لـ MTD
echo "module mtd +p" > /sys/kernel/debug/dynamic_debug/control
echo "module mtdpart +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل للملف المحدد
echo "file drivers/mtd/spi-nor/core.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# الـ flags:
# p = printk
# f = اطبع اسم الـ function
# l = اطبع رقم السطر
# m = اطبع اسم الـ module
# t = اطبع اسم الـ thread

# شوف الـ kernel log
dmesg -w | grep -E "spi|mtd|m25p|flash"

# تعديل loglevel لرؤية DEBUG messages
echo 8 > /proc/sys/kernel/printk
```

في الكود، الـ dev_dbg calls الموجودة في `drivers/mtd/spi-nor/core.c` و `drivers/spi/` بتتفعّل تلقائياً بعد الأمر ده.

#### 5. Kernel Config Options للـ Debugging

```
# SPI debugging
CONFIG_SPI_DEBUG=y               # فعّل verbose logging في SPI core
CONFIG_SPI_LOOPBACK_TEST=m       # test module للـ SPI transfers

# MTD debugging
CONFIG_MTD_TESTS=m               # مجموعة tests للـ MTD subsystem
CONFIG_MTD_DEBUG=y               # (قديم، اتاستبدل بـ dynamic debug)
CONFIG_MTD_DEBUG_VERBOSE=2       # مستوى التفاصيل (0-3)

# SPI NOR
CONFIG_MTD_SPI_NOR=y
CONFIG_MTD_SPI_NOR_USE_4K_SECTORS=y  # لو بتواجه erase issues

# DataFlash (Atmel)
CONFIG_MTD_DATAFLASH=y
CONFIG_MTD_DATAFLASH_WRITE_VERIFY=y  # verify بعد كل write

# General debug
CONFIG_DYNAMIC_DEBUG=y           # لازم يكون enabled عشان dynamic debug يشتغل
CONFIG_KALLSYMS=y                # عشان stack traces تبان صح
CONFIG_DEBUG_FS=y                # عشان debugfs يشتغل

# Memory debugging (لو مشكوك في platform_data corruption)
CONFIG_KASAN=y
CONFIG_KMSAN=y
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# mtd-utils — أهم أداة للـ MTD
apt install mtd-utils   # أو من المصدر

# اقرأ معلومات الـ flash
mtdinfo /dev/mtd0
mtdinfo -a              # كل الـ MTD devices

# اختبر القراءة
dd if=/dev/mtd0 of=/tmp/boot.bin bs=4096

# اختبر الـ erase + write
flash_erase /dev/mtd1 0 0          # erase كل الـ partition
nandwrite -p /dev/mtd1 /tmp/kernel.bin  # NAND
dd if=/tmp/kernel.bin of=/dev/mtd1      # NOR

# فحص الـ bad blocks (NAND)
nandtest -k /dev/mtd1   # -k = keep existing data

# فحص سرعة الـ flash
mtd_speedtest -d 0      # قيس read/write/erase speed

# تحقق من الـ partitions المتعرّفة
cat /proc/partitions
ls -la /dev/mtd*
```

#### 7. رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|-------------|--------|------|
| `spi_nor_read_id: error -5` | فشل JEDEC ID read — hardware مش موجود أو SPI مش شغال | افحص الـ wiring و الـ SPI mode و الـ CS |
| `m25p80 spi0.0: found unknown id bytes` | الـ chip موجود لكن JEDEC ID مش معروف | استخدم `flash_platform_data.type` لتحديد النوع يدوياً |
| `spi_nor_sr_ready: error -110` | timeout في انتظار الـ flash يخلص operation | الـ flash بطيء جداً أو مشكلة في power |
| `mtd_erase: error -5` | فشل الـ erase operation | flash تالف أو write protected |
| `spi transfer timed out` | الـ SPI controller ما ردّش في الوقت المحدد | افحص clock، DMA، أو system load |
| `cs_change delay` errors | مشكلة في timing بين الـ CS وإرسال البيانات | عدّل `cs_setup`/`cs_hold` في الـ spi_device |
| `mtdpart: partition exceeds device` | حجم الـ partition في `flash_platform_data.parts` أكبر من الـ flash | صحّح الـ sizes في board code |
| `no suitable DMA available` | الـ SPI controller حاول يستخدم DMA ومفيش | مش critical، بيرجع لـ PIO؛ افحص DMA config |
| `spi: probe of spi0.0 failed with error -19` | الـ driver المطلوب في `modalias`/`type` مش موجود | تأكد الـ driver compiled أو لود الـ module |
| `Failed to parse partition table` | الـ partition parser مش لاقي الـ magic bytes | استخدم static partitions في `flash_platform_data` |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

في `drivers/mtd/spi-nor/core.c`:

```c
/* نقطة 1: تحقق من صحة الـ flash_platform_data قبل استخدامها */
static int spi_nor_probe(struct spi_device *spi)
{
    struct flash_platform_data *data = dev_get_platdata(&spi->dev);

    if (data) {
        /* تحقق من أن nr_parts و parts متوافقين */
        WARN_ON(data->nr_parts > 0 && !data->parts);
        WARN_ON(data->nr_parts == 0 && data->parts);
    }
    // ...
}

/* نقطة 2: تحقق من حجم الـ partitions */
for (i = 0; i < data->nr_parts; i++) {
    WARN_ON(data->parts[i].offset + data->parts[i].size > nor->mtd.size);
}

/* نقطة 3: لو الـ type field اتحدد ومش موجود */
if (data->type && !spi_nor_match_id(nor, data->type)) {
    dev_warn(&spi->dev, "type '%s' not found in table\n", data->type);
    dump_stack();  // ساعات بتساعد تعرف من اللي set الـ platform_data
}
```

في كود الـ board (arch/.../board-xxx.c):

```c
/* نقطة 4: تحقق من الـ platform_data قبل تسجيل الـ device */
static int __init board_spi_init(void)
{
    BUG_ON(!flash_pdata.parts && flash_pdata.nr_parts);
    // ...
}
```

---

### Hardware Level

#### 1. التحقق من تطابق الـ Hardware مع الـ Kernel State

```bash
# kernel بيشوف إيه؟
cat /proc/mtd
cat /sys/class/mtd/mtd0/type
cat /sys/class/mtd/mtd0/size

# قارن الحجم بالـ datasheet
# مثلاً M25P64 = 8MB = 0x800000
cat /sys/class/mtd/mtd0/size  # المفروض يطلع 8388608

# تحقق من JEDEC ID اللي اتقرأ
dmesg | grep -E "JEDEC|jedec|found|Detecting"
# مثال:
# m25p80 spi0.0: w25q64 (8192 Kbytes)
#                ^^^^^^ ده اسم الـ chip الـ autodetect لقاه

# الـ SPI mode المستخدم
cat /sys/bus/spi/devices/spi0.0/mode
# 0 = CPOL=0,CPHA=0  |  1 = CPOL=0,CPHA=1
# 2 = CPOL=1,CPHA=0  |  3 = CPOL=1,CPHA=1
# المعظم NOR flash بيدعم mode 0 و 3

# الـ clock speed
cat /sys/bus/spi/devices/spi0.0/max_speed_hz
```

#### 2. Register Dump Techniques

```bash
# لو عندك /dev/mem (بيحتاج CONFIG_STRICT_DEVMEM=n أو kernel.perf_event_paranoid)
devmem2 0x40013000 w    # مثال لـ SPI controller base address على STM32

# أو استخدم /sys/bus/platform/drivers debug
# لـ Raspberry Pi (BCM2835 SPI):
devmem2 0x20204000 w    # SCS register
devmem2 0x20204004 w    # FIFO register
devmem2 0x20204008 w    # CLK divider

# لـ i.MX6 ECSPI:
devmem2 0x02008000 w    # ECSPI1_RXDATA
devmem2 0x02008008 w    # ECSPI1_CONREG

# أداة io (من package io-port-utils أو مكتوبة manually)
io -4 -r 0x40013000     # قرأ 4 bytes من عنوان الـ SPI controller

# لو مش عارف الـ base address، استخدم:
cat /proc/iomem | grep -i spi
# مثال output:
# 40013000-400133ff : 40013000.spi
```

**استخدام Python لـ register dump:**

```python
import mmap, os

def read_reg(base, offset):
    with open('/dev/mem', 'rb') as f:
        m = mmap.mmap(f.fileno(), 0x400, offset=base,
                      access=mmap.ACCESS_READ)
        val = int.from_bytes(m[offset:offset+4], 'little')
        m.close()
        return val

# SPI1 على STM32F4 مثلاً:
cr1 = read_reg(0x40013000, 0x00)   # Control Register 1
sr  = read_reg(0x40013000, 0x08)   # Status Register
print(f"CR1=0x{cr1:08x}, SR=0x{sr:08x}")
```

#### 3. Logic Analyzer / Oscilloscope Tips

**الـ signals اللي لازم تراقبها:**

```
SPI Bus:
  SCK  ─── الـ clock
  MOSI ─── Master Out Slave In (بيانات من الـ CPU للـ flash)
  MISO ─── Master In Slave Out (بيانات من الـ flash للـ CPU)
  CS#  ─── Chip Select (active low)

ASCII diagram للـ JEDEC ID read:
CS#  ‾‾\_________________________________/‾‾‾
SCK  ___/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_...
MOSI ___XXXX[ 0x9F = READ_ID command  ]XXX
MISO _____________________[ID byte 0][1][2]
```

**إعدادات الـ logic analyzer:**

```
- Sample rate: على الأقل 4x الـ SPI clock
  مثلاً: SPI @ 10MHz → sample @ 40MHz+
- Protocol decoder: SPI → بعدين MTD/NOR layer فوقيه
- Trigger: على falling edge الـ CS#
- Capture length: كافي يشمل complete transaction

نقاط المراقبة:
1. CS# setup time قبل أول SCK edge
2. CS# hold time بعد آخر SCK edge
3. MISO data valid time بعد falling SCK edge
4. وقت الـ BUSY period بعد write/erase commands
```

**في Sigrok / PulseView:**

```bash
# capture من command line
sigrok-cli -d fx2lafw \
    --config samplerate=24m \
    --samples 1000000 \
    -P spi:clk=D0:mosi=D1:miso=D2:cs=D3 \
    -A spi \
    -o flash_capture.sr

# شوف النتيجة
pulseview flash_capture.sr
```

#### 4. Hardware Issues الشائعة وأنماط الـ Kernel Log

| المشكلة | الـ Kernel Log Pattern | السبب المرجّح |
|---------|----------------------|---------------|
| Flash مش متصل | `spi_nor: unrecognized JEDEC id bytes: ff ff ff` | MISO مش متوصل أو VCC مفيش |
| SPI mode غلط | `spi_nor: unrecognized JEDEC id bytes: 00 00 00` | CPOL/CPHA مش صح |
| Clock سريع جداً | `spi transfer timed out` أو data corruption | تجاوز max frequency الـ flash |
| CS مش شغال | `spi_nor: unrecognized JEDEC id bytes: xx xx xx` (random) | GPIO CS مش configured صح |
| Power مش كافي | Device يتعرف تارة ومش تارة | decoupling caps مش كافية |
| Write Protected | `mtd_erase: -EIO` على write operations | WP# pin موصّل لـ GND |
| DataFlash size مش power-of-2 | Partition alignment errors | Atmel DataFlash خاصية معروفة، استخدم sector-aligned partitions |

#### 5. Device Tree Debugging

الـ `flash_platform_data` في الـ code الحديث بيتحوّل لـ Device Tree. التحقق:

```bash
# شوف الـ DT node للـ SPI flash
find /proc/device-tree -name "flash*" -o -name "w25q*" -o -name "m25p*" 2>/dev/null
# أو
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 30 "spi-flash\|jedec,spi-nor"

# مثال DT node صح لـ SPI flash:
# &spi0 {
#     flash@0 {
#         compatible = "jedec,spi-nor";  /* أو "atmel,at25df321" */
#         reg = <0>;                     /* CS0 */
#         spi-max-frequency = <25000000>;
#         #address-cells = <1>;
#         #size-cells = <1>;
#
#         partition@0 {
#             label = "bootloader";
#             reg = <0x0 0x100000>;
#         };
#         partition@100000 {
#             label = "kernel";
#             reg = <0x100000 0x400000>;
#         };
#     };
# };

# تحقق من القيم الفعلية اللي kernel شايفها
cat /proc/device-tree/soc/spi@40013000/flash@0/spi-max-frequency | xxd
cat /proc/device-tree/soc/spi@40013000/flash@0/compatible

# تحقق من الـ partitions المتعرّفة من DT
for d in /proc/device-tree/soc/spi*/flash*/partition*; do
    echo "=== $d ==="
    cat "$d/label" 2>/dev/null && echo
    cat "$d/reg" 2>/dev/null | xxd
done

# لو بتستخدم flash_platform_data (non-DT) تحقق من board code
dmesg | grep "platform data\|mtd_device_register\|spi_new_device"
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للـ Copy-Paste

**سيناريو 1: الـ flash مش بيتعرف**

```bash
#!/bin/bash
echo "=== SPI Flash Diagnostic ==="

# 1. تحقق الـ SPI bus موجود
ls /sys/bus/spi/devices/
echo "---"

# 2. شوف الـ kernel log للـ probe
dmesg | grep -E "(spi|m25p|w25|flash|mtd|jedec)" | tail -30
echo "---"

# 3. تحقق من الـ MTD devices
cat /proc/mtd
echo "---"

# 4. تحقق من الـ loaded modules
lsmod | grep -E "(spi|mtd|m25p|spi_nor)"
echo "---"

# 5. فعّل dynamic debug وحاول rebind
echo "module spi_nor +p" > /sys/kernel/debug/dynamic_debug/control
echo "module mtdpart +p" > /sys/kernel/debug/dynamic_debug/control

# 6. rebind الـ driver (لو الـ device موجود لكن مش شغال)
DEVICE=$(ls /sys/bus/spi/devices/ | head -1)
if [ -n "$DEVICE" ]; then
    DRIVER=$(readlink /sys/bus/spi/devices/$DEVICE/driver | xargs basename 2>/dev/null)
    [ -n "$DRIVER" ] && echo $DEVICE > /sys/bus/spi/drivers/$DRIVER/unbind
    echo $DEVICE > /sys/bus/spi/drivers/spi-nor/bind 2>/dev/null || \
    echo "Try: modprobe spi-nor && echo $DEVICE > /sys/bus/spi/drivers/spi-nor/bind"
fi
```

**سيناريو 2: تحقق من صحة الـ Partitions**

```bash
#!/bin/bash
echo "=== MTD Partition Verification ==="

# عرض كل الـ partitions
mtdinfo -a

echo ""
echo "=== /proc/mtd ==="
cat /proc/mtd

echo ""
echo "=== Partition Sizes Check ==="
TOTAL=0
for mtd in /sys/class/mtd/mtd[0-9]*; do
    # تجاهل الـ sub-partitions (روابط فقط على الـ master)
    [ -L "$mtd/master" ] && continue
    NAME=$(cat $mtd/name)
    SIZE=$(cat $mtd/size)
    OFFSET=$(cat $mtd/offset 2>/dev/null || echo "?")
    printf "%-20s size=0x%08x offset=0x%s\n" "$NAME" "$SIZE" "$OFFSET"
    TOTAL=$((TOTAL + SIZE))
done
echo "Total: 0x$(printf '%x' $TOTAL) bytes"

echo ""
echo "=== Read Test (first 512 bytes of each partition) ==="
for mtd in /dev/mtd[0-9]*; do
    [ "${mtd##*ro}" != "$mtd" ] && continue  # skip mtdroX
    NAME=$(cat /sys/class/mtd/${mtd##*/}/name 2>/dev/null)
    echo -n "Reading $mtd ($NAME)... "
    dd if=$mtd of=/dev/null bs=512 count=1 2>&1 | grep -E "copied|error"
done
```

**سيناريو 3: SPI Statistics Monitoring**

```bash
#!/bin/bash
# راقب الـ SPI statistics في real-time

DEVICE=${1:-"spi0.0"}
STATS_PATH="/sys/kernel/debug/spi0/$DEVICE/statistics"

if [ ! -f "$STATS_PATH" ]; then
    echo "No statistics at $STATS_PATH"
    echo "Available: $(ls /sys/kernel/debug/spi0/ 2>/dev/null)"
    exit 1
fi

echo "Monitoring $DEVICE statistics (Ctrl+C to stop)..."
while true; do
    clear
    echo "=== $DEVICE SPI Statistics ==="
    cat "$STATS_PATH"
    sleep 1
done
```

**سيناريو 4: ftrace لتتبع flash_platform_data usage**

```bash
#!/bin/bash
# تتبع كيف بيتم استخدام الـ platform data

TRACEDIR=/sys/kernel/tracing

# reset
echo 0 > $TRACEDIR/tracing_on
echo > $TRACEDIR/trace

# اختر الـ tracer
echo function_graph > $TRACEDIR/current_tracer

# فلتر على دوال الـ SPI NOR و MTD
echo "spi_nor_probe
spi_nor_read_id
spi_nor_setup
mtd_device_register
parse_mtd_partitions
mtdpart_add_parts" > $TRACEDIR/set_graph_function

# ابدأ
echo 1 > $TRACEDIR/tracing_on

# انتظر probe يحصل (مثلاً بعد modprobe أو device hotplug)
sleep 5

echo 0 > $TRACEDIR/tracing_on
cat $TRACEDIR/trace | head -100
```

**مثال على output و تفسيره:**

```
# بعد تشغيل: dmesg | grep -i "spi\|mtd\|flash" | tail -20

[    1.234567] spi-nor spi0.0: w25q128 (16384 Kbytes)
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
               الـ driver عرف الـ chip تلقائياً من JEDEC ID
               w25q128 = Winbond 128Mbit = 16MB

[    1.235000] 3 cmdlinepart partitions found on MTD device spi0.0
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
               الـ kernel لقى 3 partitions من flash_platform_data.parts

[    1.235100] Creating 3 MTD partitions on "spi0.0":
[    1.235200] 0x000000000000-0x000000100000 : "bootloader"
[    1.235300] 0x000000100000-0x000000500000 : "kernel"
[    1.235400] 0x000000500000-0x000001000000 : "rootfs"
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
               الـ offsets والـ sizes المعرّفة في flash_platform_data.parts
               تحقق إن مجموعها = حجم الـ flash الكلي

# لو شوفت ده:
[    1.300000] spi-nor spi0.0: unrecognized JEDEC id bytes: ff, ff, ff
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
               JEDEC ID كله FF = MISO مش واصل أو الـ flash مش powered
               الحل: استخدم flash_platform_data.type = "m25p64" مثلاً

# لو شوفت ده:
[    1.400000] spi0.0: Failed to parse 'ofpart' partition table
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
               لو الـ platform_data.parts = NULL ومفيش DT partitions
               الحل: تأكد من تحديد الـ parts في flash_platform_data
```

**سيناريو 5: اختبار write/erase صح**

```bash
#!/bin/bash
# اختبر write و erase على partition معين (احذر: بيمسح البيانات!)
MTD=${1:-/dev/mtd1}

echo "Testing $MTD ..."

# احفظ نسخة
dd if=$MTD of=/tmp/mtd_backup.bin 2>&1

# اعمل erase
flash_erase $MTD 0 0 && echo "Erase: OK" || echo "Erase: FAILED"

# اكتب pattern
dd if=/dev/urandom of=/tmp/test_pattern.bin bs=4096 count=16
dd if=/tmp/test_pattern.bin of=$MTD 2>&1 && echo "Write: OK" || echo "Write: FAILED"

# تحقق من الـ write
dd if=$MTD of=/tmp/readback.bin bs=4096 count=16
cmp /tmp/test_pattern.bin /tmp/readback.bin \
    && echo "Verify: PASS" \
    || echo "Verify: FAIL — possible hardware issue"

# استرجع النسخة
flash_erase $MTD 0 0
dd if=/tmp/mtd_backup.bin of=$MTD
echo "Restored backup."
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: SPI NOR Flash مش بيُعرَّف على Industrial Gateway بـ RK3562

#### العنوان
**Flash مش بييجي في `/proc/mtd` على gateway صناعي بـ RK3562**

#### السياق
شركة بتعمل industrial gateway بـ RK3562 للتحكم في خطوط الإنتاج. الـ flash المستخدم هو W25Q128FV — 128 Mbit SPI NOR. الـ kernel قديم نسبياً وبيستخدم legacy board file مش Device Tree خالص. الـ engineer لقى إن الـ partition map مبتظهرش وجهاز الـ gateway مش بيـ boot من الـ flash صح.

#### المشكلة
الـ `flash_platform_data` اتعرّفت في الـ board file بدون ما تتحدد الـ `type`، والـ chip مش بترد على JEDEC ID query بشكل صح بسبب مشكلة في الـ SPI timing على الـ RK3562. الـ MTD layer مش عارف يحدد نوع الـ chip فبالتالي مش بيسجل الـ partitions.

```c
/* board file — مفيش type محدد */
static struct flash_platform_data rk3562_flash_data = {
    .name    = "rk3562-nor",
    .parts   = rk3562_flash_parts,
    .nr_parts = ARRAY_SIZE(rk3562_flash_parts),
    /* .type not set — relies on JEDEC auto-detect */
};
```

#### التحليل
الـ `flash_platform_data` struct فيها field اسمه `type`:

```c
struct flash_platform_data {
    char        *name;
    struct mtd_partition *parts;
    unsigned int nr_parts;

    char        *type;   /* <-- ده اللي بيحدد نوع الـ chip يدوياً */
};
```

لما الـ `type` بيكون `NULL`، الـ driver (m25p80 أو spi-nor) بيحاول يعمل JEDEC ID read. لو الـ SPI controller timing مش صح أو الـ chip مش بيرد، الـ probing بيفشل وبالتالي الـ device مش بيتسجل. الـ `name` field بتأثر على اسم الـ MTD device في `/proc/mtd` وعلى `mtdparts=` في command line — لو الـ device مش اتسجل أصلاً، مش هتشوف حاجة.

#### الحل

**خطوة 1:** تحديد اسم الـ chip يدوياً في الـ `flash_platform_data`:

```c
static struct flash_platform_data rk3562_flash_data = {
    .name     = "rk3562-nor",
    .parts    = rk3562_flash_parts,
    .nr_parts = ARRAY_SIZE(rk3562_flash_parts),
    .type     = "w25q128",   /* force chip type, skip JEDEC */
};
```

**خطوة 2:** التأكد إن الـ SPI clock مناسب لـ W25Q128 (max 104 MHz read, 80 MHz write):

```bash
# check SPI clock rate in DT or board file
grep -r "spi-max-frequency" arch/arm64/boot/dts/rockchip/rk3562-gateway.dts
```

**خطوة 3:** بعد الـ fix، verify:

```bash
cat /proc/mtd
# المفروض تشوف:
# dev:    size   erasesize  name
# mtd0: 01000000 00001000 "rk3562-nor"
```

#### الدرس المستفاد
الـ `type` field في `flash_platform_data` مش decoration — هي escape hatch حقيقي لما الـ auto-detection بتفشل. على boards جديدة أو chips بـ timing quirks، حدد الـ type صراحةً بدل ما تعتمد على JEDEC.

---

### السيناريو الثاني: Partition Overlap بيخرب Rootfs على Android TV Box بـ Allwinner H616

#### العنوان
**Android TV box بـ H616 بيفشل في الـ boot بعد firmware update بسبب partition overlap**

#### السياق
منتج Android TV box بـ Allwinner H616 + W25Q64 SPI NOR (8 MB). الـ firmware update script بيكتب الـ kernel image في partition خاطئ بيـ overlap مع الـ rootfs. المنتج بيطلع من المصنع شغال، لكن بعد أول OTA update الجهاز بيـ brick.

#### المشكلة
الـ `mtd_partition` array اتعرّفت بـ offsets غلط — الـ kernel partition والـ rootfs partition بيتداخلوا. الـ `flash_platform_data` اللي بتبعت الـ partitions للـ MTD layer كانت غلط من الأساس.

```c
/* WRONG partition table — overlap between kernel and rootfs */
static struct mtd_partition h616_flash_parts[] = {
    {
        .name   = "uboot",
        .offset = 0x000000,
        .size   = 0x080000,   /* 512K */
    },
    {
        .name   = "kernel",
        .offset = 0x080000,
        .size   = 0x300000,   /* 3MB — ends at 0x380000 */
    },
    {
        .name   = "rootfs",
        .offset = 0x200000,   /* BUG: starts inside kernel partition! */
        .size   = MTDPART_SIZ_FULL,
    },
};

static struct flash_platform_data h616_flash_data = {
    .name     = "h616-nor",
    .parts    = h616_flash_parts,
    .nr_parts = ARRAY_SIZE(h616_flash_parts),
    .type     = "w25q64",
};
```

#### التحليل
الـ `mtd_partition` struct بتحدد:

```c
struct mtd_partition {
    const char *name;
    uint64_t size;
    uint64_t offset;    /* absolute offset from start of flash */
    uint32_t mask_flags;
    uint32_t add_flags;
    struct device_node *of_node;
};
```

الـ `flash_platform_data.parts` بتشاور على الـ array دي. لما الـ MTD layer بياخد الـ `parts` و `nr_parts`، بيـ register الـ partitions زي ما هي بدون overlap detection كاملة في بعض الـ kernel versions القديمة. النتيجة: الكتابة على `/dev/mtd2` (rootfs) بتكتب فوق نص الـ kernel في الـ flash.

#### الحل

**الـ partition table الصح:**

```c
static struct mtd_partition h616_flash_parts[] = {
    {
        .name   = "uboot",
        .offset = 0x000000,
        .size   = 0x080000,   /* 512K */
    },
    {
        .name   = "kernel",
        .offset = 0x080000,
        .size   = 0x300000,   /* 3MB — ends at 0x380000 */
    },
    {
        .name   = "rootfs",
        .offset = 0x380000,   /* FIXED: starts after kernel */
        .size   = MTDPART_SIZ_FULL,
    },
};
```

**Debug قبل الـ OTA:**

```bash
# verify partition map على الجهاز
cat /proc/mtd

# check no overlap manually
# mtd0: offset=0x0      size=0x80000
# mtd1: offset=0x80000  size=0x300000
# mtd2: offset=0x380000 size=...
```

**حماية إضافية — اعمل partition read-only بـ mask_flags:**

```c
{
    .name       = "uboot",
    .offset     = 0x000000,
    .size       = 0x080000,
    .mask_flags = MTD_WRITEABLE,  /* read-only — protect bootloader */
},
```

#### الدرس المستفاد
الـ `parts` array في `flash_platform_data` هي source of truth للـ partition map. أي غلطة في الـ offsets بتكون fatal. استخدم `MTDPART_OFS_APPEND` بدل manual offsets لتجنب الـ human error، واعمل always validation بـ `cat /proc/mtd` قبل أي OTA infrastructure.

---

### السيناريو الثالث: DataFlash Sector Alignment Panic على IoT Sensor بـ STM32MP1

#### العنوان
**kernel panic عند الكتابة على Atmel DataFlash في IoT sensor بـ STM32MP151**

#### السياق
IoT environmental sensor بـ STM32MP151 بيستخدم Atmel AT45DB161E DataFlash (SPI) لتخزين الـ readings لما يكون offline. الـ engineer عامل partitions بـ size مش aligned على الـ sector size. عند أول write operation، الـ kernel بيـ panic.

#### المشكلة
الـ DataFlash (AT45DB161E) عنده page size 512 bytes وسector size 65536 bytes — مش power of two زي NOR flash عادي. الـ comment في `flash.h` بيوضح ده صراحةً:

```c
/*
 * Note that for DataFlash, sizes for pages, blocks, and sectors are
 * rarely powers of two; and partitions should be sector-aligned.
 */
```

لكن الـ engineer عمل partitions بـ sizes arbitrary:

```c
/* WRONG — not sector-aligned for DataFlash */
static struct mtd_partition stm32_dataflash_parts[] = {
    {
        .name   = "config",
        .offset = 0,
        .size   = 0x10000,   /* 64K — مش aligned على AT45DB161E sectors */
    },
    {
        .name   = "data",
        .offset = 0x10000,
        .size   = MTDPART_SIZ_FULL,
    },
};

static struct flash_platform_data stm32_flash_data = {
    .name     = "stm32-dataflash",
    .parts    = stm32_dataflash_parts,
    .nr_parts = ARRAY_SIZE(stm32_dataflash_parts),
    .type     = "at45db161d",
};
```

#### التحليل
الـ `flash_platform_data.type` = `"at45db161d"` بيخلي الـ driver يعرف إنه DataFlash. لكن الـ MTD erase operation بتحتاج إن الـ partition boundaries تكون على حدود الـ erase sectors. لما الـ `config` partition بتنتهي في نص sector، الـ erase بيفشل أو بيـ corrupt الـ data الـ adjacent.

الـ AT45DB161E sector layout:
- Page: 512 bytes (أو 528 non-power-of-two mode)
- Block: 4096 bytes (8 pages)
- Sector: يتراوح — sector 0a هو 8 pages، الباقي 256 pages

```
Flash Layout (AT45DB161E):
+------------------+  0x000000
|   Sector 0a      |  8 pages = 4096 bytes
+------------------+  0x001000
|   Sector 0b      |  248 pages = ~127K
+------------------+  0x020000  ← sector boundary
|   Sector 1       |  256 pages
+------------------+
```

#### الحل

**خطوة 1:** تحديد الـ sector boundaries الصح للـ AT45DB161E:

```c
/* CORRECT — sector-aligned partitions for AT45DB161E */
static struct mtd_partition stm32_dataflash_parts[] = {
    {
        .name   = "config",
        .offset = 0,
        .size   = 0x020000,   /* ends at sector boundary */
    },
    {
        .name   = "data",
        .offset = 0x020000,   /* starts at sector boundary */
        .size   = MTDPART_SIZ_FULL,
    },
};
```

**خطوة 2:** التحقق من الـ erasesize بعد الـ boot:

```bash
cat /proc/mtd
# erasesize column يوضح الـ sector size الفعلي

mtdinfo /dev/mtd0
# يوضح: erasesize, writesize, etc.
```

**خطوة 3:** لو محتاج partition صغير، استخدم `MTDPART_OFS_NXTBLK`:

```c
{
    .name   = "config",
    .offset = MTDPART_OFS_APPEND,
    .size   = 0x020000,
    /* kernel will round up to next erase block boundary */
},
```

#### الدرس المستفاد
الـ comment في `flash.h` مش تزيين — هو warning حقيقي. DataFlash مختلف جوهرياً عن NOR flash العادي. دايماً راجع datasheet الـ chip لتحديد الـ sector size الحقيقي، واستخدم `mtdinfo` للـ verification بعد الـ boot.

---

### السيناريو الرابع: SPI Flash مش بيتعرف على Custom Board بـ i.MX8MM بسبب اسم غلط في `type`

#### العنوان
**SPI NOR مش بيظهر على i.MX8MM custom board بعد migrate من board file لـ Device Tree**

#### السياق
شركة عملت custom carrier board بـ i.MX8MM للـ automotive HMI. كانوا شغالين بـ legacy board file مع `flash_platform_data`، وقرروا يـ migrate لـ Device Tree. بعد الـ migration، الـ SPI flash (MX25L12835F) اختفى من الـ system تماماً.

#### المشكلة
في الـ Device Tree الجديد، الـ `compatible` string للـ flash اتكتب غلط. لكن في الـ legacy board file اللي اتشال، كانت الـ `flash_platform_data.type` بتقول `"mx25l12835f"` وكان ده بيشتغل. بعد الـ DT migration، مفيش type hint ومفيش compatible string صح:

```c
/* Old board file — this worked */
static struct flash_platform_data imx8_flash_data = {
    .name     = "imx8-nor",
    .parts    = imx8_flash_parts,
    .nr_parts = ARRAY_SIZE(imx8_flash_parts),
    .type     = "mx25l12835f",   /* exact string from spi-nor ID table */
};
```

```dts
/* NEW DT — wrong compatible string */
&ecspi1 {
    flash@0 {
        compatible = "macronix,mx25l12835";   /* WRONG — missing 'f' suffix */
        reg = <0>;
        spi-max-frequency = <50000000>;
    };
};
```

#### التحليل
الـ `flash_platform_data.type` field بيتطابق مع entries في الـ spi-nor driver's ID table:

```c
/* from drivers/mtd/spi-nor/macronix.c */
{ "mx25l12835f", INFO(0xc22018, 0, 64 * 1024, 256) },
```

الـ string `"mx25l12835f"` بيكون exact match. في الـ legacy path، الـ `flash_platform_data` بتتعدى للـ driver عبر `spi_board_info.platform_data`، والـ driver بيستخدم `pdata->type` كـ fallback لما JEDEC detection مش كافي.

في الـ DT path، الـ `compatible` string بيتحول لـ device ID مختلف. الـ `"macronix,mx25l12835"` (بدون الـ `f`) مش موجود في الـ ID table، فالـ probe بيفشل.

```
spi-nor spi0.0: unrecognized JEDEC id bytes: c2, 20, 18
```

ملاحظة: بعض versions الـ JEDEC detection بتشتغل لو الـ flash يرد على الـ RDID command — المشكلة بتظهر لما يكون في timing issue أو الـ compatible string مش موجود والـ JEDEC fallback مش مفعّل.

#### الحل

**الـ DT الصح:**

```dts
&ecspi1 {
    status = "okay";
    cs-gpios = <&gpio5 9 GPIO_ACTIVE_LOW>;

    flash@0 {
        compatible = "macronix,mx25l12835f", "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <50000000>;

        partitions {
            compatible = "fixed-partitions";
            #address-cells = <1>;
            #size-cells = <1>;

            partition@0 {
                label = "uboot";
                reg = <0x0 0x100000>;
                read-only;
            };

            partition@100000 {
                label = "kernel";
                reg = <0x100000 0x500000>;
            };

            partition@600000 {
                label = "rootfs";
                reg = <0x600000 0x0>;  /* MTDPART_SIZ_FULL equivalent */
            };
        };
    };
};
```

**Debug:**

```bash
# check what the kernel sees for SPI NOR IDs
dmesg | grep -i "spi-nor\|jedec\|mx25"

# list all SPI devices
ls /sys/bus/spi/devices/
```

#### الدرس المستفاد
لما تـ migrate من board file لـ Device Tree، الـ `flash_platform_data.type` string بيكون دليل مهم على الـ exact chip name المطلوب في الـ `compatible`. الـ string لازم يطابق بالظبط الـ entry في الـ spi-nor ID table — حتى حرف واحد فرق بيوقع المشكلة.

---

### السيناريو الخامس: Multi-Partition NOR Flash على AM62x Automotive ECU بيسبب Data Corruption

#### العنوان
**ECU data corruption على AM62x بسبب write على partition مفيش عليه `MTD_WRITEABLE` protection**

#### السياق
Automotive ECU بـ TI AM62x بيستخدم W25Q256JV (32 MB SPI NOR) لتخزين الـ calibration data، الـ bootloader، والـ application firmware. الـ bootloader partition المفروض يكون read-only لكن مفيش حماية مفعّلة. في أحد الـ stress tests، الـ application كتبت عن طريق الخطأ في الـ bootloader partition وحولت الـ ECU لـ brick.

#### المشكلة
الـ `flash_platform_data` مفيهاش `mask_flags` مضبوطة على الـ bootloader partition:

```c
/* VULNERABLE — no write protection on critical partitions */
static struct mtd_partition am62x_flash_parts[] = {
    {
        .name   = "bootloader",
        .offset = 0x000000,
        .size   = 0x100000,   /* 1MB */
        /* mask_flags = 0 — writable by anyone! */
    },
    {
        .name   = "calibration",
        .offset = 0x100000,
        .size   = 0x100000,   /* 1MB */
    },
    {
        .name   = "firmware",
        .offset = 0x200000,
        .size   = MTDPART_SIZ_FULL,
    },
};

static struct flash_platform_data am62x_flash_data = {
    .name     = "am62x-nor",
    .parts    = am62x_flash_parts,
    .nr_parts = ARRAY_SIZE(am62x_flash_parts),
    .type     = "w25q256",
};
```

#### التحليل
الـ `mtd_partition.mask_flags` بيمسح flags من الـ parent MTD device. لو `MTD_WRITEABLE` اتحط في الـ `mask_flags`، الـ partition بيبقى read-only تماماً:

```c
struct mtd_partition {
    const char  *name;
    uint64_t     size;
    uint64_t     offset;
    uint32_t     mask_flags;  /* MTD_WRITEABLE هنا = read-only partition */
    uint32_t     add_flags;
    struct device_node *of_node;
};
```

الـ `flash_platform_data.parts` بتشاور على الـ array دي. لما `mask_flags = 0`، الـ partition بيورث كل flags الـ parent بما فيها الـ write capability. أي process عندها access لـ `/dev/mtd0` تقدر تكتب فيه.

```
Attack surface:
Application → open("/dev/mtd0", O_RDWR) → write() → overwrites bootloader
```

#### الحل

**الـ partition table المحمي:**

```c
static struct mtd_partition am62x_flash_parts[] = {
    {
        .name       = "bootloader",
        .offset     = 0x000000,
        .size       = 0x100000,
        .mask_flags = MTD_WRITEABLE,  /* PROTECTED — read-only */
    },
    {
        .name       = "calibration",
        .offset     = 0x100000,
        .size       = 0x100000,
        /* writable — calibration data needs updates */
    },
    {
        .name       = "firmware",
        .offset     = 0x200000,
        .size       = MTDPART_SIZ_FULL,
        /* writable — firmware updates via OTA */
    },
};
```

**Verification:**

```bash
# check partition flags
mtdinfo -a

# try writing to protected partition — should fail
flash_erase /dev/mtd0 0 1
# Expected: flash_erase: error!: MT_ERASE: Permission denied (or Input/output error)

# verify read works
nanddump /dev/mtd0 -f /tmp/bootloader_backup.bin
```

**إضافة hardware write protect (WP# pin) في DT:**

```dts
flash@0 {
    compatible = "winbond,w25q256", "jedec,spi-nor";
    reg = <0>;
    spi-max-frequency = <80000000>;
    wp-gpios = <&gpio0 15 GPIO_ACTIVE_LOW>;  /* hardware WP */

    partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        bootloader@0 {
            label = "bootloader";
            reg = <0x0 0x100000>;
            read-only;   /* DT equivalent of mask_flags = MTD_WRITEABLE */
        };
    };
};
```

#### الدرس المستفاد
في automotive وindustrial، الـ `mask_flags = MTD_WRITEABLE` على الـ bootloader partition مش optional — هي متطلب safety. الـ `flash_platform_data` بتديك control كامل على الـ partition permissions من الـ board initialization. لازم تحمي الـ critical partitions بـ software flags، وتضيف hardware WP pin كـ defense-in-depth.
## Phase 7: مصادر ومراجع

الـ `flash_platform_data` struct موجودة في `include/linux/spi/flash.h` وهي جزء من منظومة أكبر بتشمل الـ SPI subsystem والـ MTD (Memory Technology Device) subsystem. الـ references دي متنظمة من الأساسي للمتخصص.

---

### مقالات LWN.net

| المقال | الأهمية |
|--------|---------|
| [mtd: spi-nor: generic flash driver](https://lwn.net/Articles/904474/) | يشرح الـ SFDP-based generic driver الجديد — مهم لفهم اتجاه التطوير بعد `flash_platform_data` |
| [mtd: spi-nor: Cadence QSPI Flash Controller driver](https://lwn.net/Articles/618065/) | مثال حقيقي على controller driver بيستخدم الـ spi-nor framework |
| [spi-nor: Add support for Intel SPI serial flash controller](https://lwn.net/Articles/707459/) | إضافة controller لـ Intel CPUs — بيوضح تطور الـ platform data نحو ACPI/DT |
| [mtd: spi-nor: Quad Enable and (un)lock methods](https://lwn.net/Articles/803437/) | تنظيف الـ flash register operations — مرتبط بكيفية تعريف الـ flash type |
| [mtd: spi-nor: OTP support](https://lwn.net/Articles/809081/) | إضافة One-Time Programmable support — يوضح امتداد الـ flash capabilities |
| [spi: spi-mem: Add a driver for NXP FlexSPI controller](https://lwn.net/Articles/763880/) | الـ spi-mem framework الحديث اللي بيحل محل الـ platform data القديمة |
| [add support to non-uniform SFDP SPI NOR flash memories](https://lwn.net/Articles/764577/) | SFDP Sector Map parsing — بيوضح ليه الـ static `type` field مش كافي |
| [drivers: spi: Add qspi flash controller](https://lwn.net/Articles/557211/) | TI QSPI driver — مثال على board-level integration |

---

### التوثيق الرسمي في الـ Kernel

#### SPI Subsystem
```
Documentation/driver-api/spi.rst
Documentation/spi/spidev.rst
Documentation/spi/spi-summary.rst
```

**الـ online version:**
- [Serial Peripheral Interface (SPI) — The Linux Kernel documentation](https://static.lwn.net/kerneldoc/driver-api/spi.html)
- [SPI userspace API](https://static.lwn.net/kerneldoc/spi/spidev.html)

#### MTD/SPI-NOR Subsystem
```
Documentation/driver-api/mtd/spi-nor.rst
```

**الـ online version:**
- [SPI NOR framework — The Linux Kernel documentation](https://docs.kernel.org/driver-api/mtd/spi-nor.html)

#### Device Tree Bindings
```
Documentation/devicetree/bindings/mtd/jedec,spi-nor.txt
```

**الـ online version:**
- [jedec,spi-nor.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/mtd/jedec,spi-nor.txt)

---

### الملفات الأساسية في الـ Kernel Source

الـ `flash.h` مش بتشتغل لوحدها — دي الـ files اللي لازم تقراها جنبها:

| الملف | الدور |
|-------|-------|
| `include/linux/spi/flash.h` | تعريف `flash_platform_data` — موضوع البحث |
| `include/linux/mtd/partitions.h` | تعريف `mtd_partition` اللي بتشير إليه `flash_platform_data` |
| `drivers/mtd/devices/m25p80.c` (تاريخي) | أول driver استخدم `flash_platform_data` فعلياً |
| `drivers/mtd/spi-nor/core.c` | الـ core الحديث للـ SPI NOR subsystem |
| `drivers/mtd/spi-nor/sfdp.c` | الـ SFDP parser — البديل الحديث للـ static `type` |
| `drivers/spi/spi.c` | الـ SPI core — بيدير الـ bus والـ devices |

---

### Kernel Commits المهمة

#### إضافة `flash_platform_data` الأصلية
**الـ commit الأصلي** لملف `spi/flash.h` رجعت لأيام الـ m25p80 driver في kernel 2.6:

```bash
# للبحث في git log عن تاريخ الملف
git log --follow --oneline include/linux/spi/flash.h
```

#### تحويل الـ SPI NOR لـ subsystem مستقل
الـ commit المحوري اللي أنشأ `drivers/mtd/spi-nor/`:
```
commit 8275c642a: "mtd: spi-nor: move m25p80.c to spi-nor framework"
```

#### Cleanup الـ flash_info database
- [PATCH 00/41] mtd: spi-nor: clean the flash_info database up — August 2023
  - **الأرشيف:** https://lists.infradead.org/pipermail/linux-mtd/2023-August/100339.html

---

### نقاشات Mailing List

#### linux-mtd (القائمة الرئيسية)
- **القائمة:** `linux-mtd@lists.infradead.org`
- **الأرشيف:** https://lists.infradead.org/pipermail/linux-mtd/
- **الـ lore:** https://lore.kernel.org/linux-mtd/

موضوع مهم: [PATCH v4 0/4] mtd: spi-nor: OTP support
- https://lore.kernel.org/linux-mtd/202103071618.6qOC6qMG-lkp@intel.com/T/

#### kernelnewbies mailing list
نقاش عملي عن الفرق بين `/drivers/spi` و `/drivers/mtd`:
- http://lists.kernelnewbies.org/pipermail/kernelnewbies/2019-December/020519.html

---

### مصادر eLinux.org

| المصدر | المحتوى |
|--------|---------|
| [An Introduction to SPI-NOR Subsystem (PDF)](https://elinux.org/images/4/4e/An_Introduction_to_SPI-NOR_Subsystem_-_v3_0.pdf) | شرح شامل للـ SPI-NOR subsystem من Texas Instruments — **ابدأ بيه** |
| [Minnowboard SPI Boot Flash](https://elinux.org/Minnowboard_talk:SPI_Boot_flash) | مثال board-level على استخدام SPI flash للـ boot |
| [RPi SPI](https://elinux.org/RPi_SPI) | مثال عملي على SPI على Raspberry Pi |
| [MSIOF SPI CS GPIO Testing](https://elinux.org/Tests:MSIOF-SPI-CS-GPIO) | اختبار SPI EEPROM مع AT25 devices |

---

### مصادر خارجية مفيدة

#### Bootlin
- [Managing flash storage with Linux](https://bootlin.com/blog/managing-flash-storage-with-linux/) — شرح ممتاز للـ MTD stack كامل بما فيه SPI NOR

#### Linux Foundation Presentation
- [An Introduction to SPI-NOR Subsystem — Vignesh R, Texas Instruments](http://events17.linuxfoundation.org/sites/events/files/slides/An%20Introduction%20to%20SPI-NOR%20Subsystem%20-%20v3_0.pdf) — presentation من ELC كامل عن الـ subsystem

#### Analog Devices Wiki
- [Linux MTD Driver](https://wiki.analog.com/resources/tools-software/linuxdsp/docs/linux-kernel-and-drivers/mtd/mtd) — توثيق عملي للـ MTD driver

---

### كتب موصى بيها

#### Linux Device Drivers (LDD3)
- **الفصل المهم:** Chapter 17 — Network Drivers (لفهم الـ bus model)، وChapter 14 — The Linux Device Model
- الكتاب متاح مجاناً: https://lwn.net/Kernel/LDD3/
- **ملاحظة:** LDD3 قديم (kernel 2.6.10) بس مفاهيم الـ platform data والـ SPI bus موجودة فيه

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل المهم:** Chapter 17 — Devices and Modules
- بيشرح الـ device model، الـ bus، والـ driver registration اللي `flash_platform_data` جزء منها

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المهم:** Chapter 10 — MTD Subsystem, Chapter 11 — BDI/JTAG Debugging
- بيغطي الـ flash partitioning والـ MTD layer بالتفصيل — **الأنسب مباشرة للموضوع ده**

#### Professional Linux Embedded Systems — Gene Sally
- بيغطي الـ SPI devices والـ board-level configuration بشكل عملي

---

### kernelnewbies.org — Kernel Releases ذات الصلة

| الإصدار | الإضافة المهمة |
|---------|---------------|
| [Linux 2.6.26](https://kernelnewbies.org/Linux_2_6_26) | FAST_READ لـ M25Pxx flash وAT25DF641 SPI Flash support |
| [Linux 2.6.32](https://kernelnewbies.org/Linux_2_6_32) | SST25L SPI Flash driver |
| [Linux 6.2](https://kernelnewbies.org/Linux_6.2) | SPI NOR controller updates |
| [Linux 6.6](https://kernelnewbies.org/Linux_6.6) | تحسينات على الـ spi-nor subsystem |

---

### مصطلحات البحث

للبحث عن معلومات إضافية استخدم الـ search terms دي:

```
# بحث عام
"flash_platform_data" linux kernel
"spi-nor" linux MTD subsystem
"m25p80" linux driver history
"jedec,spi-nor" device tree binding

# بحث متخصص
linux kernel "struct flash_platform_data" deprecated
SPI NOR SFDP linux kernel auto-detection
MTD partitions SPI flash linux board init
"spi_board_info" flash linux kernel

# بحث في الـ source
git log --all --oneline -- include/linux/spi/flash.h
git log --all --oneline -- drivers/mtd/devices/m25p80.c
grep -r "flash_platform_data" drivers/ include/
```

---

### خارطة الـ Subsystems

```
flash_platform_data (include/linux/spi/flash.h)
          │
          ▼
    SPI Core (drivers/spi/spi.c)
          │
          ▼
   SPI NOR Core (drivers/mtd/spi-nor/core.c)
          │
          ├── SFDP Parser (sfdp.c)
          ├── Flash Database (flash_info tables)
          └── MTD Layer (include/linux/mtd/mtd.h)
                    │
                    ▼
          MTD Partitions (include/linux/mtd/partitions.h)
                    │
                    ▼
         Userspace (/dev/mtdX, /dev/mtdblockX)
```

الـ `flash_platform_data` كانت الطريقة القديمة لتمرير الـ board-specific info — الـ SFDP والـ Device Tree حلوا محلها في الـ kernel الحديث، لكن فهمها أساسي لقراءة الـ legacy code والـ board files القديمة.
## Phase 8: Writing simple module

### الهدف

الـ `flash_platform_data` هو struct بسيط بدون functions قابلة للـ hook مباشرةً، لكن أي SPI flash device بيتسجل في النظام بيمر على `spi_register_driver()` — وده المكان المناسب للـ hook.

سنستخدم **kprobe** على `spi_register_driver` عشان نطبع اسم أي SPI driver بيتسجل في النظام، وده مفيد لتتبع متى بيُضاف SPI flash driver زي `m25p80` أو `spi-nor`.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spi_flash_probe.c
 * Hooks spi_register_driver() via kprobe to log every SPI driver registration.
 * Useful for tracing SPI flash driver (m25p80, spi-nor, etc.) load events.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* kprobe struct, register_kprobe */
#include <linux/spi/spi.h>      /* struct spi_driver */
#include <linux/printk.h>       /* pr_info */

/* ------------------------------------------------------------------ */
/* pre-handler: runs just before spi_register_driver() executes        */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument is in RDI register.
     * On ARM64 it is in x0 register.
     * regs_get_kernel_argument() abstracts both.
     */
    struct spi_driver *sdrv =
        (struct spi_driver *)regs_get_kernel_argument(regs, 0);

    if (sdrv && sdrv->driver.name)
        pr_info("[spi_flash_probe] spi_register_driver called: driver='%s'\n",
                sdrv->driver.name);
    else
        pr_info("[spi_flash_probe] spi_register_driver called (unknown driver)\n");

    return 0; /* 0 = let the real function execute normally */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                    */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "spi_register_driver", /* function to hook by name  */
    .pre_handler = handler_pre,           /* our callback               */
};

/* ------------------------------------------------------------------ */
/* module init                                                          */
/* ------------------------------------------------------------------ */
static int __init spi_flash_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[spi_flash_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[spi_flash_probe] kprobe planted on spi_register_driver @ %p\n",
            kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit spi_flash_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[spi_flash_probe] kprobe removed from spi_register_driver\n");
}

module_init(spi_flash_probe_init);
module_exit(spi_flash_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe on spi_register_driver to trace SPI flash driver registration");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|--------|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل الـ API بتاعته |
| `linux/spi/spi.h` | تعريف `struct spi_driver` عشان نقدر نقرأ اسم الـ driver من الـ argument |
| `linux/printk.h` | `pr_info` / `pr_err` لطباعة رسائل الـ kernel log |

---

#### الـ `handler_pre`

ده الـ **callback** اللي بيتشغّل قبل ما `spi_register_driver()` تتنفذ فعلاً.

بنجيب أول argument من الـ CPU registers باستخدام `regs_get_kernel_argument(regs, 0)` — وده portable على x86-64 وARM64 بدون ما نكتب `#ifdef` يدوياً. بعدين بنـ cast الـ pointer لـ `struct spi_driver *` ونطبع `driver.name` — وده بيكشف اسم الـ SPI flash driver اللي بيتسجل، سواء كان `m25p80` أو `spi-nor` أو أي حاجة تانية.

---

#### `struct kprobe kp`

بنحدد فيه اسم الـ function المستهدفة كـ string (`symbol_name`) عشان الـ kernel يحل الـ symbol تلقائياً وقت الـ register، وبنربط الـ `pre_handler` بالـ callback بتاعنا.

---

#### `module_init` — `spi_flash_probe_init`

بيستدعي `register_kprobe()` اللي بتزرع software breakpoint في بداية `spi_register_driver`. لو فشل (مثلاً الـ symbol مش موجود أو الـ CONFIG_KPROBES مش enabled) بيطبع error ويرجع كود الخطأ.

---

#### `module_exit` — `spi_flash_probe_exit`

**لازم** نعمل `unregister_kprobe()` في الـ exit عشان لو فضل الـ breakpoint مزروع بعد ما الـ module اتـ unload، الـ kernel هيـ crash لأن الـ handler بقى pointer على memory مش موجودة.

---

### Makefile لبناء الـ module

```makefile
obj-m += spi_flash_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميله
sudo insmod spi_flash_probe.ko

# تحميل SPI flash driver (إن وُجد) لمشاهدة الـ hook
sudo modprobe spi-nor

# قراءة الـ log
sudo dmesg | grep spi_flash_probe

# تفريغ الـ module
sudo rmmod spi_flash_probe
```

**مثال على الـ output المتوقع:**

```
[spi_flash_probe] kprobe planted on spi_register_driver @ ffffffffc0a12340
[spi_flash_probe] spi_register_driver called: driver='spi-nor'
```
