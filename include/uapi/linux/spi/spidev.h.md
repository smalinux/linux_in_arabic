## Phase 1: الصورة الكبيرة ببساطة

### ما هو SPI؟

**SPI (Serial Peripheral Interface)** هو بروتوكول تواصل بين الـ microcontroller/CPU والأجهزة الطرفية السريعة زي شاشات الـ LCD، شرائح الـ flash، حساسات الحرارة، وكروت الـ SD. بيشتغل بـ 4 أسلاك أساسية:

| السلك | الاسم الكامل | الدور |
|-------|-------------|-------|
| MOSI | Master Out Slave In | البيانات من الـ CPU للجهاز |
| MISO | Master In Slave Out | البيانات من الجهاز للـ CPU |
| SCLK | Serial Clock | ساعة التزامن |
| CS/CE | Chip Select | تحديد الجهاز المتكلم معاه |

الـ SPI **full-duplex** يعني بيبعت ويستقبل في نفس الوقت — مش زي UART اللي sequential.

---

### القصة: ليه محتاجين spidev؟

تخيل عندك جهاز SPI جديد — مثلاً حساس ضغط أو شريحة EEPROM. في العادة عشان تتكلم معاه من Linux لازم تكتب **kernel driver** كامل، تفهم الـ kernel API، تعمل `insmod`، وهكذا. ده صعب وبطيء وبيحتاج خبرة kernel.

**الحل:** الـ `spidev` driver بيعمل **character device** في `/dev/spidevB.C` — بالظبط زي ما بتفتح ملف عادي. برنامج الـ userspace بيفتح الملف ده وبيبعت/يستقبل بيانات عبر `ioctl()` من غير ما يحتاج يكتب ولا سطر في الـ kernel.

```
User Program
    │
    │  open("/dev/spidev0.0", ...)
    │  ioctl(fd, SPI_IOC_MESSAGE(1), &transfer)
    ▼
┌──────────────┐
│  spidev.c    │  ← kernel driver بيترجم الـ ioctl لـ spi_sync()
│  (kernel)    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  SPI Master  │  ← controller hardware (مثلاً BCM2835 في Raspberry Pi)
│  Controller  │
└──────┬───────┘
       │ MOSI/MISO/SCLK/CS
       ▼
  [SPI Device]   ← مثلاً EEPROM أو حساس
```

---

### دور ملف `spidev.h` تحديداً

الملف ده هو **الـ UAPI header** — يعني الواجهة الرسمية بين الـ userspace code والـ kernel. أي برنامج C بيتكلم مع spidev لازم يعمل:

```c
#include <linux/spi/spidev.h>
```

الملف بيعرّف:
1. **`struct spi_ioc_transfer`** — وصف transfer واحدة كاملة (بيانات بعت + بيانات استقبل + سرعة + إعدادات).
2. **`SPI_IOC_MESSAGE(N)`** — الـ ioctl command اللي بتبعت N transfers دفعة واحدة (atomic).
3. **IOCTLs للإعدادات** — قراءة وكتابة mode (0..3)، سرعة، حجم الكلمة، ترتيب البتات.

---

### `struct spi_ioc_transfer` بالتفصيل

```c
struct spi_ioc_transfer {
    __u64  tx_buf;          /* pointer لبيانات الإرسال في userspace */
    __u64  rx_buf;          /* pointer لبيانات الاستقبال في userspace */
    __u32  len;             /* طول البيانات بالـ bytes */
    __u32  speed_hz;        /* سرعة مؤقتة لهذه الـ transfer فقط */
    __u16  delay_usecs;     /* تأخير بعد انتهاء الـ transfer */
    __u8   bits_per_word;   /* حجم الكلمة (عادةً 8) */
    __u8   cs_change;       /* هل نرفع CS بعد هذه الـ transfer؟ */
    __u8   tx_nbits;        /* عدد أسلاك الإرسال (1/2/4) */
    __u8   rx_nbits;        /* عدد أسلاك الاستقبال (1/2/4) */
    __u8   word_delay_usecs;/* تأخير بين الكلمات */
    __u8   pad;             /* padding للـ alignment */
};
```

الـ `tx_buf` و`rx_buf` عبارة عن `__u64` مش pointer — عشان يشتغل صح في 32-bit userspace على 64-bit kernel (الـ ABI ثابت).

---

### الـ IOCTLs المتاحة

```
SPI_IOC_RD_MODE / SPI_IOC_WR_MODE       ← اقرأ/اكتب mode (8-bit)
SPI_IOC_RD_MODE32 / SPI_IOC_WR_MODE32   ← نفسه بس 32-bit للـ flags الحديثة
SPI_IOC_RD_LSB_FIRST / SPI_IOC_WR_LSB_FIRST  ← ترتيب البتات
SPI_IOC_RD_BITS_PER_WORD / SPI_IOC_WR_BITS_PER_WORD  ← حجم الكلمة
SPI_IOC_RD_MAX_SPEED_HZ / SPI_IOC_WR_MAX_SPEED_HZ    ← أقصى سرعة
SPI_IOC_MESSAGE(N)                       ← أرسل N transfers مرة واحدة
```

---

### الـ SPI Modes الأربعة

الـ mode بيتحدد بـ **CPOL** (قطبية الـ clock) و**CPHA** (طور الـ clock):

| Mode | CPOL | CPHA | الـ clock الافتراضي | البيانات تُقرأ عند |
|------|------|------|---------------------|-------------------|
| 0 | 0 | 0 | Low | Rising edge |
| 1 | 0 | 1 | Low | Falling edge |
| 2 | 1 | 0 | High | Falling edge |
| 3 | 1 | 1 | High | Rising edge |

---

### مثال عملي كامل

```c
#include <linux/spi/spidev.h>
#include <fcntl.h>
#include <sys/ioctl.h>

int fd = open("/dev/spidev0.0", O_RDWR);

/* اكتب الـ mode */
uint8_t mode = SPI_MODE_0;
ioctl(fd, SPI_IOC_WR_MODE, &mode);

/* اكتب السرعة */
uint32_t speed = 1000000; /* 1 MHz */
ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

/* جهّز الـ transfer */
uint8_t tx[] = {0x9F};   /* أمر "اقرأ الـ JEDEC ID" */
uint8_t rx[3] = {0};

struct spi_ioc_transfer tr = {
    .tx_buf = (unsigned long)tx,
    .rx_buf = (unsigned long)rx,
    .len    = sizeof(tx),
    .speed_hz = speed,
    .bits_per_word = 8,
};

/* نفّذ الـ transfer */
ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
```

---

### الـ Subsystem في MAINTAINERS

الملف ده تحت **SPI SUBSYSTEM** المسؤول عنه:
- **Maintainer:** Mark Brown `<broonie@kernel.org>`
- **Mailing list:** `linux-spi@vger.kernel.org`
- **Tree:** `git://git.kernel.org/pub/scm/linux/kernel/git/broonie/spi.git`

---

### الملفات المكوّنة للـ Subsystem

#### Core & Headers

| الملف | الدور |
|-------|-------|
| `include/uapi/linux/spi/spidev.h` | **الملف الحالي** — UAPI interface لـ userspace |
| `include/uapi/linux/spi/spi.h` | تعريف الـ SPI modes وفلاغز للـ userspace |
| `include/linux/spi/spi.h` | الـ kernel-internal API (structs, functions) |
| `include/linux/spi/spi-mem.h` | واجهة الذاكرة SPI (Flash/NAND) |
| `include/linux/spi/spi_bitbang.h` | Bit-banging helper |

#### Drivers

| الملف | الدور |
|-------|-------|
| `drivers/spi/spi.c` | Core SPI framework |
| `drivers/spi/spidev.c` | الـ `/dev/spidevB.C` character device driver |
| `drivers/spi/spi-bcm2835.c` | Controller driver لـ Raspberry Pi |
| `drivers/spi/spi-imx.c` | Controller driver لـ NXP i.MX |
| `drivers/spi/spi-omap2-mcspi.c` | Controller driver لـ TI OMAP |

#### Tools

| الملف | الدور |
|-------|-------|
| `tools/spi/spidev_test.c` | أداة اختبار من command line |
| `tools/spi/spidev_fdx.c` | أداة full-duplex test |

#### Documentation

| الملف | الدور |
|-------|-------|
| `Documentation/spi/spidev.rst` | شرح كامل للـ spidev API |
| `Documentation/spi/spi-summary.rst` | نظرة عامة على الـ SPI subsystem |
## Phase 2: شرح الـ SPI Subsystem Framework

### المشكلة اللي الـ SPI Subsystem بيحلها

الـ **SPI (Serial Peripheral Interface)** بروتوكول hardware synchronous serial بيُستخدم في الـ embedded systems لتوصيل الـ microcontroller بـ peripherals زي الـ flash memory، الـ ADC/DAC، الـ sensors، وشاشات الـ LCD.

المشكلة الأساسية:
- كل SPI controller (مثلاً BCM2835 في Raspberry Pi، أو i.MX6 في NXP) بيتحكم في الـ hardware بطريقة مختلفة تماماً.
- كل SPI device (مثلاً W25Q128 flash، أو MCP3208 ADC) محتاج timing، mode، وـ clock محددين.
- من غير subsystem موحّد، كل driver محتاج يكتب كود لكل combination من controller + device — الموضوع بيبقى chaos.

**الهدف:** فصل الـ "إزاي أكلم الـ hardware controller" عن الـ "إيه البروتوكول اللي الـ device محتاجه".

---

### الحل اللي الـ Kernel اتخذه

الـ kernel قسّم المسؤولية لثلاث طبقات:

1. **SPI Core** — الطبقة الوسطى، بتربط كل حاجة ببعض.
2. **SPI Controller Driver** — بيتكلم مع الـ hardware controller الفعلي (SoC-specific).
3. **SPI Device Driver** — بيتكلم مع الـ peripheral الخارجي (device-specific).

وفوق ده كمان في **spidev** — وهو generic userspace driver بيسمح لـ user-space programs إنها تتحكم في الـ SPI مباشرةً عبر الـ `/dev/spidevX.Y` بدون ما تكتب kernel driver.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Space                               │
│                                                                 │
│   Application (Python/C)                                        │
│   open("/dev/spidev0.0")                                        │
│   ioctl(fd, SPI_IOC_MESSAGE(N), transfers[])                    │
└───────────────────────────┬─────────────────────────────────────┘
                            │  syscall boundary
┌───────────────────────────▼─────────────────────────────────────┐
│                    Kernel Space                                  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               spidev driver (drivers/spi/spidev.c)       │   │
│  │  - translates ioctl → spi_sync() calls                   │   │
│  │  - maps spi_ioc_transfer → spi_transfer (kernel struct)  │   │
│  │  - manages /dev/spidevX.Y character device               │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           │                                     │
│  ┌────────────────────────▼─────────────────────────────────┐   │
│  │               SPI Core (drivers/spi/spi.c)               │   │
│  │  - spi_sync() / spi_async()                              │   │
│  │  - spi_register_controller() / spi_register_device()     │   │
│  │  - bus matching (spi_bus_type)                           │   │
│  │  - message queue management                              │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           │                                     │
│  ┌────────────────────────▼─────────────────────────────────┐   │
│  │           SPI Controller Driver (SoC-specific)           │   │
│  │  e.g. drivers/spi/spi-bcm2835.c (Raspberry Pi)          │   │
│  │       drivers/spi/spi-imx.c     (NXP i.MX)              │   │
│  │  - implements transfer_one() or transfer_one_message()   │   │
│  │  - controls CLK, MOSI, MISO, CS GPIO lines               │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           │                                     │
└───────────────────────────┼─────────────────────────────────────┘
                            │  hardware registers
┌───────────────────────────▼─────────────────────────────────────┐
│                    SPI Hardware                                  │
│   SPI Controller (in SoC)  ←→  SPI Device (W25Q128, MCP3208)   │
│   [SCLK] [MOSI] [MISO] [CS0/CS1]                               │
└─────────────────────────────────────────────────────────────────┘
```

---

### مثال واقعي: spidev مع Raspberry Pi

```bash
# الـ device node بيظهر بعد تفعيل SPI في raspi-config
ls /dev/spidev*
# /dev/spidev0.0   ← bus 0, chip-select 0
# /dev/spidev0.1   ← bus 0, chip-select 1
```

---

### الـ Real-World Analogy — مترجم في مؤتمر دولي

تخيّل إنك في مؤتمر دولي:

| عنصر المؤتمر | المقابل في الـ SPI Subsystem |
|---|---|
| المتحدث الأجنبي (الـ peripheral) | SPI Device (W25Q128 flash) |
| المترجم الفوري | SPI Core + spidev |
| الميكروفون والسماعات (الـ infrastructure) | SPI Controller Driver |
| الجمهور (الـ application) | User Space Program |
| لغة المترجم المعيارية | `struct spi_ioc_transfer` |
| قواعد المؤتمر | SPI Mode (CPOL/CPHA) |

- **الجمهور** (user-space app) بيتكلم مع **المترجم** (spidev) باللغة المعيارية (`ioctl` + `spi_ioc_transfer`).
- **المترجم** (SPI core) بيعرف يتعامل مع أي متحدث لأنه عارف القواعد المعيارية.
- **الميكروفون** (SPI controller driver) هو اللي بيوصّل الكلام فعلياً للـ hardware — ومش مهم هيكونه Raspberry Pi أو NXP i.MX.
- **قواعد المؤتمر** (SPI Mode) بتحدد إزاي بالظبط الإشارة بتتنقل — نفس الموضوع مع الـ CPOL و CPHA.
- لو المتحدث (الـ peripheral) بيحتاج كلام بـ 9-bit words — الجمهور بيحدد ده في `bits_per_word` في الـ `spi_ioc_transfer`.

---

### الـ Core Abstraction: `spi_ioc_transfer`

ده الـ struct المركزي اللي الـ file `spidev.h` بيعرّفه — ده الـ **lingua franca** بين الـ userspace والـ kernel:

```c
struct spi_ioc_transfer {
    __u64   tx_buf;          /* pointer to TX data buffer in userspace */
    __u64   rx_buf;          /* pointer to RX data buffer in userspace */

    __u32   len;             /* length of transfer in bytes */
    __u32   speed_hz;        /* override default speed for this transfer */

    __u16   delay_usecs;     /* delay after transfer before next one */
    __u8    bits_per_word;   /* override default word size */
    __u8    cs_change;       /* deassert CS between transfers? */
    __u8    tx_nbits;        /* TX bus width: 1=standard, 2=dual, 4=quad */
    __u8    rx_nbits;        /* RX bus width: 1=standard, 2=dual, 4=quad */
    __u8    word_delay_usecs;/* inter-word delay (controller must support) */
    __u8    pad;             /* explicit padding for alignment */
};
```

**نقطة مهمة جداً:** الـ `tx_buf` و `rx_buf` هما `__u64` مش pointers — عشان يكونوا نفس الحجم على 32-bit و 64-bit systems (ده **compat ABI** issue كلاسيكي في الـ kernel). الـ kernel بعدين بيعمل `copy_from_user()` أو `get_user()` باستخدام الـ address المخزن فيهم.

#### مقارنة الـ `spi_ioc_transfer` مع الـ kernel `spi_transfer`

```
userspace                          kernel
─────────────────────────────────────────────────────────
struct spi_ioc_transfer            struct spi_transfer
─────────────────────────────────────────────────────────
tx_buf  (__u64 = user addr)   →    tx_buf  (const void *)
rx_buf  (__u64 = user addr)   →    rx_buf  (void *)
len                           →    len
speed_hz                      →    speed_hz
delay_usecs                   →    delay_usecs
bits_per_word                 →    bits_per_word
cs_change                     →    cs_change
tx_nbits                      →    tx_nbits
rx_nbits                      →    rx_nbits
word_delay_usecs              →    word_delay.value (usecs)
─────────────────────────────────────────────────────────
```

الـ `spidev` driver بيعمل الـ mapping ده يدوياً — بياخد الـ user buffer، بيعمل `kmalloc` لـ kernel buffer، بيعمل `copy_from_user`، وبعدين بيبعت `spi_message` للـ SPI core.

---

### الـ IOCTL Interface — تشريح كامل

#### تركيب الـ ioctl number

الـ `_IOC` macro بتبني 32-bit number:

```
 31      30    29      16  15       8  7        0
┌───────┬──────────────┬───────────┬────────────┐
│  DIR  │    SIZE      │   TYPE    │    NR      │
│ 2 bit │   14 bit     │   8 bit   │   8 bit    │
└───────┴──────────────┴───────────┴────────────┘
  RD/WR   size of arg    'k'=0x6B    command #
```

```c
#define SPI_IOC_MAGIC  'k'   /* 0x6B — unique identifier for SPI ioctls */
```

#### جدول الـ IOCTL commands

| Macro | Direction | NR | Arg Type | الوظيفة |
|---|---|---|---|---|
| `SPI_IOC_MESSAGE(N)` | Write (user→kernel) | 0 | `char[N*sizeof(spi_ioc_transfer)]` | إرسال N transfers دفعة واحدة |
| `SPI_IOC_RD_MODE` | Read (kernel→user) | 1 | `__u8` | قراءة الـ SPI mode الحالي |
| `SPI_IOC_WR_MODE` | Write (user→kernel) | 1 | `__u8` | تغيير الـ SPI mode |
| `SPI_IOC_RD_LSB_FIRST` | Read | 2 | `__u8` | قراءة bit order |
| `SPI_IOC_WR_LSB_FIRST` | Write | 2 | `__u8` | تغيير bit order |
| `SPI_IOC_RD_BITS_PER_WORD` | Read | 3 | `__u8` | قراءة word size |
| `SPI_IOC_WR_BITS_PER_WORD` | Write | 3 | `__u8` | تغيير word size |
| `SPI_IOC_RD_MAX_SPEED_HZ` | Read | 4 | `__u32` | قراءة max clock speed |
| `SPI_IOC_WR_MAX_SPEED_HZ` | Write | 4 | `__u32` | تغيير max clock speed |
| `SPI_IOC_RD_MODE32` | Read | 5 | `__u32` | قراءة extended mode flags |
| `SPI_IOC_WR_MODE32` | Write | 5 | `__u32` | تغيير extended mode flags |

**لماذا `MODE32` بجانب `MODE`؟**
الـ `SPI_IOC_RD/WR_MODE` القديم بيستخدم `__u8` — يعني 8 bits بس — وده كان كافي للـ modes الأصلية (CPOL, CPHA, CS_HIGH, LSB_FIRST, إلخ). لما الـ kernel أضاف features زي `SPI_TX_DUAL`, `SPI_TX_QUAD`, `SPI_TX_OCTAL` (bits 8-18)، اتضافت الـ `MODE32` بـ `__u32` عشان تستوعب الـ flags الإضافية.

---

### الـ SPI Modes — شرح CPOL و CPHA

الـ `spi.h` بيعرّف 4 modes أساسية بناءً على bit علموين:

| Mode | CPOL | CPHA | Clock idle | Data sampled |
|---|---|---|---|---|
| `SPI_MODE_0` | 0 | 0 | Low | Rising edge |
| `SPI_MODE_1` | 0 | 1 | Low | Falling edge |
| `SPI_MODE_2` | 1 | 0 | High | Falling edge |
| `SPI_MODE_3` | 1 | 1 | High | Rising edge |

```
SPI_MODE_0 (CPOL=0, CPHA=0):
CLK:  ─┐ ┌─┐ ┌─┐ ┌─┐ ┌─
       └─┘ └─┘ └─┘ └─┘
DATA: ──X───X───X───X──
         ↑   ↑   ↑   ↑  sample on rising

SPI_MODE_1 (CPOL=0, CPHA=1):
CLK:  ─┐ ┌─┐ ┌─┐ ┌─┐ ┌─
       └─┘ └─┘ └─┘ └─┘
DATA: ────X───X───X───X─
           ↑   ↑   ↑   sample on falling
```

**الـ `SPI_LSB_FIRST`** بيقلب اتجاه إرسال الـ bits — معظم الـ devices بتشتغل MSB first (bit رقم 7 الأول).

---

### الـ SPI Bus Width — Standard vs Dual vs Quad vs Octal

الـ `tx_nbits` و `rx_nbits` في `spi_ioc_transfer` بيحددوا عدد الـ data lines:

```
Standard SPI (1-bit):          Quad SPI (4-bit):
   MOSI ──────────►               IO0 ◄────────►
   MISO ◄──────────               IO1 ◄────────►
                                   IO2 ◄────────►
                                   IO3 ◄────────►
```

- **Standard:** MOSI (TX) + MISO (RX) منفصلين، full duplex.
- **Dual SPI:** سلكين للـ data، half duplex، ضعف الـ throughput.
- **Quad SPI (QSPI):** 4 أسلاك، ربع ضعف الـ throughput — شائع في Flash memory زي W25Q128.
- **Octal SPI:** 8 أسلاك — بيُستخدم في الـ high-speed NOR Flash.

الـ controller driver لازم يدعم الـ multi-bit mode، غير كده الـ kernel بيـ ignore الـ field بصمت.

---

### إزاي `SPI_IOC_MESSAGE(N)` بيشتغل فعلياً

الـ macro:
```c
#define SPI_MSGSIZE(N) \
    ((((N)*(sizeof(struct spi_ioc_transfer))) < (1 << _IOC_SIZEBITS)) \
        ? ((N)*(sizeof(struct spi_ioc_transfer))) : 0)

#define SPI_IOC_MESSAGE(N)  _IOW(SPI_IOC_MAGIC, 0, char[SPI_MSGSIZE(N)])
```

- الـ `_IOC_SIZEBITS = 14` يعني الـ max size في الـ ioctl number هو `2^14 - 1 = 16383 bytes`.
- الـ `SPI_MSGSIZE(N)` بيتحقق إن `N * sizeof(spi_ioc_transfer)` مش أكبر من الـ limit — لو أكبر يرجع 0 (error).
- الـ `sizeof(struct spi_ioc_transfer) = 32 bytes` → يعني max N ≈ 512 transfer في message واحدة عبر الـ ioctl size field.

#### تسلسل العمليات في الـ kernel عند `ioctl(fd, SPI_IOC_MESSAGE(N), mesg)`

```
User Space:
  mesg[N] = [{tx_buf=0xUSER_ADDR, rx_buf=..., len=..., ...}, ...]
  ioctl(fd, SPI_IOC_MESSAGE(N), mesg)
         │
         ▼
spidev_ioctl() in spidev.c:
  1. kmalloc(N * sizeof(spi_ioc_transfer))  ← kernel buffer
  2. copy_from_user(k_xfers, mesg, total_size)
  3. for each transfer:
     a. kmalloc(tx_buf_kernel)
     b. copy_from_user(tx_buf_kernel, user_tx_buf)
     c. kmalloc(rx_buf_kernel)
     d. build spi_transfer { .tx_buf=tx_buf_kernel, .rx_buf=rx_buf_kernel, ... }
     e. spi_message_add_tail(&xfer, &msg)
  4. spi_sync(spi_device, &msg)   ← hand off to SPI core
         │
         ▼
SPI Core (spi.c):
  spi_sync() → __spi_sync() → __spi_pump_messages()
         │
         ▼
Controller Driver (e.g., spi-bcm2835.c):
  bcm2835_spi_transfer_one() → DMA or PIO transfer
         │
         ▼
  5. copy_to_user(user_rx_buf, rx_buf_kernel)  ← results back to user
  6. kfree all kernel buffers
```

---

### ملكية المسؤوليات: الـ SPI Core يملك vs يُفوّض

#### الـ SPI Core يملك:
- الـ **bus matching** — ربط الـ SPI device بالـ driver الصح.
- الـ **message queuing** — تنظيم الـ transfers في queue.
- الـ **locking** — منع تعارض الـ access من threads مختلفة.
- الـ **spi_sync() / spi_async()** API.
- الـ **chip select management** (بيستدعي `set_cs` على الـ controller).
- الـ **clock/mode validation** — التحقق إن الـ speed مش أكبر من max.

#### الـ SPI Core يُفوّض للـ Controller Driver:
- الـ **actual bit-banging أو DMA transfer** — ده hardware-specific بالكامل.
- الـ **`transfer_one()`** أو `transfer_one_message()` — اللي بيتنفذ على الـ SoC الفعلي.
- الـ **CS GPIO control** لو الـ controller مش بيعملها أوتوماتيك.
- الـ **timing constraints** زي inter-byte delays.

#### الـ spidev يُفوّض للـ SPI Core:
- كل حاجة بعد ما بيبني الـ `spi_message` — هو مش عارف الـ hardware خالص.

---

### الـ Struct Relationship Diagram

```
User Space                 Kernel Space
─────────────────────────────────────────────────────────────────
struct                     struct              struct
spi_ioc_transfer           spi_transfer        spi_message
─────────────              ─────────────       ─────────────────
tx_buf (__u64)  ──copy──►  tx_buf (void*)      transfers (list)
rx_buf (__u64)  ──copy──►  rx_buf (void*)      spi (device ptr)
len             ────────►  len                 status
speed_hz        ────────►  speed_hz            actual_length
bits_per_word   ────────►  bits_per_word            │
delay_usecs     ────────►  delay_usecs               │
cs_change       ────────►  cs_change                 │
tx_nbits        ────────►  tx_nbits                  │
rx_nbits        ────────►  rx_nbits           ┌──────▼───────┐
                                               │  SPI Core    │
                                               │  spi_sync()  │
                                               └──────┬───────┘
                                                      │
                                               ┌──────▼───────┐
                                               │  Controller  │
                                               │  transfer_   │
                                               │  one()       │
                                               └──────────────┘
```

---

### مثال كامل: قراءة W25Q128 Flash ID من User Space

```c
#include <fcntl.h>
#include <sys/ioctl.h>
#include <linux/spi/spidev.h>
#include <string.h>
#include <stdio.h>

int main(void) {
    int fd = open("/dev/spidev0.0", O_RDWR);

    /* configure: SPI_MODE_0, 8 bits/word, 10 MHz */
    uint8_t mode = SPI_MODE_0;
    uint8_t bits = 8;
    uint32_t speed = 10000000;

    ioctl(fd, SPI_IOC_WR_MODE, &mode);
    ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
    ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

    /* W25Q128: JEDEC ID command = 0x9F, returns 3 bytes */
    uint8_t tx[] = {0x9F, 0x00, 0x00, 0x00};
    uint8_t rx[4] = {0};

    struct spi_ioc_transfer tr = {
        .tx_buf        = (unsigned long)tx,
        .rx_buf        = (unsigned long)rx,
        .len           = 4,
        .speed_hz      = 10000000,
        .bits_per_word = 8,
        .cs_change     = 0,   /* keep CS asserted throughout */
    };

    /* single transfer: send command, receive JEDEC ID */
    ioctl(fd, SPI_IOC_MESSAGE(1), &tr);

    /* rx[1]=0xEF (Winbond), rx[2]=0x40, rx[3]=0x18 → W25Q128 */
    printf("JEDEC ID: %02X %02X %02X\n", rx[1], rx[2], rx[3]);

    return 0;
}
```

هنا الـ `cs_change = 0` معناها إن الـ CS هيفضل low طول الـ transfer — لو بنعمل read operation محتاجة command ثم data، ممكن نعمل `cs_change = 1` في أول transfer وبعدين second transfer بدون CS deassert لو الـ device يتطلب كده.

---

### الفرق بين `SPI_IOC_RD_MODE` و `SPI_IOC_RD_MODE32`

```c
/* old API — only lower 8 bits of mode flags */
uint8_t mode8;
ioctl(fd, SPI_IOC_RD_MODE, &mode8);   /* misses SPI_TX_QUAD etc. */

/* new API — full 32-bit mode flags */
uint32_t mode32;
ioctl(fd, SPI_IOC_RD_MODE32, &mode32); /* includes bits 8-18 */
```

الـ flags الزيادة في الـ `spi.h`:

| Flag | Bit | الوظيفة |
|---|---|---|
| `SPI_TX_DUAL` | 8 | TX على 2 wires |
| `SPI_TX_QUAD` | 9 | TX على 4 wires |
| `SPI_RX_DUAL` | 10 | RX على 2 wires |
| `SPI_RX_QUAD` | 11 | RX على 4 wires |
| `SPI_CS_WORD` | 12 | toggle CS بعد كل word |
| `SPI_TX_OCTAL` | 13 | TX على 8 wires |
| `SPI_RX_OCTAL` | 14 | RX على 8 wires |
| `SPI_3WIRE_HIZ` | 15 | high-Z turnaround لـ 3-wire |
| `SPI_RX_CPHA_FLIP` | 16 | flip CPHA على RX only |
| `SPI_MOSI_IDLE_LOW` | 17 | MOSI = 0 when idle |
| `SPI_MOSI_IDLE_HIGH` | 18 | MOSI = 1 when idle |

---

### علاقة الـ spidev بالـ Device Tree

الـ spidev driver مش بيشتغل لوحده — محتاج الـ SPI device يتعرّف في الـ Device Tree (أو ACPI/boardfile):

```dts
/* Device Tree snippet for spidev on Raspberry Pi */
&spi0 {
    status = "okay";
    spidev@0 {
        compatible = "rohm,dh2228fv";  /* generic spidev node */
        reg = <0>;                     /* chip-select 0 */
        spi-max-frequency = <500000>;  /* 500 kHz max */
    };
};
```

الـ `compatible` string بتحدد إن ده spidev device — الـ kernel بيعمل probe ويـ create الـ `/dev/spidev0.0`.

**ملاحظة مهمة:** الـ kernel من version معين بدأ يـ warn لو استخدمت `compatible = "spidev"` مباشرةً — عشان كده بيُستخدم compatible strings حقيقية لـ devices (زي `rohm,dh2228fv`) حتى لو الـ device مختلف.

---

### الأنظمة الفرعية المرتبطة

- **الـ Device Model / Driver Core (`drivers/base/`)**: الـ SPI subsystem بيعتمد عليه لـ bus registration وـ device matching.
- **الـ DMA Engine (`include/linux/dmaengine.h`)**: كتير من الـ SPI controllers بتستخدم DMA لنقل البيانات بدل الـ PIO — الـ SPI core بيدعم ده عبر `dma_map_single()`.
- **الـ GPIO Subsystem**: بعض الـ controllers بتستخدم GPIO pins عشان الـ chip select لو الـ hardware CS مش كافي.
- **الـ Clock Framework (`include/linux/clk.h`)**: كل SPI controller بيحتاج clock — الـ controller driver بيـ request الـ clock عبر `clk_get()`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ SPI Mode Flags — Cheatsheet

الـ flags دي بتتحدد في `include/uapi/linux/spi/spi.h` وبيستخدمها الـ `spidev` عشان يتحكم في طريقة الـ transfer.

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_CPHA` | 0 | Clock Phase — بيحدد إيمتى البيانات بتتسمبل على الـ clock |
| `SPI_CPOL` | 1 | Clock Polarity — بيحدد الـ idle state للـ clock |
| `SPI_CS_HIGH` | 2 | الـ chip select active high بدل low |
| `SPI_LSB_FIRST` | 3 | بيبعت الـ LSB الأول بدل الـ MSB |
| `SPI_3WIRE` | 4 | الـ MOSI والـ MISO على نفس الـ wire |
| `SPI_LOOP` | 5 | Loopback mode للتست |
| `SPI_NO_CS` | 6 | مفيش chip select (جهاز واحد على الـ bus) |
| `SPI_READY` | 7 | الـ slave بيعمل pull-low للـ pause |
| `SPI_TX_DUAL` | 8 | بيبعت على 2 wires |
| `SPI_TX_QUAD` | 9 | بيبعت على 4 wires (QSPI) |
| `SPI_RX_DUAL` | 10 | بياخد على 2 wires |
| `SPI_RX_QUAD` | 11 | بياخد على 4 wires |
| `SPI_CS_WORD` | 12 | بيعمل toggle للـ CS بعد كل word |
| `SPI_TX_OCTAL` | 13 | بيبعت على 8 wires |
| `SPI_RX_OCTAL` | 14 | بياخد على 8 wires |
| `SPI_3WIRE_HIZ` | 15 | High-impedance turnaround في 3-wire mode |
| `SPI_RX_CPHA_FLIP` | 16 | بيقلب الـ CPHA في Rx-only transfer |
| `SPI_MOSI_IDLE_LOW` | 17 | خليلي الـ MOSI low لما ميفيش data |
| `SPI_MOSI_IDLE_HIGH` | 18 | خليلي الـ MOSI high لما ميفيش data |

#### الـ SPI Modes الأساسية

| Mode | CPOL | CPHA | الـ Clock Idle | بيسمبل إيمتى |
|------|------|------|---------------|-------------|
| `SPI_MODE_0` | 0 | 0 | Low | Rising edge |
| `SPI_MODE_1` | 0 | 1 | Low | Falling edge |
| `SPI_MODE_2` | 1 | 0 | High | Falling edge |
| `SPI_MODE_3` | 1 | 1 | High | Rising edge |

---

### الـ IOCTL Commands — Cheatsheet

**الـ Magic Number**: `'k'` = `0x6B`

| IOCTL | Direction | Type | Nr | المعنى |
|-------|-----------|------|----|--------|
| `SPI_IOC_MESSAGE(N)` | Write | `char[N*sizeof(spi_ioc_transfer)]` | 0 | بيبعت array من الـ transfers |
| `SPI_IOC_RD_MODE` | Read | `__u8` | 1 | بيقرأ الـ SPI mode (8-bit) |
| `SPI_IOC_WR_MODE` | Write | `__u8` | 1 | بيكتب الـ SPI mode (8-bit) |
| `SPI_IOC_RD_LSB_FIRST` | Read | `__u8` | 2 | بيقرأ اتجاه الـ bits |
| `SPI_IOC_WR_LSB_FIRST` | Write | `__u8` | 2 | بيكتب اتجاه الـ bits |
| `SPI_IOC_RD_BITS_PER_WORD` | Read | `__u8` | 3 | بيقرأ حجم الـ word |
| `SPI_IOC_WR_BITS_PER_WORD` | Write | `__u8` | 3 | بيكتب حجم الـ word |
| `SPI_IOC_RD_MAX_SPEED_HZ` | Read | `__u32` | 4 | بيقرأ أقصى سرعة |
| `SPI_IOC_WR_MAX_SPEED_HZ` | Write | `__u32` | 4 | بيكتب أقصى سرعة |
| `SPI_IOC_RD_MODE32` | Read | `__u32` | 5 | بيقرأ الـ mode كاملاً (32-bit) |
| `SPI_IOC_WR_MODE32` | Write | `__u32` | 5 | بيكتب الـ mode كاملاً (32-bit) |

> الفرق بين `RD_MODE` (8-bit) و `RD_MODE32` (32-bit): الأول للـ flags الأساسية بس (bits 0-7)، التاني بيشمل الـ extended flags زي `SPI_TX_QUAD` وغيره.

---

### الـ Structs

#### `struct spi_ioc_transfer`

الـ struct الوحيدة في الملف دا، وهي بتمثل **transfer واحدة** على الـ SPI bus من الـ userspace.

```c
struct spi_ioc_transfer {
    __u64  tx_buf;           /* pointer to TX buffer in userspace (or 0) */
    __u64  rx_buf;           /* pointer to RX buffer in userspace (or 0) */

    __u32  len;              /* length of tx/rx buffers in bytes */
    __u32  speed_hz;         /* per-transfer speed override (0 = use default) */

    __u16  delay_usecs;      /* delay after last bit before CS deassert */
    __u8   bits_per_word;    /* per-transfer word size override (0 = default) */
    __u8   cs_change;        /* deselect CS before next transfer if nonzero */
    __u8   tx_nbits;         /* number of wires for TX (1/2/4/8) */
    __u8   rx_nbits;         /* number of wires for RX (1/2/4/8) */
    __u8   word_delay_usecs; /* delay between words within transfer */
    __u8   pad;              /* explicit padding — must be zero */
};
```

##### شرح الـ Fields

| Field | الحجم | الـ Default | المعنى |
|-------|-------|------------|--------|
| `tx_buf` | 64-bit | `NULL` | عنوان الـ userspace buffer للبيانات المبعوتة — لو `NULL` بيبعت zeros |
| `rx_buf` | 64-bit | `NULL` | عنوان الـ userspace buffer للبيانات المستقبَلة — لو `NULL` بيتجاهل الـ RX |
| `len` | 32-bit | مطلوب | حجم الـ transfer بالـ bytes — بينطبق على الـ tx والـ rx |
| `speed_hz` | 32-bit | 0 (default) | تجاوز مؤقت لسرعة الـ device — بيرجع للـ default بعد الـ transfer |
| `delay_usecs` | 16-bit | 0 | انتظار بعد آخر bit قبل ما يشيل الـ CS — مفيد للـ devices اللي محتاجة settling time |
| `bits_per_word` | 8-bit | 0 (default) | تجاوز مؤقت لحجم الـ word |
| `cs_change` | 8-bit | 0 | لو 1 بيعمل deselect للـ CS قبل الـ transfer الجاية |
| `tx_nbits` | 8-bit | 1 | عدد الـ wires للإرسال (1=standard, 2=dual, 4=quad, 8=octal) |
| `rx_nbits` | 8-bit | 1 | عدد الـ wires للاستقبال |
| `word_delay_usecs` | 8-bit | 0 | تأخير بين الـ words — محتاج دعم في الـ controller |
| `pad` | 8-bit | 0 | padding صريح — لازم يكون zero |

##### نقطة مهمة في الـ Layout

الـ struct اتصممت عشان **layout واحد في 32-bit و 64-bit userspace**:
- الـ `tx_buf` و `rx_buf` هما `__u64` مش pointers — عشان على 32-bit kernel بيجي userspace pointer 32-bit فيتحط في `__u64` بدون مشكلة alignment.
- الـ total size = 8+8+4+4+2+1+1+1+1+1+1 = **32 bytes**.

---

### علاقة الـ Structs ببعض

#### الـ Userspace View

```
userspace application
        |
        | open("/dev/spidev0.0", O_RDWR)
        v
  file descriptor (fd)
        |
        | ioctl(fd, SPI_IOC_MESSAGE(N), transfers[])
        v
  struct spi_ioc_transfer[N]   <-- الـ struct اللي في الملف دا
   ┌─────────────────────────┐
   │ tx_buf  ──→ user TX buf │
   │ rx_buf  ──→ user RX buf │
   │ len                     │
   │ speed_hz                │
   │ delay_usecs             │
   │ cs_change               │
   │ ...                     │
   └─────────────────────────┘
```

#### الـ Kernel View (بعد ما الـ ioctl يوصل للـ kernel)

```
struct spi_ioc_transfer (uapi)
        |
        | kernel copies + validates
        v
struct spi_transfer (kernel internal)    [include/linux/spi/spi.h]
   ┌─────────────────────────────────┐
   │ tx_buf   ──→ kernel TX buffer   │
   │ rx_buf   ──→ kernel RX buffer   │
   │ len                             │
   │ speed_hz                        │
   │ delay_usecs                     │
   │ cs_change                       │
   │ list ──→ next transfer in msg   │
   └─────────────────────────────────┘
        |
        | added to
        v
struct spi_message
   ┌───────────────────────────┐
   │ transfers (linked list)   │
   │ spi ──→ struct spi_device │
   │ complete callback         │
   │ status                    │
   └───────────────────────────┘
        |
        | spi_sync() or spi_async()
        v
struct spi_controller (master)
   ┌─────────────────────────────────┐
   │ transfer_one_message()          │
   │ bus_num                         │
   │ dev ──→ struct device           │
   └─────────────────────────────────┘
        |
        v
   Hardware SPI registers
```

---

### Lifecycle Diagram

```
=== OPEN ===
userspace: open("/dev/spidevB.D")
    │
    ▼
kernel: spidev_open()
    ├─ بيخلي reference على struct spi_device
    ├─ بيعمل alloc لـ spidev_data struct
    └─ بيحط file->private_data = spidev

=== CONFIGURE (optional) ===
ioctl(fd, SPI_IOC_WR_MODE, &mode)
    │
    ▼
kernel: spidev_ioctl()
    ├─ بيقرأ mode من userspace
    ├─ بيعمل validate للـ flags
    └─ بيكتب على spi_device->mode + spi_setup()

=== TRANSFER ===
ioctl(fd, SPI_IOC_MESSAGE(N), transfers)
    │
    ▼
kernel: spidev_message()
    ├─ copy_from_user() — بيجيب struct spi_ioc_transfer[] من userspace
    ├─ بيعمل alloc لـ kernel buffers
    ├─ copy_from_user() للـ TX data
    ├─ بيحول كل spi_ioc_transfer → spi_transfer
    ├─ بيحط كل transfer في spi_message
    ├─ spi_sync() — بيبعت المسج للـ controller
    ├─ copy_to_user() للـ RX data
    └─ بيحرر الـ buffers

=== CLOSE ===
close(fd)
    │
    ▼
kernel: spidev_release()
    └─ بيحرر الـ spidev_data وبيشيل الـ reference
```

---

### Call Flow Diagrams

#### Transfer كاملة من الـ userspace للـ hardware

```
User App
  │
  │  ioctl(fd, SPI_IOC_MESSAGE(2), mesg[2])
  ▼
spidev_ioctl()                    [drivers/spi/spidev.c]
  │
  │  spidev_message(spidev, ioc, n_ioc)
  ▼
spidev_message()
  ├── copy_from_user(ioc, u_ioc, n*sizeof)   /* بياخد الـ transfers من userspace */
  ├── for each transfer:
  │     ├── alloc tx/rx kernel buffers
  │     ├── copy_from_user(tx_buf)            /* بياخد الـ TX data */
  │     └── spi_message_add_tail(k_xfer, &msg)
  │
  │  spi_sync(spi, &msg)
  ▼
spi_sync()                        [drivers/spi/spi.c]
  │
  │  __spi_sync() → spi_start_transfer()
  ▼
spi_controller->transfer_one_message()       /* controller driver */
  │
  ├── for each spi_transfer in message:
  │     ├── setup CS (assert)
  │     ├── configure clock speed
  │     ├── controller->transfer_one(xfer)
  │     │       └── writes to hardware registers (SPI_DR, SPI_CR, ...)
  │     ├── wait for completion (interrupt or polling)
  │     └── delay_usecs / cs_change logic
  │
  └── complete callback → spi_sync() wakes up
  │
  ▼
spidev_message() resumes
  ├── copy_to_user(rx_buf)         /* بيرجع الـ RX data للـ userspace */
  └── returns total bytes
```

#### IOCTL بسيطة — تغيير الـ mode

```
User App
  │  ioctl(fd, SPI_IOC_WR_MODE32, &new_mode)
  ▼
spidev_ioctl()
  ├── get_user(tmp, (__u32 __user *)arg)
  ├── spi->mode = (spi->mode & ~SPI_MODE_USER_MASK) | (tmp & SPI_MODE_USER_MASK)
  └── spi_setup(spi)
        │
        ▼
      controller->setup(spi)      /* بيحدث hardware registers */
```

---

### الـ `SPI_MSGSIZE` Macro

```c
#define SPI_MSGSIZE(N) \
    ((((N)*(sizeof(struct spi_ioc_transfer))) < (1 << _IOC_SIZEBITS)) \
        ? ((N)*(sizeof(struct spi_ioc_transfer))) : 0)
```

الـ macro دا بيحسب الحجم الكلي للـ array من الـ transfers وبيتأكد إنه مش بيتعدى الـ limit المسموح في الـ `_IOC_SIZEBITS` (عادةً 14 bit = 16383 bytes). لو عدى الـ limit بيرجع 0 وبيخلي الـ ioctl يفشل.

**الـ `_IOC_SIZEBITS`** = 14 bits على معظم الـ architectures، يعني أقصى حجم = 16383 bytes.

بما إن `sizeof(spi_ioc_transfer)` = 32 bytes، يعني أقصى عدد transfers في message واحدة = **511 transfer**.

---

### Locking Strategy

الـ file دا uapi header — مفيش locks فيه. بس لازم نفهم الـ locking اللي بيحصل في الـ kernel لما الـ ioctl يتنفذ:

#### الـ Locks في `spidev` driver

```
┌─────────────────────────────────────────────────────┐
│               spidev_data                           │
│                                                     │
│  buf_lock (mutex)  ──── يحمي tx_buffer, rx_buffer  │
│  spi_lock (spinlock) ── يحمي spi pointer نفسه      │
└─────────────────────────────────────────────────────┘
```

| Lock | النوع | بيحمي إيه |
|------|-------|-----------|
| `spidev->buf_lock` | `mutex` | الـ TX/RX kernel buffers — منع concurrent ioctls |
| `spidev->spi_lock` | `spinlock` | الـ `spidev->spi` pointer نفسه — منع race مع الـ unbind |
| `spi_controller->bus_lock_mutex` | `mutex` | الـ SPI bus كله — serialize الـ transfers |

#### الـ Lock Ordering

```
spi_lock (spinlock)          [أسرع — بيحمي pointer فقط]
    ↓
buf_lock (mutex)             [بيحمي buffers + transfer كاملة]
    ↓
bus_lock_mutex (controller)  [بيحمي الـ hardware bus]
```

#### سيناريو الـ Race المهم

لما الـ spidev device بتتشال (مثلاً الـ SPI device اتفصل):

```
Thread 1 (ioctl in progress)     Thread 2 (device removal)
─────────────────────────────    ──────────────────────────
spidev_ioctl()
  spin_lock(spi_lock)
  spi = spidev->spi              spidev_remove()
  spin_unlock(spi_lock)            spin_lock(spi_lock)
                                   spidev->spi = NULL   ← safe, spi still valid
  spi_sync(spi, ...)               spin_unlock(spi_lock)
  ...transfer completes...
                                 spi_dev_put(spi)      ← after ioctl done
```

الـ `spin_lock` بيحمي الـ pointer من الـ NULL assignment، والـ reference counting بيضمن إن الـ `spi_device` مش بيتحرر وإحنا لسه بنستخدمه.

---

### ASCII Diagram — العلاقة الكاملة

```
╔══════════════════════════════════════════════╗
║              USERSPACE                       ║
║                                              ║
║  app                                         ║
║   │                                          ║
║   │ open("/dev/spidev0.0")                   ║
║   │ ioctl(fd, SPI_IOC_MESSAGE(N), xfers[])   ║
║   │                                          ║
║   └─── struct spi_ioc_transfer[] ───────────→║
╚══════════════════════════════════════════════╝
                    │  syscall boundary
╔══════════════════════════════════════════════╗
║              KERNEL SPACE                    ║
║                                              ║
║  spidev_data                                 ║
║  ┌─────────────────┐                         ║
║  │ spi ────────────┼──→ spi_device           ║
║  │ buf_lock        │     ┌──────────────┐    ║
║  │ tx_buffer       │     │ master ──────┼──→ spi_controller ║
║  │ rx_buffer       │     │ mode         │    │ ┌───────────┐║
║  └─────────────────┘     │ max_speed_hz │    │ │ transfer_ ││
║                          └──────────────┘    │ │ one_msg() ││
║                                              │ └───────────┘║
║  spi_message                                 │              ║
║  ┌───────────────────────┐                   │              ║
║  │ transfers (list_head) │                   │              ║
║  │  ├→ spi_transfer[0]   │                   │              ║
║  │  ├→ spi_transfer[1]   │                   │              ║
║  │  └→ spi_transfer[N-1] │                   │              ║
║  └───────────────────────┘                   │              ║
╚══════════════════════════════════════════════╝
                    │
╔══════════════════════════════════════════════╗
║              HARDWARE                        ║
║                                              ║
║  SPI Controller Registers                    ║
║  ┌──────────────────────┐                    ║
║  │ CR1: mode, speed     │                    ║
║  │ DR:  data register   │                    ║
║  │ SR:  status register │                    ║
║  └──────────────────────┘                    ║
║         │ MOSI/MISO/SCK/CS                   ║
║         ▼                                    ║
║  SPI Slave Device (sensor, flash, ...)       ║
╚══════════════════════════════════════════════╝
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ API — Cheatsheet

#### الـ IOCTLs — جدول سريع

| IOCTL | Direction | Type | Nr | وصف مختصر |
|---|---|---|---|---|
| `SPI_IOC_MESSAGE(N)` | Write (to kernel) | `char[N*sizeof(spi_ioc_transfer)]` | 0 | تنفيذ N transfers دفعة واحدة (userspace spi_sync) |
| `SPI_IOC_RD_MODE` | Read | `__u8` | 1 | قراءة SPI mode (0..3) — 8-bit |
| `SPI_IOC_WR_MODE` | Write | `__u8` | 1 | كتابة SPI mode (0..3) — 8-bit |
| `SPI_IOC_RD_LSB_FIRST` | Read | `__u8` | 2 | قراءة bit justification |
| `SPI_IOC_WR_LSB_FIRST` | Write | `__u8` | 2 | كتابة bit justification |
| `SPI_IOC_RD_BITS_PER_WORD` | Read | `__u8` | 3 | قراءة word size بالـ bits |
| `SPI_IOC_WR_BITS_PER_WORD` | Write | `__u8` | 3 | كتابة word size بالـ bits |
| `SPI_IOC_RD_MAX_SPEED_HZ` | Read | `__u32` | 4 | قراءة max clock speed |
| `SPI_IOC_WR_MAX_SPEED_HZ` | Write | `__u32` | 4 | كتابة max clock speed |
| `SPI_IOC_RD_MODE32` | Read | `__u32` | 5 | قراءة SPI mode — 32-bit كامل |
| `SPI_IOC_WR_MODE32` | Write | `__u32` | 5 | كتابة SPI mode — 32-bit كامل |

#### الـ Macros المساعدة

| Macro | الغرض |
|---|---|
| `SPI_IOC_MAGIC` | الـ magic number `'k'` لكل IOCTLs الـ spidev |
| `SPI_MSGSIZE(N)` | حساب حجم array الـ transfers بأمان مع overflow check |
| `SPI_IOC_MESSAGE(N)` | بناء الـ IOCTL number لـ N transfers |

#### الـ SPI Mode Flags (من `linux/spi/spi.h`)

| Flag | Bit | المعنى |
|---|---|---|
| `SPI_CPHA` | 0 | Clock phase |
| `SPI_CPOL` | 1 | Clock polarity |
| `SPI_MODE_0..3` | — | combinations of CPHA/CPOL |
| `SPI_CS_HIGH` | 2 | Chip select active high |
| `SPI_LSB_FIRST` | 3 | LSB أول على السلك |
| `SPI_3WIRE` | 4 | MOSI/MISO على نفس الخط |
| `SPI_LOOP` | 5 | Loopback mode |
| `SPI_TX_DUAL/QUAD/OCTAL` | 8,9,13 | Multi-wire TX |
| `SPI_RX_DUAL/QUAD/OCTAL` | 10,11,14 | Multi-wire RX |
| `SPI_CS_WORD` | 12 | Toggle CS بعد كل word |
| `SPI_MOSI_IDLE_LOW/HIGH` | 17,18 | حالة MOSI وقت الـ idle |

---

### Group 1: Transfer Execution — `SPI_IOC_MESSAGE`

الغرض من الـ group ده هو تنفيذ واحدة أو أكثر من الـ SPI transfers في syscall واحدة، وده المكافئ المباشر لـ `spi_sync()` في kernel space.

---

#### `SPI_MSGSIZE(N)`

```c
#define SPI_MSGSIZE(N) \
    ((((N)*(sizeof (struct spi_ioc_transfer))) < (1 << _IOC_SIZEBITS)) \
        ? ((N)*(sizeof (struct spi_ioc_transfer))) : 0)
```

**الـ macro بيحسب الحجم الكلي لـ array من N transfers مع guard ضد overflow في الـ IOCTL size field.**

- **`N`** — عدد الـ transfers المطلوبة.
- **`_IOC_SIZEBITS`** — عادةً 14 bit، يعني الحجم الأقصى 16383 bytes. لو الحجم تجاوز الحد بترجع صفر، وده بيخلي `SPI_IOC_MESSAGE` يولد IOCTL number غلط وبالتالي kernel بيرفضه بـ `ENOTTY`.
- **Return** — حجم الـ array بالـ bytes أو صفر لو overflow.
- **لازم تعمله zero-initialize** على كل عناصر الـ array عشان حقول مستقبلية محتملة ميأثرش عليها garbage.

---

#### `SPI_IOC_MESSAGE(N)`

```c
#define SPI_IOC_MESSAGE(N) _IOW(SPI_IOC_MAGIC, 0, char[SPI_MSGSIZE(N)])
```

```c
/* userspace usage */
struct spi_ioc_transfer xfer[2];
memset(xfer, 0, sizeof(xfer));

xfer[0].tx_buf = (unsigned long)tx_cmd;
xfer[0].len    = 1;
xfer[0].cs_change = 1; /* deselect then reselect before next transfer */

xfer[1].rx_buf = (unsigned long)rx_data;
xfer[1].len    = 16;

int ret = ioctl(fd, SPI_IOC_MESSAGE(2), xfer);
/* ret == total bytes transferred, or -1 on error */
```

**الـ IOCTL ده بيبعت array من `spi_ioc_transfer` للـ kernel اللي بيحولها لـ `spi_message` فيها متعدد `spi_transfer` nodes ويستدعي `spi_sync()` عليها بشكل atomic.**

- **`N`** — عدد الـ transfers في الـ array، لازم يكون ثابت وقت compile عشان الـ macro يحسب حجم الـ IOCTL.
- **Direction** — `_IOW` يعني userspace بيكتب البيانات للـ kernel (الـ array نفسها)، لكن الـ rx_buf جواها بيتملاه الـ kernel read من الـ SPI bus.
- **Return** — عدد الـ bytes المنقولة بنجاح، أو `errno` سالب.
- **Locking** — الـ spidev driver في الـ kernel بياخد `spi->controller->bus_lock_mutex` طول فترة تنفيذ الـ message.
- **Error paths** — `EMSGSIZE` لو `SPI_MSGSIZE(N)` رجع صفر. `EFAULT` لو أي pointer في الـ array invalid. `EIO` لو الـ controller hardware فشل.
- **الفرق عن kernel API** — معادل مباشر لـ `spi_sync()` لكن الـ pointers هنا userspace virtual addresses بتنقلها الـ kernel بـ `copy_from_user` / `copy_to_user`.

##### Pseudocode Flow (kernel-side spidev_ioctl):

```
ioctl(SPI_IOC_MESSAGE(N)):
    copy_from_user(u_tmp, user_array, N * sizeof(spi_ioc_transfer))
    spi_message_init(&msg)
    for each transfer in u_tmp:
        k_xfer[i].tx_buf = memdup_user(u_tmp[i].tx_buf) if non-null
        k_xfer[i].rx_buf = kmalloc(len)             if non-null
        k_xfer[i].len    = u_tmp[i].len
        k_xfer[i].speed_hz  = u_tmp[i].speed_hz  ?: spi->max_speed_hz
        k_xfer[i].bits_per_word = u_tmp[i].bits_per_word ?: spi->bits_per_word
        k_xfer[i].cs_change = u_tmp[i].cs_change
        spi_message_add_tail(&k_xfer[i], &msg)
    spi_sync(spi, &msg)                          /* blocks until done */
    for each rx transfer:
        copy_to_user(u_tmp[i].rx_buf, k_xfer[i].rx_buf, len)
    return total_bytes
```

---

### Group 2: Device Configuration IOCTLs — Read/Write Settings

الـ group ده بيوفر control على الـ default settings للـ SPI device. الـ settings دي بتتطبق على كل transfer ما لمش في الـ `spi_ioc_transfer` نفسه override.

---

#### `SPI_IOC_RD_MODE` / `SPI_IOC_WR_MODE`

```c
#define SPI_IOC_RD_MODE   _IOR(SPI_IOC_MAGIC, 1, __u8)
#define SPI_IOC_WR_MODE   _IOW(SPI_IOC_MAGIC, 1, __u8)
```

```c
__u8 mode;

/* read current mode */
ioctl(fd, SPI_IOC_RD_MODE, &mode);

/* set mode 3 with CS active-high */
mode = SPI_MODE_3 | SPI_CS_HIGH;
ioctl(fd, SPI_IOC_WR_MODE, &mode);
```

**بيقرأ أو بيكتب الـ SPI mode الخاص بالـ device — الـ 8 bits السفلى بس من حقل الـ mode.**

- **`__u8 *`** — pointer لـ byte واحدة تحمل الـ mode flags (combination من `SPI_CPHA`, `SPI_CPOL`, وغيرهم).
- **Return** — صفر للنجاح، سالب للـ error.
- **القيود** — الـ Write version بيعمل `spi_setup()` في الـ kernel، وده ممكن يفشل لو الـ controller مش بيدعم الـ mode المطلوب (بيرجع `EINVAL`).
- **8-bit limitation** — بيدعم فقط الـ flags من bit 0 لـ 7، يعني مش بيدعم `SPI_TX_DUAL/QUAD` وغيرهم — استخدم `SPI_IOC_RD/WR_MODE32` للـ extended flags.

---

#### `SPI_IOC_RD_MODE32` / `SPI_IOC_WR_MODE32`

```c
#define SPI_IOC_RD_MODE32   _IOR(SPI_IOC_MAGIC, 5, __u32)
#define SPI_IOC_WR_MODE32   _IOW(SPI_IOC_MAGIC, 5, __u32)
```

```c
__u32 mode;

/* enable quad TX */
ioctl(fd, SPI_IOC_RD_MODE32, &mode);
mode |= SPI_TX_QUAD;
ioctl(fd, SPI_IOC_WR_MODE32, &mode);
```

**النسخة الـ 32-bit من الـ mode IOCTL — بيدعم كل الـ flags المعرفة في `SPI_MODE_USER_MASK` (bits 0..18).**

- **`__u32 *`** — pointer لـ 32-bit word يحمل كل الـ mode flags.
- **Return** — صفر للنجاح.
- **الفرق عن `_MODE`** — بيقدر يتعامل مع multi-wire modes زي `SPI_TX_DUAL`, `SPI_TX_QUAD`, `SPI_TX_OCTAL`, `SPI_RX_*`, `SPI_CS_WORD`, `SPI_MOSI_IDLE_*`.
- **`SPI_MODE_USER_MASK`** — بيعمل mask على الـ bits المسموح لـ userspace يكتبها، أي bit خارج الـ mask بيتجاهل أو بيرجع `EINVAL`.

---

#### `SPI_IOC_RD_LSB_FIRST` / `SPI_IOC_WR_LSB_FIRST`

```c
#define SPI_IOC_RD_LSB_FIRST   _IOR(SPI_IOC_MAGIC, 2, __u8)
#define SPI_IOC_WR_LSB_FIRST   _IOW(SPI_IOC_MAGIC, 2, __u8)
```

```c
__u8 lsb;
lsb = 1; /* LSB first */
ioctl(fd, SPI_IOC_WR_LSB_FIRST, &lsb);
```

**بيتحكم في ترتيب الـ bits على السلك — MSB first (default) أو LSB first.**

- **`__u8 *`** — قيمة non-zero تعني LSB أول، صفر تعني MSB أول.
- **الـ kernel mapping** — بيضبط أو بيمسح `SPI_LSB_FIRST` flag في `spi->mode` ثم بيستدعي `spi_setup()`.
- **Return** — صفر للنجاح، `EINVAL` لو الـ controller مش بيدعم LSB-first.

---

#### `SPI_IOC_RD_BITS_PER_WORD` / `SPI_IOC_WR_BITS_PER_WORD`

```c
#define SPI_IOC_RD_BITS_PER_WORD   _IOR(SPI_IOC_MAGIC, 3, __u8)
#define SPI_IOC_WR_BITS_PER_WORD   _IOW(SPI_IOC_MAGIC, 3, __u8)
```

```c
__u8 bits = 16; /* 16-bit words */
ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
```

**بيضبط الـ default word size بالـ bits للـ device.**

- **`__u8 *`** — حجم الـ word من 1 لـ N bits. القيمة صفر تعني "استخدم الـ default الخاص بالـ controller" (غالباً 8 bits).
- **Override per-transfer** — ممكن تـ override الـ setting دي في أي `spi_ioc_transfer` بـ `bits_per_word` field.
- **Return** — صفر للنجاح، `EINVAL` لو الـ controller مش بيدعم الـ word size المطلوبة.

---

#### `SPI_IOC_RD_MAX_SPEED_HZ` / `SPI_IOC_WR_MAX_SPEED_HZ`

```c
#define SPI_IOC_RD_MAX_SPEED_HZ   _IOR(SPI_IOC_MAGIC, 4, __u32)
#define SPI_IOC_WR_MAX_SPEED_HZ   _IOW(SPI_IOC_MAGIC, 4, __u32)
```

```c
__u32 speed;

/* read current max speed */
ioctl(fd, SPI_IOC_RD_MAX_SPEED_HZ, &speed);
printf("current speed: %u Hz\n", speed);

/* set to 10 MHz */
speed = 10000000;
ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);
```

**بيقرأ أو بيضبط الـ maximum clock speed للـ SPI device بالـ Hz.**

- **`__u32 *`** — الـ speed بالـ Hertz. الـ actual clock ممكن يبقى أقل بعد تقريب الـ controller للـ clock divider.
- **Override per-transfer** — `speed_hz` في `spi_ioc_transfer` بيـ override القيمة دي لـ transfer واحدة.
- **Return** — صفر للنجاح. الـ kernel بيستدعي `spi_setup()` بعد الكتابة، لو فشل بيرجع `EINVAL`.
- **Real-world tip** — الـ Read بعد الـ Write ممكن ترجع قيمة مختلفة عن اللي كتبتها لأن الـ controller بيقرب لـ أقرب divisor.

---

### Group 3: الـ `struct spi_ioc_transfer` — Transfer Descriptor

الـ struct ده هو محور الـ API، كل transfer في الـ `SPI_IOC_MESSAGE` بيتوصف بـ instance منه.

---

#### `struct spi_ioc_transfer`

```c
struct spi_ioc_transfer {
    __u64  tx_buf;          /* userspace TX buffer pointer (or 0) */
    __u64  rx_buf;          /* userspace RX buffer pointer (or 0) */

    __u32  len;             /* buffer length in bytes */
    __u32  speed_hz;        /* per-transfer clock override (0 = use default) */

    __u16  delay_usecs;     /* delay after last bit before CS change */
    __u8   bits_per_word;   /* per-transfer word size override (0 = default) */
    __u8   cs_change;       /* deselect CS before next transfer? */
    __u8   tx_nbits;        /* number of wires for TX (1/2/4/8) */
    __u8   rx_nbits;        /* number of wires for RX (1/2/4/8) */
    __u8   word_delay_usecs;/* delay between words within transfer */
    __u8   pad;             /* reserved — always zero */
};
```

**الـ struct ده بيصف transfer واحدة على الـ SPI bus — بيتم تحويله مباشرة لـ `kernel spi_transfer` داخل الـ kernel.**

##### شرح الحقول

| Field | Type | الوصف |
|---|---|---|
| `tx_buf` | `__u64` | عنوان الـ userspace buffer اللي فيه البيانات للإرسال. لو صفر، الـ kernel بيبعت zeros. **لازم يبقى `__u64` مش pointer عشان يشتغل 32/64-bit compat.** |
| `rx_buf` | `__u64` | عنوان الـ userspace buffer لاستقبال البيانات. لو صفر، البيانات المستقبلة بتتحط. |
| `len` | `__u32` | حجم الـ tx و rx buffers بالـ bytes — لازم يكونوا نفس الحجم. |
| `speed_hz` | `__u32` | Override مؤقت للـ clock speed للـ transfer دي بس. صفر = استخدم الـ device default. |
| `delay_usecs` | `__u16` | وقت الانتظار بالـ microseconds بعد آخر bit قبل ما الـ CS يتغير. |
| `bits_per_word` | `__u8` | Override مؤقت للـ word size للـ transfer دي. صفر = استخدم الـ device default. |
| `cs_change` | `__u8` | لو non-zero، بيعمل deassert للـ CS بعد الـ transfer دي وقبل اللي بعدها. |
| `tx_nbits` | `__u8` | عدد الـ data lines للـ TX: 1 (single), 2 (dual), 4 (quad), 8 (octal). صفر = 1. |
| `rx_nbits` | `__u8` | عدد الـ data lines للـ RX. |
| `word_delay_usecs` | `__u8` | تأخير بين كل word وبعضه داخل نفس الـ transfer — محتاج support صريح في الـ SPI controller. |
| `pad` | `__u8` | Reserved — لازم يبقى صفر، عشان compatibility مستقبلية. |

##### نقاط مهمة

- **ABI stability** — الـ struct layout متطابق في 32-bit و64-bit userspace عشان الـ pointers اتعملوا `__u64` صريح مش `unsigned long`.
- **Zero-initialize دايماً** — أي field جديد في الـ kernel هيفترض قيمة صفر تعني "default"، الـ garbage values ممكن تسبب behavior غير متوقع.
- **Half-duplex** — لو `tx_buf` بس non-null والـ `rx_buf` صفر، الـ transfer TX-only. العكس بالعكس.
- **Full-duplex** — الاتنين non-null في نفس الوقت، الـ len نفسه للاتنين.
- **`cs_change` على آخر transfer** — لو set على آخر transfer في الـ message، الـ kernel بيعمل deassert-then-assert للـ CS (useful لـ protocols محتاجة pulse).

##### مثال full-duplex عملي (SPI flash read):

```c
struct spi_ioc_transfer xfers[2];
uint8_t cmd[4]  = { 0x03, 0x00, 0x10, 0x00 }; /* READ cmd + 3-byte address */
uint8_t data[256];

memset(xfers, 0, sizeof(xfers));

/* Transfer 0: send READ command */
xfers[0].tx_buf   = (unsigned long)cmd;
xfers[0].len      = 4;
xfers[0].cs_change = 0; /* keep CS asserted */

/* Transfer 1: clock out 256 bytes of data */
xfers[1].rx_buf   = (unsigned long)data;
xfers[1].len      = 256;

int ret = ioctl(fd, SPI_IOC_MESSAGE(2), xfers);
if (ret < 0)
    perror("SPI transfer failed");
```

---

### Group 4: الـ Magic Number و IOCTL Construction

#### `SPI_IOC_MAGIC`

```c
#define SPI_IOC_MAGIC 'k'
```

**الـ magic byte اللي بيميز كل IOCTLs الخاصة بالـ spidev driver عن IOCTLs الـ subsystems التانية.**

- قيمته `'k'` = `0x6B`.
- بيدخل كـ `type` في `_IOR`/`_IOW` macros اللي بتبني رقم الـ IOCTL.
- الـ kernel بيـ dispatch الـ ioctl calls على base الـ magic number والـ number field.

##### كيف بيتبني رقم الـ IOCTL

```
 31      30-29   28-16    15-8    7-0
 ┌──────┬───────┬────────┬───────┬──────┐
 │unused│ dir   │  size  │ type  │  nr  │
 └──────┴───────┴────────┴───────┴──────┘
    dir: 00=none, 10=write, 01=read, 11=read+write
   size: sizeof(data type) — من _IOC_SIZEBITS bits
   type: SPI_IOC_MAGIC = 'k' = 0x6B
     nr: رقم الـ command (1..5 للـ settings، 0 للـ message)
```

مثال — `SPI_IOC_WR_MAX_SPEED_HZ`:
```
_IOW('k', 4, __u32)
→ dir=write(10), size=4 bytes, type=0x6B, nr=4
→ 0x40046B04
```

---

### نقاط Integration مهمة

**الـ spidev driver في الـ kernel** (`drivers/spi/spidev.c`) هو اللي بيـ implement كل الـ IOCTLs دي في `spidev_ioctl()`. الـ flow العام:

```
userspace ioctl()
    → VFS → char device → spidev_ioctl()
        → SPI_IOC_MESSAGE:  spi_sync()  ← atomic, blocks
        → SPI_IOC_WR_*:     spi_setup() ← updates spi->mode/speed/bits
        → SPI_IOC_RD_*:     read from spi->mode/max_speed_hz/bits_per_word
```

**الـ concurrency** — الـ spidev driver بيستخدم `mutex` داخلي لكل open instance، بس الـ SPI controller lock هو الـ serialization الحقيقي على مستوى الـ bus.

**الـ 32-bit compat** — الـ kernel عنده `spidev_compat_ioctl()` بيتعامل مع الـ struct بشكل صحيح لأن الـ `tx_buf`/`rx_buf` `__u64` ثابتين في الاتنين، ما فيش thunk خاص محتاج.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs entries

**الـ spidev** بيستخدم الـ SPI subsystem، والـ debugfs بتاعته موجود تحت `/sys/kernel/debug/spi/`.

```bash
# Mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل الـ SPI controllers المتاحة
ls /sys/kernel/debug/spi/

# اقرأ stats الخاص بـ controller معين (مثلاً spi0)
cat /sys/kernel/debug/spi/spi0/statistics
```

**الـ entries المهمة:**

| Entry | المعنى |
|-------|---------|
| `/sys/kernel/debug/spi/spiX/statistics` | عدد الـ transfers، الـ errors، الـ bytes المرسلة/المستقبلة |
| `/sys/kernel/debug/spi/spiX.Y/` | device-specific stats لكل slave على الـ bus |

```bash
# مثال: اقرأ stats لـ spi0 slave 0
cat /sys/kernel/debug/spi/spi0.0/statistics
# Output مثال:
# transfers:    1042
# errors:       3
# timedout:     0
# bytes_tx:     8336
# bytes_rx:     8336
```

---

#### 2. sysfs entries

```bash
# اعرض كل الـ SPI devices المسجلة في النظام
ls /sys/bus/spi/devices/

# اقرأ modalias لمعرفة driver المرتبط بالـ device
cat /sys/bus/spi/devices/spi0.0/modalias
# Output: spi:spidev  (أو اسم الـ driver المناسب)

# اعرض الـ driver المستخدم
ls -la /sys/bus/spi/devices/spi0.0/driver

# تحقق من الـ /dev node المرتبط
ls -la /dev/spidev*
# Output: crw-rw---- 1 root spi 153, 0 Feb 23 10:00 /dev/spidev0.0

# اقرأ معلومات الـ SPI controller
cat /sys/class/spi_master/spi0/statistics/bytes_transferred
```

**الـ sysfs attributes المهمة:**

| Path | القيمة |
|------|--------|
| `/sys/bus/spi/devices/spiX.Y/modalias` | اسم الـ driver المطلوب |
| `/sys/bus/spi/devices/spiX.Y/of_node` | رابط لـ Device Tree node |
| `/sys/class/spi_master/spiX/` | attributes للـ controller |

---

#### 3. ftrace — tracepoints وإزاي تفعلها

```bash
# اعرض كل الـ SPI tracepoints المتاحة
ls /sys/kernel/debug/tracing/events/spi/

# الـ events المهمة:
# spi_controller_busy   — الـ controller بدأ شغل
# spi_controller_idle   — الـ controller خلص
# spi_message_start     — بدأ message جديد
# spi_message_done      — انتهى message
# spi_transfer_start    — بدأ transfer
# spi_transfer_stop     — انتهى transfer

# فعّل كل SPI events
echo 1 > /sys/kernel/debug/tracing/events/spi/enable

# أو فعّل event معين بس
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_transfer_start/enable
echo 1 > /sys/kernel/debug/tracing/events/spi/spi_message_done/enable

# شغّل الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# نفّذ عملية SPI من userspace (مثلاً spi-pipe أو برنامجك)
# ...

# اقرأ النتائج
cat /sys/kernel/debug/tracing/trace

# مثال output:
# spi-test-1234  [000] .... 123.456789: spi_transfer_start: spi0.0 len=8
# spi-test-1234  [000] .... 123.456901: spi_transfer_stop:  spi0.0 len=8 status=0

# وقف الـ trace وامسح
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
```

---

#### 4. printk و dynamic debug

**تفعيل الـ dynamic debug للـ spidev و SPI core:**

```bash
# فعّل debug messages للـ spidev driver
echo "file drivers/spi/spidev.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل debug messages للـ SPI core كله
echo "file drivers/spi/spi.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل debug messages لكل الـ SPI subsystem
echo "module spi +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق إيه اللي اتفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep spi

# اقرأ الـ kernel log في real-time
dmesg -w | grep -i spi

# أو من journald
journalctl -k -f | grep -i spi
```

**لو محتاج printk level أعلى:**

```bash
# اضبط log level عشان تشوف كل الـ debug messages
echo 8 > /proc/sys/kernel/printk
# Format: console_loglevel default_msg_loglevel min_console_loglevel default_console_loglevel
```

---

#### 5. Kernel Config options للـ debugging

| Config Option | الوظيفة |
|---------------|---------|
| `CONFIG_SPI_DEBUG` | يفعّل الـ verbose debug messages في الـ SPI core |
| `CONFIG_DEBUG_SPI_STUB` | stub driver للتجربة بدون hardware |
| `CONFIG_SPI_SPIDEV` | لازم يكون enabled عشان `/dev/spidevX.Y` يظهر |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug()` و `dev_dbg()` dynamically |
| `CONFIG_FTRACE` | لازم للـ tracepoints |
| `CONFIG_TRACING` | الـ tracing infrastructure |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في الـ SPI bus locking |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف spinlock misuse في الـ SPI controller drivers |
| `CONFIG_PROVE_LOCKING` | يتحقق من صحة الـ locking order |
| `CONFIG_KASAN` | يكتشف memory errors في الـ transfer buffers |

```bash
# تحقق إيه الـ configs المفعّلة في الكرنل الحالي
zcat /proc/config.gz | grep -E "CONFIG_SPI|CONFIG_DEBUG_SPI"
# أو
grep -E "CONFIG_SPI|CONFIG_DEBUG_SPI" /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ SPI Subsystem

**spi-tools / spidev utilities:**

```bash
# تثبيت الأدوات
apt install spi-tools  # Debian/Ubuntu

# اقرأ معلومات الـ SPI device
spi-config -d /dev/spidev0.0 -q
# Output:
# /dev/spidev0.0: mode=0, lsb=0, bits=8, speed=500000, spiready=0

# اضبط الـ mode والـ speed
spi-config -d /dev/spidev0.0 --mode=0 --speed=1000000 --bits=8

# اعمل loopback test (MOSI متوصل بـ MISO)
spidev_test -D /dev/spidev0.0 -v -l
# -v: verbose, -l: loopback

# أرسل bytes معينة
spidev_test -D /dev/spidev0.0 -p "hello"

# استخدم spi-pipe لإرسال/استقبال بيانات
echo -ne "\x9F\x00\x00\x00" | spi-pipe -d /dev/spidev0.0 -s 1000000 | xxd
```

**Python للـ rapid prototyping والـ debugging:**

```python
import spidev
import time

spi = spidev.SpiDev()
spi.open(0, 0)  # bus=0, device=0

# اقرأ الـ JEDEC ID من flash chip كمثال
spi.max_speed_hz = 1000000
spi.mode = 0

response = spi.xfer2([0x9F, 0x00, 0x00, 0x00])
print(f"JEDEC ID: {[hex(b) for b in response]}")
# Output: JEDEC ID: ['0x0', 'xef', '0x40', '0x18']  (Winbond W25Q128)

spi.close()
```

---

#### 7. جدول أخطاء شائعة

| رسالة الخطأ | المعنى | الحل |
|-------------|--------|-------|
| `spidev: SPI_IOC_MESSAGE: 22` (EINVAL) | حجم الـ transfer أكبر من `bufsiz` أو `len=0` | تقليل `len` أو زيادة `bufsiz` module parameter |
| `spidev: copy_from_user: Bad address` | الـ tx_buf/rx_buf pointer غلط في userspace | تحقق من الـ buffer allocation وصحة الـ pointer |
| `spi_master spi0: transfer failed` | الـ controller فشل في إتمام الـ transfer | تحقق من الـ hardware connections والـ clock speed |
| `spidev: can't setup spi0.0` | فشل في إعداد الـ SPI device أثناء probe | تحقق من الـ Device Tree وصحة الـ SPI parameters |
| `spi0.0: timeout waiting for completion` | الـ transfer لم يكتمل في الوقت المحدد | تحقق من الـ hardware، قلل الـ speed، تحقق من الـ CS |
| `spidev spi0.0: GPIO chip select not found` | الـ GPIO المحدد للـ CS مش موجود | تحقق من الـ Device Tree pinmux وتعريف الـ GPIO |
| `bad ioctl (0xXX)` | الـ IOCTL command غير معروف | تحقق إن قيمة الـ magic `'k'` والـ number صح |
| `spidev: fifo overrun` | بيانات اتفقدت بسبب سرعة عالية | قلل الـ speed أو فعّل DMA |
| `-ENOMEM` عند فتح الـ device | مفيش ذاكرة كافية لـ buffers | راقب الـ memory أو قلل الـ bufsiz |
| `modalias mismatch` | اسم الـ driver في DT مش `spidev` | غيّر `compatible = "linux,spidev"` في الـ DT |

---

#### 8. أماكن مناسبة لـ dump_stack() و WARN_ON()

```c
/* في drivers/spi/spidev.c — أماكن استراتيجية للـ debugging */

/* 1. تحقق من صحة الـ ioctl transfer قبل التنفيذ */
static int spidev_message(struct spidev_data *spidev,
                          struct spi_ioc_transfer *u_xfers,
                          unsigned n_xfers)
{
    /* تحقق إن الـ len مش zero */
    WARN_ON(n_xfers == 0);

    /* تحقق إن الـ speed مش أعلى من الـ max */
    WARN_ON(u_xfers->speed_hz > spidev->spi->max_speed_hz);
    ...
}

/* 2. تحقق من الـ CS state بعد كل transfer */
static void spidev_complete(void *arg)
{
    /* لو الـ completion جاء بدون active transfer — مشكلة */
    WARN_ON(!spidev->tx_buffer && !spidev->rx_buffer);
    complete(arg);
}

/* 3. في الـ open() لو أكثر من الـ limit فتحوا الـ device */
static int spidev_open(struct inode *inode, struct file *filp)
{
    if (spidev->users > MAX_SPI_USERS) {
        WARN_ON(1);  /* يطبع stack trace */
        return -EBUSY;
    }
    ...
}

/* 4. تحقق من alignment للـ DMA buffers */
WARN_ON((unsigned long)spidev->tx_buffer & (dma_get_cache_alignment() - 1));
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State موافق الـ Kernel State

```bash
# اقرأ الـ mode الحالي من kernel
spi-config -d /dev/spidev0.0 -q
# Output: mode=0, lsb=0, bits=8, speed=1000000

# قارن مع ما تشوفه على الـ oscilloscope:
# - Mode 0: CPOL=0 (clock idle low), CPHA=0 (sample on rising edge)
# - Mode 1: CPOL=0, CPHA=1 (sample on falling edge)
# - Mode 2: CPOL=1 (clock idle high), CPHA=0 (sample on falling)
# - Mode 3: CPOL=1, CPHA=1 (sample on rising edge)

# تحقق من الـ speed الفعلية على الـ bus
# (kernel بيقرّب لأقرب قيمة تدعمها الـ hardware)
dmesg | grep "spi0: registered master"
# Output: spi0: registered master, speed_hz=25000000 max

# تحقق من chipselect polarity
cat /sys/bus/spi/devices/spi0.0/of_node/spi-cs-high
# لو الملف موجود: CS active high
# لو مش موجود: CS active low (الـ default)
```

---

#### 2. Register Dump Techniques

```bash
# devmem2 — اقرأ/اكتب registers مباشرة (تحتاج root)
# مثال: اقرأ status register لـ Raspberry Pi SPI controller
# Base address لـ BCM2835 SPI0 = 0x3F204000

# Read CS register (offset 0x00)
devmem2 0x3F204000 w
# Output: Value at address 0x3F204000 (0xb6620000): 0x00040000

# Read FIFO register (offset 0x04)
devmem2 0x3F204004 w

# Read CLK register (offset 0x08)
devmem2 0x3F204008 w

# استخدام /dev/mem مباشرة مع Python
python3 -c "
import mmap, os

# BCM2835 SPI0 base
SPI_BASE = 0x3F204000
PAGE_SIZE = 4096

mem = open('/dev/mem', 'rb')
spi_map = mmap.mmap(mem.fileno(), PAGE_SIZE,
                    mmap.MAP_SHARED, mmap.PROT_READ,
                    offset=SPI_BASE)
import struct
cs_reg = struct.unpack('<I', spi_map[0:4])[0]
clk_reg = struct.unpack('<I', spi_map[8:12])[0]
print(f'CS reg:  0x{cs_reg:08x}')
print(f'CLK reg: 0x{clk_reg:08x}')
spi_map.close()
mem.close()
"

# استخدام io utility لو متاح
io -4 -r 0x3F204000  # قرأ 4 bytes
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس:**

```
SPI Bus Signals:
                    ┌─────────────────────────────────┐
  SCLK (SCK) ───────┤  CLK  │  قيس هنا: frequency, duty cycle, idle state
  MOSI       ───────┤  DATA │  قيس هنا: bit pattern, alignment
  MISO       ───────┤  DATA │  قيس هنا: response timing
  CS (SS)    ───────┤  EN   │  قيس هنا: pulse width, polarity
                    └─────────────────────────────────┘

  ┌─── CS low (active) ───────────────────────────────┐
  │                                                   │
CS: ─┘                                               └─
         ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
CLK:  ───┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─
MOSI: ─[D7][D6][D5][D4][D3][D2][D1][D0]──
```

**إعدادات الـ Logic Analyzer:**

```
- Sample rate: على الأقل 10x الـ SPI clock speed
  مثال: لـ 1MHz SPI → sample at 10MHz
- Trigger: على الـ CS falling edge (active low)
- Decode: اختر "SPI" protocol
  - CPOL وCPHA حسب الـ mode المستخدم
  - bit order: MSB first (default)
  - word size: 8 bits (default)
```

**أشياء تبحث عنها:**

```
1. CS glitches — الـ CS بيتفعّل وبيتعطّل بشكل غير متوقع
   → في الـ kernel log: تكرار الـ transfers بشكل غير متوقع

2. Clock stretching — الـ CLK بيتوقف في النص
   → في الـ kernel log: timeout messages

3. MISO floating — الـ MISO مش موصل بشكل صح
   → في الـ kernel log: قيم عشوائية في الـ rx_buf

4. متأخرة في الـ MISO response — الـ slave بطيء
   → في الـ kernel log: data mismatch في loopback test
```

---

#### 4. Common Hardware Issues وـ Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ kernel log | التشخيص |
|----------------------|--------------------------|----------|
| مفيش voltage على VCC للـ slave | `spi0.0: timeout` | افحص الـ power rails |
| الـ SCLK مش بيوصل | `spi_transfer_stop: status=-110` (ETIMEDOUT) | افحص الـ pinmux في DT |
| الـ CS دايماً high (مش بيتفعّل) | الـ device مش بيرد، `rx_buf` كله 0xFF | افحص GPIO polarity |
| الـ MOSI وMISO متعكسين | loopback data مش matching | اعكس الـ wiring |
| impedance mismatch في speeds عالية | بيانات غلط عند speed عالية، صح عند منخفضة | حط termination resistor، قلل الـ speed |
| الـ GND مش متوصل | إشارات unstable | وصّل الـ GND |
| الـ clock too fast للـ slave | data errors عند speed عالية | قلل `max-frequency` في DT |

---

#### 5. Device Tree Debugging

```bash
# اقرأ الـ DT المُطبَّق على الـ hardware الحالي
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "spi@"

# تحقق من الـ spidev node في الـ DT
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 10 "spidev"

# مثال DT node صح:
# &spi0 {
#     status = "okay";
#     cs-gpios = <&gpio 8 GPIO_ACTIVE_LOW>;
#
#     spidev@0 {
#         compatible = "linux,spidev";
#         reg = <0>;                    /* CS0 */
#         spi-max-frequency = <1000000>;
#     };
# };

# تحقق من الـ pinctrl (الـ pins مضبوطة صح؟)
cat /sys/kernel/debug/pinctrl/pinctrl-handles

# أو باستخدام device tree compiler
# قارن الـ compiled DTB مع الـ source DTS
dtc -I dtb -O dts /boot/dtb/your-board.dtb > /tmp/current.dts
diff your-original.dts /tmp/current.dts

# تحقق من الـ of_node للـ device
ls -la /sys/bus/spi/devices/spi0.0/of_node/
# Output يجب يظهر: compatible, reg, spi-max-frequency

# اقرأ قيم الـ DT للـ device مباشرة
cat /sys/bus/spi/devices/spi0.0/of_node/compatible
# Output: linux,spidev

# تحقق من الـ clocks المربوطة بالـ SPI controller
cat /sys/kernel/debug/clk/spi0/clk_rate
# Output: 25000000  (25MHz)
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

```bash
# ============================================================
# 1. تحقق سريع إن الـ spidev متاح
# ============================================================
ls -la /dev/spidev* && echo "spidev OK" || echo "spidev NOT found"

# ============================================================
# 2. اقرأ configuration الحالية للـ device
# ============================================================
spi-config -d /dev/spidev0.0 -q

# ============================================================
# 3. اعمل loopback test (وصّل MOSI بـ MISO أولاً)
# ============================================================
spidev_test -D /dev/spidev0.0 -v -l -p "Hello SPI"
# Expected output:
# TX: 48 65 6C 6C 6F 20 53 50 49
# RX: 48 65 6C 6C 6F 20 53 50 49  ← نفس البيانات لو loopback

# ============================================================
# 4. فعّل كل SPI debug messages
# ============================================================
echo "module spi +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/spi/spidev.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg -w &

# ============================================================
# 5. تتبع كل SPI transfers مع ftrace
# ============================================================
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo 1 > /sys/kernel/debug/tracing/events/spi/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل التطبيق ...
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# ============================================================
# 6. قياس latency للـ SPI transfers
# ============================================================
# باستخدام perf
perf trace -e spi:spi_transfer_start,spi:spi_transfer_stop \
    spidev_test -D /dev/spidev0.0 -p "test"
# Output مثال:
# 0.000 spi:spi_transfer_start(spi0.0 len=4)
# 0.052 spi:spi_transfer_stop(spi0.0 len=4 status=0)
# Transfer time: 52 microseconds

# ============================================================
# 7. اكتشف الـ ioctl numbers المستخدمة
# ============================================================
strace -e ioctl spidev_test -D /dev/spidev0.0 -p "test" 2>&1 | grep ioctl
# Output مثال:
# ioctl(3, SPI_IOC_WR_MODE, [0])       = 0
# ioctl(3, SPI_IOC_WR_BITS_PER_WORD, [8]) = 0
# ioctl(3, SPI_IOC_WR_MAX_SPEED_HZ, [500000]) = 0
# ioctl(3, SPI_IOC_MESSAGE(1), 0x...) = 1

# ============================================================
# 8. تحقق من الـ SPI statistics
# ============================================================
watch -n 1 'cat /sys/kernel/debug/spi/spi0/statistics 2>/dev/null || \
            cat /sys/class/spi_master/spi0/statistics/* 2>/dev/null'

# ============================================================
# 9. فحص الـ DT للـ SPI nodes
# ============================================================
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    grep -B2 -A15 "compatible.*spidev"

# ============================================================
# 10. اكتشف الـ bufsiz المستخدم (default 4096 bytes)
# ============================================================
dmesg | grep -i spidev | grep bufsiz
# أو
cat /sys/module/spidev/parameters/bufsiz 2>/dev/null || \
    grep bufsiz /sys/module/spidev/parameters/ 2>/dev/null
# لو محتاج تزود الـ buffer size:
# modprobe spidev bufsiz=65536

# ============================================================
# 11. فحص الـ SPI mode flags
# ============================================================
python3 - << 'EOF'
import fcntl, struct

SPI_IOC_MAGIC = ord('k')
SPI_IOC_RD_MODE32 = (2 << 30) | (SPI_IOC_MAGIC << 8) | (5 << 0) | (4 << 16)

with open('/dev/spidev0.0', 'rb') as f:
    buf = struct.pack('I', 0)
    result = fcntl.ioctl(f, SPI_IOC_RD_MODE32, bytearray(4))
    mode = struct.unpack('I', result)[0]
    print(f"Mode32: 0x{mode:08x}")
    flags = {
        0x001: 'CPHA', 0x002: 'CPOL', 0x004: 'CS_HIGH',
        0x008: 'LSB_FIRST', 0x010: '3WIRE', 0x020: 'LOOP',
        0x040: 'NO_CS', 0x080: 'READY', 0x100: 'TX_DUAL',
        0x200: 'TX_QUAD', 0x400: 'RX_DUAL', 0x800: 'RX_QUAD'
    }
    active = [name for bit, name in flags.items() if mode & bit]
    print(f"Active flags: {active if active else ['MODE_0 (no flags)']}")
EOF
# Output مثال:
# Mode32: 0x00000000
# Active flags: ['MODE_0 (no flags)']
```

---

#### تفسير الـ Output

```
# ftrace output تفسيره:
#
#   process-PID   [CPU]  timestamp    event: details
#    spi-test-42  [000] .... 1234.567890: spi_message_start: spi0.0
#    spi-test-42  [000] .... 1234.567901: spi_transfer_start: spi0.0 len=8
#    spi-test-42  [000] .... 1234.567953: spi_transfer_stop:  spi0.0 len=8 status=0
#    spi-test-42  [000] .... 1234.567960: spi_message_done:  spi0.0 status=0 bytes=8
#
#   الـ timestamp بالثواني، الـ status=0 معناه نجاح
#   الفرق بين transfer_start وtransfer_stop = 52µs = وقت الـ transfer الفعلي
#
# strace ioctl output تفسيره:
#
#   ioctl(fd, SPI_IOC_MESSAGE(1), ptr) = N
#   N = عدد الـ bytes المنقولة لو نجح، أو -1 لو فشل
#   errno بيديك سبب الفشل: EINVAL=22, ENOMEM=12, EFAULT=14
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — بيانات اتلت بسبب غلط في SPI Mode

#### العنوان
**Corrupted SPI data على industrial gateway بسبب wrong CPOL/CPHA configuration**

#### السياق
شركة بتعمل industrial IoT gateway بالـ **TI AM62x** SoC. الـ gateway بتتصل بـ **SPI temperature sensor** من نوع MAX31855. المنتج اتشحن وبدأ الـ field engineers يشتكوا إن قراءات الحرارة عشوائية تماماً.

#### المشكلة
الـ userspace application بتفتح `/dev/spidev1.0` وبتقرأ من الـ sensor. الـ readings مش بس غلط — في حالات بتجي negative temperatures في بيئة حرارتها 80°C.

#### التحليل
الـ MAX31855 بيشتغل على **SPI_MODE_1** (CPOL=0, CPHA=1). الـ developer حط في الكود:

```c
uint8_t mode = SPI_MODE_0; /* wrong! should be SPI_MODE_1 */
ioctl(fd, SPI_IOC_WR_MODE, &mode);
```

الـ `SPI_MODE_0` معناه `CPOL=0, CPHA=0` — اللي بيخلي الـ master يسمبل الـ data على الـ rising edge، بس الـ MAX31855 بيطلعها على الـ rising edge كمان، يعني الـ master بيسمبل الـ data قبل ما تتستقر. النتيجة: **bit shifting** في الـ 32-bit frame، والـ sign bit بيتقرأ غلط.

من `spidev.h`:
```c
/* Read / Write of SPI mode (SPI_MODE_0..SPI_MODE_3) (limited to 8 bits) */
#define SPI_IOC_RD_MODE   _IOR(SPI_IOC_MAGIC, 1, __u8)
#define SPI_IOC_WR_MODE   _IOW(SPI_IOC_MAGIC, 1, __u8)
```

الـ `ioctl` ده بيكتب byte واحد — القيمة `SPI_CPOL | SPI_CPHA` من `spi.h`. المشكلة إن الـ developer اعتقد إن الـ sensor بيشتغل على MODE_0 لأن الـ datasheet الـ diagram بيبدو كده بدون قراءة دقيقة للـ CPHA definition.

**Debug command:**
```bash
# اقرأ الـ mode الحالي
python3 -c "
import fcntl, struct
fd = open('/dev/spidev1.0', 'rb+', buffering=0)
buf = bytearray(1)
fcntl.ioctl(fd, 0x80016b01, buf)  # SPI_IOC_RD_MODE
print('Current mode:', buf[0])
"
```

#### الحل
```c
#include <linux/spi/spidev.h>
#include <linux/spi/spi.h>

int fd = open("/dev/spidev1.0", O_RDWR);

/* MAX31855 requires MODE_1: CPOL=0, CPHA=1 */
uint8_t mode = SPI_MODE_1;
ioctl(fd, SPI_IOC_WR_MODE, &mode);

/* verify */
uint8_t actual_mode;
ioctl(fd, SPI_IOC_RD_MODE, &actual_mode);
assert(actual_mode == SPI_MODE_1);
```

أو من الـ Device Tree:
```dts
&spi1 {
    max31855@0 {
        compatible = "maxim,max31855";
        reg = <0>;
        spi-max-frequency = <5000000>;
        spi-cpol;   /* CPOL=0 → remove this */
        spi-cpha;   /* CPHA=1 → keep this = MODE_1 */
    };
};
```

#### الدرس المستفاد
الـ `SPI_IOC_WR_MODE` بيكتب `__u8` واحد بس — لو الـ sensor محتاج flags زيادة (زي `SPI_CS_HIGH`) لازم تستخدم `SPI_IOC_WR_MODE32` اللي بيكتب `__u32`. دايما راجع الـ datasheet للـ CPHA definition — بعض الـ vendors بيقولوا "data sampled on rising edge" وده ممكن يبقى MODE_0 أو MODE_1 حسب الـ CPOL.

---

### السيناريو 2: Android TV Box على Allwinner H616 — SPI Flash تقيل بسبب wrong speed

#### العنوان
**SPI NOR Flash read latency عالية جداً على Android TV box بسبب محدودية الـ speed_hz**

#### السياق
منتج Android TV box على **Allwinner H616**. الـ SPI NOR flash (W25Q128) موصولة على `spidev0.0`. الـ application بتعمل firmware OTA update عن طريق userspace spidev. المشكلة: الـ flash write بتاخد 45 دقيقة بدل 3 دقيقة.

#### المشكلة
الـ developer بيستخدم `SPI_IOC_MESSAGE` مع هيكل `spi_ioc_transfer` من غير ما يحدد الـ `speed_hz`:

```c
struct spi_ioc_transfer tr = {
    .tx_buf = (unsigned long)tx,
    .rx_buf = (unsigned long)rx,
    .len    = 256,
    /* speed_hz not set → stays 0 */
};
ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
```

#### التحليل
من `spidev.h`:
```c
struct spi_ioc_transfer {
    __u64   tx_buf;
    __u64   rx_buf;
    __u32   len;
    __u32   speed_hz;     /* ← لو 0 بيستخدم الـ default من الـ device */
    __u16   delay_usecs;
    __u8    bits_per_word;
    __u8    cs_change;
    ...
};
```

لما `speed_hz = 0`، الـ kernel بيرجع للـ `spi_device->max_speed_hz` اللي اتعملت configure في الـ Device Tree. على الـ H616 board ده، الـ DT مفيهاش `spi-max-frequency` صريح، فالـ kernel بيستخدم قيمة conservative جداً — **400 kHz** بدل الـ 80 MHz اللي الـ W25Q128 يقدر يشتغل بيها.

الـ flash size 16 MB. بـ 400 kHz وبأخذ في الاعتبار الـ protocol overhead:
- زمن النقل = (16 * 1024 * 1024 * 8) / 400,000 ≈ 335 ثانية ≈ 5.6 دقيقة (للـ read فقط)
- بالـ erase cycles والـ page programming → 45 دقيقة منطقية

#### الحل
```c
struct spi_ioc_transfer tr = {
    .tx_buf      = (unsigned long)tx,
    .rx_buf      = (unsigned long)rx,
    .len         = 256,
    .speed_hz    = 50000000,  /* 50 MHz — safe for W25Q128 over PCB trace */
    .bits_per_word = 8,
    .delay_usecs = 0,
};

/* set device-level max speed too */
uint32_t speed = 50000000;
ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
```

الـ `speed_hz` في `spi_ioc_transfer` ده **per-transfer override** — بيغير الـ speed لهذا الـ transfer بس، مش بيعدل الـ device default. لو عايز تغير الـ default، استخدم `SPI_IOC_WR_MAX_SPEED_HZ`.

**Verify:**
```bash
# اقرأ الـ speed الحالي
python3 -c "
import fcntl, struct
fd = open('/dev/spidev0.0', 'rb+', buffering=0)
buf = bytearray(4)
fcntl.ioctl(fd, 0x80046b04, buf)  # SPI_IOC_RD_MAX_SPEED_HZ
print('Speed:', struct.unpack('<I', buf)[0], 'Hz')
"
```

#### الدرس المستفاد
الـ `speed_hz = 0` في `spi_ioc_transfer` مش معناه "أسرع ما يمكن" — معناه "استخدم الـ device default". دايما set الـ `speed_hz` explicitly في الـ production code، وراجع الـ DT إن الـ `spi-max-frequency` مضبوط صح.

---

### السيناريو 3: i.MX8 Automotive ECU — cs_change بيأخر الـ transaction

#### العنوان
**SPI multi-transfer transaction فاشلة على automotive ECU بسبب إساءة استخدام الـ `cs_change` field**

#### السياق
شركة automotive بتعمل ECU على **NXP i.MX8M Plus**. الـ ECU بتتكلم مع **TLE9263 SBC** (System Basis Chip) عن طريق SPI. الـ TLE9263 محتاج الـ CS يفضل active (low) طول فترة الـ transaction اللكونة من 3 transfers. بعد كل بيتا النظام بيعمل reset غير متوقع.

#### المشكلة
الـ developer عمل array من `spi_ioc_transfer` وفعّل `cs_change = 1` في كل transfer معتقداً إن ده بيعمل "sync" بين الـ transfers:

```c
struct spi_ioc_transfer mesg[3];
memset(mesg, 0, sizeof(mesg));

/* transfer 1: send command */
mesg[0].tx_buf = (unsigned long)cmd;
mesg[0].len    = 2;
mesg[0].cs_change = 1; /* WRONG: this deselects CS after transfer! */

/* transfer 2: send data */
mesg[1].tx_buf = (unsigned long)data;
mesg[1].len    = 4;
mesg[1].cs_change = 1; /* WRONG */

/* transfer 3: read status */
mesg[2].rx_buf = (unsigned long)status;
mesg[2].len    = 1;
mesg[2].cs_change = 0;

ioctl(fd, SPI_IOC_MESSAGE(3), mesg);
```

#### التحليل
من comment الـ `spidev.h`:

```c
__u8    cs_change;
/*
 * cs_change: True to deselect device before starting the next transfer.
 */
```

يعني `cs_change = 1` في `mesg[0]` بيعمل:
1. ينهي transfer[0]
2. **يرفع CS** (deselect = CS goes HIGH)
3. يبدأ transfer[1]

الـ TLE9263 بيفسر الـ CS high كـ "end of frame" → بيعتقد إن الـ command اتكمل من غير data → بيدخل في undefined state → الـ SBC بيعمل watchdog reset للـ system.

الـ `SPI_IOC_MESSAGE(3)` بيعمل الـ 3 transfers كـ atomic group من وجهة نظر الـ userspace، بس الـ CS toggling جوا بيتحدد بـ `cs_change`:

```c
#define SPI_IOC_MESSAGE(N) _IOW(SPI_IOC_MAGIC, 0, char[SPI_MSGSIZE(N)])
```

الـ `SPI_MSGSIZE(N)` بيحسب total size:
```c
#define SPI_MSGSIZE(N) \
    ((((N)*(sizeof(struct spi_ioc_transfer))) < (1 << _IOC_SIZEBITS)) \
        ? ((N)*(sizeof(struct spi_ioc_transfer))) : 0)
```

لو الـ N كبير جداً وحجم الـ array تعدى `(1 << _IOC_SIZEBITS)` = 16384 bytes، الـ macro بيرجع 0 والـ ioctl بيفشل. ده edge case مهم للـ bulk transfers.

#### الحل
```c
struct spi_ioc_transfer mesg[3];
memset(mesg, 0, sizeof(mesg)); /* zero-init including cs_change */

/* CS stays LOW across all 3 transfers */
mesg[0].tx_buf = (unsigned long)cmd;
mesg[0].len    = 2;
mesg[0].cs_change = 0; /* keep CS active */

mesg[1].tx_buf = (unsigned long)data;
mesg[1].len    = 4;
mesg[1].cs_change = 0; /* keep CS active */

mesg[2].rx_buf = (unsigned long)status;
mesg[2].len    = 1;
mesg[2].cs_change = 0; /* CS releases after this (last transfer) */

int ret = ioctl(fd, SPI_IOC_MESSAGE(3), mesg);
if (ret < 0) perror("SPI_IOC_MESSAGE failed");
```

لو محتاج CS يتحرر بعد transfer معين ثم يرجع، ابقى فهم إن ده ممكن يبقى by design (زي EEPROM اللي بتحتاج CS toggle بين write enable وwrite command).

**Debug:**
```bash
# scope على CS line أو استخدم logic analyzer
# تأكد إن CS بيفضل low طول الـ 3 transfers
```

#### الدرس المستفاد
الـ `cs_change = 1` معناه "deselect بعد الـ transfer ده" — مش "sync point". الـ `memset` لـ 0 على الـ struct ضروري لأن `cs_change = 0` هو الـ default المطلوب في معظم الحالات. دايما اقرأ الـ comment في `spidev.h` بتاع كل field قبل ما تحط قيمة.

---

### السيناريو 4: STM32MP1 IoT Sensor — 32-bit mode مش شغال مع flags متقدمة

#### العنوان
**SPI_CS_HIGH مش بيتفعل على IoT sensor board بسبب استخدام `SPI_IOC_WR_MODE` بدل `SPI_IOC_WR_MODE32`**

#### السياق
board بـ **STM32MP1** بتتصل بـ **ADXL362 accelerometer** اللي بـ active-high chip select (نادر بس موجود في بعض الـ variants). الـ CS مش بيشتغل صح والـ sensor مش بيستجاوب.

#### المشكلة
الـ developer بيحاول يفعل `SPI_CS_HIGH`:

```c
uint8_t mode = SPI_MODE_0 | SPI_CS_HIGH;
ioctl(fd, SPI_IOC_WR_MODE, &mode);
```

الكود بيرن من غير error، بس الـ CS ما اتغيرش.

#### التحليل
`SPI_IOC_WR_MODE` معرف كـ:
```c
#define SPI_IOC_WR_MODE   _IOW(SPI_IOC_MAGIC, 1, __u8)
```

بياخد `__u8` — 8 bits فقط. بص على `spi.h`:
```c
#define SPI_CPHA    _BITUL(0)
#define SPI_CPOL    _BITUL(1)
#define SPI_CS_HIGH _BITUL(2)   /* bit 2 → value = 4 */
#define SPI_LSB_FIRST _BITUL(3)
```

الـ `SPI_CS_HIGH` = `_BITUL(2)` = 4. ده بيعدي في الـ 8 bits بسهولة. إذن المشكلة مش في الحجم.

المشكلة الحقيقية: الـ kernel في بعض النسخ بيرفض تغيير `SPI_CS_HIGH` من خلال `SPI_IOC_WR_MODE` (الـ 8-bit version) بسبب security concern — الـ CS polarity محتاجة يتحددها الـ DT أو الـ driver مش الـ userspace في runtime. الـ kernel بيعمل mask على الـ flags المسموحة.

الحل الصح هو استخدام `SPI_IOC_WR_MODE32`:
```c
#define SPI_IOC_RD_MODE32   _IOR(SPI_IOC_MAGIC, 5, __u32)
#define SPI_IOC_WR_MODE32   _IOW(SPI_IOC_MAGIC, 5, __u32)
```

ده بيدعم الـ 32-bit mode field اللي فيه كل الـ flags من `SPI_MODE_USER_MASK`:
```c
/* من spi.h */
#define SPI_MODE_USER_MASK  (_BITUL(19) - 1)  /* bits 0..18 */
```

```c
uint32_t mode32;
/* اقرأ الـ mode الحالي الأول */
ioctl(fd, SPI_IOC_RD_MODE32, &mode32);
mode32 |= SPI_CS_HIGH;
ioctl(fd, SPI_IOC_WR_MODE32, &mode32);
```

لكن حتى `SPI_IOC_WR_MODE32` ممكن يرفض `SPI_CS_HIGH` في بعض الـ kernel versions. الحل الأضمن:

```dts
/* Device Tree */
&spi3 {
    adxl362@0 {
        compatible = "adi,adxl362";
        reg = <0>;
        spi-max-frequency = <5000000>;
        spi-cs-high;  /* active-high CS */
    };
};
```

#### الحل النهائي
```bash
# تأكد إن DT فيه spi-cs-high
dtc -I dtb -O dts /boot/dtbs/stm32mp15xx-myboard.dtb | grep -A5 adxl362

# لو الـ DT صح، اتحقق من الـ hardware:
# بعض الـ STM32MP1 SPI controllers بتحتاج inversion hardware إضافية
# لأن الـ native CS دايماً active-low
```

#### الدرس المستفاد
الـ `SPI_IOC_WR_MODE` و`SPI_IOC_WR_MODE32` مش متطابقين في الـ semantics — الـ 8-bit version موجود للـ backward compatibility. الـ flags المتقدمة زي `SPI_CS_HIGH` يُفضل تتحدد في الـ Device Tree مش في الـ userspace runtime، لأن بعض الـ controllers محتاجة hardware configuration لازم تحصل في الـ probe time.

---

### السيناريو 5: RK3562 Custom Board Bring-up — SPI_IOC_MESSAGE بيفشل بـ EINVAL

#### العنوان
**`SPI_IOC_MESSAGE` بيرجع EINVAL على RK3562 custom board أثناء bring-up**

#### السياق
engineer بيعمل bring-up لـ custom board على **Rockchip RK3562** موصول بيها **ADS8688 ADC** عن طريق SPI. الـ userspace test script بيفشل فوراً.

#### المشكلة
```bash
python3 spi_test.py
# Error: [Errno 22] Invalid argument
```

#### التحليل خطوة بخطوة

**الخطوة 1 — تحقق من الـ device node:**
```bash
ls -la /dev/spidev*
# /dev/spidev2.0  → موجود، طيب
```

**الخطوة 2 — شوف الكود:**
```python
import spidev
# developer بيستخدم python-spidev مش ioctl مباشرة
# python-spidev بيبعت SPI_IOC_MESSAGE داخلياً

# الكود:
spi = spidev.SpiDev()
spi.open(2, 0)
spi.max_speed_hz = 100000000  # 100 MHz ← مشكلة!
response = spi.xfer2([0x00, 0x00, 0x00])
```

**الخطوة 3 — فهم المشكلة:**

الـ `SPI_IOC_WR_MAX_SPEED_HZ` معرف كـ:
```c
#define SPI_IOC_WR_MAX_SPEED_HZ   _IOW(SPI_IOC_MAGIC, 4, __u32)
```

الـ 100 MHz = 100,000,000 Hz. ده عدد يعدي الـ `__u32` Max؟ لا (Max = ~4.2 GHz). المشكلة في مكان تاني.

الـ RK3562 SPI controller الـ hardware بيدعم max speed معين — في الـ kernel driver الـ `master->max_speed_hz` محدود. لما الـ userspace بيطلب أكتر من اللي الـ hardware يدعمه، الـ spidev driver بيرجع `EINVAL`.

**الخطوة 4 — تشخيص دقيق:**
```bash
# شوف الـ kernel log
dmesg | grep spi
# [  12.345] spi2.0: requested speed 100000000 exceeds limit 80000000

# أو جرب سرعة أقل يدوياً
python3 -c "
import fcntl, struct
fd = open('/dev/spidev2.0', 'rb+', buffering=0)
speed = struct.pack('<I', 50000000)  # 50 MHz
fcntl.ioctl(fd, 0x40046b04, speed)  # SPI_IOC_WR_MAX_SPEED_HZ
print('OK')
"
```

**خطأ تاني محتمل — حجم الـ message:**

من `spidev.h`:
```c
#define SPI_MSGSIZE(N) \
    ((((N)*(sizeof(struct spi_ioc_transfer))) < (1 << _IOC_SIZEBITS)) \
        ? ((N)*(sizeof(struct spi_ioc_transfer))) : 0)
```

الـ `_IOC_SIZEBITS` = 14 → max size = 16384 bytes. الـ `sizeof(spi_ioc_transfer)` = 32 bytes. يعني max N = 16384/32 = **512 transfer**. لو حد بعت N > 512، الـ `SPI_MSGSIZE` بيرجع 0، والـ `SPI_IOC_MESSAGE(N)` بييجي invalid ioctl.

#### الحل
```c
#include <linux/spi/spidev.h>

int fd = open("/dev/spidev2.0", O_RDWR);

/* set safe speed within RK3562 SPI controller limits */
uint32_t speed = 50000000; /* 50 MHz */
if (ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed) < 0) {
    perror("SPI_IOC_WR_MAX_SPEED_HZ");
    return -1;
}

uint8_t tx[] = {0x00, 0x00, 0x00};
uint8_t rx[3] = {0};

struct spi_ioc_transfer tr = {
    .tx_buf      = (unsigned long)tx,
    .rx_buf      = (unsigned long)rx,
    .len         = sizeof(tx),
    .speed_hz    = 50000000,
    .bits_per_word = 8,
    .delay_usecs = 0,
    .cs_change   = 0,
};

int ret = ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
if (ret < 0) {
    perror("SPI_IOC_MESSAGE");
    return -1;
}
```

**تحقق من الـ DT:**
```dts
&spi2 {
    status = "okay";
    ads8688@0 {
        compatible = "ti,ads8688";
        reg = <0>;
        spi-max-frequency = <50000000>; /* 50 MHz max */
    };
};
```

**Debug script شامل:**
```bash
#!/bin/bash
# spi_bring_up_check.sh — RK3562 SPI diagnostic

DEV=/dev/spidev2.0
MAGIC=0x6b  # 'k' = SPI_IOC_MAGIC

echo "=== SPI Bring-up Diagnostic ==="

# Check device exists
[ -c "$DEV" ] || { echo "ERROR: $DEV not found"; exit 1; }

# Read current config via python
python3 << 'EOF'
import fcntl, struct

fd = open('/dev/spidev2.0', 'rb+', buffering=0)

# Read mode (SPI_IOC_RD_MODE = 0x80016b01)
buf = bytearray(1)
fcntl.ioctl(fd, 0x80016b01, buf)
print(f"Mode: {buf[0]} (0=MODE_0, 1=MODE_1, 2=MODE_2, 3=MODE_3)")

# Read speed (SPI_IOC_RD_MAX_SPEED_HZ = 0x80046b04)
buf = bytearray(4)
fcntl.ioctl(fd, 0x80046b04, buf)
speed = struct.unpack('<I', buf)[0]
print(f"Max Speed: {speed/1e6:.1f} MHz")

# Read bits per word (SPI_IOC_RD_BITS_PER_WORD = 0x80016b03)
buf = bytearray(1)
fcntl.ioctl(fd, 0x80016b03, buf)
print(f"Bits per word: {buf[0]}")

fd.close()
print("=== Config OK ===")
EOF
```

#### الدرس المستفاد
أثناء الـ board bring-up، ابدأ دايماً بـ speed منخفض (1 MHz) وزوّد تدريجياً. الـ `EINVAL` من `SPI_IOC_MESSAGE` ممكن يجي من 3 أسباب: (1) speed تعدى الـ hardware limit، (2) N تعدى 512 transfer في `SPI_IOC_MESSAGE(N)`، (3) الـ mode flags مش مدعومة من الـ controller. اقرأ `dmesg` فوراً بعد الـ ioctl الفاشل — الـ kernel بيكتب السبب الحقيقي.
## Phase 7: مصادر ومراجع

### مصادر رسمية من kernel.org

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| SPI Userspace API | [docs.kernel.org/spi/spidev.html](https://docs.kernel.org/spi/spidev.html) | الوثيقة الرسمية للـ spidev userspace API |
| SPI Driver API | [static.lwn.net/kerneldoc/driver-api/spi.html](https://static.lwn.net/kerneldoc/driver-api/spi.html) | وثيقة الـ SPI driver API الكاملة |
| kernel.org Documentation | [kernel.org/doc/Documentation/spi/spidev](https://www.kernel.org/doc/Documentation/spi/spidev) | الـ documentation القديم (plain text) |

### مسارات الـ Documentation داخل الـ Kernel

```
Documentation/spi/spidev.rst        ← وثيقة الـ spidev userspace API
Documentation/spi/spi-summary.rst   ← ملخص شامل عن subsystem الـ SPI
Documentation/spi/spi-ldisc-support.rst
include/uapi/linux/spi/spidev.h     ← الملف الـ UAPI header الرئيسي
drivers/spi/spidev.c                ← الـ driver implementation
```

### مقالات LWN.net

دي أهم المقالات اللي غطّت تطور الـ SPI subsystem في Linux:

| المقال | الرابط | السنة |
|--------|--------|-------|
| Simple SPI framework | [lwn.net/Articles/154425](https://lwn.net/Articles/154425/) | 2005 — David Brownell يقدم الـ SPI framework الأول |
| SPI redux — driver model support | [lwn.net/Articles/149698](https://lwn.net/Articles/149698/) | 2005 — دعم الـ driver model |
| SPI core refresh | [lwn.net/Articles/162268](https://lwn.net/Articles/162268/) | 2006 — تحديث الـ core |
| SPI over GPIO driver | [lwn.net/Articles/290068](https://lwn.net/Articles/290068/) | 2008 — bitbanging على GPIO |
| SPI slave mode support | [lwn.net/Articles/723440](https://lwn.net/Articles/723440/) | 2017 — دعم الـ slave mode |
| Introduction to SPI NAND framework | [lwn.net/Articles/715120](https://lwn.net/Articles/715120/) | 2017 — NAND فوق SPI |
| Deterministic SPI latency with NXP DSPI | [lwn.net/Articles/798467](https://lwn.net/Articles/798467/) | 2020 — latency محددة |
| SPI userspace API (static mirror) | [static.lwn.net/kerneldoc/spi/spidev.html](https://static.lwn.net/kerneldoc/spi/spidev.html) | مرجع سريع |

### مقالات kernelnewbies.org

الـ kernelnewbies بيوثق التغييرات في كل إصدار kernel. أهم الـ releases اللي فيها تغييرات على spidev:

| الإصدار | الرابط | التغيير |
|---------|--------|---------|
| Linux 3.15 | [kernelnewbies.org/Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) | دعم Dual/Quad SPI transfers في spidev |
| Linux 4.8 | [kernelnewbies.org/Linux_4.8](https://kernelnewbies.org/Linux_4.8) | تفعيل spidev على Intel Edison |
| Linux 4.13 | [kernelnewbies.org/Linux_4.13](https://kernelnewbies.org/Linux_4.13) | تحسينات SPI |
| Linux 4.14 | [kernelnewbies.org/Linux_4.14](https://kernelnewbies.org/Linux_4.14) | تحسينات SPI |
| Linux 5.8 | [kernelnewbies.org/Linux_5.8](https://kernelnewbies.org/Linux_5.8) | دعم Octal mode في spidev |
| Linux 6.3 | [kernelnewbies.org/Linux_6.3](https://kernelnewbies.org/Linux_6.3) | إضافة spidev لـ Silicon Labs SI3210 |

### مقالات elinux.org

موارد عملية للـ embedded systems:

| المقال | الرابط | الوصف |
|--------|--------|-------|
| RPi SPI | [elinux.org/RPi_SPI](https://elinux.org/RPi_SPI) | استخدام spidev على Raspberry Pi |
| BeagleBone Black Enable SPIDEV | [elinux.org/BeagleBone_Black_Enable_SPIDEV](https://elinux.org/BeagleBone_Black_Enable_SPIDEV) | تفعيل SPIDEV على BeagleBone Black بالـ Device Tree |
| ECE497 SPI Project | [elinux.org/ECE497_SPI_Project](https://elinux.org/ECE497_SPI_Project) | مشروع تعليمي شامل |

### الـ Source Code على GitHub

| الملف | الرابط |
|-------|--------|
| `include/uapi/linux/spi/spidev.h` | [github.com/torvalds/linux — spidev.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/spi/spidev.h) |
| `drivers/spi/spidev.c` | [github.com/torvalds/linux — spidev.c](https://github.com/torvalds/linux/blob/master/drivers/spi/spidev.c) |

### قنوات التواصل مع المجتمع

- **Mailing List**: `linux-spi@vger.kernel.org` — القائمة الرسمية لكل patches وnقاشات الـ SPI subsystem
- **Patchwork**: [patchwork.kernel.org/project/linux-spi](https://patchwork.kernel.org/project/linux-spi/) — تتبع الـ patches المرسلة
- **Git log**: `git log --follow -- drivers/spi/spidev.c` لمتابعة تاريخ التعديلات

### مصادر إضافية

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| SPIdev — linux-sunxi.org | [linux-sunxi.org/SPIdev](https://linux-sunxi.org/SPIdev) | تفاصيل تقنية للـ Allwinner SoCs |
| STM32 MPU — How to use SPI from Linux userland | [wiki.st.com — SPI userland](https://wiki.st.com/stm32mpu/wiki/How_to_use_SPI_from_Linux_userland_with_spidev) | دليل عملي لـ STM32MP |
| Peripheral Interaction Without a Kernel Driver | [embeddedrelated.com](https://www.embeddedrelated.com/showarticle/1485.php) | مقال عملي عن spidev بدون kernel driver |

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 15**: Memory Mapping and DMA — مفيد لفهم `mmap` و`copy_to_user`/`copy_from_user` في الـ driver
- **الفصل 3**: Char Drivers — الأساس اللي قام عليه spidev driver
- **الفصل 6**: Advanced Char Driver Operations — `ioctl`, `poll`, `fasync`
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل 17**: Devices and Modules — فهم الـ device model وربط الـ driver بالـ device
- **الفصل 13**: The Virtual Filesystem — فهم الـ file_operations structure

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل 8**: Device Driver Basics — مقدمة عملية للـ character device drivers
- **الفصل 15**: Debugging Embedded Linux Applications — debugging الـ userspace SPI code

### مصطلحات البحث

للبحث عن معلومات إضافية استخدم الـ search terms دي:

```
"spidev" linux kernel ioctl userspace
"SPI_IOC_MESSAGE" linux example
"spi_ioc_transfer" struct tutorial
linux spidev device tree overlay
spidev.c driver implementation
linux SPI userspace half-duplex full-duplex
"SPI_IOC_RD_MODE" "SPI_IOC_WR_MODE" linux
linux spidev python example
spidev major number 153
linux-spi mailing list spidev patch
```

### ملاحق مفيدة — أدوات الاختبار

الـ kernel نفسه بيشيل أداة اختبار جاهزة:

```bash
# موجود في tools/spi/spidev_test.c
ls tools/spi/

# تشغيل loopback test (MISO موصل بـ MOSI)
./spidev_test -D /dev/spidev0.0 -v -p "Hello"

# اختبار بـ speed محددة
./spidev_test -D /dev/spidev0.0 -s 500000 -v
```

**الـ Python library** للـ spidev:

```bash
pip install spidev
# repo: https://github.com/doceme/py-spidev
# PyPI: https://pypi.org/project/spidev/
```
## Phase 8: Writing simple module

### الفكرة

هنعمل **kprobe** على الدالة `spidev_ioctl` اللي موجودة في `drivers/spi/spidev.c`. الـ `spidev_ioctl` هي نقطة الدخول لكل ioctl calls اللي بيعملها userspace على `/dev/spidevB.C`، يعني كل مرة أي تطبيق يبعت `SPI_IOC_MESSAGE` أو يقرأ speed أو mode — بتتفير الـ kprobe دي وبنطبع اللي بيحصل.

### ليه `spidev_ioctl` تحديدًا؟

الدالة دي هي الـ `unlocked_ioctl` handler في `file_operations` الخاصة بـ spidev. كل الـ IOCTL commands المُعرّفة في `spidev.h` (زي `SPI_IOC_MESSAGE`, `SPI_IOC_RD_MAX_SPEED_HZ`, إلخ) بتعدي من هنا. ده بيخليها نقطة مراقبة مثالية لأي نشاط SPI من userspace.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spidev_kprobe.c
 *
 * Hooks spidev_ioctl() via kprobe to log every SPI ioctl
 * issued by userspace through /dev/spidevB.C devices.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* kprobe struct and API */
#include <linux/fs.h>          /* struct file */
#include <linux/spi/spidev.h>  /* SPI_IOC_MAGIC, SPI_IOC_MESSAGE, etc. */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe on spidev_ioctl to trace SPI userspace ioctls");

/* ------------------------------------------------------------------ */
/* Helper: decode the ioctl command number into a human-readable name  */
/* ------------------------------------------------------------------ */
static const char *spi_ioctl_name(unsigned int cmd)
{
    /*
     * _IOC_TYPE(cmd) must equal SPI_IOC_MAGIC ('k').
     * _IOC_NR(cmd)   gives the sub-command number.
     * SPI_IOC_MESSAGE uses NR=0, the rest use NR=1..5.
     */
    if (_IOC_TYPE(cmd) != SPI_IOC_MAGIC)
        return "UNKNOWN";

    switch (_IOC_NR(cmd)) {
    case 0: return "SPI_IOC_MESSAGE";      /* full-duplex transfer */
    case 1: return "SPI_IOC_RD/WR_MODE";  /* SPI mode 0..3 */
    case 2: return "SPI_IOC_LSB_FIRST";   /* bit order */
    case 3: return "SPI_IOC_BITS_PER_WORD";
    case 4: return "SPI_IOC_MAX_SPEED_HZ";
    case 5: return "SPI_IOC_MODE32";
    default: return "SPI_IOC_UNKNOWN";
    }
}

/* ------------------------------------------------------------------ */
/* pre-handler: fires just BEFORE spidev_ioctl() executes              */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Arguments of spidev_ioctl(struct file *filp, unsigned int cmd,
     *                           unsigned long arg):
     *
     * On x86-64 the calling convention (System V AMD64 ABI) passes:
     *   arg0 (filp) -> rdi
     *   arg1 (cmd)  -> rsi
     *   arg2 (arg)  -> rdx
     *
     * regs->si  == cmd  (unsigned int)
     * regs->dx  == arg  (unsigned long, usually a userspace pointer)
     */
    unsigned int  cmd      = (unsigned int)regs->si;
    unsigned long uarg     = regs->dx;
    const char   *cmd_name = spi_ioctl_name(cmd);
    unsigned int  ioc_dir  = _IOC_DIR(cmd);
    unsigned int  ioc_size = _IOC_SIZE(cmd);

    /*
     * _IOC_DIR flags:
     *   _IOC_NONE  = 0  (no data transfer)
     *   _IOC_READ  = 2  (kernel -> user)
     *   _IOC_WRITE = 1  (user -> kernel)
     */
    pr_info("spidev_kprobe: ioctl intercepted\n"
            "  cmd=0x%08x  name=%s\n"
            "  dir=%s%s  size=%u bytes  uarg=0x%lx\n",
            cmd,
            cmd_name,
            (ioc_dir & _IOC_READ)  ? "READ "  : "",
            (ioc_dir & _IOC_WRITE) ? "WRITE"  : "",
            ioc_size,
            uarg);

    /* Return 0 → let the real function execute normally */
    return 0;
}

/* ------------------------------------------------------------------ */
/* kprobe definition                                                    */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    /*
     * symbol_name: الـ kernel هيدور على العنوان من kallsyms.
     * لو الـ function اتعملتلها inline ممكن الـ kprobe ما يلاقيهاش،
     * لكن spidev_ioctl كبيرة بما يكفي إنها ما تتعملش inline.
     */
    .symbol_name = "spidev_ioctl",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init: register the kprobe                                     */
/* ------------------------------------------------------------------ */
static int __init spidev_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("spidev_kprobe: register_kprobe failed, ret=%d\n", ret);
        /*
         * أسباب شائعة للفشل:
         *  -EINVAL : الـ symbol مش موجود أو الـ kprobes مش مفعّل
         *  -ENOENT : الـ function مش موجودة في kallsyms (مثلاً spidev
         *            مش loaded أو اتعملتلها inline)
         */
        return ret;
    }

    pr_info("spidev_kprobe: planted at %pS\n", kp.addr);
    /*
     * %pS بيطبع رمز الـ kernel مع offset، مثلاً:
     *   spidev_ioctl+0x0/0x1e0
     * ده بيأكدلنا إن الـ kprobe اتزرع في المكان الصح.
     */
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit: MUST unregister before the module is unloaded           */
/* ------------------------------------------------------------------ */
static void __exit spidev_kprobe_exit(void)
{
    unregister_kprobe(&kp);
    /*
     * لازم ندي register قبل ما الـ module يتشال من الذاكرة،
     * لأن لو الـ kprobe اشتغل بعد ما الـ handler اتشال
     * هيحصل kernel panic (execution في عنوان مش موجود).
     */
    pr_info("spidev_kprobe: removed\n");
}

module_init(spidev_kprobe_init);
module_exit(spidev_kprobe_exit);
```

---

### Makefile

```makefile
obj-m += spidev_kprobe.o

# Replace with your actual kernel build directory
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
| `linux/module.h` | ماكروز `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/fs.h` | `struct file` — نوع الـ argument الأول لـ `spidev_ioctl` |
| `linux/spi/spidev.h` | ماكروز `SPI_IOC_MAGIC`, `_IOC_NR`, `_IOC_DIR`, `_IOC_SIZE` اللي بنستخدمها لفك الـ cmd |

الـ `linux/kernel.h` بتديه `pr_info` و `pr_err` للـ logging.

#### الـ `handler_pre` والـ Arguments

**الـ `pt_regs`** بيحتوي على حالة registers الـ CPU لحظة الـ kprobe. على x86-64:
- `regs->si` = الـ argument التاني (`cmd` — رقم الـ ioctl)
- `regs->dx` = الـ argument التالت (`arg` — pointer من userspace)

بنستخرج منهم معلومات عن الـ command نوعه (read/write)، حجم البيانات، والعنوان اللي جاي من userspace — من غير ما نتلمس الـ userspace pointer نفسه (ده أأمن في pre-handler).

#### الـ `spi_ioctl_name` Helper

الـ ioctl `cmd` في SPI بيتكوّن من `_IOC(dir, type, nr, size)`. الـ `_IOC_NR` بيطلعلك الـ sub-command number (0 لـ `SPI_IOC_MESSAGE`، 1-5 للقراءة/كتابة الـ mode والـ speed إلخ). بدل ما نطبع رقم hex خام، بنحوّله لاسم مقروء.

#### الـ `kprobe` struct

الـ `symbol_name = "spidev_ioctl"` بيخلي الـ kernel يبحث في `kallsyms` عن عنوان الدالة تلقائيًا. مش محتاجين نحدد العنوان يدويًا. الـ `pre_handler` بيشتغل قبل تنفيذ أول تعليمة في الـ function المستهدفة.

#### الـ `module_init` و `module_exit`

- الـ `init` بيسجّل الـ kprobe، ولو فشل بيرجع الـ error code مباشرة بدل ما يكمّل.
- الـ `exit` بيشيل الـ kprobe **قبل** ما الـ module code يتحذف من الذاكرة — ده ضروري لتفادي الـ kernel panic.

---

### تجربة الـ Module

```bash
# بناء الـ module
make

# تحميل الـ module
sudo insmod spidev_kprobe.ko

# تأكد إنه اتحمّل وشاف الـ symbol
sudo dmesg | tail -5
# المفروض تشوف: spidev_kprobe: planted at spidev_ioctl+0x0/...

# شغّل أي userspace tool على /dev/spidev0.0 (لو موجود)
# مثلاً:
python3 -c "
import spidev, time
spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 1000000   # triggers SPI_IOC_WR_MAX_SPEED_HZ
spi.xfer2([0xAA, 0x55])      # triggers SPI_IOC_MESSAGE
spi.close()
"

# راقب الـ kernel log
sudo dmesg | grep spidev_kprobe
# مثال على الـ output:
# spidev_kprobe: ioctl intercepted
#   cmd=0x40046b04  name=SPI_IOC_MAX_SPEED_HZ
#   dir=WRITE  size=4 bytes  uarg=0x7ffd3a21b4dc
# spidev_kprobe: ioctl intercepted
#   cmd=0x40206b00  name=SPI_IOC_MESSAGE
#   dir=WRITE  size=32 bytes  uarg=0x7ffd3a21b480

# إزالة الـ module
sudo rmmod spidev_kprobe
sudo dmesg | tail -3
# spidev_kprobe: removed
```

---

### ملاحظات مهمة

- **`CONFIG_KPROBES=y`** لازم يكون مفعّل في الـ kernel config (موجود في معظم distros).
- لو `spidev` مش مُحمَّل أو الـ kernel عمل inline للدالة، الـ `register_kprobe` هيرجع `-EINVAL`. تقدر تتحقق بـ `grep spidev_ioctl /proc/kallsyms`.
- الـ kprobe بيشتغل في **interrupt context** مع preemption disabled، يعني `pr_info` مقبولة لكن ممنوع تعمل sleep أو تاخد lock.
- الـ module ده للـ debugging والـ tracing فقط — في production استخدم `perf probe` أو `ftrace` بدل kprobes.
