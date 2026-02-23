## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **I2C Subsystem** في Linux kernel، اللي بيتبعه Wolfram Sang وبيتوزع على القائمة `linux-i2c@vger.kernel.org`.

---

### قصة المشكلة — ليه I2C موجود أصلاً؟

تخيل عندك microcontroller أو processor وعايز تكلم chip تانية — مثلاً sensor حرارة، أو شاشة OLED صغيرة، أو RTC (ساعة)، أو EEPROM. المشكلة إن كل chip بتتكلم ببروتوكول معين، وكل واحد بيحتاج وصلات كتير.

**الـ I2C** (Inter-Integrated Circuit) جاء سنة 1982 من Philips عشان يحل المشكلة دي ببساطة شديدة:

- **سلكين بس**: `SDA` (data) و `SCL` (clock)
- ممكن تحط على نفس السلكين **128 device مختلف** (7-bit address)
- واحد بيبقى **master** (CPU) والباقي **slaves** (sensors, EEPROMs, إلخ)

```
CPU (Master)
    |
  SDA + SCL
    |
  +---------+---------+---------+
  |         |         |         |
Sensor   EEPROM     RTC     Display
(0x48)   (0x50)   (0x68)   (0x3C)
```

---

### الـ i2c-dev.h — الـ File ده بيعمل إيه؟

الـ kernel عنده طبقتين للتعامل مع I2C:

1. **Kernel space**: kernel drivers بيكلموا devices مباشرةً
2. **User space**: برنامج عادي (Python, C, bash) عايز يكلم I2C device بدون ما يكتب kernel driver

**الـ `include/linux/i2c-dev.h`** ده هو **الجسر الكرنلي** اللي بيعرّف رقم الـ major device اللي بيتعامل مع `/dev/i2c-X`.

الـ file نفسه صغير جداً — سطرين مفيدين بس:

```c
#include <uapi/linux/i2c-dev.h>   /* يجيب تعريفات الـ ioctl للـ userspace */

#define I2C_MAJOR  89              /* رقم الـ major للـ char device */
```

---

### الـ `uapi/linux/i2c-dev.h` — اللي بيتضمنه

الـ kernel-facing header ده بيضم الـ UAPI (User API) الحقيقي، يعني التعريفات اللي شايفها userspace وكمان kernel:

#### الـ ioctl Commands

| الـ ioctl | الرقم | الوظيفة |
|-----------|--------|---------|
| `I2C_SLAVE` | `0x0703` | تحديد عنوان الـ slave اللي هتكلمه |
| `I2C_SLAVE_FORCE` | `0x0706` | نفس الأول لكن حتى لو الـ driver شاغله |
| `I2C_TENBIT` | `0x0704` | تفعيل 10-bit addressing |
| `I2C_FUNCS` | `0x0705` | اسأل الـ adapter بيدعم إيه |
| `I2C_RDWR` | `0x0707` | إرسال واستقبال في نفس الوقت (combined transfer) |
| `I2C_SMBUS` | `0x0720` | تنفيذ SMBus transaction |
| `I2C_RETRIES` | `0x0701` | عدد المحاولات لو الـ device مش بيرد |
| `I2C_TIMEOUT` | `0x0702` | الـ timeout بوحدة 10ms |
| `I2C_PEC` | `0x0708` | تفعيل Packet Error Checking في SMBus |

#### الـ Structs المهمة

```c
/* لـ I2C_SMBUS ioctl — بيعمل SMBus transaction */
struct i2c_smbus_ioctl_data {
    __u8  read_write;          /* I2C_SMBUS_READ أو I2C_SMBUS_WRITE */
    __u8  command;             /* رقم الـ register */
    __u32 size;                /* نوع البيانات: byte, word, block... */
    union i2c_smbus_data __user *data;  /* البيانات نفسها */
};

/* لـ I2C_RDWR ioctl — بيعمل combined read/write */
struct i2c_rdwr_ioctl_data {
    struct i2c_msg __user *msgs;  /* array من الرسائل */
    __u32 nmsgs;                  /* عدد الرسائل (max 42) */
};
```

---

### السيناريو الكامل — برنامج Python بيكلم sensor

```
برنامج Python
    |
    | open("/dev/i2c-1")
    v
 VFS (Virtual Filesystem)
    |
    | يروح لـ char device رقم 89
    v
 drivers/i2c/i2c-dev.c  ← الـ driver اللي بيعمل /dev/i2c-X
    |
    | ioctl(fd, I2C_SLAVE, 0x48)   ← اختار الـ sensor على عنوان 0x48
    | ioctl(fd, I2C_RDWR, &data)   ← ابعت/استقبل البيانات
    v
 drivers/i2c/i2c-core-base.c  ← الـ core اللي يدير الـ bus
    |
    v
 drivers/i2c/busses/i2c-*.c   ← الـ hardware driver للـ controller على الـ SoC
    |
    v
 SDA/SCL pins  ← على الـ hardware فعلاً
    |
    v
 Sensor/EEPROM/RTC
```

---

### ليه الـ File ده مهم؟

الـ `I2C_MAJOR = 89` ده رقم ثابت من زمان في Linux — أي `/dev/i2c-0`, `/dev/i2c-1` إلخ بيتسجلوا تحت الرقم ده. من غيره الـ kernel مش هيعرف يربط الـ char device file بالـ I2C subsystem.

الـ file ده بسيط لكن ضروري لأنه:
1. بيفصل الـ kernel-internal definition (الـ major number) عن الـ UAPI
2. بيسمح للـ kernel code إنه يعمل `#include <linux/i2c-dev.h>` ويوصل للرقم ده بدون ما يدخل في تفاصيل UAPI

---

### الفرق بين I2C و SMBus

**الـ SMBus** (System Management Bus) هو subset من I2C، اخترعته Intel للـ PC motherboards عشان يكلم voltage regulators وfan controllers وbattery chips. أصغر، أبطأ، وأكثر قيوداً من I2C العادي.

الـ `I2C_SMBUS` ioctl بيسمح لأي برنامج يعمل SMBus transactions عبر `/dev/i2c-X`.

---

### الملفات المكونة للـ Subsystem

#### الـ Headers (Kernel Internal)

| الملف | الوظيفة |
|-------|---------|
| `include/linux/i2c-dev.h` | **الملف ده** — major number للـ char device |
| `include/linux/i2c.h` | التعريفات الأساسية: `i2c_adapter`, `i2c_client`, `i2c_driver` |
| `include/linux/i2c-smbus.h` | SMBus alert و notification |
| `include/linux/i2c-mux.h` | I2C multiplexer support |
| `include/linux/i2c-atr.h` | I2C Address Translator |

#### الـ UAPI Headers (Kernel + Userspace)

| الملف | الوظيفة |
|-------|---------|
| `include/uapi/linux/i2c-dev.h` | الـ ioctl commands والـ structs للـ `/dev/i2c-X` |
| `include/uapi/linux/i2c.h` | `i2c_msg` struct و flags |

#### الـ Core Drivers

| الملف | الوظيفة |
|-------|---------|
| `drivers/i2c/i2c-core-base.c` | القلب — registration, transfer, locking |
| `drivers/i2c/i2c-core-smbus.c` | SMBus emulation فوق I2C |
| `drivers/i2c/i2c-core-of.c` | Device Tree probing |
| `drivers/i2c/i2c-core-acpi.c` | ACPI probing |
| `drivers/i2c/i2c-dev.c` | الـ `/dev/i2c-X` char device driver |
| `drivers/i2c/i2c-boardinfo.c` | Static board-level device registration |
| `drivers/i2c/i2c-mux.c` | I2C bus multiplexing |
| `drivers/i2c/i2c-smbus.c` | SMBus alert IRQ handling |
| `drivers/i2c/i2c-stub.c` | Fake I2C adapter للـ testing |

#### الـ Hardware Bus Drivers (أمثلة)

| الملف | الـ Hardware |
|-------|------------|
| `drivers/i2c/busses/i2c-bcm2835.c` | Raspberry Pi |
| `drivers/i2c/busses/i2c-i801.c` | Intel PCH (PC motherboards) |
| `drivers/i2c/busses/i2c-qcom-geni.c` | Qualcomm SoCs |
| `drivers/i2c/busses/i2c-imx.c` | NXP i.MX |
| `drivers/i2c/busses/i2c-omap.c` | TI OMAP |
## Phase 2: شرح الـ I2C-dev Framework

### المشكلة — ليه الـ i2c-dev موجود أصلاً؟

في اللينوكس، الطريقة الطبيعية للتعامل مع I2C device زي sensor أو EEPROM هي إنك تكتب **kernel driver** كامل بـ `struct i2c_driver` و`probe()`/`remove()` إلخ. ده مناسب في production، لكن فيه سيناريوهات كتير بيبقى فيها مشكلة:

- **التطوير والـ prototyping**: محتاج تقرأ register من sensor جديد من غير ما تكتب driver كامل.
- **الـ testing والـ debugging**: محتاج تتحقق إن الـ hardware شغال صح من الـ userspace قبل ما تبدأ تكتب driver.
- **الـ factory calibration**: tools مكتوبة بـ Python أو C تحتاج تكتب بيانات على EEPROM على خط التجميع.
- **الـ proprietary software**: شركات بتفضل تحتفظ بالـ logic في userspace بدل ما تفتح kernel driver.

بدون i2c-dev، الـ userspace لا يملك أي وسيلة للتحدث مع I2C bus مباشرة — الـ kernel يحتكر الوصول التام للـ bus.

**الـ i2c-dev** يحل المشكلة دي بإنه يفتح **نافذة مباشرة** من الـ userspace على كل I2C bus موجود في النظام عبر **character device**.

---

### الحل — الـ approach اللي بياخده الـ kernel

الـ kernel بيعمل الآتي:

1. لكل `i2c_adapter` (يعني كل I2C bus controller) مسجّل في النظام، الـ i2c-dev module بيعمل **character device** باسم `/dev/i2c-N` (N = رقم الـ bus).
2. الـ major number ثابت = **89** (ده اللي في `i2c-dev.h`: `#define I2C_MAJOR 89`).
3. الـ userspace يفتح الـ file، يستخدم `ioctl()` لتحديد عنوان الـ slave، ثم يقرأ/يكتب بـ `read()`/`write()` — أو يبعت transactions مركبة بـ `I2C_RDWR`.
4. الـ kernel يترجم العمليات دي لـ `i2c_msg` structs ويمررها لـ `i2c_transfer()` الحقيقية على الـ adapter.

**نقطة مهمة**: الـ i2c-dev مش driver لـ device معين — هو **bridge** بين الـ userspace وبين subsystem الـ I2C كله.

---

### الـ Architecture — الصورة الكبيرة

```
┌─────────────────────────────────────────────────────────────────┐
│                        USERSPACE                                │
│                                                                 │
│  Python script      C app          i2ctools (i2cget/i2cset)    │
│       │                │                    │                  │
│       └────────────────┴────────────────────┘                  │
│                         │  open/read/write/ioctl               │
│                  /dev/i2c-0   /dev/i2c-1   /dev/i2c-N         │
└─────────────────────────────────────────────────────────────────┘
                          │  VFS syscall layer
┌─────────────────────────────────────────────────────────────────┐
│                   KERNEL — i2c-dev module                       │
│                                                                 │
│   i2cdev_fops { .open, .read, .write, .unlocked_ioctl, ... }   │
│        │                                                        │
│   يترجم read/write/ioctl → struct i2c_msg[]                    │
│        │                                                        │
│   يستدعي i2c_transfer(adapter, msgs, nmsgs)  ◄── i2c-core API │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                   KERNEL — i2c-core (i2c subsystem)             │
│                                                                 │
│   struct i2c_adapter   struct i2c_algorithm   struct i2c_client │
│        │                      │                                 │
│   bus_lock (rt_mutex)   algo->xfer()                           │
│        │                      │                                 │
└───────────────────────────────┼─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│             KERNEL — Platform/SoC I2C Controller Driver         │
│                                                                 │
│   مثلاً: i2c-imx.c / i2c-designware.c / i2c-bcm2835.c         │
│   يتحكم في الـ hardware registers مباشرة                       │
│   يولّد START / STOP / ACK/NAK على الـ wire                    │
└─────────────────────────────────────────────────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │     I2C Wire (HW)     │
                    │   SDA ─────────────── │
                    │   SCL ─────────────── │
                    └───────────────────────┘
                         │           │
                    [Sensor]      [EEPROM]
                    0x48          0x50
```

---

### الـ ioctl Commands — واجهة الـ userspace

الـ UAPI header `uapi/linux/i2c-dev.h` بيعرّف كل الـ ioctl codes اللي الـ userspace بيستخدمها:

| ioctl | Code | الغرض |
|---|---|---|
| `I2C_SLAVE` | `0x0703` | تحديد عنوان الـ slave (7-bit) |
| `I2C_SLAVE_FORCE` | `0x0706` | نفس السابق لكن حتى لو الـ kernel driver حاجز العنوان |
| `I2C_TENBIT` | `0x0704` | تفعيل 10-bit addressing |
| `I2C_FUNCS` | `0x0705` | جلب قدرات الـ adapter (bitmask) |
| `I2C_RDWR` | `0x0707` | إرسال transactions مركبة (multi-message بـ STOP واحد) |
| `I2C_SMBUS` | `0x0720` | SMBus transaction (أبسط من raw I2C) |
| `I2C_RETRIES` | `0x0701` | عدد المحاولات عند عدم الـ ACK |
| `I2C_TIMEOUT` | `0x0702` | timeout بوحدة 10ms |
| `I2C_PEC` | `0x0708` | تفعيل Packet Error Checking لـ SMBus |

---

### الـ Core Structs — كيف بيتكلموا مع بعض

#### `struct i2c_rdwr_ioctl_data` — الأهم لـ raw I2C

```c
/* userspace بيحط pointer على array من messages */
struct i2c_rdwr_ioctl_data {
    struct i2c_msg __user *msgs;  /* array of I2C messages */
    __u32 nmsgs;                  /* عددهم، max = 42 */
};
```

الـ `__user` macro ده hint للـ sparse tool إن الـ pointer ده من الـ userspace — يعني لازم `copy_from_user()` قبل استخدامه في الـ kernel.

**ليه 42 بالذات؟** `I2C_RDWR_IOCTL_MAX_MSGS = 42` — حد عملي لمنع الـ userspace من استهلاك memory لا نهاية فيها في kernel space.

#### `struct i2c_smbus_ioctl_data` — لـ SMBus

```c
struct i2c_smbus_ioctl_data {
    __u8 read_write;              /* I2C_SMBUS_READ أو I2C_SMBUS_WRITE */
    __u8 command;                 /* register address أو command byte */
    __u32 size;                   /* نوع التراسيل: byte/word/block */
    union i2c_smbus_data __user *data; /* البيانات نفسها */
};
```

**الـ SMBus**: بروتوكول فوق I2C، Intel طوّرته، بيضيف conventions محددة زي "اكتب byte على register"، "اقرأ word". الـ kernel يقدر يـ emulate SMBus على أي I2C adapter.

---

### الـ Real-World Analogy — مفصّل ومترجم للـ kernel concepts

**تخيل مستشفى كبير:**

| جزء المستشفى | الـ kernel concept | التفصيل |
|---|---|---|
| **مبنى المستشفى** | الـ kernel | الإطار اللي كل حاجة تشتغل جواه |
| **قسم الطوارئ** | `i2c-dev` module | بوابة مفتوحة للزوار (userspace) بدون موعد |
| **الأطباء المتخصصون** | kernel I2C drivers (e.g. `bmp280-i2c.c`) | بيتعاملوا مع مرضى محددين بخبرة كاملة |
| **الممرضة الاستقبال** | `i2cdev_ioctl()` | بتاخد طلبك وبتوجهك للأدوات الصح |
| **الجناح الطبي (السرير رقم N)** | `/dev/i2c-N` | كل bus بيبقاله entry خاصة |
| **وصفة الدواء** | `struct i2c_msg` | التعليمات المحددة: اقرأ/اكتب، عنوان الـ slave، الـ data |
| **طاقم العمليات** | `i2c_algorithm->xfer()` | الناس اللي بتنفذ الوصفة فعلياً على الـ hardware |
| **غرفة العمليات** | SoC I2C controller | الـ hardware اللي بيولّد إشارات SDA/SCL |
| **المريض** | I2C slave device | الـ sensor/EEPROM على الـ bus |

**الـ analogy بيشتغل على مستوى أعمق**:
- زي ما الطوارئ بيعالجوا مريض جديد من غير إنك تعرف اسمه مسبقاً — `I2C_SLAVE_FORCE` بيتيح التواصل مع device حتى لو ملهوش driver مسجّل.
- الطبيب المتخصص (kernel driver) أكفأ وأأمن في التعامل مع مريض محدد لأنه يعرف تفاصيله — لكن الطوارئ (i2c-dev) أسرع للحالات الطارئة أو التجريبية.
- الوصفة (`i2c_msg`) بتعدّي من الممرضة (ioctl handler) لغرفة العمليات (الـ controller driver) بنفس الشكل بغض النظر عن مصدرها.

---

### الـ Core Abstraction — الفكرة المحورية

**الـ i2c-dev مش subsystem جديد — هو thin translation layer.**

الـ abstraction المحوري هو:

> كل عملية userspace (open/read/write/ioctl) بتتترجم لـ `struct i2c_msg` واحدة أو أكتر، وتُمرر لـ `i2c_transfer()` — نفس الـ function اللي أي kernel driver بيستخدمها.

```
userspace read(fd, buf, len)
        │
        ▼
i2cdev_read()   ← في i2c-dev.c
        │
        │  بيبني:
        │  struct i2c_msg msg = {
        │      .addr  = client->addr,   /* من I2C_SLAVE ioctl */
        │      .flags = I2C_M_RD,
        │      .len   = len,
        │      .buf   = buf,
        │  };
        ▼
i2c_transfer(adap, &msg, 1)  ← نفس الـ API اللي kernel drivers بتستخدمه
        │
        ▼
adap->algo->xfer(adap, msgs, num)  ← الـ SoC driver
```

**ده معناه**: الـ i2c-dev مش بيدور حواليه — هو just **userspace-to-kernel-API bridge** بشكل character device.

---

### الـ Kernel I2C Subsystem — المكونات الرئيسية

عشان تفهم i2c-dev صح، لازم تعرف الـ building blocks اللي فوقيها بيشتغل:

#### `struct i2c_adapter` — يمثّل الـ physical I2C bus

```c
struct i2c_adapter {
    const struct i2c_algorithm *algo; /* xfer(), smbus_xfer(), functionality() */
    const struct i2c_lock_operations *lock_ops; /* lock_bus/unlock_bus */
    struct rt_mutex bus_lock;         /* يمنع race conditions على الـ bus */
    int nr;                           /* رقم الـ bus = N في /dev/i2c-N */
    char name[48];
    struct list_head userspace_clients; /* clients المسجلة من userspace */
    struct device dev;                /* في الـ device model */
    /* ... */
};
```

**الـ `rt_mutex`**: ده real-time mutex — مش عادي، بيدعم **priority inheritance** عشان يمنع **priority inversion** في الـ real-time scenarios. لازم تعرف الـ locking subsystem لتفهم ليه ده مش `mutex` عادي.

#### `struct i2c_algorithm` — الـ transport implementation

```c
struct i2c_algorithm {
    int (*xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs, int num);
    int (*smbus_xfer)(struct i2c_adapter *adap, u16 addr, ...);
    u32 (*functionality)(struct i2c_adapter *adap); /* I2C_FUNC_* bitmask */
};
```

الـ `functionality()` مهم جداً للـ i2c-dev: الـ `I2C_FUNCS` ioctl بيستدعيه عشان الـ userspace يعرف إيه اللي الـ adapter بيدعمه (10-bit addressing, SMBus, إلخ).

#### `struct i2c_client` — يمثّل slave device واحد

```c
struct i2c_client {
    unsigned short flags;    /* I2C_CLIENT_TEN, I2C_CLIENT_PEC, ... */
    unsigned short addr;     /* 7-bit address في الـ lower 7 bits */
    char name[I2C_NAME_SIZE];
    struct i2c_adapter *adapter; /* الـ bus اللي هو عليه */
    struct device dev;           /* في الـ device model */
    int irq;
};
```

في حالة الـ i2c-dev، الـ kernel بيعمل **temporary i2c_client** في الـ memory لكل file descriptor مفتوح — ده الـ "virtual client" اللي بيمثل العنوان اللي حددته بـ `I2C_SLAVE` ioctl.

---

### الـ Struct Relationships — كيف بيتربطوا

```
struct i2c_adapter
    │
    ├── algo ──────────────► struct i2c_algorithm
    │                            ├── xfer()         ← SoC driver ينفذه
    │                            ├── smbus_xfer()
    │                            └── functionality()
    │
    ├── lock_ops ──────────► struct i2c_lock_operations
    │                            ├── lock_bus()
    │                            └── unlock_bus()
    │
    ├── bus_recovery_info ─► struct i2c_bus_recovery_info
    │                            ├── recover_bus()
    │                            ├── get_scl() / set_scl()
    │                            └── scl_gpiod / sda_gpiod
    │
    ├── quirks ────────────► struct i2c_adapter_quirks
    │                            ├── max_num_msgs
    │                            ├── max_write_len
    │                            └── flags (I2C_AQ_*)
    │
    └── userspace_clients ─► [list of i2c_client created by i2c-dev]


struct i2c_client
    ├── adapter ──────────► struct i2c_adapter (الـ bus اللي هو عليه)
    └── dev ──────────────► struct device (في الـ driver model)
                                └── driver ───► struct i2c_driver
                                                    ├── probe()
                                                    ├── remove()
                                                    └── id_table[]
```

---

### ما الذي يملكه الـ i2c-dev وما الذي يفوّضه

| المسؤولية | i2c-dev يملكها | يفوّضها لـ |
|---|---|---|
| إنشاء `/dev/i2c-N` | نعم | — |
| تحديد الـ major number (89) | نعم — محدد في الـ header | — |
| تسجيل الـ char device | نعم | — |
| ترجمة `read()`/`write()` لـ `i2c_msg` | نعم | — |
| تنفيذ الـ ioctl commands | نعم | — |
| الـ bus locking | لا | `i2c_transfer()` في i2c-core |
| إرسال البيانات فعلياً على الـ wire | لا | `algo->xfer()` في الـ SoC driver |
| SMBus protocol emulation | لا | `i2c_smbus_xfer()` في i2c-core |
| إدارة الـ device tree / ACPI | لا | i2c-core + platform code |
| error recovery (bus stuck) | لا | `i2c_bus_recovery_info` في الـ adapter driver |
| الـ clock stretching / timing | لا | الـ SoC driver و`i2c_timings` |

---

### مثال عملي — قراءة temperature sensor على `/dev/i2c-1`

**الـ sensor**: TMP102 على عنوان `0x48`، الـ temperature register هو `0x00`، بيرجع 2 bytes.

```c
#include <fcntl.h>
#include <sys/ioctl.h>
#include <linux/i2c-dev.h>  /* I2C_SLAVE, I2C_RDWR */
#include <linux/i2c.h>      /* struct i2c_msg */

int fd = open("/dev/i2c-1", O_RDWR);

/* تحديد عنوان الـ slave — يترجم لـ i2cdev_ioctl() */
ioctl(fd, I2C_SLAVE, 0x48);

/* combined transaction: write register address, then read 2 bytes */
struct i2c_msg msgs[2] = {
    {
        .addr  = 0x48,
        .flags = 0,       /* write */
        .len   = 1,
        .buf   = (uint8_t[]){0x00},  /* register address */
    },
    {
        .addr  = 0x48,
        .flags = I2C_M_RD, /* read */
        .len   = 2,
        .buf   = temp_buf,
    },
};

struct i2c_rdwr_ioctl_data data = {
    .msgs  = msgs,
    .nmsgs = 2,
};

/* بعت الـ transaction للـ kernel عبر I2C_RDWR ioctl */
ioctl(fd, I2C_RDWR, &data);

/* temp_buf[0] و temp_buf[1] دلوقتي فيهم الـ raw temperature */
int16_t raw = (temp_buf[0] << 4) | (temp_buf[1] >> 4);
float celsius = raw * 0.0625f;
```

**اللي بيحصل جوه الـ kernel**:
1. `ioctl(I2C_RDWR)` → `i2cdev_ioctl()` → `copy_from_user()` للـ msgs
2. validation: `nmsgs <= 42`، كل msg فيها `len > 0`، إلخ
3. `i2c_transfer(adapter, msgs, 2)` → `rt_mutex_lock(&adapter->bus_lock)`
4. `adapter->algo->xfer(adapter, msgs, 2)` → الـ SoC driver بيولّد:
   - START → address 0x48+W → 0x00 → repeated START → 0x48+R → byte1 → byte2 → STOP
5. `rt_mutex_unlock()`
6. `copy_to_user()` بالنتيجة

---

### ملاحظات مهمة للـ embedded engineer

**`I2C_SLAVE` vs `I2C_SLAVE_FORCE`**:
- `I2C_SLAVE` بيفشل لو فيه kernel driver حاجز العنوان ده بالفعل (`-EBUSY`).
- `I2C_SLAVE_FORCE` بيتجاهل ده — خطير في production لأنه ممكن يتعارض مع kernel driver شغال.

**الـ `userspace_clients` list في `i2c_adapter`**:
الـ i2c-dev بيضيف كل client (عنوان) بيتفتح من الـ userspace لـ `adapter->userspace_clients` — ده يخلي الـ i2c-core يعرف مين شغال على الـ bus ويقدر يمنع التعارض.

**الـ `I2C_FUNCS` ioctl مهم جداً**:
قبل ما تستخدم `I2C_RDWR` أو SMBus، تأكد إن الـ adapter بيدعمهم:

```c
unsigned long funcs;
ioctl(fd, I2C_FUNCS, &funcs);
if (!(funcs & I2C_FUNC_I2C))
    /* الـ adapter مش بيدعم raw I2C، بس SMBus */
```

**الـ `I2C_MAJOR = 89`**:
ده الـ major number المحجوز تاريخياً في الـ kernel لـ I2C character devices. الـ minor number = رقم الـ adapter (0, 1, 2, ...). `/dev/i2c-1` يعني major=89, minor=1.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### نظرة عامة على الملف

الـ `include/linux/i2c-dev.h` هو header بسيط جداً — بيعرّف بس `I2C_MAJOR` (رقم 89) وبيعمل `#include` للـ uapi header الفعلي.
الـ uapi header هو `uapi/linux/i2c-dev.h` وهو اللي بيعرّف الـ structs والـ ioctls الخاصة بـ `/dev/i2c-X`.
الـ `uapi/linux/i2c.h` بيعرّف `struct i2c_msg` و `union i2c_smbus_data` اللي بيستخدمهم الـ uapi header.

---

### 0. الـ Flags والـ Defines — Cheatsheet

#### ioctl Commands على `/dev/i2c-X`

| Define | Value | المعنى |
|---|---|---|
| `I2C_RETRIES` | `0x0701` | عدد مرات polling الـ slave لو مش بيرد بـ ACK |
| `I2C_TIMEOUT` | `0x0702` | timeout بوحدة 10ms |
| `I2C_SLAVE` | `0x0703` | تحديد عنوان الـ slave المستخدم |
| `I2C_SLAVE_FORCE` | `0x0706` | زي `I2C_SLAVE` بس بيتجاوز حتى لو driver تاني شايله |
| `I2C_TENBIT` | `0x0704` | 0 = 7-bit address، غير صفر = 10-bit address |
| `I2C_FUNCS` | `0x0705` | اقرأ الـ functionality mask بتاع الـ adapter |
| `I2C_RDWR` | `0x0707` | Combined read/write transfer بـ STOP واحد بس |
| `I2C_PEC` | `0x0708` | فعّل Packet Error Checking مع SMBus |
| `I2C_SMBUS` | `0x0720` | SMBus transfer |

#### الـ `I2C_RDWR_IOCTL_MAX_MSGS`

| Define | Value | المعنى |
|---|---|---|
| `I2C_RDWR_IOCTL_MAX_MSGS` | 42 | أقصى عدد `i2c_msg` في طلب `I2C_RDWR` واحد |
| `I2C_RDRW_IOCTL_MAX_MSGS` | نفس 42 | alias قديم فيه typo — محتفظ به للـ compatibility |

---

#### flags الـ `struct i2c_msg`

| Flag | Value | الشرط المطلوب | المعنى |
|---|---|---|---|
| `I2C_M_RD` | `0x0001` | دايماً | قراءة من الـ slave (مش كتابة) |
| `I2C_M_TEN` | `0x0010` | `I2C_FUNC_10BIT_ADDR` | عنوان 10-bit |
| `I2C_M_DMA_SAFE` | `0x0200` | kernel space فقط | الـ buffer آمن للـ DMA |
| `I2C_M_RECV_LEN` | `0x0400` | `I2C_FUNC_SMBUS_READ_BLOCK_DATA` | الطول بييجي من أول byte مستلم |
| `I2C_M_NO_RD_ACK` | `0x0800` | `I2C_FUNC_PROTOCOL_MANGLING` | تجاهل الـ ACK/NACK في القراءة |
| `I2C_M_IGNORE_NAK` | `0x1000` | `I2C_FUNC_PROTOCOL_MANGLING` | تعامل مع الـ NACK كـ ACK |
| `I2C_M_REV_DIR_ADDR` | `0x2000` | `I2C_FUNC_PROTOCOL_MANGLING` | اقلب bit الـ R/W في العنوان |
| `I2C_M_NOSTART` | `0x4000` | `I2C_FUNC_NOSTART` | متبعتش repeated START |
| `I2C_M_STOP` | `0x8000` | `I2C_FUNC_PROTOCOL_MANGLING` | إجبار STOP بعد الـ message |

---

#### الـ I2C_FUNC — Adapter Functionality Bits

| Define | Value | المعنى |
|---|---|---|
| `I2C_FUNC_I2C` | `0x00000001` | raw I2C transfers |
| `I2C_FUNC_10BIT_ADDR` | `0x00000002` | عناوين 10-bit |
| `I2C_FUNC_PROTOCOL_MANGLING` | `0x00000004` | تعديل البروتوكول |
| `I2C_FUNC_SMBUS_PEC` | `0x00000008` | Packet Error Checking |
| `I2C_FUNC_NOSTART` | `0x00000010` | تخطي START |
| `I2C_FUNC_SLAVE` | `0x00000020` | الـ adapter يشتغل كـ slave |
| `I2C_FUNC_SMBUS_QUICK` | `0x00010000` | SMBus Quick Command |
| `I2C_FUNC_SMBUS_READ_BYTE` | `0x00020000` | قراءة byte بدون register |
| `I2C_FUNC_SMBUS_WRITE_BYTE` | `0x00040000` | كتابة byte بدون register |
| `I2C_FUNC_SMBUS_READ_BYTE_DATA` | `0x00080000` | قراءة byte من register |
| `I2C_FUNC_SMBUS_WRITE_BYTE_DATA` | `0x00100000` | كتابة byte في register |
| `I2C_FUNC_SMBUS_READ_WORD_DATA` | `0x00200000` | قراءة word من register |
| `I2C_FUNC_SMBUS_WRITE_WORD_DATA` | `0x00400000` | كتابة word في register |
| `I2C_FUNC_SMBUS_PROC_CALL` | `0x00800000` | Process Call |
| `I2C_FUNC_SMBUS_READ_BLOCK_DATA` | `0x01000000` | قراءة block (مطلوب لـ `I2C_M_RECV_LEN`) |
| `I2C_FUNC_SMBUS_WRITE_BLOCK_DATA` | `0x02000000` | كتابة block |
| `I2C_FUNC_SMBUS_READ_I2C_BLOCK` | `0x04000000` | قراءة block بـ I2C style |
| `I2C_FUNC_SMBUS_WRITE_I2C_BLOCK` | `0x08000000` | كتابة block بـ I2C style |
| `I2C_FUNC_SMBUS_HOST_NOTIFY` | `0x10000000` | SMBus 2.0 Host Notify |
| `I2C_FUNC_SMBUS_BLOCK_PROC_CALL` | `0x00008000` | SMBus 2.0 Block Process Call |

#### Composite FUNC Macros

| Macro | المكونات |
|---|---|
| `I2C_FUNC_SMBUS_BYTE` | READ_BYTE \| WRITE_BYTE |
| `I2C_FUNC_SMBUS_BYTE_DATA` | READ_BYTE_DATA \| WRITE_BYTE_DATA |
| `I2C_FUNC_SMBUS_WORD_DATA` | READ_WORD_DATA \| WRITE_WORD_DATA |
| `I2C_FUNC_SMBUS_BLOCK_DATA` | READ_BLOCK_DATA \| WRITE_BLOCK_DATA |
| `I2C_FUNC_SMBUS_I2C_BLOCK` | READ_I2C_BLOCK \| WRITE_I2C_BLOCK |
| `I2C_FUNC_SMBUS_EMUL` | كل عمليات SMBus الأساسية معاً + PEC |
| `I2C_FUNC_SMBUS_EMUL_ALL` | EMUL + READ_BLOCK_DATA + BLOCK_PROC_CALL |

---

#### SMBus Transaction Types (الـ `size` field في `i2c_smbus_ioctl_data`)

| Define | Value | النوع |
|---|---|---|
| `I2C_SMBUS_QUICK` | 0 | Quick Command فقط (no data) |
| `I2C_SMBUS_BYTE` | 1 | Byte بدون register |
| `I2C_SMBUS_BYTE_DATA` | 2 | Byte مع register |
| `I2C_SMBUS_WORD_DATA` | 3 | Word مع register |
| `I2C_SMBUS_PROC_CALL` | 4 | Process Call |
| `I2C_SMBUS_BLOCK_DATA` | 5 | Block حقيقي |
| `I2C_SMBUS_I2C_BLOCK_BROKEN` | 6 | I2C block (legacy broken) |
| `I2C_SMBUS_BLOCK_PROC_CALL` | 7 | Block Process Call (SMBus 2.0) |
| `I2C_SMBUS_I2C_BLOCK_DATA` | 8 | I2C block صحيح |

#### Read/Write Markers

| Define | Value |
|---|---|
| `I2C_SMBUS_READ` | 1 |
| `I2C_SMBUS_WRITE` | 0 |

---

### 1. الـ Structs المهمة

#### `struct i2c_msg`

**الغرض:** تمثيل segment واحد من transaction الـ I2C — كل message بتبدأ بـ START على الـ bus.

```c
struct i2c_msg {
    __u16 addr;    /* عنوان الـ slave (7-bit أو 10-bit) */
    __u16 flags;   /* I2C_M_RD وغيره — بيحدد طبيعة الـ transfer */
    __u16 len;     /* عدد الـ bytes في buf */
    __u8 *buf;     /* البيانات المرسلة أو المستلمة */
};
```

| Field | النوع | الشرح |
|---|---|---|
| `addr` | `__u16` | عنوان الـ slave على الـ bus، 7-bit عادةً |
| `flags` | `__u16` | مجموعة من الـ `I2C_M_*` flags |
| `len` | `__u16` | حجم الـ `buf` بالـ bytes |
| `buf` | `__u8 *` | pointer للبيانات — kernel بيستخدمه، userspace بيتنسخ |

**الاتصال بالـ structs التانية:** الـ `i2c_rdwr_ioctl_data` بيحتوي على array من `i2c_msg`.

---

#### `struct i2c_rdwr_ioctl_data`

**الغرض:** الـ parameter المُمرَّر لـ `ioctl(fd, I2C_RDWR, ...)` — بيحمل مجموعة messages اتبعت كـ combined transfer بـ STOP واحد في النهاية.

```c
struct i2c_rdwr_ioctl_data {
    struct i2c_msg __user *msgs;  /* array of i2c_msg في userspace */
    __u32 nmsgs;                  /* عدد الـ messages (max 42) */
};
```

| Field | النوع | الشرح |
|---|---|---|
| `msgs` | `struct i2c_msg __user *` | pointer لـ array من الـ messages |
| `nmsgs` | `__u32` | عدد الـ messages — لازم ≤ `I2C_RDWR_IOCTL_MAX_MSGS` (42) |

**الاتصال بالـ structs التانية:** بيعتمد على `struct i2c_msg` لكل message.

---

#### `struct i2c_smbus_ioctl_data`

**الغرض:** الـ parameter المُمرَّر لـ `ioctl(fd, I2C_SMBUS, ...)` — بيصف عملية SMBus كاملة.

```c
struct i2c_smbus_ioctl_data {
    __u8 read_write;              /* I2C_SMBUS_READ أو I2C_SMBUS_WRITE */
    __u8 command;                 /* رقم الـ register أو الـ command */
    __u32 size;                   /* نوع الـ transaction: I2C_SMBUS_BYTE_DATA إلخ */
    union i2c_smbus_data __user *data; /* البيانات نفسها */
};
```

| Field | النوع | الشرح |
|---|---|---|
| `read_write` | `__u8` | `I2C_SMBUS_READ`=1 أو `I2C_SMBUS_WRITE`=0 |
| `command` | `__u8` | رقم الـ command / register address |
| `size` | `__u32` | نوع الـ transaction من الـ `I2C_SMBUS_*` defines |
| `data` | `union i2c_smbus_data __user *` | pointer للبيانات في userspace |

---

#### `union i2c_smbus_data`

**الغرض:** container موحد لكل أنواع بيانات SMBus.

```c
#define I2C_SMBUS_BLOCK_MAX  32

union i2c_smbus_data {
    __u8  byte;                          /* byte واحد */
    __u16 word;                          /* word واحد */
    __u8  block[I2C_SMBUS_BLOCK_MAX + 2]; /* block[0] = الطول */
};
```

| Member | الحجم | الاستخدام |
|---|---|---|
| `byte` | 1 byte | `I2C_SMBUS_BYTE` أو `BYTE_DATA` |
| `word` | 2 bytes | `I2C_SMBUS_WORD_DATA` أو `PROC_CALL` |
| `block` | 34 bytes | `I2C_SMBUS_BLOCK_DATA` — `block[0]` بيحمل الطول |

---

### 2. مخطط العلاقات بين الـ Structs

```
userspace process
      │
      │  ioctl(fd, I2C_RDWR, &rdwr_data)
      ▼
┌─────────────────────────────┐
│  i2c_rdwr_ioctl_data        │
│  ┌───────────────────────┐  │
│  │ msgs  ──────────────► │──┼──► [ i2c_msg #0 ] ──► buf (data)
│  │ nmsgs = N (max 42)    │  │    [ i2c_msg #1 ] ──► buf (data)
│  └───────────────────────┘  │    [ i2c_msg #2 ] ──► buf (data)
└─────────────────────────────┘         ...

      │  ioctl(fd, I2C_SMBUS, &smbus_data)
      ▼
┌─────────────────────────────────┐
│  i2c_smbus_ioctl_data           │
│  ┌──────────────────────────┐   │
│  │ read_write (R or W)      │   │
│  │ command    (reg addr)    │   │
│  │ size       (xfer type)   │   │
│  │ data ───────────────────►│───┼──► union i2c_smbus_data
│  └──────────────────────────┘   │    ┌──────────────────┐
└─────────────────────────────────┘    │ .byte  (u8)      │
                                       │ .word  (u16)     │
                                       │ .block (u8 x 34) │
                                       └──────────────────┘
```

---

### 3. مخطط الـ Lifecycle

#### lifecycle الـ `/dev/i2c-X` من منظور userspace

```
open("/dev/i2c-0", O_RDWR)
        │
        ▼
  [kernel: i2cdev_open()]
  ينشئ i2c_client مؤقت مرتبط بالـ adapter
        │
        ▼
ioctl(fd, I2C_SLAVE, 0x50)       ← تحديد عنوان الـ slave
        │
        ▼
ioctl(fd, I2C_FUNCS, &funcs)     ← استعلام capabilities الـ adapter
        │
        ├── I2C_RDWR ──────────────────────────────────────────────┐
        │   userspace يملأ i2c_rdwr_ioctl_data                     │
        │   + array من i2c_msg                                      │
        │           │                                               │
        │           ▼                                               │
        │   [kernel: i2cdev_ioctl_rdwr()]                          │
        │   - ينسخ struct من userspace (copy_from_user)            │
        │   - ينسخ كل i2c_msg                                      │
        │   - ينسخ كل buf                                          │
        │   - يستدعي i2c_transfer(adapter, msgs, nmsgs)            │
        │   - ينسخ النتائج لـ userspace (copy_to_user)             │
        │                                                           │
        └── I2C_SMBUS ─────────────────────────────────────────────┘
            userspace يملأ i2c_smbus_ioctl_data
                    │
                    ▼
            [kernel: i2cdev_ioctl_smbus()]
            - ينسخ struct من userspace
            - يستدعي i2c_smbus_xfer()
            - ينسخ النتائج لـ userspace
                    │
                    ▼
close(fd)
[kernel: i2cdev_release()]
يحرر الـ i2c_client المؤقت
```

---

### 4. مخطط Call Flow

#### I2C_RDWR ioctl flow

```
userspace: ioctl(fd, I2C_RDWR, &data)
    │
    ▼
i2cdev_ioctl()                        [drivers/i2c/i2c-dev.c]
    │
    ▼
i2cdev_ioctl_rdwr(client, data)
    ├── copy_from_user(&rdwr_arg, data, sizeof)
    ├── copy_from_user(rdwr_pa, rdwr_arg.msgs, nmsgs * sizeof(i2c_msg))
    ├── for each msg: copy_from_user(buf, msg.buf, msg.len)
    │
    ▼
i2c_transfer(client->adapter, msgs, nmsgs)
    │
    ▼
adap->algo->master_xfer(adap, msgs, nmsgs)   ← hardware driver
    │
    ▼
[I2C hardware registers — START, addr, data bytes, STOP]
    │
    ▼
النتيجة ترجع للـ i2cdev_ioctl_rdwr
    └── copy_to_user() للـ read buffers
```

#### I2C_SMBUS ioctl flow

```
userspace: ioctl(fd, I2C_SMBUS, &data)
    │
    ▼
i2cdev_ioctl()
    │
    ▼
i2cdev_ioctl_smbus(client, data)
    ├── copy_from_user(&data_arg, data, sizeof)
    ├── copy_from_user(&temp, data_arg.data, smbus_data_size)
    │
    ▼
i2c_smbus_xfer(adapter, addr, flags, rw, command, size, &data)
    │
    ├── [إذا adapter عنده smbus_xfer]
    │       adap->algo->smbus_xfer()    ← native SMBus hardware
    │
    └── [إذا مفيش smbus_xfer — emulation]
            i2c_smbus_xfer_emulated()
                └── i2c_transfer() بـ msgs مكونة يدوياً
    │
    ▼
copy_to_user() للنتيجة
```

#### I2C_FUNCS ioctl flow

```
userspace: ioctl(fd, I2C_FUNCS, &funcs)
    │
    ▼
i2cdev_ioctl()
    │
    ▼
i2c_get_functionality(client->adapter)
    └── يرجع adapter->algo->functionality
    │
    ▼
put_user(funcs, (unsigned long __user *)arg)
```

---

### 5. استراتيجية الـ Locking

الـ headers نفسها (uapi) مش بتعرّف locks — دي مسؤولية الـ kernel core في `i2c-dev.c` و `i2c-core.c`. بس المهم نفهم الصورة الكاملة:

#### الـ i2c_dev (struct داخل kernel)

```
struct i2c_dev {
    struct list_head list;   /* محمي بـ i2c_dev_lock (spinlock) */
    struct i2c_adapter *adap;
    struct device *dev;
    struct cdev cdev;
};
```

| Lock | النوع | بيحمي إيه |
|---|---|---|
| `i2c_dev_lock` | `spinlock` | الـ `i2c_dev_list` — قائمة كل devices |
| `adap->bus_lock` | `rt_mutex` | الـ I2C bus نفسه — منع تعارض الـ transfers |
| `adap->mux_lock` | `mutex` | الـ mux adapters لو موجودة |

#### ترتيب الـ Locks (Lock Ordering)

```
i2c_dev_lock   (spinlock — أقصر scope ممكن)
    │
    ▼ (يتحرر قبل)
adap->bus_lock  (rt_mutex — يُحجز طول فترة الـ transfer)
    │
    ▼ (اختياري)
adap->mux_lock  (mutex — لو الـ adapter عليه mux)
```

**قاعدة مهمة:** الـ `copy_from_user` / `copy_to_user` بتحصل **خارج** `bus_lock` — مينفعش تمسك lock وتتكلم مع userspace memory لأن ممكن تحصل page fault وتتعلق.

#### خلاصة الـ locking في `I2C_RDWR`

```
i2cdev_ioctl_rdwr()
    │
    ├── copy_from_user(...)              ← بدون أي lock
    │
    ├── i2c_transfer()
    │       └── rt_mutex_lock(&adap->bus_lock)
    │               └── master_xfer()   ← hardware access
    │           rt_mutex_unlock(&adap->bus_lock)
    │
    └── copy_to_user(...)               ← بدون أي lock
```
## Phase 4: شرح الـ Functions

> الـ `include/linux/i2c-dev.h` هو header بسيط جداً — بيعمل include لـ UAPI header ويعرّف الـ major number بس. الـ core الحقيقي هو `include/uapi/linux/i2c-dev.h` اللي بيعرّف الـ ioctl commands والـ data structures اللي بتتيح لـ userspace يتكلم مع I2C adapters عبر `/dev/i2c-X`.

---

### ملخص: الـ Macros والـ Structures (Cheatsheet)

#### الـ ioctl Commands

| Macro | Value | الـ Parameter Type | الغرض |
|---|---|---|---|
| `I2C_RETRIES` | `0x0701` | `unsigned long` | عدد مرات polling الـ device لما ميرد |
| `I2C_TIMEOUT` | `0x0702` | `unsigned long` | timeout بوحدة 10ms |
| `I2C_SLAVE` | `0x0703` | `unsigned long` | تحديد slave address (7-bit أو 10-bit) |
| `I2C_SLAVE_FORCE` | `0x0706` | `unsigned long` | تحديد slave address حتى لو driver تاني شايله |
| `I2C_TENBIT` | `0x0704` | `unsigned long` | تفعيل 10-bit addressing (0 = 7-bit) |
| `I2C_FUNCS` | `0x0705` | `unsigned long *` | جلب الـ functionality mask للـ adapter |
| `I2C_RDWR` | `0x0707` | `struct i2c_rdwr_ioctl_data *` | Combined R/W transfers بـ STOP واحد |
| `I2C_PEC` | `0x0708` | `unsigned long` | تفعيل Packet Error Checking مع SMBus |
| `I2C_SMBUS` | `0x0720` | `struct i2c_smbus_ioctl_data *` | SMBus transaction |

#### الـ Data Structures

| Structure | الغرض |
|---|---|
| `struct i2c_smbus_ioctl_data` | بيانات الـ SMBus transaction |
| `struct i2c_rdwr_ioctl_data` | مصفوفة من i2c_msg للـ combined transfer |
| `struct i2c_msg` | (من uapi/linux/i2c.h) segment واحد من الـ I2C transaction |
| `union i2c_smbus_data` | (من uapi/linux/i2c.h) بيانات SMBus: byte أو word أو block |

#### الـ Constants المهمة

| Macro | Value | المعنى |
|---|---|---|
| `I2C_MAJOR` | `89` | الـ major number للـ char device `/dev/i2c-X` |
| `I2C_RDWR_IOCTL_MAX_MSGS` | `42` | أقصى عدد i2c_msg في transfer واحد |
| `I2C_SMBUS_BLOCK_MAX` | `32` | أقصى حجم block data بالـ bytes (SMBus standard) |

---

### الـ i2c_msg Flags (من uapi/linux/i2c.h)

| Flag | Value | الشرط | المعنى |
|---|---|---|---|
| `I2C_M_RD` | `0x0001` | دايماً | قراءة من slave |
| `I2C_M_TEN` | `0x0010` | `I2C_FUNC_10BIT_ADDR` | 10-bit address |
| `I2C_M_DMA_SAFE` | `0x0200` | kernel only | الـ buffer آمن للـ DMA |
| `I2C_M_RECV_LEN` | `0x0400` | `I2C_FUNC_SMBUS_READ_BLOCK_DATA` | أول byte هو الـ length |
| `I2C_M_NO_RD_ACK` | `0x0800` | `I2C_FUNC_PROTOCOL_MANGLING` | skip ACK/NACK في القراءة |
| `I2C_M_IGNORE_NAK` | `0x1000` | `I2C_FUNC_PROTOCOL_MANGLING` | تجاهل الـ NAK |
| `I2C_M_REV_DIR_ADDR` | `0x2000` | `I2C_FUNC_PROTOCOL_MANGLING` | عكس Rd/Wr bit |
| `I2C_M_NOSTART` | `0x4000` | `I2C_FUNC_NOSTART` | حذف الـ repeated START |
| `I2C_M_STOP` | `0x8000` | `I2C_FUNC_PROTOCOL_MANGLING` | إجبار STOP بعد الـ message |

---

### Category 1: Device Configuration ioctls

هاي الـ ioctls بتتحكم في إعدادات الـ I2C adapter نفسه قبل ما نبدأ أي transfer. الـ userspace بتبعتهم على `/dev/i2c-X` file descriptor.

---

#### `I2C_SLAVE` / `I2C_SLAVE_FORCE`

```c
/* في userspace */
ioctl(fd, I2C_SLAVE, 0x50);       /* set slave address = 0x50 */
ioctl(fd, I2C_SLAVE_FORCE, 0x50); /* force حتى لو driver آخر شايله */
```

**الـ `I2C_SLAVE`** بيحدد الـ slave address اللي هيتكلم معاه الـ userspace في الـ transfers الجاية. لو الـ address بيستخدمه kernel driver تاني، بيرجع `-EBUSY`.

**الـ `I2C_SLAVE_FORCE`** نفس الوظيفة لكن بيتجاوز فحص الـ `-EBUSY` — خطير لأنه ممكن يتعارض مع kernel driver تاني بيتكلم مع نفس الـ device.

**Parameters:**
- الـ `arg` في `ioctl()`: قيمة الـ slave address — 7-bit أو 10-bit (لو `I2C_TENBIT` مفعّل)

**Return:** `0` نجاح، `-EINVAL` لو الـ address خارج المدى، `-EBUSY` (لـ `I2C_SLAVE` فقط)

**Key details:** الـ i2c-dev driver بيخزن الـ address في `struct i2c_client` مؤقت جوا الـ `i2cdev_state`. مش بيبعت أي إشارة على الـ bus.

---

#### `I2C_TENBIT`

```c
ioctl(fd, I2C_TENBIT, 1); /* enable 10-bit addressing */
ioctl(fd, I2C_TENBIT, 0); /* back to 7-bit (default) */
```

بيحدد هل الـ slave address هيُفسَّر كـ 7-bit أو 10-bit. لازم الـ adapter يدعم `I2C_FUNC_10BIT_ADDR` قبل الاستخدام.

**Parameters:**
- `arg`: `0` للـ 7-bit، أي قيمة تانية للـ 10-bit

**Return:** `0` نجاح، `-EINVAL` لو الـ adapter مش داعم 10-bit وحاولت تفعّله

---

#### `I2C_RETRIES`

```c
ioctl(fd, I2C_RETRIES, 3); /* حاول 3 مرات لو الـ slave ما رد */
```

بيحدد عدد مرات إعادة المحاولة لما الـ slave device ما يعمل ACK. بيتحكم في الـ `i2c_adapter.retries` field مباشرة.

**Parameters:**
- `arg`: عدد المحاولات (unsigned long)

**Return:** `0` دايماً

---

#### `I2C_TIMEOUT`

```c
ioctl(fd, I2C_TIMEOUT, 20); /* 200ms timeout (20 × 10ms) */
```

بيضبط الـ timeout للـ adapter بوحدة 10ms. بيعدّل `i2c_adapter.timeout` اللي بيستخدمها الـ adapter driver داخلياً.

**Parameters:**
- `arg`: قيمة الـ timeout بوحدة 10ms

**Return:** `0` دايماً

---

#### `I2C_PEC`

```c
ioctl(fd, I2C_PEC, 1); /* enable Packet Error Checking */
```

بيفعّل/يعطّل الـ **Packet Error Checking** في الـ SMBus transactions. الـ PEC عبارة عن CRC-8 checksum بيتحسب على كل الـ bytes في الـ transaction ويتبعت في آخره.

**Parameters:**
- `arg`: `!= 0` تفعيل، `0` تعطيل

**Return:** `0` نجاح، `-ENOTTY` لو الـ adapter ما يدعم PEC

---

#### `I2C_FUNCS`

```c
unsigned long funcs;
ioctl(fd, I2C_FUNCS, &funcs);
if (funcs & I2C_FUNC_SMBUS_READ_BYTE_DATA) {
    /* adapter supports SMBus byte read */
}
```

بيرجع الـ **functionality bitmask** للـ adapter. الـ userspace لازم يستعلم عن الـ capabilities قبل ما يستخدم features معينة.

**Parameters:**
- `arg`: pointer لـ `unsigned long` — بيُكتب فيها الـ mask

**Return:** `0` نجاح، `-EFAULT` لو الـ pointer إنفالد

**Key details:** القيم المرجعة من `I2C_FUNC_*` flags (مُعرّفة في `uapi/linux/i2c.h`). الـ i2c-dev بيقرأها من `i2c_get_functionality(adapter)`.

---

### Category 2: Data Transfer ioctls

---

#### `I2C_RDWR` — Combined R/W Transfer

```c
/* Signature (userspace استخدام) */
int ioctl(int fd, unsigned long I2C_RDWR, struct i2c_rdwr_ioctl_data *data);
```

**الـ `struct i2c_rdwr_ioctl_data`:**

```c
struct i2c_rdwr_ioctl_data {
    struct i2c_msg __user *msgs;  /* array of i2c_msg segments */
    __u32 nmsgs;                  /* number of messages in array */
};
```

الـ `I2C_RDWR` هو الـ ioctl الأقوى في الـ i2c-dev interface. بيسمح بإرسال مصفوفة من الـ `i2c_msg` segments في transaction واحدة — كل segment ممكن يكون read أو write، وبينهم repeated STARTs، وفي النهاية STOP واحد بس.

مثال: كتابة register address ثم قراءة البيانات منه (write-then-read):

```c
struct i2c_msg msgs[2];
struct i2c_rdwr_ioctl_data rdwr;
__u8 reg = 0x10;
__u8 buf[4];

/* message 1: write register address */
msgs[0].addr  = 0x50;
msgs[0].flags = 0;          /* write */
msgs[0].len   = 1;
msgs[0].buf   = &reg;

/* message 2: read data */
msgs[1].addr  = 0x50;
msgs[1].flags = I2C_M_RD;  /* read */
msgs[1].len   = 4;
msgs[1].buf   = buf;

rdwr.msgs  = msgs;
rdwr.nmsgs = 2;

ioctl(fd, I2C_RDWR, &rdwr);
```

**الـ i2c-dev kernel side flow (مبسّط):**

```
ioctl(I2C_RDWR)
  │
  ├─ copy_from_user(i2c_rdwr_ioctl_data)
  ├─ validate: nmsgs <= I2C_RDWR_IOCTL_MAX_MSGS (42)
  ├─ for each msg:
  │   ├─ copy_from_user(i2c_msg header)
  │   ├─ validate len (I2C_M_RECV_LEN: max I2C_SMBUS_BLOCK_MAX + 2)
  │   └─ copy_from_user(msg->buf) إذا write
  ├─ i2c_transfer(adapter, msgs, nmsgs)
  └─ for each read msg: copy_to_user(msg->buf)
```

**Parameters:**
- `msgs`: مصفوفة من `struct i2c_msg`، كل message بتحدد: slave address، flags، length، data buffer
- `nmsgs`: عدد الـ messages — أقصى قيمة `I2C_RDWR_IOCTL_MAX_MSGS = 42`

**Return:** عدد الـ messages المنفّذة نجاحاً، أو قيمة سالبة للخطأ

**Key details:**
- الـ `__user` annotations ضرورية — الـ kernel بيعمل `copy_from_user` / `copy_to_user` بشكل صريح
- الـ `I2C_M_DMA_SAFE` ما يتستخدم من userspace — الـ kernel بيعمل copy للـ buffers تلقائياً
- الـ `I2C_RDWR_IOCTL_MAX_MSGS = 42` بيمنع stack overflow عند تخصيص مصفوفة كبيرة في الـ kernel
- الـ `I2C_RDRW_IOCTL_MAX_MSGS` هو alias قديم بسبب typo — محفوظ للـ backward compatibility

---

#### `I2C_SMBUS` — SMBus Transaction

```c
/* Signature (userspace استخدام) */
int ioctl(int fd, unsigned long I2C_SMBUS, struct i2c_smbus_ioctl_data *data);
```

**الـ `struct i2c_smbus_ioctl_data`:**

```c
struct i2c_smbus_ioctl_data {
    __u8 read_write;               /* I2C_SMBUS_READ=1 أو I2C_SMBUS_WRITE=0 */
    __u8 command;                  /* register/command byte */
    __u32 size;                    /* transaction type */
    union i2c_smbus_data __user *data; /* بيانات التبادل */
};
```

**الـ `union i2c_smbus_data`:**

```c
union i2c_smbus_data {
    __u8  byte;                         /* single byte */
    __u16 word;                         /* 16-bit word */
    __u8  block[I2C_SMBUS_BLOCK_MAX+2]; /* block[0] = length */
};
```

الـ `I2C_SMBUS` بيوفّر SMBus protocol transfers المُهيكلة. SMBus فوق I2C — بيضيف command bytes وأنواع transactions محددة. الـ i2c-dev بيترجم الـ ioctl لـ `i2c_smbus_xfer()` kernel call.

**أنواع الـ SMBus transactions (حقل `size`):**

| قيمة `size` | Macro | الوصف |
|---|---|---|
| `0` | `I2C_SMBUS_QUICK` | Send/Receive bit فقط |
| `1` | `I2C_SMBUS_BYTE` | Read/Write single byte |
| `2` | `I2C_SMBUS_BYTE_DATA` | Write command + Read/Write byte |
| `3` | `I2C_SMBUS_WORD_DATA` | Write command + Read/Write word |
| `4` | `I2C_SMBUS_PROC_CALL` | Write word, Read word |
| `5` | `I2C_SMBUS_BLOCK_DATA` | Write/Read block (max 32 bytes) |
| `7` | `I2C_SMBUS_BLOCK_PROC_CALL` | SMBus 2.0: Write block, Read block |
| `8` | `I2C_SMBUS_I2C_BLOCK_DATA` | I2C-style block |

مثال: قراءة byte data من register:

```c
struct i2c_smbus_ioctl_data args;
union i2c_smbus_data data;

args.read_write = I2C_SMBUS_READ;  /* 1 */
args.command    = 0x10;            /* register address */
args.size       = I2C_SMBUS_BYTE_DATA; /* 2 */
args.data       = &data;

ioctl(fd, I2C_SMBUS, &args);
printf("value = 0x%02x\n", data.byte);
```

**الـ kernel flow (مبسّط):**

```
ioctl(I2C_SMBUS)
  │
  ├─ copy_from_user(i2c_smbus_ioctl_data)
  ├─ validate: read_write ∈ {0,1}, size ∈ valid range
  ├─ allocate local union i2c_smbus_data
  ├─ if write: copy_from_user(args.data → local_data)
  ├─ i2c_smbus_xfer(adapter, addr, flags, rw, cmd, size, &local_data)
  └─ if read: copy_to_user(local_data → args.data)
```

**Parameters:**
- `read_write`: `I2C_SMBUS_READ` (=1) أو `I2C_SMBUS_WRITE` (=0)
- `command`: الـ register أو command byte — بيتبعت كأول byte بعد الـ address
- `size`: نوع الـ transaction من الجدول فوق
- `data`: pointer لـ `union i2c_smbus_data` — الـ kernel بيقرأ منها أو بيكتب فيها حسب الـ `read_write`

**Return:** `0` نجاح، قيمة سالبة للخطأ (`-EOPNOTSUPP`, `-EINVAL`, `-ETIMEDOUT`, إلخ)

**Key details:**
- الـ PEC بيتحسب تلقائياً لو `I2C_PEC` مفعّل
- الـ `I2C_SMBUS_BLOCK_DATA` read: `block[0]` بيرجّع فيها الـ length، الـ data في `block[1..]`
- الـ adapter ممكن يـemulate الـ SMBus عبر raw I2C transfers لو مفيش hardware SMBus support

---

### Category 3: الـ Kernel-Internal Macros والـ Constants

---

#### `I2C_MAJOR`

```c
/* من include/linux/i2c-dev.h */
#define I2C_MAJOR  89
```

الـ **major device number** للـ i2c-dev char devices. لما الـ i2c-dev module بيتسجّل، بيطلب minor numbers ديناميكياً لكل I2C adapter، لكن الـ major number ثابت = 89. ده بيخلّي `/dev/i2c-0`, `/dev/i2c-1`, إلخ تتعرّف بـ:

```
crw-rw---- 1 root i2c 89, 0 /dev/i2c-0
crw-rw---- 1 root i2c 89, 1 /dev/i2c-1
```

**من يستخدمه:** الـ i2c-dev kernel driver (`drivers/i2c/i2c-dev.c`) عند استدعاء `register_chrdev()` أو `cdev_add()`. الـ udev/mdev بيستخدم الـ major number ده لإنشاء الـ device nodes تلقائياً.

---

#### `I2C_RDWR_IOCTL_MAX_MSGS` و `I2C_RDRW_IOCTL_MAX_MSGS`

```c
#define I2C_RDWR_IOCTL_MAX_MSGS   42
/* Originally defined with a typo, keep it for compatibility */
#define I2C_RDRW_IOCTL_MAX_MSGS   I2C_RDWR_IOCTL_MAX_MSGS
```

بيحدد الحد الأقصى لعدد الـ `i2c_msg` في أي `I2C_RDWR` ioctl call. القيمة 42 مش عشوائية — الـ i2c-dev driver بيعمل `kmalloc` لمصفوفة من الـ `i2c_msg` على الـ heap، والحد ده بيمنع exhaustion غير مقصود للـ kernel memory.

الـ `I2C_RDRW_IOCTL_MAX_MSGS` موجود فقط للـ backward compatibility — في نسخ قديمة كان مكتوب بـ typo ("RDRW" بدل "RDWR")، وكتير من الـ userspace programs القديمة بتستخدم النسخة الغلطة.

---

#### `I2C_SMBUS_BLOCK_MAX`

```c
#define I2C_SMBUS_BLOCK_MAX  32  /* As specified in SMBus standard */
```

الـ SMBus standard بيحدد maximum block size = 32 bytes. الـ `union i2c_smbus_data.block` حجمها `I2C_SMBUS_BLOCK_MAX + 2`:
- `block[0]`: الـ length byte (بيتبعت/بيتستقبل من الـ slave)
- `block[1..32]`: الـ actual data
- `block[33]`: byte إضافي للـ PEC (عشان userspace compatibility)

---

### Architecture Overview: كيف بيشتغل الـ i2c-dev Interface

```
Userspace                    Kernel (i2c-dev driver)            I2C Core
─────────                    ────────────────────────           ────────
open("/dev/i2c-0")           i2cdev_open()
                              └─ alloc i2cdev_state
                              └─ get i2c_adapter ref

ioctl(I2C_SLAVE, 0x50)  ──►  i2cdev_ioctl()
                              └─ store addr in state

ioctl(I2C_RDWR, &rdwr)  ──►  i2cdev_ioctl()
                              └─ i2cdev_ioctl_rdwr()
                                  └─ copy_from_user(msgs)
                                  └─ i2c_transfer()       ──►  adapter->master_xfer()

ioctl(I2C_SMBUS, &smb)  ──►  i2cdev_ioctl()
                              └─ i2cdev_ioctl_smbus()
                                  └─ copy_from_user(data)
                                  └─ i2c_smbus_xfer()     ──►  adapter->smbus_xfer()
                                                               أو emulated via master_xfer()

close(fd)                    i2cdev_release()
                              └─ put i2c_adapter ref
```

---

### الـ I2C_FUNC_* Capability Flags (من uapi/linux/i2c.h)

**الـ Functionality Groups:**

```c
/* Basic I2C */
I2C_FUNC_I2C                  /* 0x00000001 - raw I2C transfers */
I2C_FUNC_10BIT_ADDR           /* 0x00000002 - 10-bit addressing */
I2C_FUNC_PROTOCOL_MANGLING    /* 0x00000004 - non-standard protocol tricks */
I2C_FUNC_NOSTART              /* 0x00000010 - I2C_M_NOSTART support */
I2C_FUNC_SLAVE                /* 0x00000020 - slave mode */

/* SMBus */
I2C_FUNC_SMBUS_QUICK          /* 0x00010000 */
I2C_FUNC_SMBUS_BYTE           /* READ | WRITE byte */
I2C_FUNC_SMBUS_BYTE_DATA      /* READ | WRITE byte_data */
I2C_FUNC_SMBUS_WORD_DATA      /* READ | WRITE word_data */
I2C_FUNC_SMBUS_BLOCK_DATA     /* READ | WRITE block_data */
I2C_FUNC_SMBUS_I2C_BLOCK      /* READ | WRITE i2c_block */
I2C_FUNC_SMBUS_PEC            /* 0x00000008 - PEC support */

/* Compound emulation groups */
I2C_FUNC_SMBUS_EMUL           /* كل الـ SMBus basic types + PEC */
I2C_FUNC_SMBUS_EMUL_ALL       /* + BLOCK_DATA + BLOCK_PROC_CALL */
```

**الـ `I2C_FUNC_SMBUS_EMUL`** هو الـ bitmask الأكثر استخداماً — الـ adapters اللي بتعمل emulate للـ SMBus عبر raw I2C بترجع الـ mask ده. أي adapter عنده `I2C_FUNC_I2C` يقدر يـemulate كل الـ SMBus transactions تقريباً.

---

### ملاحظات مهمة للـ Embedded Developer

**1. الـ `I2C_SLAVE` vs `I2C_SLAVE_FORCE`:**
لو شغّال على production system، لازم تتأكد إن الـ address مش مستخدم من kernel driver قبل ما تستخدم `I2C_SLAVE_FORCE` — ممكن يسبب data corruption لو kernel driver بيكتب على نفس الـ device.

**2. الـ `I2C_RDWR` أكفأ من SMBus:**
لو الـ adapter يدعم `I2C_FUNC_I2C`، استخدم `I2C_RDWR` بدل `I2C_SMBUS` لأنه أكثر مرونة ويقلل الـ overhead — ممكن تعمل write-then-read في syscall واحد.

**3. فحص الـ `I2C_FUNCS` أول شيء:**
```c
unsigned long funcs;
ioctl(fd, I2C_FUNCS, &funcs);
assert(funcs & I2C_FUNC_I2C); /* أو I2C_FUNC_SMBUS_* حسب الاحتياج */
```

**4. الـ `I2C_M_RECV_LEN` للـ block reads:**
لما تقرأ block data وأنت مش عارف الـ length مسبقاً، ضبط `msg.len = 1` وحط `I2C_M_RECV_LEN` في الـ flags — الـ driver هيقرأ أول byte كـ length ويكمل القراءة. الـ buffer لازم يكون على الأقل `I2C_SMBUS_BLOCK_MAX + 1` bytes.
## Phase 5: دليل الـ Debugging الشامل

الـ `i2c-dev` subsystem بيوفر **character device interface** (`/dev/i2c-X`) للـ userspace عشان يتكلم مع الـ I2C slaves مباشرة عبر ioctls زي `I2C_RDWR` و`I2C_SMBUS`. الـ debugging بتاعه بينقسم لـ software level وـ hardware level.

---

### Software Level

#### 1. debugfs Entries

الـ **debugfs** بيكشف معلومات مفيدة عن الـ I2C adapters والـ devices:

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل entries خاصة بـ I2C
ls /sys/kernel/debug/i2c-*/

# اقرأ الـ fault injector (لو CONFIG_I2C_FAULT_INJECTOR=y)
ls /sys/kernel/debug/i2c-0/fault-injector/
cat /sys/kernel/debug/i2c-0/fault-injector/corrupt_nak
```

| Entry | المعنى |
|-------|--------|
| `/sys/kernel/debug/i2c-N/fault-injector/` | inject فشل صناعي في الـ bus |
| `/sys/kernel/debug/i2c-N/fault-injector/corrupt_nak` | اجبر NACK على الـ transfer |
| `/sys/kernel/debug/i2c-N/fault-injector/lose_arbitration` | simulate arbitration loss |
| `/sys/kernel/debug/i2c-N/fault-injector/inject_panic` | trigger kernel panic عند transfer |

#### 2. sysfs Entries

الـ **sysfs** بيعطيك حالة الـ adapter والـ devices المربوطة:

```bash
# قائمة كل I2C adapters في النظام
ls /sys/class/i2c-adapter/

# معلومات adapter معين
cat /sys/class/i2c-adapter/i2c-0/name

# الـ devices المربوطة على الـ bus
ls /sys/bus/i2c/devices/

# معلومات device معينة (مثلاً sensor على address 0x48)
cat /sys/bus/i2c/devices/0-0048/name
cat /sys/bus/i2c/devices/0-0048/modalias

# الـ driver المرتبط بالـ device
ls -l /sys/bus/i2c/devices/0-0048/driver

# timeout وretries الخاصة بالـ adapter
cat /sys/class/i2c-adapter/i2c-0/timeout
cat /sys/class/i2c-adapter/i2c-0/retries
```

#### 3. ftrace — Tracepoints والـ Events

الـ **ftrace** بيوفر tracepoints خاصة بالـ I2C:

```bash
# شوف كل I2C tracepoints المتاحة
ls /sys/kernel/debug/tracing/events/i2c/

# فعّل كل I2C events
echo 1 > /sys/kernel/debug/tracing/events/i2c/enable

# أو فعّل events محددة
echo 1 > /sys/kernel/debug/tracing/events/i2c/i2c_write/enable
echo 1 > /sys/kernel/debug/tracing/events/i2c/i2c_read/enable
echo 1 > /sys/kernel/debug/tracing/events/i2c/i2c_reply/enable
echo 1 > /sys/kernel/debug/tracing/events/i2c/i2c_result/enable
echo 1 > /sys/kernel/debug/tracing/events/i2c/smbus_write/enable
echo 1 > /sys/kernel/debug/tracing/events/i2c/smbus_read/enable
echo 1 > /sys/kernel/debug/tracing/events/i2c/smbus_reply/enable
echo 1 > /sys/kernel/debug/tracing/events/i2c/smbus_result/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل التطبيق اللي بيستخدم /dev/i2c-X
./my_i2c_app

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

مثال على output:

```
my_app-1234  [000] .... i2c_write: i2c-0 #0 a=048 f=0000 l=1 [00]
my_app-1234  [000] .... i2c_read:  i2c-0 #1 a=048 f=0001 l=2
my_app-1234  [000] .... i2c_reply: i2c-0 #1 a=048 f=0001 l=2 [1a 00]
my_app-1234  [000] .... i2c_result: i2c-0 n=2 ret=2
```

**التفسير:**
- `a=048` → الـ slave address (hex)
- `f=0000` → flags (0x0001 = I2C_M_RD)
- `l=1` → عدد البايتات
- `ret=2` → عدد الـ messages اللي اتعملت بنجاح

#### 4. printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لكل i2c-dev messages
echo 'module i2c_dev +p' > /sys/kernel/debug/dynamic_debug/control

# أو لـ i2c core
echo 'module i2c_core +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل debug في i2c subsystem (verbose جداً)
echo 'file drivers/i2c/* +p' > /sys/kernel/debug/dynamic_debug/control

# شوف اللي فعّلته
grep i2c /sys/kernel/debug/dynamic_debug/control

# level الـ kernel messages
dmesg -w | grep i2c
```

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_I2C_DEBUG_CORE` | يطبع debug messages من i2c core |
| `CONFIG_I2C_DEBUG_ALGO` | debug للـ bit-banging algorithms |
| `CONFIG_I2C_DEBUG_BUS` | debug لكل bus transactions |
| `CONFIG_I2C_FAULT_INJECTOR` | يضيف debugfs interface لـ fault injection |
| `CONFIG_I2C_SLAVE` | يفعّل slave mode support |
| `CONFIG_I2C_STUB` | virtual I2C adapter للـ testing بدون hardware |
| `CONFIG_I2C_SMBUS` | SMBus emulation layer |
| `CONFIG_DEBUG_FS` | مطلوب لـ debugfs entries |
| `CONFIG_TRACING` | مطلوب لـ ftrace tracepoints |
| `CONFIG_I2C_CHARDEV` | الـ /dev/i2c-X interface نفسه |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'I2C_DEBUG|I2C_FAULT|I2C_STUB'
# أو
grep -E 'I2C_DEBUG|I2C_FAULT|I2C_STUB' /boot/config-$(uname -r)
```

#### 6. أدوات الـ Userspace المتخصصة

**i2c-tools** هي الأداة الأساسية للتفاعل مع `/dev/i2c-X`:

```bash
# ثبّت i2c-tools
apt install i2c-tools   # Debian/Ubuntu
dnf install i2c-tools   # Fedora

# اعرض كل I2C buses
i2cdetect -l

# scan لكل devices على bus 0 (بيستخدم I2C_FUNC_SMBUS_QUICK internally)
i2cdetect -y 0

# scan بـ read byte بدلاً من quick write (أأمن لبعض الـ devices)
i2cdetect -y -r 0

# اقرأ byte من register 0x00 في device على address 0x48
i2cget -y 0 0x48 0x00

# اكتب byte
i2cset -y 0 0x48 0x01 0xFF

# dump كل registers (0x00 - 0xFF)
i2cdump -y 0 0x48

# تحقق من الـ functionality المدعومة من الـ adapter
i2cdetect -F 0
```

مثال على output لـ `i2cdetect -y 0`:

```
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- 48 -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
```

**الـ `48`** معناه في device موجود على address `0x48`.

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|----------------------|--------|------|
| `i2c i2c-0: adapter timed out` | الـ adapter ما ردش في الوقت المحدد | تحقق من الـ clock speed — زوّد `I2C_TIMEOUT` عبر ioctl |
| `i2c-dev: i2c-%d is not a valid interface` | رقم الـ bus غلط أو الـ adapter مش موجود | تحقق بـ `ls /dev/i2c-*` و`i2cdetect -l` |
| `i2c i2c-0: slave 0x48 not found` | الـ device مش بيرد على الـ address | تحقق من الـ hardware connection والـ address |
| `i2c i2c-0: message too long` | `i2c_msg.len` تجاوز حد الـ adapter | قسّم الـ transfer لـ messages أصغر |
| `i2c i2c-0: NAK from device` | الـ slave رفض الـ transaction | تحقق من الـ address والـ register والـ device state |
| `i2c-dev: combined message not supported` | الـ adapter مش بيدعم `I2C_FUNC_I2C` | استخدم SMBus operations بدلاً منها |
| `i2c i2c-0: bus seems to be busy` | الـ bus مش في حالة IDLE (SDA أو SCL = 0) | reset الـ bus أو افصل الـ device المشكلة |
| `i2c i2c-0: arbitration lost` | collision على الـ bus (multi-master) | تحقق من الـ bus topology |
| `i2c-dev: address not allowed` | محاولة access لـ reserved address | استخدم `I2C_SLAVE_FORCE` بحذر |
| `EREMOTEIO (-121)` | الـ slave أرسل NACK بعد الـ address byte | الـ device مش موجود أو مش مستعد |
| `ETIMEDOUT (-110)` | انتهى الـ timeout | زوّد timeout أو تحقق من الـ hardware |
| `ENXIO (-6)` | الـ adapter مش موجود أو الـ bus مش مهيّأ | تحقق من `modprobe i2c_dev` وـ DT |

#### 8. نقاط استراتيجية لـ dump_stack() وـ WARN_ON()

```c
/* في i2c-dev.c — عند فشل ioctl */
static long i2cdev_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct i2c_client *client = file->private_data;

    /* تحقق من صحة الـ client */
    if (WARN_ON(!client || !client->adapter))
        return -ENODEV;

    switch (cmd) {
    case I2C_RDWR: {
        struct i2c_rdwr_ioctl_data rdwr_arg;
        if (copy_from_user(&rdwr_arg, (struct i2c_rdwr_ioctl_data __user *)arg,
                           sizeof(rdwr_arg)))
            return -EFAULT;

        /* تحقق من الحد الأقصى */
        if (WARN_ON(rdwr_arg.nmsgs > I2C_RDWR_IOCTL_MAX_MSGS))
            return -EINVAL;
        break;
    }
    case I2C_SMBUS: {
        struct i2c_smbus_ioctl_data data_arg;
        if (copy_from_user(&data_arg, (struct i2c_smbus_ioctl_data __user *)arg,
                           sizeof(data_arg)))
            return -EFAULT;

        /* size يجب أن يكون valid */
        if (WARN_ON(data_arg.size > I2C_SMBUS_I2C_BLOCK_DATA)) {
            dump_stack(); /* طباعة الـ call stack عشان نعرف مين بيبعت الطلب */
            return -EINVAL;
        }
        break;
    }
    }
    /* ... */
}
```

**أماكن مهمة للـ WARN_ON:**
- بعد `copy_from_user` لـ `i2c_rdwr_ioctl_data` → تحقق من `nmsgs <= I2C_RDWR_IOCTL_MAX_MSGS`
- في `i2c_transfer` → تحقق من أن الـ adapter مش NULL
- عند فشل `i2c_master_xfer` مع ret < 0 → log الـ error

---

### Hardware Level

#### 1. التحقق من أن الـ Hardware State يطابق الـ Kernel State

```bash
# خطوة 1: تحقق من الـ kernel يشوف الـ adapter
i2cdetect -l
# Output: i2c-0    i2c        DesignWare HDMI                  I2C adapter

# خطوة 2: تحقق من الـ devices المكتشفة
i2cdetect -y 0
# لازم تشوف الـ slave address اللي متوقعه

# خطوة 3: قارن مع الـ DT / ACPI
# لو device مش ظاهر → مشكلة hardware أو DT
cat /sys/bus/i2c/devices/0-0048/of_node/compatible

# خطوة 4: تحقق من الـ clock speed
cat /sys/bus/i2c/devices/i2c-0/of_node/clock-frequency
# لازم يطابق الـ device datasheet (100000 = 100kHz, 400000 = 400kHz)
```

#### 2. Register Dump Techniques

```bash
# استخدام devmem2 لقراءة registers الـ I2C controller مباشرة
# مثال: DesignWare I2C على Raspberry Pi (base address 0xFE205000)
devmem2 0xFE205000 w   # IC_CON register
devmem2 0xFE205004 w   # IC_TAR (target address)
devmem2 0xFE20501C w   # IC_INTR_STAT

# استخدام /dev/mem (requires root)
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 0x1000, offset=0xFE205000)
    ic_con = struct.unpack('<I', m[0:4])[0]
    print(f'IC_CON: 0x{ic_con:08x}')
    m.close()
"

# استخدام io utility (من i2c-tools)
io -4 -r 0xFE205000   # قراءة 4 bytes من الـ base address
```

**مثال: DesignWare I2C Registers المهمة:**

| Register | Offset | المعنى |
|----------|--------|--------|
| `IC_CON` | 0x00 | Control register — speed, master mode |
| `IC_TAR` | 0x04 | Target address |
| `IC_DATA_CMD` | 0x10 | Data buffer + read/write command |
| `IC_SS_SCL_HCNT` | 0x14 | Standard speed SCL high count |
| `IC_FS_SCL_HCNT` | 0x1C | Fast speed SCL high count |
| `IC_INTR_STAT` | 0x2C | Interrupt status |
| `IC_STATUS` | 0x70 | Bus status (TFNF, RFNE, etc.) |
| `IC_TXFLR` | 0x74 | TX FIFO level |
| `IC_RXFLR` | 0x78 | RX FIFO level |
| `IC_ABRT_SOURCE` | 0x80 | Abort source (NACK, ARB_LOST, etc.) |

#### 3. Logic Analyzer والـ Oscilloscope

**نقاط القياس:**

```
Raspberry Pi I2C:
  SDA ← GPIO2 (Pin 3)
  SCL ← GPIO3 (Pin 5)
  GND ← Pin 6

Pull-up resistors:
  لـ 100kHz: 4.7kΩ إلى VCC
  لـ 400kHz: 2.2kΩ إلى VCC
  لـ 1MHz:   1kΩ إلى VCC
```

**أشياء تتحقق منها على الـ scope:**

```
1. Rise time للـ SDA/SCL:
   - 100kHz: يجب أن يكون < 1μs
   - 400kHz: يجب أن يكون < 300ns

2. شكل الـ ACK/NACK:
   - ACK: SDA = LOW عند الـ 9th clock pulse
   - NACK: SDA = HIGH عند الـ 9th clock pulse

3. الـ START condition:
   - SDA ينزل بينما SCL = HIGH

4. الـ STOP condition:
   - SDA يرتفع بينما SCL = HIGH

5. Clock stretching:
   - الـ slave بيمسك SCL = LOW
   - الـ master لازم يستنى
```

**Sigrok/PulseView commands:**

```bash
# decode I2C بـ sigrok-cli
sigrok-cli -d fx2lafw \
  --channels D0=SCL,D1=SDA \
  -P i2c:scl=SCL:sda=SDA \
  --time 100ms \
  -O ascii
```

#### 4. Common Hardware Issues وـ Kernel Log Patterns

| مشكلة الـ Hardware | Pattern في الـ Kernel Log | التشخيص والحل |
|-------------------|--------------------------|---------------|
| Pull-up مفقود أو كبير جداً | `i2c i2c-0: timeout waiting for bus ready` | قس الـ SDA/SCL بالـ scope — لازم يكونوا HIGH في idle |
| الـ slave مش مغذّى (no power) | `i2c i2c-0: NACK on address` | تحقق من VCC للـ device |
| كابل طويل جداً (capacitance عالية) | Intermittent `EREMOTEIO` / `ETIMEDOUT` | قصّر الكابل أو استخدم pull-up أصغر |
| Multi-master collision | `i2c i2c-0: arbitration lost` | تحقق من الـ bus topology — مين اللي بيتكلم بالتوازي |
| الـ slave في حالة خطأ (stuck SDA=0) | `i2c i2c-0: bus is busy` في كل transfer | أرسل 9 clock pulses يدوياً لـ reset الـ slave |
| SCL stuck low | `i2c i2c-0: timeout, scl is low` | fault في الـ slave — افصله وأعد تشغيل الـ adapter |
| Noise على الـ bus | CRC errors في SMBus (مع `I2C_PEC`) | أضف capacitor 100nF على VCC للـ slave |

**Bus stuck recovery:**

```bash
# بعض الـ I2C controllers بيدعموا bus recovery تلقائياً
# تحقق من الـ kernel log
dmesg | grep -i "bus recovery"

# إذا الـ adapter بيدعمه:
# الـ kernel بيبعت 9 SCL pulses تلقائياً عبر i2c_recover_bus()
# لو مش شغّال، أعد تشغيل الـ adapter driver
modprobe -r i2c_designware_platform && modprobe i2c_designware_platform
```

#### 5. Device Tree Debugging

```bash
# شوف الـ DT nodes الخاصة بالـ I2C
ls /sys/firmware/devicetree/base/soc/i2c*/

# اقرأ الـ properties المهمة
cat /sys/firmware/devicetree/base/soc/i2c@fe205000/clock-frequency
# لازم يطابق: 100000 أو 400000 أو 1000000

# تحقق من الـ compatible string
cat /sys/firmware/devicetree/base/soc/i2c@fe205000/compatible
# مثال: snps,designware-i2c

# شوف الـ i2c devices المعرّفة في الـ DT
ls /sys/firmware/devicetree/base/soc/i2c@fe205000/

# قارن مع الـ devices المكتشفة فعلاً
i2cdetect -y 0

# decode الـ DTB
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "i2c@"
```

**DT مثال صح:**

```dts
/* Device Tree source snippet */
&i2c1 {
    clock-frequency = <400000>;  /* 400kHz fast mode */
    status = "okay";

    /* Temperature sensor على address 0x48 */
    lm75: temperature@48 {
        compatible = "national,lm75";
        reg = <0x48>;
        /* الـ reg لازم يطابق الـ physical address pins */
    };
};
```

```bash
# تحقق من أن الـ reg في DT يطابق الـ scan
i2cdetect -y 1
# لازم تشوف 0x48

# لو الـ address في DT 0x48 بس scan بيلاقيه على 0x49
# → خطأ في DT أو hardware addressing
```

---

### Practical Commands — Ready-to-Copy

#### سيناريو 1: الـ /dev/i2c-X مش موجود

```bash
# خطوة 1: تحقق من الـ module
lsmod | grep i2c_dev
# لو مش موجود:
modprobe i2c_dev
ls /dev/i2c-*

# خطوة 2: تحقق من الـ major number
# لازم يكون 89 (I2C_MAJOR)
ls -la /dev/i2c-*
# crw-rw-r-- 1 root i2c 89, 0 Feb 23 10:00 /dev/i2c-0
#                        ^^ هو الـ major number

# خطوة 3: تحقق من الـ adapter driver
dmesg | grep "i2c"
```

#### سيناريو 2: ioctl يرجع ENODEV أو ENXIO

```bash
# تحقق من وجود الـ device
i2cdetect -y 0

# تحقق من الـ functionality
i2cdetect -F 0
# مثال output:
# Functionalities implemented by /dev/i2c-0:
# I2C                              yes
# SMBus Quick Command              yes
# SMBus Send Byte                  yes
# ...

# لو device مش بيرد → ابدأ بـ SLAVE_FORCE
# في C code:
# ioctl(fd, I2C_SLAVE_FORCE, 0x48);
```

#### سيناريو 3: تتبع I2C_RDWR ioctl كاملاً

```bash
# استخدم strace لتتبع الـ ioctls من userspace
strace -e trace=ioctl -v ./my_i2c_app 2>&1 | grep -A 5 "I2C"

# مثال output:
# ioctl(3, I2C_SLAVE, 0x48)          = 0
# ioctl(3, I2C_RDWR, {msgs=[
#   {addr=0x48, flags=0, len=1, buf=[0x00]},
#   {addr=0x48, flags=I2C_M_RD, len=2, buf=[...]}
# ], nmsgs=2}) = 2

# لو رجع -1 → اقرأ errno
strace -e trace=ioctl,open ./my_i2c_app 2>&1 | grep -E "i2c|EREMOTE|ETIMEDOUT"
```

#### سيناريو 4: فعّل كل الـ I2C tracing في خطوة واحدة

```bash
#!/bin/bash
# i2c_trace.sh — فعّل I2C tracing وسجّل النتيجة

TRACE=/sys/kernel/debug/tracing

# صفّر الـ trace buffer
echo > $TRACE/trace

# فعّل I2C events
for event in i2c_write i2c_read i2c_reply i2c_result \
             smbus_write smbus_read smbus_reply smbus_result; do
    echo 1 > $TRACE/events/i2c/$event/enable 2>/dev/null
done

echo 1 > $TRACE/tracing_on
echo "I2C tracing enabled. Running: $@"
"$@"
echo 0 > $TRACE/tracing_on

echo "=== I2C Trace Output ==="
cat $TRACE/trace
```

```bash
# الاستخدام:
chmod +x i2c_trace.sh
./i2c_trace.sh i2cdump -y 0 0x48
```

#### سيناريو 5: تحقق من الـ PEC (Packet Error Checking)

```bash
# فعّل PEC عبر ioctl في C:
# ioctl(fd, I2C_PEC, 1);

# أو في Python:
python3 << 'EOF'
import fcntl, os

I2C_PEC = 0x0708
I2C_SLAVE = 0x0703

fd = os.open("/dev/i2c-0", os.O_RDWR)
fcntl.ioctl(fd, I2C_SLAVE, 0x48)
fcntl.ioctl(fd, I2C_PEC, 1)   # فعّل PEC
# الآن كل SMBus transactions هيحسب ويتحقق من الـ CRC
os.close(fd)
EOF

# لو في PEC errors:
dmesg | grep -i "pec\|crc"
```

#### سيناريو 6: استخدام i2c-stub للـ Testing بدون Hardware

```bash
# فعّل i2c-stub مع addresses
modprobe i2c-stub chip_addr=0x48,0x50

# هيظهر adapter جديد
i2cdetect -l
# i2c-8    smbus   SMBus stub driver           SMBUS adapter

# اكتب وافتحص بدون hardware
i2cset -y 8 0x48 0x00 0xAB
i2cget -y 8 0x48 0x00
# 0xab
```

#### سيناريو 7: تشخيص Bus Hang كاملاً

```bash
#!/bin/bash
# i2c_diagnose.sh — فحص شامل لـ I2C bus

BUS=${1:-0}
echo "=== I2C Bus $BUS Diagnostic ==="

echo "[1] Adapter info:"
i2cdetect -l | grep "i2c-$BUS"

echo "[2] Functionality:"
i2cdetect -F $BUS 2>&1 | grep -E "I2C|SMBus Quick|10-bit"

echo "[3] Device scan:"
i2cdetect -y $BUS

echo "[4] Kernel messages:"
dmesg | grep -i "i2c-$BUS" | tail -20

echo "[5] sysfs state:"
cat /sys/class/i2c-adapter/i2c-$BUS/name 2>/dev/null

echo "[6] Active tracepoints:"
grep -l "1" /sys/kernel/debug/tracing/events/i2c/*/enable 2>/dev/null
```

```bash
chmod +x i2c_diagnose.sh
./i2c_diagnose.sh 0
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — جهاز I2C محجوز من Kernel Driver

#### العنوان
`I2C_SLAVE_FORCE` لما يكون الـ sensor محجوز من kernel driver

#### السياق
بتشتغل على industrial IoT gateway مبني على **TI AM62x**. الجهاز فيه temperature/humidity sensor **SHT31** على `/dev/i2c-1` بـ address `0x44`. الـ product team عايزة تكتب userspace daemon بـ Python يقرأ القراءات كل ثانية مباشرةً من `/dev/i2c-1`.

#### المشكلة
```bash
$ python3 sensor_daemon.py
OSError: [Errno 16] Device or resource busy
```
الـ daemon مش قادر يفتح الـ slave address. السبب إن kernel module `sht3x` موجود في Device Tree وبيشتغل هو كمان على نفس الـ address.

#### التحليل
لما بتعمل `ioctl(fd, I2C_SLAVE, 0x44)`:
```c
/* من i2c-dev.c في kernel */
/* بيتحقق إن في driver تاني شايل الـ address */
if (i2c_check_addr_validity(addr, msg.flags))
    return -EBUSY;  /* errno 16 */
```
الماكرو اللي بيحدد السلوك ده معرّف في الـ header:
```c
#define I2C_SLAVE       0x0703  /* Use this slave address */
#define I2C_SLAVE_FORCE 0x0706  /* Use this slave address, even if it
                                   is already in use by a driver! */
```
**الـ `I2C_SLAVE`** بيرفض لو في driver شايل الـ address. **الـ `I2C_SLAVE_FORCE`** بيتجاوز الـ check ده.

#### الحل
**الأفضل:** إزالة `sht3x` من Device Tree لو الـ userspace daemon هو المسؤول الوحيد:
```dts
/* احذف أو comment الـ node ده من am62x board .dts */
/*
&i2c1 {
    sht31: humidity@44 {
        compatible = "sensirion,sht3x";
        reg = <0x44>;
    };
};
*/
```

**البديل السريع (للـ debugging فقط):**
```python
import fcntl
I2C_SLAVE_FORCE = 0x0706
fcntl.ioctl(fd, I2C_SLAVE_FORCE, 0x44)  # تجاوز الـ busy check
```

**تحقق إن مفيش kernel driver شايل الـ address:**
```bash
ls /sys/bus/i2c/devices/1-0044/driver
# لو موجود → في driver kernel شايل الـ address
```

#### الدرس المستفاد
**`I2C_SLAVE`** vs **`I2C_SLAVE_FORCE`** فرق مهم جداً في بيئات production. الـ `FORCE` مش للـ production — هو للـ debug والـ bring-up بس. في production، قرر مين المسؤول عن الـ device: kernel driver أو userspace، مش الاتنين.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI EDID فشل بسبب I2C Timeout

#### العنوان
`I2C_TIMEOUT` غلط بيخلي HDMI EDID read يفشل على H616

#### السياق
بتعمل bring-up لـ Android TV box على **Allwinner H616**. الـ HDMI DDC bus (اللي بيقرأ الـ EDID من الشاشة) بيشتغل على `/dev/i2c-5`. بكتبت test tool بـ C يقرأ الـ EDID من الشاشة مباشرةً من userspace للـ debugging.

#### المشكلة
```bash
$ ./read_edid /dev/i2c-5
ioctl I2C_RDWR failed: Connection timed out (errno 110)
```
الـ EDID read بيفشل بـ timeout، لكن `i2cdetect -y 5` بيشوف الـ address `0x50` بدون مشكلة.

#### التحليل
الـ DDC spec بتقول إن الشاشة ممكن تاخد وقت أطول في الـ acknowledge الأول. الـ default timeout في kernel هو **1 جثانية** (100 × 10ms). بس بعض الشاشات — خصوصاً الـ cheap panels — بتاخد أكتر.

من الـ header:
```c
#define I2C_TIMEOUT 0x0702  /* set timeout in units of 10 ms */
```
الـ unit مهم جداً: **القيمة بـ 10ms وحدات**، مش milliseconds ومش seconds.

الكود الأصلي كان:
```c
/* خطأ: ظن إن الـ timeout بالـ ms */
ioctl(fd, I2C_TIMEOUT, 1000);  /* ده في الحقيقة 10 ثانية! */
/* أو */
ioctl(fd, I2C_TIMEOUT, 1);     /* ده 10ms بس — قليل جداً */
```

#### الحل
```c
#include <linux/i2c-dev.h>
#include <sys/ioctl.h>

int setup_i2c_for_edid(int fd)
{
    /* I2C_TIMEOUT unit = 10ms, set 500ms = 50 units */
    if (ioctl(fd, I2C_TIMEOUT, 50) < 0) {
        perror("I2C_TIMEOUT");
        return -1;
    }

    /* retry 3 times before giving up */
    if (ioctl(fd, I2C_RETRIES, 3) < 0) {
        perror("I2C_RETRIES");
        return -1;
    }

    return 0;
}
```

```bash
# تحقق من الـ current timeout
cat /sys/bus/i2c/devices/i2c-5/timeout
# unit هنا بالـ ms في sysfs — مختلف عن الـ ioctl unit!
```

#### الدرس المستفاد
**`I2C_TIMEOUT`** بيستخدم وحدات **10ms** مش ms. ده مصدر bugs شائعة. دايماً اقرأ الـ comment في الـ header بعناية. كمان الـ sysfs interface بيعرض نفس الـ timeout بوحدات مختلفة — متخلطش بينهم.

---

### السيناريو 3: Automotive ECU على i.MX8 — SMBus PEC Verification لـ Safety-Critical Sensor

#### العنوان
تفعيل **`I2C_PEC`** للتحقق من integrity البيانات في automotive ECU

#### السياق
بتشتغل على automotive ECU مبني على **NXP i.MX8**. الـ ECU بيقرأ battery voltage من **BQ76952** battery management IC على الـ SMBus. الـ ISO 26262 functional safety requirements بتقول لازم يكون في error detection على الـ communication bus.

#### المشكلة
بدون **PEC (Packet Error Code)**، لو حصل bit flip في الـ I2C bus بسبب electrical noise، الـ ECU ممكن يقرأ قيمة غلط للـ voltage ويعمل قرار خاطئ.

#### التحليل
الـ SMBus PEC هو CRC-8 byte بيتحسب على كل الـ transaction وبيتبعت في الآخر. الـ controller بيتحقق منه. من الـ header:
```c
#define I2C_PEC   0x0708  /* != 0 to use PEC with SMBus */
#define I2C_SMBUS 0x0720  /* SMBus transfer */

struct i2c_smbus_ioctl_data {
    __u8 read_write;   /* I2C_SMBUS_READ أو I2C_SMBUS_WRITE */
    __u8 command;      /* register address */
    __u32 size;        /* نوع الـ transaction */
    union i2c_smbus_data __user *data;
};
```

#### الحل
```c
#include <linux/i2c-dev.h>
#include <linux/i2c.h>
#include <sys/ioctl.h>
#include <stdint.h>

/* قراءة voltage register من BQ76952 مع PEC */
int read_battery_voltage(int fd, uint16_t *voltage_mv)
{
    struct i2c_smbus_ioctl_data args;
    union i2c_smbus_data data;

    /* تفعيل PEC للـ safety */
    if (ioctl(fd, I2C_PEC, 1) < 0) {
        perror("I2C_PEC enable");
        return -1;
    }

    /* set slave address */
    if (ioctl(fd, I2C_SLAVE, 0x08) < 0) {  /* BQ76952 address */
        perror("I2C_SLAVE");
        return -1;
    }

    args.read_write = I2C_SMBUS_READ;
    args.command    = 0x14;               /* CellVoltage1 register */
    args.size       = I2C_SMBUS_WORD_DATA;
    args.data       = &data;

    if (ioctl(fd, I2C_SMBUS, &args) < 0) {
        /* PEC mismatch → errno = EBADMSG */
        if (errno == EBADMSG)
            syslog(LOG_CRIT, "BQ76952: PEC error — data corrupted!");
        return -1;
    }

    *voltage_mv = data.word;
    return 0;
}
```

```bash
# تحقق إن الـ adapter بيدعم PEC
i2cdetect -F 1 | grep PEC
# Output: PEC                                yes
```

#### الدرس المستفاد
**`I2C_PEC`** مش بس feature اختيارية — في safety-critical systems هي requirement. لو الـ adapter مش بيدعمها، `ioctl(I2C_FUNCS)` بيرجعلك الـ capabilities قبل ما تحاول تفعّلها. تحقق دايماً من الـ `I2C_FUNC_SMBUS_PEC` flag.

---

### السيناريو 4: IoT Sensor Board على STM32MP1 — Combined R/W بـ I2C_RDWR بدون STOP

#### العنوان
استخدام `I2C_RDWR` لـ register read بـ repeated START على STM32MP1

#### السياق
بتعمل IoT sensor hub على **STM32MP157**. فيه **MPU-6050** IMU sensor على `/dev/i2c-2`. الـ MPU-6050 لقراءة register بيتطلب: write register address بدون STOP، ثم repeated START، ثم read البيانات. الـ `I2C_SMBUS` ioctl مش كافي هنا عشان محتاج control كامل على الـ transaction.

#### المشكلة
الكود القديم بيعمل write transaction منفصلة ثم read منفصلة:
```c
/* ده بيبعت STOP بين الـ write والـ read */
write(fd, &reg_addr, 1);   /* STOP */
read(fd, buf, 6);           /* START جديد */
```
بعض الـ IMU sensors بتغير قيمة الـ register pointer لو جاه STOP بينهم، فالـ data بتبقى غلط.

#### التحليل
من الـ header:
```c
#define I2C_RDWR 0x0707  /* Combined R/W transfer (one STOP only) */

struct i2c_rdwr_ioctl_data {
    struct i2c_msg __user *msgs;  /* array of i2c_msgs */
    __u32 nmsgs;                  /* number of i2c_msgs */
};

#define I2C_RDWR_IOCTL_MAX_MSGS 42
```
الـ `I2C_RDWR` بيسمح بإرسال multiple messages في transaction واحدة مع **STOP واحد بس في الآخر**. ده هو الـ "repeated START" mechanism.

#### الحل
```c
#include <linux/i2c-dev.h>
#include <linux/i2c.h>
#include <sys/ioctl.h>

/* قراءة 6 bytes من MPU-6050 بـ repeated START */
int mpu6050_read_regs(int fd, uint8_t reg, uint8_t *buf, int len)
{
    struct i2c_msg msgs[2];
    struct i2c_rdwr_ioctl_data rdwr;

    /* Message 1: write register address (no STOP) */
    msgs[0].addr  = 0x68;       /* MPU-6050 address */
    msgs[0].flags = 0;          /* write */
    msgs[0].len   = 1;
    msgs[0].buf   = &reg;

    /* Message 2: read data (repeated START, then STOP) */
    msgs[1].addr  = 0x68;
    msgs[1].flags = I2C_M_RD;  /* read */
    msgs[1].len   = len;
    msgs[1].buf   = buf;

    rdwr.msgs  = msgs;
    rdwr.nmsgs = 2;             /* must be <= I2C_RDWR_IOCTL_MAX_MSGS (42) */

    if (ioctl(fd, I2C_RDWR, &rdwr) < 0) {
        perror("I2C_RDWR");
        return -1;
    }

    return 0;
}

/* استخدام: قراءة accelerometer (6 bytes من reg 0x3B) */
uint8_t accel_data[6];
mpu6050_read_regs(fd, 0x3B, accel_data, 6);
```

```bash
# تحقق إن الـ adapter بيدعم I2C_RDWR
i2cdetect -F 2 | grep "I2C"
# Output: I2C                                yes
```

#### الدرس المستفاد
**`I2C_RDWR_IOCTL_MAX_MSGS`** = 42 هو حد مهم — لو حاولت تبعت أكتر من كده بـ transaction واحدة هيرجعلك `-EINVAL`. كمان الـ `I2C_RDWR` هو الطريقة الصحيحة الوحيدة للـ repeated START من userspace — مش ممكن تعملها بـ `write`/`read` منفصلين.

---

### السيناريو 5: Custom Board Bring-up على RK3562 — I2C_FUNCS لاكتشاف Adapter Capabilities

#### العنوان
استخدام `I2C_FUNCS` في bring-up script لاكتشاف قدرات الـ I2C adapter على RK3562

#### السياق
بتعمل bring-up لـ custom board على **Rockchip RK3562** لـ smart display product. الـ board فيه 4 I2C buses. بتكتب bring-up validation script بـ C يتأكد إن كل bus معمول configure صح في الـ Device Tree وبيدعم الـ features المطلوبة قبل ما يبدأ shipment testing.

#### المشكلة
فريق الـ hardware غيّر bus assignment في آخر spin — الـ EEPROM اللي كان على `i2c-2` اتنقل لـ `i2c-3`، وبعض الـ buses اتضاف ليها 10-bit addressing support. الـ test scripts القديمة بتفشل بطرق غريبة بدون error واضحة.

#### التحليل
من الـ header:
```c
#define I2C_FUNCS 0x0705  /* Get the adapter functionality mask */
```
**الـ `I2C_FUNCS`** بيرجع `unsigned long` mask فيها bits بتوصف كل الـ features اللي الـ adapter بيدعمها. دي الـ validation tool الحقيقية في الـ bring-up.

#### الحل
```c
#include <linux/i2c-dev.h>
#include <linux/i2c.h>
#include <sys/ioctl.h>
#include <stdio.h>
#include <fcntl.h>

/* structure لوصف الـ required capabilities لكل bus */
struct i2c_bus_requirement {
    const char *bus_path;
    unsigned long required_funcs;
    const char *description;
};

/* requirements للـ RK3562 custom board */
static const struct i2c_bus_requirement board_requirements[] = {
    {
        "/dev/i2c-0",
        I2C_FUNC_SMBUS_BYTE_DATA | I2C_FUNC_SMBUS_WORD_DATA,
        "PMIC (RK806) — SMBus only"
    },
    {
        "/dev/i2c-2",
        I2C_FUNC_I2C | I2C_FUNC_SMBUS_PEC,
        "Battery gauge — needs raw I2C + PEC"
    },
    {
        "/dev/i2c-3",
        I2C_FUNC_I2C,
        "EEPROM + touch controller — raw I2C"
    },
};

int validate_i2c_bus(const struct i2c_bus_requirement *req)
{
    int fd;
    unsigned long funcs;

    fd = open(req->bus_path, O_RDWR);
    if (fd < 0) {
        printf("[FAIL] %s: cannot open — %s\n", req->bus_path, strerror(errno));
        return -1;
    }

    /* I2C_FUNCS: parameter هو pointer لـ unsigned long */
    if (ioctl(fd, I2C_FUNCS, &funcs) < 0) {
        perror("I2C_FUNCS");
        close(fd);
        return -1;
    }

    close(fd);

    if ((funcs & req->required_funcs) != req->required_funcs) {
        printf("[FAIL] %s (%s)\n", req->bus_path, req->description);
        printf("       required: 0x%08lX\n", req->required_funcs);
        printf("       got:      0x%08lX\n", funcs);
        printf("       missing:  0x%08lX\n",
               req->required_funcs & ~funcs);
        return -1;
    }

    printf("[OK]   %s (%s) — funcs=0x%08lX\n",
           req->bus_path, req->description, funcs);
    return 0;
}

int main(void)
{
    int i, failed = 0;
    int n = sizeof(board_requirements) / sizeof(board_requirements[0]);

    printf("=== RK3562 I2C Bus Validation ===\n");
    for (i = 0; i < n; i++) {
        if (validate_i2c_bus(&board_requirements[i]) < 0)
            failed++;
    }

    printf("\n%d/%d buses passed\n", n - failed, n);
    return failed ? 1 : 0;
}
```

```bash
# تشغيل الـ validation
./i2c_validate
# =============================================================================
# [OK]   /dev/i2c-0 (PMIC) — funcs=0x0EFF0009
# [FAIL] /dev/i2c-2 (Battery gauge)
#        required: 0x00010008
#        got:      0x00000008
#        missing:  0x00010000   ← I2C_FUNC_SMBUS_PEC مش متدعم
# [OK]   /dev/i2c-3 (EEPROM) — funcs=0x0EFF0009

# لو الـ PEC مش متدعم → تحقق من الـ DT controller config
cat /sys/bus/i2c/devices/i2c-2/../driver/module/parameters/
```

**تحقق من الـ DT لو الـ bus مش بيدعم الـ feature المطلوبة:**
```dts
/* في rk3562-custom-board.dts */
&i2c2 {
    status = "okay";
    clock-frequency = <400000>;
    /* تأكد إن الـ compatible string صح */
};
```

#### الدرس المستفاد
**`I2C_FUNCS`** هو أول حاجة تعملها في أي bring-up script. بيوفر ساعات debugging — بدله بتقعد تسأل "ليه الـ ioctl ده بيرجع -EOPNOTSUPP؟" وانت مش عارف إن الـ adapter أصلاً مش بيدعم الـ operation دي. لاحظ إن الـ parameter هنا **pointer** لـ `unsigned long` مش قيمة مباشرة — ده مختلف عن باقي الـ ioctls في نفس الـ header.
## Phase 7: مصادر ومراجع

### توضيح على الملف المدروس

الـ `include/linux/i2c-dev.h` ملف kernel-internal بسيط جداً — بيعرّف بس `I2C_MAJOR` (رقم 89) وبيعمل include للـ `uapi/linux/i2c-dev.h`. الـ uapi header ده هو اللي بيحتوي على كل الـ `ioctl` definitions والـ structs اللي بتخلي userspace يتكلم مع الـ I2C bus. المصادر دي بتغطي الـ subsystem كله.

---

### مصادر LWN.net

| المقال | الوصف |
|--------|-------|
| [i2c: Add i2c-pseudo driver for userspace I2C adapters](https://lwn.net/Articles/820320/) | بيشرح إزاي ممكن تعمل I2C adapter في userspace يستخدم `/dev/i2c-N` عبر i2c-dev للتكلم مع DUT على شبكة أو debug interface |
| [i2c-tools: add new tool 'i2ctransfer'](https://lwn.net/Articles/716917/) | patch بيضيف `i2ctransfer` لـ i2c-tools — بيستخدم `I2C_RDWR` ioctl مباشرة عبر `linux/i2c-dev.h` |
| [Changes to the I2C driver to support a non-blocking interface](https://lwn.net/Articles/121655/) | نقاش تاريخي مهم عن تغييرات الـ i2c driver interface |
| [Linux drivers in user space — a survey](https://lwn.net/Articles/703785/) | مقال شامل بيغطي مفهوم userspace drivers ومنه I2C عبر i2c-dev |
| [Documentation/i2c/slave-interface](https://lwn.net/Articles/640640/) | الـ I2C slave interface — تكملة مهمة لفهم الصورة الكاملة للـ subsystem |
| [driver core support for i2c bus and drivers](https://lwn.net/Articles/24973/) | patch قديم بيوضح إزاي اتدمج I2C في driver model |

---

### توثيق الـ Kernel الرسمي

#### مسارات `Documentation/` في الـ kernel source

```
Documentation/i2c/
├── dev-interface.rst        ← الأهم: شرح /dev/i2c-N وكل ioctls
├── summary.rst              ← مقدمة I2C وSMBus
├── writing-clients.rst      ← كتابة I2C client drivers
├── instantiating-devices.rst
├── fault-injection.rst
├── slave-interface.rst
└── busses/                  ← توثيق كل bus controller driver
```

#### روابط مباشرة على docs.kernel.org

- [Implementing I2C device drivers in userspace](https://docs.kernel.org/i2c/dev-interface.html) — الصفحة الرئيسية لفهم `i2c-dev` والـ ioctls
- [Introduction to I2C and SMBus](https://docs.kernel.org/i2c/summary.html)
- [I2C and SMBus Subsystem (driver-api)](https://static.lwn.net/kerneldoc/driver-api/i2c.html)
- [I2C/SMBus Subsystem index](https://static.lwn.net/kerneldoc/i2c/index.html)
- [Implementing I2C device drivers (writing-clients)](https://static.lwn.net/kerneldoc/i2c/writing-clients.html)

---

### Kernel Commits المهمة

الـ `I2C_MAJOR = 89` اتحدد من زمان — ممكن تتتبع التاريخ عبر:

```bash
# تتبع تاريخ الملف في kernel git
git log --follow -- include/linux/i2c-dev.h
git log --follow -- drivers/i2c/i2c-dev.c

# أول commit للـ i2c-dev character device
git log --diff-filter=A -- drivers/i2c/i2c-dev.c
```

**Commits مهمة تاريخياً:**

- الـ `i2c-dev.c` اتضاف من Simon G. Vogl وFrodo Looijaard في 1995-1999
- الـ [Subsystem History wiki](https://archive.kernel.org/oldwiki/i2c.wiki.kernel.org/index.php/Subsystem_History.html) بيوثق كل التحولات المهمة في الـ i2c subsystem

---

### نقاشات Mailing List

| المصدر | الرابط |
|--------|--------|
| **lore.kernel.org** (الأرشيف الرسمي) | [linux-i2c@vger.kernel.org](https://lore.kernel.org/linux-i2c/) |
| **LKML Archive** | [lkml.org](https://lkml.org/) — ابحث عن `i2c-dev` |
| **KernelNewbies mailing list** — نقاش i2c-stub | [kernelnewbies.org/2013](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2013-April/008084.html) |
| **KernelNewbies** — I2C مع FTDI/GPIO | [kernelnewbies.org/2015](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-September/015153.html) |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 8**: I2C/SMBus الأساسيات — بيغطي `i2c_driver`, `i2c_client`, `i2c_adapter`
- الكتاب متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Ed.)
- **الفصل 14**: The Block I/O Layer — فيه background على character devices والـ major/minor numbers
- بيساعد تفهم ليه `I2C_MAJOR = 89` موجود

#### Embedded Linux Primer — Christopher Hallinan (2nd Ed.)
- **الفصل 15**: I2C على embedded systems — أمثلة عملية على استخدام `/dev/i2c-N` من userspace
- بيغطي `i2cdetect`, `i2cdump`, `i2cget`, `i2cset`

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- **الفصل 8**: I2C — من أفضل الشروحات العملية للـ subsystem

---

### مصادر eLinux.org

| الصفحة | المحتوى |
|--------|---------|
| [EBC I2C](https://elinux.org/EBC_I2C) | شرح عملي على BeagleBoard — `i2cdetect`, `/dev/i2c-N`, Python examples |
| [Hammer I2C Driver](https://elinux.org/Hammer_I2C_Driver) | kernel config وuserspace I2C access |
| [Tests: I2C-fault-injection](https://elinux.org/Tests:I2C-fault-injection) | اختبار الـ fault injection في i2c subsystem |
| [Tests: I2C-core-DMA](https://elinux.org/Tests:I2C-core-DMA) | اختبار DMA buffer handling في i2c-core |
| [RPi ADC I2C Python](https://elinux.org/RPi_ADC_I2C_Python) | مثال عملي على Raspberry Pi — Python + `/dev/i2c-1` |
| [EVM I2C Mods](https://elinux.org/I2C_Mods) | تعديلات hardware وconnect devices على I2C bus |

---

### مصادر إضافية

| المصدر | الرابط |
|--------|--------|
| **kernel.org — dev-interface** | [kernel.org/doc/Documentation/i2c/dev-interface](https://www.kernel.org/doc/Documentation/i2c/dev-interface) |
| **i2c-tools source** | `git clone https://git.kernel.org/pub/scm/utils/i2c-tools/i2c-tools.git` |
| **uapi header** (الأهم لـ userspace) | `include/uapi/linux/i2c-dev.h` في kernel source |
| **i2c-dev.c** (الـ driver نفسه) | `drivers/i2c/i2c-dev.c` في kernel source |
| **Practical guide** | [hackerbikepacker.com/i2c-on-linux](https://hackerbikepacker.com/i2c-on-linux) |
| **ctrlinux.com — Part II** | [I2C Communication from Linux Userspace](https://www.ctrlinux.com/blog/?p=38) |

---

### Search Terms للبحث عن معلومات أكتر

```
# بحث عام
"i2c-dev" linux kernel ioctl
"I2C_RDWR" userspace linux
"/dev/i2c" character device major 89
"i2c_smbus_xfer" vs "i2c_transfer"

# بحث في lore.kernel.org
site:lore.kernel.org "i2c-dev"
site:lore.kernel.org "I2C_MAJOR"

# بحث في LWN
site:lwn.net "i2c-dev"
site:lwn.net "I2C_RDWR"

# بحث في kernel docs
site:docs.kernel.org i2c dev-interface
site:static.lwn.net i2c
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `i2c-dev.h` نفسه صغير جداً — بس الحقيقي الإثارة في `i2c_transfer` اللي هي الـ function المركزية اللي بتتنفذ كل عملية قراءة/كتابة على الـ I2C bus. هنعمل **kprobe** عليها عشان نشوف كل transaction بيعدي: مين الـ adapter، كام message، وعناوين الـ slave devices.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * i2c_transfer_probe.c
 * kprobe on i2c_transfer() — logs every I2C transaction:
 * adapter name, number of messages, and slave addresses.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/i2c.h>       /* struct i2c_adapter, struct i2c_msg */
#include <linux/printk.h>

/* ------------------------------------------------------------------ */
/* pre-handler: runs just before i2c_transfer() executes              */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * i2c_transfer(struct i2c_adapter *adap,
     *              struct i2c_msg    *msgs,
     *              int                num);
     *
     * On x86-64:  rdi = adap,  rsi = msgs,  rdx = num
     * On ARM64:   x0  = adap,  x1  = msgs,  x2  = num
     */
#if defined(CONFIG_X86_64)
    struct i2c_adapter *adap = (struct i2c_adapter *)regs->di;
    struct i2c_msg     *msgs = (struct i2c_msg     *)regs->si;
    int                 num  = (int)                 regs->dx;
#elif defined(CONFIG_ARM64)
    struct i2c_adapter *adap = (struct i2c_adapter *)regs->regs[0];
    struct i2c_msg     *msgs = (struct i2c_msg     *)regs->regs[1];
    int                 num  = (int)                 regs->regs[2];
#else
    /* unsupported arch — skip */
    return 0;
#endif

    int i;

    if (!adap || !msgs || num <= 0)
        return 0;

    pr_info("i2c_probe: adapter=\"%s\" nr_msgs=%d\n",
            adap->name, num);

    /* print each message's slave address and direction */
    for (i = 0; i < num; i++) {
        pr_info("  msg[%d]: addr=0x%02x %s len=%u\n",
                i,
                msgs[i].addr,
                (msgs[i].flags & I2C_M_RD) ? "READ" : "WRITE",
                msgs[i].len);
    }

    return 0; /* 0 = let the real function continue */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "i2c_transfer", /* function to hook by name */
    .pre_handler = handler_pre,    /* our callback             */
};

/* ------------------------------------------------------------------ */
/* module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init i2c_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("i2c_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("i2c_probe: hooked i2c_transfer() at %p\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit i2c_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("i2c_probe: unhooked i2c_transfer()\n");
}

module_init(i2c_probe_init);
module_exit(i2c_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on i2c_transfer — trace all I2C bus transactions");
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
#include <linux/i2c.h>
```

**الـ** `kprobes.h` بيجيب `struct kprobe` وكل الـ API اللازم للـ hooking.
**الـ** `i2c.h` بيعرّف `struct i2c_adapter` و`struct i2c_msg` اللي هنقرأ منهم البيانات جوه الـ handler.

---

#### الـ Pre-handler وقراءة الـ Arguments

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

**الـ** `pt_regs` بيحتوي على الـ CPU registers لحظة الـ probe — منها بنقرأ الـ arguments بتاعة الـ function اللي اتـ hook عليها.
الـ `#ifdef` بيفرق بين x86-64 (اللي بتبعت arguments في `rdi/rsi/rdx`) وARM64 (اللي بتحطهم في `x0/x1/x2`) عشان الـ module يشتغل على الاتنين.

---

#### طباعة بيانات كل Message

```c
pr_info("  msg[%d]: addr=0x%02x %s len=%u\n", ...);
```

بنطبع عنوان الـ slave بالـ hex، اتجاه الـ transfer (READ/WRITE من bit `I2C_M_RD`)، وطول البيانات.
ده بيخلينا نشوف في `dmesg` كل device على الـ bus وإيه اللي بيتبعتله.

---

#### الـ kprobe struct

```c
static struct kprobe kp = {
    .symbol_name = "i2c_transfer",
    .pre_handler = handler_pre,
};
```

بنحدد الـ function بالاسم — الـ kernel بيـ resolve العنوان تلقائياً وقت `register_kprobe`.
الـ `pre_handler` بيتشغل **قبل** تنفيذ `i2c_transfer` بمعنى بنشوف الـ arguments قبل ما حاجة تتغير.

---

#### الـ Init والـ Exit

```c
ret = register_kprobe(&kp);
```

**الـ** `module_init` بتسجّل الـ kprobe — لو فشلت (مثلاً الـ function مش exported أو الـ kprobes disabled) بترجع الـ error فوراً.

```c
unregister_kprobe(&kp);
```

**الـ** `module_exit` لازم تـ unregister — لو سبنا الـ hook بعد `rmmod` هيبقى فيه dangling pointer لـ handler مش موجود وده kernel panic مضمون.

---

### تشغيل الـ Module

```bash
# build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# load
sudo insmod i2c_transfer_probe.ko

# trigger some I2C activity (e.g. read a sensor or run i2cdetect)
sudo i2cdetect -y 1

# watch output
sudo dmesg | grep i2c_probe

# unload
sudo rmmod i2c_transfer_probe
```

مثال على الـ output:

```
[  42.111] i2c_probe: adapter="i2c-1" nr_msgs=2
[  42.111]   msg[0]: addr=0x50 WRITE len=1
[  42.111]   msg[1]: addr=0x50 READ  len=32
[  42.115] i2c_probe: adapter="i2c-1" nr_msgs=1
[  42.115]   msg[0]: addr=0x68 WRITE len=2
```

**الـ** `addr=0x50` غالباً EEPROM وده combined write-then-read (register address ثم data).
**الـ** `addr=0x68` ممكن يكون RTC زي DS3231.
