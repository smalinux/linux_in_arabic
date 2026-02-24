## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ I2C Subsystem؟

**الـ I2C** (Inter-Integrated Circuit) هو بروتوكول تواصل بين الـ chips داخل نفس الجهاز — اخترعه فيليبس سنة 1982. الملف `include/linux/i2c.h` هو الـ **public API** الرئيسي لكل الـ subsystem ده في الـ kernel.

الـ maintainer الأساسي هو **Wolfram Sang**، والـ subsystem موجود في `drivers/i2c/` وملفاته الرئيسية في `include/linux/i2c.h` و`include/uapi/linux/i2c.h`.

---

### القصة: ليه I2C موجود أصلاً؟

تخيّل إنك بتصمم لاب توب. جوّاه في:
- sensor حرارة
- EEPROM بيحفظ الـ config
- شاشة صغيرة
- real-time clock
- battery fuel gauge

كل chip ده محتاج يتكلم مع الـ CPU. لو عملت لكل chip كابلات منفصلة، هتحتاج عشرات الـ pins. **الـ I2C حل المشكلة دي:** سلكين بس (SCL للـ clock، SDA للـ data) وعليهم ممكن يتعلق 127 device مختلف، كل واحد بـ address خاص.

```
CPU (Master)
   |
   +--SCL (clock) ──────────────────────────────┐
   |                                             |
   +--SDA (data)  ────┬─────────┬──────────┬────┘
                      |         |          |
                   Temp       EEPROM     RTC
                 Sensor      0x50       0x68
                  0x48
```

الـ CPU (أو الـ SoC) هو الـ **master** — هو اللي يبدأ الكلام. الـ devices هم الـ **slaves** — بيستنوا أوامر.

---

### المشكلة اللي `i2c.h` بيحلها

في الـ kernel، في آلاف الـ I2C devices من شركات مختلفة. لو كل driver كتب الـ logic بتاعه من الصفر:
- كل driver هيتعامل مع الـ hardware بشكل مختلف
- نفس الـ bus ممكن يتقرأ منه اتنين في نفس الوقت → corruption
- لما الـ board يتغير، كل الـ drivers هتتكسر

**الـ `i2c.h` عرّف abstraction layer كامل:**

```
┌──────────────────────────────────────────────┐
│         Device Drivers (e.g. tmp102, at24)   │  ← بيستخدموا i2c_client + i2c_driver
├──────────────────────────────────────────────┤
│              I2C Core (i2c-core-base.c)       │  ← الـ framework نفسه
├──────────────────────────────────────────────┤
│         Bus Adapters (i2c_adapter)            │  ← hardware-specific controller drivers
├──────────────────────────────────────────────┤
│      Real Hardware (DesignWare, OMAP, etc.)   │
└──────────────────────────────────────────────┘
```

---

### الـ Structures الأساسية في `i2c.h`

#### 1. `struct i2c_msg` — وحدة الكلام الأساسية
```c
struct i2c_msg {
    __u16 addr;   /* address of the slave device */
    __u16 flags;  /* I2C_M_RD for read, etc. */
    __u16 len;    /* how many bytes to transfer */
    __u8  *buf;   /* the data buffer */
};
```
ده الـ "جملة" اللي بتتبعت على الـ bus. كل transaction ممكن يكون فيه أكتر من message.

#### 2. `struct i2c_client` — تمثيل الـ device
```c
struct i2c_client {
    unsigned short addr;         /* 7-bit address on the bus */
    char name[I2C_NAME_SIZE];    /* chip type, e.g. "tmp102" */
    struct i2c_adapter *adapter; /* which bus it lives on */
    struct device dev;           /* Linux device model node */
    int irq;                     /* optional interrupt line */
    /* ... */
};
```
كل chip موصول على الـ bus عنده `i2c_client` يمثّله. ده اللي الـ driver بيشتغل عليه.

#### 3. `struct i2c_driver` — الـ driver نفسه
```c
struct i2c_driver {
    int (*probe)(struct i2c_client *client);   /* device found, init it */
    void (*remove)(struct i2c_client *client); /* device removed */
    const struct i2c_device_id *id_table;      /* list of supported chips */
    /* ... */
};
```
كل driver بيعرّف نفسه بالـ struct ده ويقول "أنا بدعم الـ chips دي".

#### 4. `struct i2c_adapter` — الـ controller hardware
```c
struct i2c_adapter {
    const struct i2c_algorithm *algo; /* how to actually talk on the bus */
    struct rt_mutex bus_lock;         /* one transfer at a time */
    int nr;                           /* bus number, e.g. /dev/i2c-0 */
    struct i2c_bus_recovery_info *bus_recovery_info; /* if bus hangs */
    /* ... */
};
```
ده يمثل الـ physical I2C controller في الـ SoC. كل جهاز ممكن يبقى فيه عشرات الـ adapters.

#### 5. `struct i2c_algorithm` — كيفية الإرسال الفعلي
```c
struct i2c_algorithm {
    int (*xfer)(struct i2c_adapter *adap,
                struct i2c_msg *msgs, int num); /* do the actual transfer */
    u32 (*functionality)(struct i2c_adapter *adap); /* what can we do? */
    /* ... */
};
```
الـ adapter بيقول للـ core "إزاي تبعت bits على الـ wire الخاصة بيّا".

---

### الـ SMBus: ابن I2C

**الـ SMBus** (System Management Bus) هو I2C بـ rules إضافية — شائع في الـ PCs لإدارة الـ power والـ sensors. الـ kernel بيدعم الاتنين. لو الـ adapter مش بيدعم SMBus natively، الـ core بيعمل **emulation** فوق I2C عادي.

```c
/* High-level SMBus API — driver writer uses these */
s32 i2c_smbus_read_byte_data(const struct i2c_client *client, u8 reg);
s32 i2c_smbus_write_byte_data(const struct i2c_client *client,
                              u8 reg, u8 value);
```

---

### السيناريو الكامل: driver لـ temperature sensor

```
1. Board boots → i2c_register_board_info() يعرف إن في tmp102 على addr 0x48

2. i2c_core يلاقي الـ device → ينشئ i2c_client

3. tmp102 driver يكون registered بـ module_i2c_driver(tmp102_driver)

4. Core يعمل match → يستدعي tmp102_driver.probe(client)

5. الـ probe بتعمل i2c_smbus_read_byte_data() لتقرأ temperature register

6. الـ call بتنزل:
   i2c_smbus_read_byte_data()
     → i2c_smbus_xfer()
       → i2c_transfer()
         → adapter->algo->xfer()  ← الـ hardware-specific code هنا
```

---

### الـ Bus Recovery

لو الـ SDA اتعلق في حالة LOW (slave لسه بيبعت)، الـ bus هيبقى locked. الـ `struct i2c_bus_recovery_info` بيدّي الـ core آلية للـ recovery: بيبعت clock pulses إضافية عشان يخليه يكمّل.

---

### الـ Slave Mode

الـ `i2c.h` بيدعم كمان إن الـ adapter نفسه يشتغل كـ **slave** (مش بس master). ده مهم لبعض الـ embedded use cases زي PMIC أو microcontroller بيتكلم مع host.

---

### الـ Frequency Modes المعرفة

| Mode | Max Frequency |
|------|--------------|
| Standard | 100 kHz |
| Fast | 400 kHz |
| Fast Mode Plus | 1 MHz |
| Turbo | 1.4 MHz |
| High Speed | 3.4 MHz |
| Ultra Fast | 5 MHz |

---

### ملفات الـ Subsystem

#### الـ Core Headers
| الملف | الدور |
|-------|-------|
| `include/linux/i2c.h` | الـ main API — كل اللي بيكتب driver محتاجه |
| `include/uapi/linux/i2c.h` | الـ `i2c_msg`, `i2c_smbus_data` — مشترك مع userspace |
| `include/linux/i2c-smbus.h` | SMBus alert protocol |
| `include/linux/i2c-dev.h` | userspace `/dev/i2c-N` interface |

#### الـ Core Implementation
| الملف | الدور |
|-------|-------|
| `drivers/i2c/i2c-core-base.c` | قلب الـ framework: registration, transfer, matching |
| `drivers/i2c/i2c-core-smbus.c` | SMBus emulation والـ transactions |
| `drivers/i2c/i2c-core-of.c` | Device Tree support |
| `drivers/i2c/i2c-core-acpi.c` | ACPI support |
| `drivers/i2c/i2c-core-slave.c` | Slave mode support |
| `drivers/i2c/i2c-boardinfo.c` | Static board info registration |
| `drivers/i2c/i2c-dev.c` | `/dev/i2c-N` character device |
| `drivers/i2c/i2c-mux.c` | I2C bus multiplexers |

#### الـ Algorithms (software bit-banging)
| الملف | الدور |
|-------|-------|
| `drivers/i2c/algos/i2c-algo-bit.c` | GPIO bit-banging algorithm |
| `drivers/i2c/algos/i2c-algo-pcf.c` | PCF8584 chip algorithm |

#### أمثلة على الـ HW Bus Drivers
| الملف | الدور |
|-------|-------|
| `drivers/i2c/busses/i2c-designware-*.c` | DesignWare IP core (شائع في Intel/AMD/ARM) |
| `drivers/i2c/busses/i2c-i801.c` | Intel PCH SMBus |
| `drivers/i2c/busses/i2c-omap.c` | TI OMAP/AM series |
| `drivers/i2c/busses/i2c-imx.c` | NXP i.MX SoCs |
| `drivers/i2c/busses/i2c-gpio.c` | generic GPIO bit-bang |
| `drivers/i2c/busses/i2c-bcm2835.c` | Raspberry Pi |

---

### ملفات مهمة للقارئ

- **`Documentation/i2c/writing-clients.rst`** — دليل كتابة I2C driver من الصفر
- **`Documentation/i2c/fault-codes.rst`** — error codes المعيارية
- **`Documentation/i2c/instantiating-devices.rst`** — طرق إنشاء الـ I2C devices
- **`include/uapi/linux/i2c-dev.h`** — للـ userspace tools زي `i2cdetect` و`i2cdump`
## Phase 2: شرح الـ I2C Framework

---

### المشكلة — ليه الـ I2C subsystem موجود أصلاً؟

في أي SoC ARM عندك عشرات الـ peripherals: temperature sensors، EEPROMs، PMICs، accelerometers، codecs، RTC chips — كلهم بيتكلموا على نفس الـ protocol: **I2C (Inter-Integrated Circuit)**.

المشكلة من غير framework:
- كل driver لازم يـimplement منطق الـ bus لوحده (START/STOP/ACK/NACK)
- كل SoC عنده I2C controller مختلف (BCM2835، i.MX6، STM32) — كود مكرر في كل driver
- مفيش طريقة موحدة للـ device enumeration على I2C
- مفيش حماية من الـ concurrent access لو أكتر من driver اشتغل في نفس الوقت على نفس الـ bus

**الحل المطلوب:** طبقة abstraction تفصل بين:
1. "مين يحتاج يبعت data" — الـ **consumer driver** (مثلاً driver الـ temperature sensor)
2. "كيف البيانات دي تتبعت فعلاً على الـ wire" — الـ **adapter driver** (driver الـ I2C controller في الـ SoC)

---

### الحل — نهج الـ kernel

الـ kernel حل المشكلة بـ **device/driver model** مبني على الـ `bus_type` abstraction اللي موجودة في الـ Driver Model subsystem.

> **الـ Driver Model subsystem:** هو الإطار العام في الـ kernel اللي بيدير الـ devices والـ drivers وعلاقتهم ببعض من خلال `struct bus_type` و`struct device` و`struct device_driver`. كل subsystem زي I2C أو SPI بيستخدمه.

الـ I2C framework بيعمل:
1. **`i2c_bus_type`** — bus افتراضي يشوف كل الـ I2C devices والـ drivers
2. **`i2c_adapter`** — يمثل الـ hardware controller (مثلاً I2C0 على الـ SoC)
3. **`i2c_client`** — يمثل كل slave device على الـ bus (مثلاً EEPROM عند address 0x50)
4. **`i2c_driver`** — الـ driver اللي بيـmanage نوع معين من الـ devices
5. **`i2c_algorithm`** — الـ vtable اللي بيـabstract الـ hardware-specific transfer

---

### تشبيه واقعي — شبكة الهاتف الأرضي

تخيل شبكة تليفون أرضي في مبنى:

| عنصر الشبكة | المقابل في I2C Framework |
|---|---|
| **السنترال** (exchange) | `struct i2c_adapter` — الـ controller الـ hardware |
| **خط التليفون** (wire) | الـ SDA/SCL lines على الـ PCB |
| **رقم التليفون** (phone number) | `client->addr` — الـ 7-bit I2C address |
| **جهاز التليفون** (handset) | `struct i2c_client` — الـ slave device |
| **شركة الاتصالات** (telecom co.) | `struct i2c_algorithm` — بروتوكول الإرسال الفعلي |
| **كتاب التليفونات** (directory) | `struct i2c_board_info` / Device Tree nodes |
| **الشخص اللي بيتكلم** (caller) | `struct i2c_driver` — الـ device driver |
| **قواعد المكالمة** (call rules) | `struct i2c_lock_operations` — mutual exclusion |
| **مشكلة الخط** (line fault) | `struct i2c_bus_recovery_info` — bus recovery |

**التفصيل الدقيق للتشبيه:**
- لما الـ temperature sensor driver (الشخص) يريد يقرأ درجة الحرارة (يتصل بحد) — بيستخدم `i2c_transfer()` (يرفع سماعة ويطلب رقم)
- الـ `i2c_transfer()` بيطلب من الـ `i2c_adapter` (السنترال) يوصّل الاتصال
- الـ adapter بيستخدم `algo->xfer()` (شبكة الكابلات الفعلية) لإرسال الـ bits على الـ SDA/SCL
- الـ slave (جهاز التليفون في الطرف التاني) عنده address ثابت — زي ما رقم التليفون ثابت في الحائط
- لو خطأ حصل (NACK/bus stuck) — الـ `i2c_bus_recovery_info` (فني الإصلاح) بيتدخل يـtoggle الـ SCL ويـclear الـ bus

---

### الصورة الكبيرة — Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Consumer Drivers                           │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│   │ lm75 (temp)  │  │ at24 (eeprom)│  │ bq27xxx (fuel gauge) │ │
│   │ i2c_driver   │  │ i2c_driver   │  │ i2c_driver           │ │
│   └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │
│          │                 │                      │             │
│          └────────────┬────┘                      │             │
│                       │    i2c_transfer() / SMBus  │             │
└───────────────────────┼────────────────────────────┼────────────┘
                        │                            │
┌───────────────────────▼────────────────────────────▼────────────┐
│                     I2C Core  (drivers/i2c/i2c-core-*.c)        │
│                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────────┐ │
│  │  bus_type   │  │  device mgmt │  │  SMBus emulation layer  │ │
│  │  matching   │  │  (add/del)   │  │  (I2C msgs → SMBus)     │ │
│  └─────────────┘  └──────────────┘  └─────────────────────────┘ │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               struct i2c_adapter                         │   │
│  │  .algo → struct i2c_algorithm { .xfer, .smbus_xfer }     │   │
│  │  .lock_ops → struct i2c_lock_operations                  │   │
│  │  .bus_recovery_info → struct i2c_bus_recovery_info       │   │
│  │  .quirks → struct i2c_adapter_quirks                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────────────────┐
│                    Adapter Drivers (HAL)                         │
│                                                                  │
│  ┌────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │ i2c-bcm2835.c  │  │ i2c-imx.c        │  │ i2c-gpio.c       │ │
│  │ (Raspberry Pi) │  │ (NXP i.MX)       │  │ (bit-banging)    │ │
│  └────────┬───────┘  └────────┬─────────┘  └────────┬─────────┘ │
└───────────┼───────────────────┼────────────────────┼────────────┘
            │                   │                    │
            ▼                   ▼                    ▼
         SDA/SCL             SDA/SCL              GPIO pins
         (BCM SoC)           (i.MX SoC)           (any SoC)
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ core abstraction في الـ I2C framework هي **الفصل بين الـ "ماذا تريد" والـ "كيف يتم"** من خلال الـ `struct i2c_algorithm`.

لما تكتب:
```c
i2c_transfer(client->adapter, msgs, num);
```

الـ core بيعمل:
1. Lock الـ bus (via `lock_ops->lock_bus()`)
2. Check الـ adapter مش suspended
3. Retry loop حسب `adap->retries`
4. استدعاء `adap->algo->xfer(adap, msgs, num)` — ده الـ hardware-specific code
5. Unlock الـ bus

الـ driver بتاعك ما بيشوفش أي SoC هو شغال عليه.

---

### شرح مفصل للـ Structs الأساسية

#### `struct i2c_msg` — الوحدة الأساسية للنقل

```c
struct i2c_msg {
    __u16 addr;    /* 7-bit أو 10-bit address للـ slave */
    __u16 flags;   /* I2C_M_RD للقراءة، I2C_M_TEN لـ 10-bit، إلخ */
    __u16 len;     /* عدد البايتات في buf */
    __u8  *buf;    /* الـ data buffer */
};
```

كل `i2c_msg` بيمثل **segment واحد** في الـ transaction. Transaction كاملة ممكن تكون أكتر من msg:

```
Write-then-Read (register read pattern):
┌─────────────────────────────────────────────────┐
│ START | ADDR+W | REG_NUM | rSTART | ADDR+R | DATA | STOP │
└────────────────── msg[0] ──────┘└──────── msg[1] ────────┘
     flags=0, buf=[0x10]              flags=I2C_M_RD, buf=result
```

الـ `i2c_transfer()` بتاخد array من الـ `i2c_msg` — ده اللي بيخليك تعمل combined transactions بـ repeated START من غير STOP في النص.

---

#### `struct i2c_algorithm` — الـ Hardware Abstraction Layer

```c
struct i2c_algorithm {
    /* الـ I2C raw transfer — الأهم */
    union {
        int (*xfer)(struct i2c_adapter *adap,
                    struct i2c_msg *msgs, int num);
        int (*master_xfer)(...);  /* deprecated alias */
    };

    /* نفس xfer بس في atomic context (مثلاً أثناء shutdown لـ PMIC) */
    union {
        int (*xfer_atomic)(struct i2c_adapter *adap,
                           struct i2c_msg *msgs, int num);
        int (*master_xfer_atomic)(...);  /* deprecated */
    };

    /* SMBus native — لو الـ hardware يدعمه مباشرة */
    int (*smbus_xfer)(struct i2c_adapter *adap, u16 addr,
                      unsigned short flags, char read_write,
                      u8 command, int size,
                      union i2c_smbus_data *data);

    /* SMBus atomic variant */
    int (*smbus_xfer_atomic)(...);

    /* بيرجع bitmask من I2C_FUNC_* — الـ core بيستخدمه للـ validation */
    u32 (*functionality)(struct i2c_adapter *adap);

    /* slave mode — لو الـ adapter نفسه يقدر يبقى slave */
    int (*reg_target)(struct i2c_client *client);
    int (*unreg_target)(struct i2c_client *client);
};
```

**نقطة مهمة جداً — SMBus Emulation:**
لو `smbus_xfer` مش موجود (NULL)، الـ core بيـemulate الـ SMBus تلقائياً باستخدام `xfer` وبيبعت الـ I2C messages المناسبة. ده معناه أي adapter بيـimplements `xfer` بس، بيحصل على SMBus support مجاناً.

```
i2c_smbus_read_byte_data(client, reg)
        │
        ▼
i2c_smbus_xfer()  ─── adap->algo->smbus_xfer موجود؟ ───► native SMBus HW
        │                      NO
        ▼
i2c_smbus_xfer_emulated()
        │
        ▼  builds i2c_msg array
adap->algo->xfer(adap, msgs, 2)  ── عبر الـ I2C raw transfer
```

---

#### `struct i2c_adapter` — قلب الـ Framework

```c
struct i2c_adapter {
    struct module *owner;
    unsigned int class;              /* للـ detection (hwmon, إلخ) */
    const struct i2c_algorithm *algo; /* الـ vtable للـ HW ops */
    void *algo_data;                 /* private data للـ adapter driver */

    const struct i2c_lock_operations *lock_ops; /* bus locking */
    struct rt_mutex bus_lock;        /* الـ default lock */
    struct rt_mutex mux_lock;        /* للـ I2C mux trees */

    int timeout;                     /* بـ jiffies */
    int retries;
    struct device dev;               /* في driver model tree */
    unsigned long locked_flags;      /* I2C_ALF_IS_SUSPENDED إلخ */

    int nr;                          /* رقم الـ adapter (i2c-0, i2c-1, ...) */
    char name[48];
    struct completion dev_released;

    struct i2c_bus_recovery_info *bus_recovery_info;
    const struct i2c_adapter_quirks *quirks;

    struct irq_domain *host_notify_domain;  /* للـ SMBus Host Notify */
    struct regulator *bus_regulator;         /* VCC للـ bus */

    /* bitmap للـ addresses المستخدمة حالياً على الـ 7-bit space */
    DECLARE_BITMAP(addrs_in_instantiation, 1 << 7);
};
```

لاحظ الـ `struct device dev` جوا الـ adapter — ده اللي بيخلي الـ adapter نفسه جزء من الـ driver model tree. الـ `to_i2c_adapter(d)` ماكرو بيستخدم `container_of` للرجوع من الـ `struct device` للـ `struct i2c_adapter`.

---

#### `struct i2c_client` — تمثيل الـ Slave Device

```c
struct i2c_client {
    unsigned short flags;     /* I2C_CLIENT_TEN, I2C_CLIENT_PEC, إلخ */
    unsigned short addr;      /* الـ 7-bit address في الـ lower 7 bits */
    char name[I2C_NAME_SIZE]; /* اسم الـ device type */
    struct i2c_adapter *adapter; /* الـ bus اللي هو عليه */
    struct device dev;           /* في driver model */
    int init_irq;
    int irq;
    struct list_head detected;
    i2c_slave_cb_t slave_cb;  /* لو الـ client نفسه بيشتغل كـ slave */
    void *devres_group_id;
    struct dentry *debugfs;
};
```

العلاقة بين الـ structs:

```
struct i2c_adapter                struct i2c_client
┌──────────────────┐              ┌──────────────────┐
│  .nr = 0         │◄─────────────│  .adapter        │
│  .algo ──────────┼──►algorithm  │  .addr = 0x48    │
│  .dev            │              │  .dev            │
│  .bus_lock       │              │  .irq            │
└──────────────────┘              └────────┬─────────┘
                                           │  matched by i2c_bus_type
                                  ┌────────▼─────────┐
                                  │  struct i2c_driver│
                                  │  .probe()         │
                                  │  .remove()        │
                                  │  .id_table        │
                                  │  .driver          │
                                  └──────────────────┘
```

---

#### `struct i2c_driver` — الـ Device Driver

```c
struct i2c_driver {
    unsigned int class;  /* I2C_CLASS_HWMON مثلاً — للـ detection */

    int   (*probe)(struct i2c_client *client);
    void  (*remove)(struct i2c_client *client);
    void  (*shutdown)(struct i2c_client *client);

    /* SMBus Alert / Host Notify */
    void  (*alert)(struct i2c_client *client,
                   enum i2c_alert_protocol protocol,
                   unsigned int data);

    int   (*command)(struct i2c_client *client,
                     unsigned int cmd, void *arg);

    struct device_driver driver;       /* base class */
    const struct i2c_device_id *id_table;

    /* للـ auto-detection */
    int   (*detect)(struct i2c_client *client,
                    struct i2c_board_info *info);
    const unsigned short *address_list; /* العناوين اللي يجرب عليها */
    struct list_head clients;
    u32 flags;
};
```

الـ `struct device_driver driver` جوا الـ `i2c_driver` هو اللي بيخلي الـ I2C driver يـregister نفسه في الـ driver model. الـ `to_i2c_driver(d)` بيعمل `container_of` بالعكس.

---

#### `struct i2c_board_info` — وصف الـ Device

```c
struct i2c_board_info {
    char             type[I2C_NAME_SIZE]; /* "lm75a", "at24c32", إلخ */
    unsigned short   flags;
    unsigned short   addr;                /* 0x48، 0x50، إلخ */
    const char      *dev_name;
    void            *platform_data;
    struct fwnode_handle *fwnode;         /* DT أو ACPI node */
    const struct software_node *swnode;
    const struct resource *resources;
    unsigned int     num_resources;
    int              irq;
};
```

الـ `i2c_board_info` هو "الوصفة" لإنشاء `i2c_client`. الـ core بياخد الوصفة دي ويخلق الـ client object ويـregister إياه على الـ bus.

**طرق إنشاء الـ clients:**

```
┌──────────────────────────────────────────────────────────┐
│           Methods to Create i2c_client                   │
│                                                          │
│  1. Device Tree:                                         │
│     &i2c0 { lm75@48 { compatible = "national,lm75a" } } │
│     → of_i2c_register_devices() → i2c_new_client_device()│
│                                                          │
│  2. ACPI: DSDT I2CSerialBus() resource                   │
│     → i2c_acpi_register_devices()                        │
│                                                          │
│  3. Static board file (old style):                       │
│     i2c_register_board_info(0, my_devs, ARRAY_SIZE(...)) │
│                                                          │
│  4. Runtime (add-on boards):                             │
│     i2c_new_client_device(adapter, &info)                │
│                                                          │
│  5. Auto-detection:                                      │
│     i2c_driver.detect() + i2c_driver.address_list       │
└──────────────────────────────────────────────────────────┘
```

---

#### `struct i2c_bus_recovery_info` — إصلاح الـ Bus المعلق

```c
struct i2c_bus_recovery_info {
    int  (*recover_bus)(struct i2c_adapter *adap);  /* entry point */

    /* للـ generic SCL recovery */
    int  (*get_scl)(struct i2c_adapter *adap);
    void (*set_scl)(struct i2c_adapter *adap, int val);
    int  (*get_sda)(struct i2c_adapter *adap);
    void (*set_sda)(struct i2c_adapter *adap, int val);
    int  (*get_bus_free)(struct i2c_adapter *adap);

    void (*prepare_recovery)(struct i2c_adapter *adap);
    void (*unprepare_recovery)(struct i2c_adapter *adap);

    /* GPIO recovery */
    struct gpio_desc    *scl_gpiod;
    struct gpio_desc    *sda_gpiod;
    struct pinctrl      *pinctrl;
    struct pinctrl_state *pins_default;
    struct pinctrl_state *pins_gpio;
};
```

**متى تُستخدم؟** لو الـ slave crash أثناء read transaction وبقى بيـholds الـ SDA low (الـ bus stuck). الـ recovery procedure بيعمل:
1. Toggle الـ SCL 9 مرات (يخلي الـ slave يـfinish الـ byte اللي هو في النص)
2. يبعت STOP condition
3. يرجع الـ bus لـ idle state (SDA و SCL high)

ده بيتطلب الـ **GPIO subsystem** و**pinctrl subsystem** لتحويل الـ dedicated I2C pins لـ GPIO مؤقتاً.

> **الـ pinctrl subsystem:** بيـmanage الـ pin multiplexing على الـ SoC — يقدر يحوّل نفس الـ physical pin من I2C function لـ GPIO function وبالعكس.

---

#### `struct i2c_adapter_quirks` — محدودية الـ Hardware

```c
struct i2c_adapter_quirks {
    u64 flags;
    int max_num_msgs;       /* أقصى عدد msgs في transaction واحدة */
    u16 max_write_len;      /* أقصى حجم write message */
    u16 max_read_len;       /* أقصى حجم read message */
    u16 max_comb_1st_msg_len;
    u16 max_comb_2nd_msg_len;
};
```

الـ flags المهمة:

| Flag | المعنى |
|---|---|
| `I2C_AQ_COMB_WRITE_THEN_READ` | الـ controller يدعم بس write-then-read combined |
| `I2C_AQ_NO_CLK_STRETCH` | الـ slave مش مسموح له يـstretch الـ clock |
| `I2C_AQ_NO_REP_START` | الـ controller مش بيدعم repeated START |
| `I2C_AQ_NO_ZERO_LEN` | الـ messages لازم تكون > 0 bytes |

الـ core بيـcheck الـ quirks قبل أي transfer ويـreject الـ msgs اللي مش compatible.

---

### I2C مقابل SMBus — الفرق الحقيقي

**SMBus (System Management Bus)** هو subset من I2C مع قيود إضافية وبعض الـ extensions:

```
I2C (General Protocol)
├── أي speed من 10kHz لـ 5MHz
├── أي message size لـ 64KB
├── أي sequence من msgs
└── SMBus (Restricted Subset)
    ├── 100kHz فقط (Standard Mode)
    ├── Message types محددة (byte، word، block)
    ├── PEC (Packet Error Checking) — CRC-8
    ├── Alert protocol (slave يطلب attention)
    └── Host Notify (slave بيـact كـ master لإرسال notification)
```

الـ kernel بيدعم الاتنين — لما driver يستخدم `i2c_smbus_read_byte_data()` والـ adapter مدعوش `smbus_xfer`، الـ core بيعمل:

```c
/* emulation: converts SMBus read_byte_data to 2 i2c_msgs */
msg[0].addr  = client->addr;
msg[0].flags = 0;           /* write */
msg[0].len   = 1;
msg[0].buf   = &command;    /* register address */

msg[1].addr  = client->addr;
msg[1].flags = I2C_M_RD;   /* read */
msg[1].len   = 1;
msg[1].buf   = &data->byte;

i2c_transfer(adapter, msg, 2);
```

---

### الـ Slave Mode — حين يكون الـ Adapter هو الـ Slave

```c
enum i2c_slave_event {
    I2C_SLAVE_READ_REQUESTED,   /* master بدأ read من الـ adapter */
    I2C_SLAVE_WRITE_REQUESTED,  /* master بدأ write للـ adapter */
    I2C_SLAVE_READ_PROCESSED,   /* تم إرسال byte، جاهز للـ next */
    I2C_SLAVE_WRITE_RECEIVED,   /* استقبلنا byte */
    I2C_SLAVE_STOP,             /* STOP condition */
};
```

لما تشغّل الـ kernel على microcontroller أو embedded SoC ويكون محتاج يستجيب كـ slave (مثلاً BMC في servers)، الـ I2C adapter نفسه ممكن يشتغل كـ slave. الـ `i2c_slave_cb_t` callback في الـ `i2c_client` بيتعامل مع الـ events دي.

---

### ما يمتلكه الـ Framework مقابل ما يفوّضه للـ Drivers

#### ما يمتلكه الـ I2C Core (يعمله بنفسه):

| المسؤولية | الآلية |
|---|---|
| **Bus locking** | `rt_mutex bus_lock` + `lock_ops` |
| **SMBus emulation** | `i2c_smbus_xfer_emulated()` |
| **Device enumeration من DT/ACPI** | `of_i2c_register_devices()`, `i2c_acpi_register_devices()` |
| **Retry logic** | `adap->retries` loop في `i2c_transfer()` |
| **Quirks validation** | `i2c_check_quirks()` قبل كل transfer |
| **Auto-detection** | `i2c_detect()` |
| **Bus recovery trigger** | `i2c_recover_bus()` |
| **Suspend/Resume gating** | `I2C_ALF_IS_SUSPENDED` flag |
| **DMA buffer management** | `i2c_get_dma_safe_msg_buf()` |
| **debugfs entries** | تلقائياً لكل adapter وclient |
| **userspace access (i2c-dev)** | `/dev/i2c-N` interface |

#### ما يفوّضه للـ Adapter Driver:

| المسؤولية | الـ callback |
|---|---|
| **الإرسال الفعلي على الـ wire** | `algo->xfer()` |
| **Atomic transfer** | `algo->xfer_atomic()` |
| **Bus frequency setup** | في `probe()` بتاع الـ adapter |
| **Timing parameters** | `struct i2c_timings` + `i2c_parse_fw_timings()` |
| **Hardware recovery** | `bus_recovery_info->recover_bus()` |
| **Custom locking** | `lock_ops` (للـ mux adapters) |
| **Functionality advertisement** | `algo->functionality()` |

#### ما يفوّضه للـ Device Driver:

| المسؤولية | الآلية |
|---|---|
| **تفسير الـ data** | في `probe()` وعمليات القراءة/الكتابة |
| **Register map** | `regmap` subsystem (اختياري) |
| **Device-specific protocol** | SMBus أو raw I2C calls |
| **Power management** | `dev_pm_ops` في الـ `i2c_driver.driver` |
| **IRQ handling** | `client->irq` + standard IRQ APIs |

---

### مثال عملي — Driver الـ LM75 Temperature Sensor

```c
/* Consumer driver يستخدم الـ framework */
static int lm75_probe(struct i2c_client *client)
{
    /* check إن الـ adapter يقدر يعمل الـ operations المطلوبة */
    if (!i2c_check_functionality(client->adapter,
                                  I2C_FUNC_SMBUS_BYTE_DATA |
                                  I2C_FUNC_SMBUS_WORD_DATA))
        return -EOPNOTSUPP;

    /* قراءة temperature register (register 0x00) */
    s32 temp = i2c_smbus_read_word_swapped(client, 0x00);
    if (temp < 0)
        return temp;

    /* temp الآن هو raw 16-bit value من الـ sensor */
    /* الـ driver بيفسّر الـ data، مش الـ framework */
}

static const struct i2c_device_id lm75_ids[] = {
    { "lm75a", lm75a },
    { "tmp75", tmp75 },
    { }
};

static struct i2c_driver lm75_driver = {
    .driver = {
        .name = "lm75",
        .of_match_table = lm75_of_match,
    },
    .probe    = lm75_probe,
    .id_table = lm75_ids,
};

module_i2c_driver(lm75_driver);
/* ده بيـexpand لـ module_init/module_exit بيستخدموا i2c_add_driver/i2c_del_driver */
```

---

### الـ Locking Architecture — تفصيل مهم

```
i2c_transfer()
    │
    ▼
i2c_lock_bus(adap, I2C_LOCK_ROOT_ADAPTER)
    │                 │
    │                 ▼
    │          lock_ops->lock_bus()
    │                 │
    │          ┌──────▼──────────────────────────────────┐
    │          │  Default: rt_mutex_lock(&adap->bus_lock)│
    │          │  MUX:     lock root + lock segment       │
    │          └─────────────────────────────────────────┘
    │
    ▼
__i2c_transfer() ──► algo->xfer()  [hardware access]
    │
    ▼
i2c_unlock_bus(adap, I2C_LOCK_ROOT_ADAPTER)
```

الـ `rt_mutex` (realtime mutex) بدل الـ عادي — عشان الـ I2C بيُستخدم كتير من drivers زي PMIC اللي بيتـaccess من realtime contexts، والـ rt_mutex بيـhandles الـ priority inversion.

الـ `mux_lock` منفصل — لما عندك I2C mux (switch زي PCA9548 بيوصل 8 buses على bus واحدة)، الـ mux lock بيـprotect الـ segment بس، والـ root lock بيـprotect الـ physical bus كاملاً.

---

### Device Tree Integration — كيف بيتـenumerate الـ Devices

```dts
/* في Device Tree للـ SoC */
i2c0: i2c@40004000 {
    compatible = "nxp,lpc1778-i2c";
    reg = <0x40004000 0x1000>;
    clock-frequency = <400000>;   /* → i2c_timings.bus_freq_hz */
    #address-cells = <1>;
    #size-cells = <0>;

    /* الـ slaves على الـ bus */
    temp_sensor: lm75@48 {
        compatible = "national,lm75a";
        reg = <0x48>;              /* → i2c_board_info.addr */
        interrupt-parent = <&gpio>;
        interrupts = <5 IRQ_TYPE_LEVEL_LOW>;  /* → client->irq */
    };

    eeprom@50 {
        compatible = "atmel,at24c32";
        reg = <0x50>;
        pagesize = <32>;
    };
};
```

لما الـ adapter driver بيـcall `i2c_add_adapter()`:
1. الـ core بيـcall `of_i2c_register_devices(adap)`
2. بيـiterate على كل child nodes في الـ DT
3. بيـcall `of_i2c_get_board_info()` لكل node → يملأ `i2c_board_info`
4. بيـcall `i2c_new_client_device(adap, &info)` → ينشئ `i2c_client`
5. الـ `i2c_bus_type.match()` بيربط الـ client مع الـ driver المناسب
6. بيـcall `driver->probe(client)`
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — Flags, Enums, Config Options

#### `i2c_msg` Flags (`I2C_M_*`)

| Flag | Value | الاستخدام |
|------|-------|-----------|
| `I2C_M_RD` | `0x0001` | قراءة من الـ slave (master receive) |
| `I2C_M_TEN` | `0x0010` | عنوان 10-bit — يتطلب `I2C_FUNC_10BIT_ADDR` |
| `I2C_M_DMA_SAFE` | `0x0200` | الـ buffer آمن للـ DMA |
| `I2C_M_RECV_LEN` | `0x0400` | أول byte مستلم = طول الرسالة |
| `I2C_M_NO_RD_ACK` | `0x0800` | skip الـ ACK/NACK في القراءة |
| `I2C_M_IGNORE_NAK` | `0x1000` | تعامل مع الـ NACK كـ ACK |
| `I2C_M_REV_DIR_ADDR` | `0x2000` | قلب bit الـ R/W |
| `I2C_M_NOSTART` | `0x4000` | skip الـ repeated START |
| `I2C_M_STOP` | `0x8000` | أجبر STOP بعد الرسالة |

#### `i2c_client` Flags (`I2C_CLIENT_*`)

| Flag | Value | المعنى |
|------|-------|--------|
| `I2C_CLIENT_PEC` | `0x04` | استخدم Packet Error Checking |
| `I2C_CLIENT_TEN` | `0x10` | عنوان 10-bit chip |
| `I2C_CLIENT_SLAVE` | `0x20` | الـ client شغال كـ slave |
| `I2C_CLIENT_HOST_NOTIFY` | `0x40` | يستخدم I2C host notify |
| `I2C_CLIENT_WAKE` | `0x80` | يقدر يصحّي الـ system |
| `I2C_CLIENT_SCCB` | `0x9000` | Omnivision SCCB protocol |

#### Adapter Quirks Flags (`I2C_AQ_*`)

| Flag | المعنى |
|------|--------|
| `I2C_AQ_COMB` | الـ adapter يدعم combined messages فقط (max 2 msgs) |
| `I2C_AQ_COMB_WRITE_FIRST` | أول رسالة في الـ combined لازم تكون write |
| `I2C_AQ_COMB_READ_SECOND` | ثاني رسالة لازم تكون read |
| `I2C_AQ_COMB_SAME_ADDR` | كلا الرسالتين لنفس العنوان |
| `I2C_AQ_COMB_WRITE_THEN_READ` | الـ macro الجاهز لأشهر حالة: write-then-read |
| `I2C_AQ_NO_CLK_STRETCH` | الـ hardware مش بيدعم clock stretching |
| `I2C_AQ_NO_ZERO_LEN_READ` | ممنوع رسالة read بطول صفر |
| `I2C_AQ_NO_ZERO_LEN_WRITE` | ممنوع رسالة write بطول صفر |
| `I2C_AQ_NO_REP_START` | الـ adapter مش بيدعم repeated START |

#### Adapter Locked Flags (`I2C_ALF_*`)

| Flag | Bit | المعنى |
|------|-----|--------|
| `I2C_ALF_IS_SUSPENDED` | 0 | الـ adapter suspended — الـ core يرفض transfers |
| `I2C_ALF_SUSPEND_REPORTED` | 1 | تم الإبلاغ عن الـ suspend بالفعل |

#### Bus Locking Flags

| Flag | المعنى |
|------|--------|
| `I2C_LOCK_ROOT_ADAPTER` | lock الـ root adapter (فوق شجرة الـ mux) |
| `I2C_LOCK_SEGMENT` | lock الـ segment الحالي فقط |

#### `enum i2c_slave_event`

| Event | المعنى |
|-------|--------|
| `I2C_SLAVE_READ_REQUESTED` | الـ master طلب قراءة منّا |
| `I2C_SLAVE_WRITE_REQUESTED` | الـ master هيكتب لينا |
| `I2C_SLAVE_READ_PROCESSED` | تم إرسال byte — جهّز التالي |
| `I2C_SLAVE_WRITE_RECEIVED` | استلمنا byte |
| `I2C_SLAVE_STOP` | الـ master بعت STOP |

#### `enum i2c_alert_protocol`

| Value | المعنى |
|-------|--------|
| `I2C_PROTOCOL_SMBUS_ALERT` | SMBus alert عبر خط منفصل |
| `I2C_PROTOCOL_SMBUS_HOST_NOTIFY` | الـ slave يعمل master ويبعت notify |

#### `enum i2c_driver_flags`

| Flag | المعنى |
|------|--------|
| `I2C_DRV_ACPI_WAIVE_D0_PROBE` | لا تحط الجهاز في D0 قبل الـ probe (ACPI) |

#### I2C Frequency Modes

| Macro | القيمة | الوضع |
|-------|--------|-------|
| `I2C_MAX_STANDARD_MODE_FREQ` | 100 kHz | Standard Mode |
| `I2C_MAX_FAST_MODE_FREQ` | 400 kHz | Fast Mode |
| `I2C_MAX_FAST_MODE_PLUS_FREQ` | 1 MHz | Fast Mode Plus |
| `I2C_MAX_TURBO_MODE_FREQ` | 1.4 MHz | Turbo Mode |
| `I2C_MAX_HIGH_SPEED_MODE_FREQ` | 3.4 MHz | High Speed Mode |
| `I2C_MAX_ULTRA_FAST_MODE_FREQ` | 5 MHz | Ultra Fast Mode |

#### `I2C_FUNC_*` — Functionality Bits (أهمها)

| Flag | المعنى |
|------|--------|
| `I2C_FUNC_I2C` | يدعم raw I2C transfers |
| `I2C_FUNC_10BIT_ADDR` | يدعم عناوين 10-bit |
| `I2C_FUNC_PROTOCOL_MANGLING` | يدعم تعديل البروتوكول |
| `I2C_FUNC_SMBUS_PEC` | يدعم Packet Error Checking |
| `I2C_FUNC_NOSTART` | يدعم `I2C_M_NOSTART` |
| `I2C_FUNC_SLAVE` | يدعم وضع الـ slave |
| `I2C_FUNC_SMBUS_QUICK` | SMBus Quick Command |
| `I2C_FUNC_SMBUS_READ_BLOCK_DATA` | قراءة block مع `I2C_M_RECV_LEN` |
| `I2C_FUNC_SMBUS_HOST_NOTIFY` | يدعم SMBus Host Notify |
| `I2C_FUNC_SMBUS_EMUL` | مجموعة كاملة يمكن محاكاتها عبر I2C msgs |

#### SMBus Transaction Types

| Constant | Value | العملية |
|----------|-------|---------|
| `I2C_SMBUS_QUICK` | 0 | Quick Command — بس R/W bit |
| `I2C_SMBUS_BYTE` | 1 | Send/Receive Byte |
| `I2C_SMBUS_BYTE_DATA` | 2 | Read/Write Byte Data |
| `I2C_SMBUS_WORD_DATA` | 3 | Read/Write Word Data |
| `I2C_SMBUS_PROC_CALL` | 4 | Process Call |
| `I2C_SMBUS_BLOCK_DATA` | 5 | Read/Write Block Data |
| `I2C_SMBUS_I2C_BLOCK_DATA` | 8 | I2C Block Transfer |

---

### 1. الـ Structs المهمة

#### `struct i2c_msg`
**الغرض:** الوحدة الأساسية لأي transaction على الـ bus — كل رسالة واحدة (segment) تبدأ بـ START.

```c
struct i2c_msg {
    __u16 addr;   /* عنوان الـ slave: 7-bit أو 10-bit */
    __u16 flags;  /* I2C_M_RD, I2C_M_TEN, ... */
    __u16 len;    /* عدد البيانات في buf */
    __u8 *buf;    /* البيانات المرسلة أو المستقبلة */
};
```

**الاتصال بـ structs أخرى:**
- يُمرَّر كـ array لـ `i2c_algorithm.xfer(adap, msgs, num)`
- يُستخدم في `i2c_transfer()` و `__i2c_transfer()`

---

#### `struct i2c_client`
**الغرض:** يمثّل جهاز I2C slave واحد (chip) متصل بالـ bus. ده الـ handle اللي الـ driver بيتعامل معاه.

```c
struct i2c_client {
    unsigned short flags;       /* I2C_CLIENT_PEC, TEN, SLAVE, ... */
    unsigned short addr;        /* عنوان الـ chip — 7-bit في الـ lower bits */
    char name[I2C_NAME_SIZE];   /* نوع الجهاز — مثلاً "tmp102" */
    struct i2c_adapter *adapter;/* الـ adapter اللي الجهاز متصل بيه */
    struct device dev;          /* integration مع device model */
    int init_irq;               /* IRQ اتحدد وقت الـ init */
    int irq;                    /* IRQ الجهاز بيطلعه */
    struct list_head detected;  /* عضو في i2c_driver.clients أو userspace_devices */
    i2c_slave_cb_t slave_cb;    /* callback لو شغالين كـ slave */
    void *devres_group_id;      /* ID لمجموعة الـ devres بتاعة الـ probe */
    struct dentry *debugfs;     /* debugfs directory */
};
```

**الاتصال:**
- `client->adapter` يشير لـ `i2c_adapter`
- `client->dev` هو embedded `struct device` (الـ parent هو `adapter->dev`)
- عضو في `i2c_driver.clients` list لو اتاكتشف automatically

---

#### `struct i2c_driver`
**الغرض:** يمثّل الـ driver نفسه — يحتوي على الـ callbacks وجدول الأجهزة المدعومة.

```c
struct i2c_driver {
    unsigned int class;          /* ماسك لتصنيف الأجهزة (للـ detect) */
    int (*probe)(struct i2c_client *);          /* bind: جهاز اتلاقى */
    void (*remove)(struct i2c_client *);        /* unbind: جهاز اتشال */
    void (*shutdown)(struct i2c_client *);      /* قبل إغلاق النظام */
    void (*alert)(struct i2c_client *,
                  enum i2c_alert_protocol, unsigned int);
    int (*command)(struct i2c_client *, unsigned int cmd, void *arg);
    struct device_driver driver;                /* embedded driver model */
    const struct i2c_device_id *id_table;       /* جدول الأجهزة المدعومة */
    int (*detect)(struct i2c_client *,
                  struct i2c_board_info *);     /* اكتشاف تلقائي */
    const unsigned short *address_list;         /* عناوين يجرّب عليها الـ detect */
    struct list_head clients;                   /* الأجهزة اللي اتاكتشفت */
    u32 flags;                                  /* I2C_DRV_ACPI_WAIVE_D0_PROBE */
};
```

**الاتصال:**
- `driver.driver` embedded في `struct device_driver` — الـ bus core بيستخدمه
- `clients` list تحتوي على `i2c_client.detected` nodes

---

#### `struct i2c_adapter`
**الغرض:** يمثّل الـ I2C bus controller نفسه (الـ hardware). هو اللي بيتحكم في الـ SCL/SDA وبينفّذ الـ transfers الفعلية.

```c
struct i2c_adapter {
    struct module *owner;
    unsigned int class;                          /* أنواع الأجهزة اللي يقدر يكتشفها */
    const struct i2c_algorithm *algo;            /* طريقة الوصول للـ bus */
    void *algo_data;                             /* بيانات خاصة بالـ algorithm */
    const struct i2c_lock_operations *lock_ops;  /* عمليات الـ locking */
    struct rt_mutex bus_lock;                    /* lock الـ bus الكامل */
    struct rt_mutex mux_lock;                    /* lock لشجرة الـ mux */
    int timeout;                                 /* timeout بالـ jiffies */
    int retries;                                 /* عدد مرات إعادة المحاولة */
    struct device dev;                           /* embedded device */
    unsigned long locked_flags;                  /* I2C_ALF_IS_SUSPENDED, ... */
    int nr;                                      /* رقم الـ bus */
    char name[48];
    struct completion dev_released;
    struct mutex userspace_clients_lock;
    struct list_head userspace_clients;
    struct i2c_bus_recovery_info *bus_recovery_info;
    const struct i2c_adapter_quirks *quirks;
    struct irq_domain *host_notify_domain;
    struct regulator *bus_regulator;
    struct dentry *debugfs;
    DECLARE_BITMAP(addrs_in_instantiation, 128); /* عناوين 7-bit جارٍ تسجيلها */
};
```

**الاتصال:**
- `algo` يشير لـ `i2c_algorithm` — الـ ops table للـ hardware
- `lock_ops` يشير لـ `i2c_lock_operations`
- `bus_recovery_info` يشير لـ `i2c_bus_recovery_info`
- `quirks` يشير لـ `i2c_adapter_quirks`
- `dev.parent` هو الـ platform device اللي يملك الـ controller

---

#### `struct i2c_algorithm`
**الغرض:** الـ ops table للـ adapter — زي الـ `file_operations` للـ VFS. كل adapter بيوفّر تطبيق لهذه الـ callbacks.

```c
struct i2c_algorithm {
    /* raw I2C transfer — الأساسي */
    union {
        int (*xfer)(struct i2c_adapter *, struct i2c_msg *, int num);
        int (*master_xfer)(...); /* deprecated alias */
    };
    /* نفس xfer بس في atomic context (لـ PMICs وقت الـ shutdown) */
    union {
        int (*xfer_atomic)(...);
        int (*master_xfer_atomic)(...); /* deprecated */
    };
    /* SMBus native — لو NULL الـ core بيحاكي عبر I2C msgs */
    int (*smbus_xfer)(struct i2c_adapter *, u16 addr,
                      unsigned short flags, char read_write,
                      u8 command, int size, union i2c_smbus_data *);
    int (*smbus_xfer_atomic)(...);
    /* ما الـ functionality المدعومة */
    u32 (*functionality)(struct i2c_adapter *);
    /* slave/target mode */
    union { int (*reg_target)(...); int (*reg_slave)(...); };
    union { int (*unreg_target)(...); int (*unreg_slave)(...); };
};
```

---

#### `struct i2c_lock_operations`
**الغرض:** يسمح لأنظمة الـ mux بتخصيص سلوك الـ locking بدل الـ rt_mutex الافتراضي.

```c
struct i2c_lock_operations {
    void (*lock_bus)(struct i2c_adapter *, unsigned int flags);
    int  (*trylock_bus)(struct i2c_adapter *, unsigned int flags);
    void (*unlock_bus)(struct i2c_adapter *, unsigned int flags);
};
```

---

#### `struct i2c_board_info`
**الغرض:** template لإنشاء `i2c_client` — بيحتوي كل المعلومات اللازمة لتعريف جهاز على الـ bus.

```c
struct i2c_board_info {
    char type[I2C_NAME_SIZE];           /* اسم الجهاز → i2c_client.name */
    unsigned short flags;               /* → i2c_client.flags */
    unsigned short addr;                /* → i2c_client.addr */
    const char *dev_name;               /* يلغي الاسم الافتراضي <busnr>-<addr> */
    void *platform_data;
    struct fwnode_handle *fwnode;       /* من الـ firmware (DT/ACPI) */
    const struct software_node *swnode;
    const struct resource *resources;
    unsigned int num_resources;
    int irq;
};
```

---

#### `struct i2c_timings`
**الغرض:** تخزين قيم التوقيت المقروءة من الـ firmware (Device Tree) لتهيئة الـ controller hardware.

```c
struct i2c_timings {
    u32 bus_freq_hz;               /* تردد الـ bus */
    u32 scl_rise_ns;               /* زمن صعود SCL — t(r) */
    u32 scl_fall_ns;               /* زمن هبوط SCL — t(f) */
    u32 scl_int_delay_ns;          /* تأخير إضافي للـ IP core على SCL */
    u32 sda_fall_ns;               /* زمن هبوط SDA */
    u32 sda_hold_ns;               /* hold time إضافي على SDA */
    u32 digital_filter_width_ns;   /* عرض الـ spike الممكن تصفيته رقمياً */
    u32 analog_filter_cutoff_freq_hz; /* تردد قطع الـ low-pass analog filter */
};
```

---

#### `struct i2c_bus_recovery_info`
**الغرض:** معلومات وأدوات استرداد الـ bus لو علق في حالة SDA=low (bus hung). فيه طريقتين: GPIO recovery أو custom.

```c
struct i2c_bus_recovery_info {
    int (*recover_bus)(struct i2c_adapter *);       /* الدخول للـ recovery */
    int (*get_scl)(struct i2c_adapter *);
    void (*set_scl)(struct i2c_adapter *, int val);
    int (*get_sda)(struct i2c_adapter *);
    void (*set_sda)(struct i2c_adapter *, int val);
    int (*get_bus_free)(struct i2c_adapter *);      /* optional */
    void (*prepare_recovery)(struct i2c_adapter *); /* إعداد الـ pinmux */
    void (*unprepare_recovery)(struct i2c_adapter *);
    /* GPIO recovery */
    struct gpio_desc *scl_gpiod;
    struct gpio_desc *sda_gpiod;
    struct pinctrl *pinctrl;
    struct pinctrl_state *pins_default; /* الحالة الطبيعية للـ pins */
    struct pinctrl_state *pins_gpio;    /* حالة الـ recovery */
};
```

---

#### `struct i2c_adapter_quirks`
**الغرض:** يوصف قيود الـ hardware للـ adapter — الـ core بيستخدمها للـ validation قبل ما يبعت الـ transfer.

```c
struct i2c_adapter_quirks {
    u64 flags;                 /* I2C_AQ_* bitmask */
    int max_num_msgs;          /* أقصى عدد رسائل في transfer واحد */
    u16 max_write_len;         /* أقصى طول رسالة write */
    u16 max_read_len;          /* أقصى طول رسالة read */
    u16 max_comb_1st_msg_len;  /* أقصى طول الرسالة الأولى في combined mode */
    u16 max_comb_2nd_msg_len;  /* أقصى طول الرسالة الثانية */
};
```

---

#### `struct i2c_device_identity`
**الغرض:** تعريف الجهاز عبر بروتوكول SMBus — manufacturer, part, die revision.

```c
struct i2c_device_identity {
    u16 manufacturer_id; /* 0-4095 — قاعدة بيانات NXP */
    u16 part_id;         /* 0-511 */
    u8  die_revision;    /* 0-7 */
};
```

---

#### `union i2c_smbus_data`
**الغرض:** يمثّل payload بيانات SMBus — byte أو word أو block بحد أقصى 32 byte.

```c
union i2c_smbus_data {
    __u8  byte;
    __u16 word;
    __u8  block[I2C_SMBUS_BLOCK_MAX + 2]; /* block[0] = length */
};
```

---

### 2. مخطط علاقات الـ Structs

```
  ┌────────────────────────────────────────────────────────────┐
  │                    struct i2c_adapter                      │
  │                                                            │
  │  algo ──────────────────────► struct i2c_algorithm         │
  │  lock_ops ──────────────────► struct i2c_lock_operations   │
  │  quirks ────────────────────► struct i2c_adapter_quirks    │
  │  bus_recovery_info ─────────► struct i2c_bus_recovery_info │
  │  host_notify_domain ────────► struct irq_domain            │
  │  bus_regulator ─────────────► struct regulator             │
  │  struct device dev          (embedded)                     │
  │  struct rt_mutex bus_lock   (embedded)                     │
  │  struct rt_mutex mux_lock   (embedded)                     │
  │  struct list_head userspace_clients                        │
  └────────────────────┬───────────────────────────────────────┘
                       │ ▲ client->adapter (back-pointer)
                       │
  ┌────────────────────▼───────────────────────────────────────┐
  │                    struct i2c_client                       │
  │                                                            │
  │  addr, flags, name[I2C_NAME_SIZE]                          │
  │  struct i2c_adapter *adapter ──────────────────────────────┘
  │  struct device dev          (embedded, parent=adapter->dev)│
  │  i2c_slave_cb_t slave_cb ──► callback function             │
  │  struct list_head detected  (member of driver->clients)    │
  └────────────────────┬───────────────────────────────────────┘
                       │ client->dev.driver
                       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    struct i2c_driver                        │
  │                                                             │
  │  probe / remove / shutdown / alert / command / detect       │
  │  struct device_driver driver  (embedded)                    │
  │  const struct i2c_device_id *id_table                       │
  │  struct list_head clients ──► i2c_client.detected nodes     │
  └─────────────────────────────────────────────────────────────┘

  Transfer data path:
  ─────────────────────────────────────────────────────────────
  i2c_transfer()
    └──► struct i2c_msg[] (addr + flags + len + buf)
           └──► i2c_adapter.algo->xfer(adap, msgs, num)

  SMBus data payload:
  ─────────────────────────────────────────────────────────────
  i2c_smbus_xfer()
    └──► union i2c_smbus_data { byte | word | block[] }
           └──► i2c_algorithm.smbus_xfer() [native]
               or __i2c_transfer() [emulated via i2c_msg]

  Board info → Client creation:
  ─────────────────────────────────────────────────────────────
  struct i2c_board_info
    └──► i2c_new_client_device()
           └──► struct i2c_client (heap allocated)

  Timing configuration:
  ─────────────────────────────────────────────────────────────
  i2c_parse_fw_timings(dev, &timings, true)
    └──► struct i2c_timings
           └──► adapter driver reads fields to program hardware
```

---

### 3. Lifecycle Diagrams

#### Lifecycle الـ Adapter

```
  Platform driver probe()
        │
        ▼
  alloc + fill i2c_adapter:
  ┌─────────────────────────────┐
  │  adap.algo  = &my_algo      │
  │  adap.owner = THIS_MODULE   │
  │  adap.name  = "my-i2c"     │
  │  adap.dev.parent = &pdev->dev│
  └─────────────────────────────┘
        │
        ├──► i2c_add_adapter(adap)        [رقم bus تلقائي]
        │    أو
        └──► i2c_add_numbered_adapter()   [رقم bus محدد مسبقاً]
                   │
                   ▼
        i2c-core: device_register(&adap->dev)
        /sys/bus/i2c/devices/i2c-N يظهر
                   │
                   ▼
        i2c-core: يمشي على board_info المسجّلة
        i2c_new_client_device() لكل جهاز
                   │
                   ▼
        الـ adapter جاهز لاستقبال transfers
                   │
           [system suspend]
                   │
                   ▼
        i2c_mark_adapter_suspended(adap)
        └── i2c_lock_bus() + set_bit(I2C_ALF_IS_SUSPENDED) + i2c_unlock_bus()
                   │
           [system resume]
                   │
                   ▼
        i2c_mark_adapter_resumed(adap)
        └── i2c_lock_bus() + clear_bit(I2C_ALF_IS_SUSPENDED) + i2c_unlock_bus()
                   │
           [remove / module unload]
                   │
                   ▼
        i2c_del_adapter(adap)
        └── يمسح كل i2c_client مسجّل
        └── device_unregister(&adap->dev)
```

---

#### Lifecycle الـ Driver

```
  module_i2c_driver(my_driver)
  = module_init: i2c_add_driver(&my_driver)
  = i2c_register_driver(THIS_MODULE, &my_driver)
        │
        ▼
  driver_register(&my_driver.driver)
  على i2c_bus_type
        │
        ▼
  bus core: match كل i2c_client موجود
  (عبر id_table أو DT compatible أو ACPI _HID)
        │
        ▼
  my_driver.probe(client) لكل جهاز مطابق:
  ┌───────────────────────────────────────┐
  │  priv = devm_kzalloc(...)             │
  │  i2c_set_clientdata(client, priv)     │
  │  alloc resources (IRQ, regmap, ...)   │
  │  register with subsystem (hwmon, etc) │
  └───────────────────────────────────────┘
        │
   [device removed or driver unloaded]
        │
        ▼
  my_driver.remove(client)
  └── devres فرّغت الـ resources تلقائياً
        │
        ▼
  module_exit: i2c_del_driver(&my_driver)
```

---

#### Lifecycle الـ Client (Dynamic Creation)

```
  [board code أو driver يريد إضافة جهاز برمجياً]

  struct i2c_board_info info = {
      I2C_BOARD_INFO("tmp102", 0x48),
      .irq = gpio_to_irq(GPIO_TMP_ALERT),
  };
        │
        ▼
  client = i2c_new_client_device(adapter, &info)
  ┌────────────────────────────────────────────┐
  │  client = kzalloc(sizeof(*client))         │
  │  client->addr    = info.addr               │
  │  client->flags   = info.flags              │
  │  strlcpy(client->name, info.type)          │
  │  client->adapter = adapter                 │
  │  client->dev.parent = &adapter->dev        │
  │  device_register(&client->dev)             │
  └────────────────────────────────────────────┘
        │
        ▼
  bus core يبحث عن driver مطابق → probe()
        │
   [teardown]
        │
        ▼
  i2c_unregister_device(client)
  └── device_unregister(&client->dev)
  └── driver->remove(client) يُستدعى تلقائياً
```

---

### 4. Call Flow Diagrams

#### Raw I2C Write (Master Transmit)

```
driver calls:
  i2c_master_send(client, buf, count)
    │
    └──► i2c_transfer_buffer_flags(client, buf, count, flags=0)
           │
           ├── build: struct i2c_msg msg = {
           │       .addr  = client->addr,
           │       .flags = 0,        /* write */
           │       .len   = count,
           │       .buf   = buf,
           │   };
           │
           └──► i2c_transfer(client->adapter, &msg, 1)
                  │
                  ├── i2c_lock_bus(adap, I2C_LOCK_ROOT_ADAPTER)
                  │     └── adap->lock_ops->lock_bus()
                  │           └── rt_mutex_lock(&adap->bus_lock)
                  │
                  ├── check: I2C_ALF_IS_SUSPENDED? → return -EACCES
                  │
                  ├── check: adap->quirks validation
                  │     (max_num_msgs, max_write_len, ...)
                  │
                  ├── __i2c_transfer(adap, &msg, 1)
                  │     └── adap->algo->xfer(adap, msgs, num)
                  │           └── [hardware driver]
                  │                 write to MMIO TX FIFO
                  │                 wait for interrupt/completion
                  │                 return msgs_transferred or -errno
                  │
                  └── i2c_unlock_bus(adap, I2C_LOCK_ROOT_ADAPTER)
                        └── rt_mutex_unlock(&adap->bus_lock)
```

---

#### SMBus Read Byte Data

```
driver calls:
  i2c_smbus_read_byte_data(client, reg=0x01)
    │
    └──► i2c_smbus_xfer(adap, addr=0x48, flags=0,
                        read_write=I2C_SMBUS_READ,
                        command=0x01,
                        size=I2C_SMBUS_BYTE_DATA,
                        &data)
           │
           ├── i2c_lock_bus(adap, I2C_LOCK_ROOT_ADAPTER)
           │
           ├── if (adap->algo->smbus_xfer):
           │     └──► adap->algo->smbus_xfer(...)   [native SMBus hardware]
           │
           └── else (emulation via raw I2C):
                 i2c_smbus_xfer_emulated(...)
                 ├── msg[0]: { addr=0x48, flags=WRITE, len=1, buf=&reg }
                 │          [send register address]
                 ├── msg[1]: { addr=0x48, flags=READ,  len=1, buf=&data.byte }
                 │          [receive value]
                 └──► __i2c_transfer(adap, msgs, 2)
                             └──► adap->algo->xfer(adap, msgs, 2)
```

---

#### Bus Recovery (SDA Stuck Low)

```
  i2c_transfer() fails or adapter driver detects bus hang
    │
    └──► i2c_recover_bus(adap)
           │
           ├── adap->bus_recovery_info == NULL? → return -EOPNOTSUPP
           │
           ├── bri->prepare_recovery(adap)        [optional]
           │     └── pinctrl_select_state(bri->pins_gpio)
           │           [SCL/SDA switched from I2C mode to GPIO mode]
           │
           ├── bri->recover_bus(adap)
           │   [usually i2c_generic_scl_recovery()]
           │     loop up to 9 times:
           │     ├── bri->set_scl(adap, 0)
           │     ├── udelay(...)
           │     ├── bri->set_scl(adap, 1)
           │     ├── udelay(...)
           │     └── if bri->get_sda(adap) == 1: SDA free → break
           │     └── send STOP: SDA low → SCL high → SDA high
           │
           └── bri->unprepare_recovery(adap)      [optional]
                 └── pinctrl_select_state(bri->pins_default)
                       [pins restored to I2C function]
```

---

#### Slave Mode Event Flow

```
  [external I2C master sends START + our address]
    │
    └──► adapter hardware interrupt
           └──► adapter driver
                  └──► i2c_slave_event(client,
                                       I2C_SLAVE_WRITE_REQUESTED, NULL)
                         └──► client->slave_cb(client, event, val)
                                └──► slave driver: prepare receive buffer

  [master sends data byte 0xAB]
    └──► adapter hardware interrupt
           └──► adapter driver has byte 0xAB
                  └──► i2c_slave_event(client,
                                       I2C_SLAVE_WRITE_RECEIVED, &byte)
                         └──► slave_cb: store 0xAB in ring buffer

  [master sends STOP]
    └──► adapter hardware interrupt
           └──► i2c_slave_event(client, I2C_SLAVE_STOP, NULL)
                  └──► slave_cb: finalize transaction

  [master wants to READ from us]
    └──► I2C_SLAVE_READ_REQUESTED
           └──► slave_cb: *val = next_byte_to_send
         [master clocks out the byte, needs next one]
    └──► I2C_SLAVE_READ_PROCESSED
           └──► slave_cb: *val = next_byte_to_send
```

---

### 5. Locking Strategy

#### الـ Locks الموجودة

| Lock | النوع | المكان | ما يحميه |
|------|-------|--------|----------|
| `adap->bus_lock` | `rt_mutex` | `struct i2c_adapter` | الوصول الكامل للـ bus — كل الـ transfers |
| `adap->mux_lock` | `rt_mutex` | `struct i2c_adapter` | تسلسل access لشجرة الـ mux |
| `adap->userspace_clients_lock` | `mutex` | `struct i2c_adapter` | قائمة `userspace_clients` |
| `i2c_lock_operations` | custom callbacks | per-adapter | يمكن استبدال الـ rt_mutex بـ custom logic لشجرة الـ mux |

---

#### ليه `rt_mutex` وليس `mutex` عادي؟

الـ `rt_mutex` بيدعم **priority inheritance**:

```
  Thread A (low priority)  — يمسك bus_lock ويعمل I2C transfer
  Thread B (high priority) — يحتاج bus_lock لـ PMIC access أثناء suspend

  بدون RT mutex:  Thread B ينتظر وقت غير محدد (priority inversion)
  مع    RT mutex: الـ kernel يرفع priority Thread A مؤقتاً
                  حتى يخلّص ويفرج عن الـ lock → Thread B يمشي
```

ده حيوي لأن I2C بيُستخدم لـ PMICs اللي بتتحكم في الـ power أثناء system suspend/resume في وقت حرج.

---

#### Lock Ordering (ترتيب الـ Locks — لازم تتبعه)

```
  [في حالة وجود MUX adapter tree]

  root_adapter->bus_lock     ← أول حاجة تمسكها
       └──► mux_lock         ← تاني
              └──► child_adapter->bus_lock   ← تالت

  الترتيب: root قبل child — دايماً بدون استثناء.

  [Transfer عادي بدون mux]

  i2c_lock_bus(adap, I2C_LOCK_ROOT_ADAPTER)
      ↓ rt_mutex_lock(&adap->bus_lock)
      ↓
      [adap->algo->xfer() — critical section]
      ↓
  i2c_unlock_bus(adap, I2C_LOCK_ROOT_ADAPTER)
      ↓ rt_mutex_unlock(&adap->bus_lock)

  [Suspend marking]

  i2c_lock_bus(adap, I2C_LOCK_ROOT_ADAPTER)
      set_bit(I2C_ALF_IS_SUSPENDED, &adap->locked_flags)
      [atomic op على locked_flags — مش محتاج lock إضافي]
  i2c_unlock_bus(adap, I2C_LOCK_ROOT_ADAPTER)
  [يضمن مفيش transfer شغال وقت ما بنحدد الـ suspended flag]
```

---

#### `I2C_LOCK_SEGMENT` vs `I2C_LOCK_ROOT_ADAPTER`

```
   root adapter (i2c-0)         ← bus_lock هنا يحمي كل الـ tree
         │
    ┌────┴────┐
  mux-A     mux-B               ← كل mux هو i2c_adapter منفصل
    │          │
  dev-A1    dev-B1

  I2C_LOCK_ROOT_ADAPTER:
  └── يمسك bus_lock الـ root
  └── يمنع أي transfer على الـ bus كله (root + كل الـ children)
  └── يُستخدم في: suspend, recovery, atomic transfers

  I2C_LOCK_SEGMENT:
  └── يمسك bus_lock الـ segment الحالي فقط
  └── يسمح لـ devices على segments تانية بالعمل بالتوازي
  └── يُستخدم في: MUX drivers لتقليل الـ contention
```

---

#### `addrs_in_instantiation` Bitmap

الـ `DECLARE_BITMAP(addrs_in_instantiation, 128)` في الـ adapter بيحمي من race condition لو اتنين contexts بيحاولوا يسجّلوا client بنفس العنوان في نفس الوقت. الـ core بيشيك على الـ bit قبل التسجيل ويمسحه لو فشل — كل ده جوه `userspace_clients_lock` mutex.
## Phase 4: شرح الـ Functions

---

### ملخص عام — Cheatsheet

#### مجموعات الـ Functions

| المجموعة | الـ Functions الرئيسية |
|---|---|
| **Master Transfer** | `i2c_transfer`, `__i2c_transfer`, `i2c_transfer_buffer_flags`, `i2c_master_send`, `i2c_master_recv` |
| **SMBus Access** | `i2c_smbus_xfer`, `__i2c_smbus_xfer`, `i2c_smbus_read/write_byte/byte_data/word_data/block_data` |
| **Adapter Registration** | `i2c_add_adapter`, `i2c_add_numbered_adapter`, `devm_i2c_add_adapter`, `i2c_del_adapter` |
| **Driver Registration** | `i2c_register_driver`, `i2c_del_driver`, `i2c_add_driver` (macro) |
| **Client/Device Mgmt** | `i2c_new_client_device`, `i2c_new_scanned_device`, `i2c_new_dummy_device`, `i2c_unregister_device` |
| **Bus Locking** | `i2c_lock_bus`, `i2c_trylock_bus`, `i2c_unlock_bus` |
| **Slave Mode** | `i2c_slave_register`, `i2c_slave_unregister`, `i2c_slave_event`, `i2c_detect_slave_mode` |
| **Adapter Lookup** | `i2c_get_adapter`, `i2c_put_adapter`, `i2c_find_adapter_by_fwnode`, `of_find_i2c_adapter_by_node` |
| **Helper/Utility** | `i2c_get_clientdata`, `i2c_set_clientdata`, `i2c_check_functionality`, `i2c_get_dma_safe_msg_buf` |
| **Bus Recovery** | `i2c_recover_bus`, `i2c_generic_scl_recovery` |
| **ACPI** | `i2c_acpi_find_bus_speed`, `i2c_acpi_new_device_by_fwnode`, `i2c_acpi_find_adapter_by_handle` |

---

### المجموعة الأولى: Master Transfer Functions

دي المجموعة المسؤولة عن إرسال واستقبال البيانات على I2C bus كـ master. كل الـ high-level helpers بتتحول في النهاية لـ `i2c_transfer` أو `__i2c_transfer`.

---

#### `i2c_transfer_buffer_flags`

```c
int i2c_transfer_buffer_flags(const struct i2c_client *client,
                              char *buf, int count, u16 flags);
```

بتبني `struct i2c_msg` واحدة من الـ buffer والـ flags المحددة، وبتعمل transfer واحدة على الـ adapter اللي الـ client متصل بيه. دي الـ core function اللي `i2c_master_send/recv` بيبنوا عليها.

| Parameter | الوصف |
|---|---|
| `client` | الـ slave device handle |
| `buf` | الـ data buffer (read أو write حسب الـ flags) |
| `count` | عدد الـ bytes، لازم < 64KB (لأن `msg.len` هو `u16`) |
| `flags` | `I2C_M_RD` للقراءة، `I2C_M_DMA_SAFE` لو الـ buffer DMA-safe |

**Return:** عدد الـ bytes اللي اتنقلت، أو negative errno في حالة error.

**Key details:** بتكوّن `struct i2c_msg` على الـ stack، بعدين بتعمل call لـ `i2c_transfer`. لو الـ buffer مش DMA-safe والـ adapter محتاج DMA، الـ core ممكن يعمل copy.

---

#### `i2c_master_send`

```c
static inline int i2c_master_send(const struct i2c_client *client,
                                  const char *buf, int count);
```

الـ inline wrapper الأبسط لإرسال data للـ slave. بتستدعي `i2c_transfer_buffer_flags` بـ `flags = 0` (write direction).

**Return:** عدد الـ bytes اللي اتكتبت، أو negative errno.

**Caller context:** Process context فقط — بتأخذ الـ bus lock.

---

#### `i2c_master_recv`

```c
static inline int i2c_master_recv(const struct i2c_client *client,
                                  char *buf, int count);
```

نفس `i2c_master_send` بس بتمرر `I2C_M_RD` flag عشان تقرأ من الـ slave.

**Return:** عدد الـ bytes اللي اتقرأت، أو negative errno.

---

#### `i2c_master_send_dmasafe` / `i2c_master_recv_dmasafe`

```c
static inline int i2c_master_send_dmasafe(const struct i2c_client *client,
                                          const char *buf, int count);
static inline int i2c_master_recv_dmasafe(const struct i2c_client *client,
                                          char *buf, int count);
```

نفس `send/recv` بس بيضيفوا `I2C_M_DMA_SAFE` flag للـ message. بتستخدمهم لما بتعرف إن الـ buffer اتعمله allocate بـ `kmalloc` أو مكان DMA-accessible، عشان تتجنب الـ bounce buffer overhead.

---

#### `i2c_transfer`

```c
int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num);
```

الـ workhorse الأساسي للـ I2C transfers. بتأخذ الـ bus lock، بتتحقق من الـ adapter state (suspended؟)، وبتعدي الـ messages للـ `adap->algo->xfer()` مع retry logic لو حصل EAGAIN.

| Parameter | الوصف |
|---|---|
| `adap` | الـ I2C adapter |
| `msgs` | array من الـ I2C messages |
| `num` | عدد الـ messages |

**Return:** عدد الـ messages اللي اتنفذت بنجاح، أو negative errno.

**Locking:** بتأخذ `adap->bus_lock` (rt_mutex) للـ duration كامل. لو الـ adapter suspended (`I2C_ALF_IS_SUSPENDED` set)، بترجع `-EACCES`.

**Key details:**
- بتعمل retry عدد `adap->retries` مرات لو الـ transfer فشل بـ `-EAGAIN`
- بتتحقق من `adap->quirks` قبل التنفيذ
- لا تستخدمها من atomic context — استخدم `__i2c_transfer` لو عندك الـ lock مسبقاً

**Pseudocode:**
```c
i2c_transfer(adap, msgs, num):
    i2c_lock_bus(adap, I2C_LOCK_SEGMENT)
    if adapter_is_suspended(adap):
        unlock; return -EACCES
    for retry in range(adap->retries + 1):
        ret = adap->algo->xfer(adap, msgs, num)
        if ret != -EAGAIN: break
    i2c_unlock_bus(adap, I2C_LOCK_SEGMENT)
    return ret
```

---

#### `__i2c_transfer`

```c
int __i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num);
```

الـ unlocked version من `i2c_transfer`. بتفترض إن الـ caller عنده الـ bus lock مسبقاً. بتستخدمها في الـ mux drivers أو لو عايز تعمل multiple transfers تحت نفس الـ lock.

**Caller context:** لازم تكون أخدت الـ bus lock بـ `i2c_lock_bus()` قبل الاستدعاء.

---

### المجموعة الثانية: SMBus Access Functions

الـ SMBus protocol هو subset من I2C بـ higher-level transactions. الـ kernel بيوفر functions جاهزة لكل نوع من الـ SMBus transactions. لو الـ adapter مش عنده `smbus_xfer`، الـ core بيعمل emulation عبر I2C messages.

---

#### `i2c_smbus_xfer`

```c
s32 i2c_smbus_xfer(struct i2c_adapter *adapter, u16 addr,
                   unsigned short flags, char read_write, u8 command,
                   int protocol, union i2c_smbus_data *data);
```

الـ generic SMBus access function. بتأخذ الـ bus lock وبتستدعي `__i2c_smbus_xfer`.

| Parameter | الوصف |
|---|---|
| `adapter` | الـ I2C adapter |
| `addr` | عنوان الـ slave (7-bit) |
| `flags` | client flags (مثلاً PEC enable) |
| `read_write` | `I2C_SMBUS_READ` أو `I2C_SMBUS_WRITE` |
| `command` | الـ register/command byte |
| `protocol` | نوع الـ transaction (`I2C_SMBUS_BYTE_DATA`, `I2C_SMBUS_WORD_DATA`, إلخ) |
| `data` | pointer لـ `union i2c_smbus_data` |

**Return:** `0` أو positive value للنجاح، negative errno للفشل.

**Key details:** لو الـ adapter عنده `smbus_xfer`، بيستخدمها مباشرة. غير كده، بيعمل emulation عبر `i2c_transfer`. بتتحقق من الـ functionality قبل التنفيذ.

---

#### `__i2c_smbus_xfer`

```c
s32 __i2c_smbus_xfer(struct i2c_adapter *adapter, u16 addr,
                     unsigned short flags, char read_write, u8 command,
                     int protocol, union i2c_smbus_data *data);
```

الـ unlocked version. نفس المنطق بس بتفترض إن الـ caller عنده الـ lock.

---

#### `i2c_smbus_read_byte`

```c
s32 i2c_smbus_read_byte(const struct i2c_client *client);
```

بتقرأ byte واحدة من الـ slave من غير إرسال command byte. بتنفذ **SMBus Receive Byte** protocol. الـ return value: القيمة المقروءة كـ unsigned في الـ 8 bits الـ lower، أو negative errno.

---

#### `i2c_smbus_write_byte`

```c
s32 i2c_smbus_write_byte(const struct i2c_client *client, u8 value);
```

بتكتب byte واحدة على الـ bus من غير command byte. **SMBus Send Byte** protocol. بتستخدمها للـ simple control flags زي power-on لبعض الـ sensors.

---

#### `i2c_smbus_read_byte_data`

```c
s32 i2c_smbus_read_byte_data(const struct i2c_client *client, u8 command);
```

بتعمل write للـ command byte أولاً، بعدين repeated START، وبتقرأ byte واحدة. ده الـ **SMBus Read Byte Data** وده الـ pattern الشائع جداً لقراءة register من device زي accelerometer أو temperature sensor.

**Return:** الـ byte المقروءة (0-255) كـ `s32`، أو negative errno.

---

#### `i2c_smbus_write_byte_data`

```c
s32 i2c_smbus_write_byte_data(const struct i2c_client *client,
                              u8 command, u8 value);
```

بتكتب command byte وبعدها data byte في نفس الـ transaction. **SMBus Write Byte Data** — الطريقة القياسية لكتابة register.

**Return:** `0` للنجاح، negative errno للفشل.

---

#### `i2c_smbus_read_word_data`

```c
s32 i2c_smbus_read_word_data(const struct i2c_client *client, u8 command);
```

بتقرأ 16-bit word من الـ slave. الـ word بييجي **little-endian** (low byte أولاً) حسب SMBus spec. **Return:** الـ 16-bit value كـ `s32`، أو negative errno.

---

#### `i2c_smbus_write_word_data`

```c
s32 i2c_smbus_write_word_data(const struct i2c_client *client,
                              u8 command, u16 value);
```

بتكتب 16-bit word. الـ byte order: little-endian.

---

#### `i2c_smbus_read_word_swapped` / `i2c_smbus_write_word_swapped`

```c
static inline s32 i2c_smbus_read_word_swapped(const struct i2c_client *client,
                                               u8 command);
static inline s32 i2c_smbus_write_word_swapped(const struct i2c_client *client,
                                                u8 command, u16 value);
```

نفس `read/write_word_data` بس بيعملوا `swab16()` على الـ value. بيستخدمهم مع الـ devices اللي بترسل الـ word بـ big-endian order (مخالفة لـ SMBus spec)، زي بعض الـ sensors القديمة.

---

#### `i2c_smbus_read_block_data`

```c
s32 i2c_smbus_read_block_data(const struct i2c_client *client,
                              u8 command, u8 *values);
```

بتقرأ block من الـ data حسب **SMBus Block Read** protocol. الـ slave بيرسل byte أولاً فيه عدد الـ bytes اللي هيتبعها (max 32 byte). **Return:** عدد الـ bytes المقروءة، أو negative errno.

**Key details:** بتتطلب `I2C_FUNC_SMBUS_READ_BLOCK_DATA` في الـ adapter.

---

#### `i2c_smbus_write_block_data`

```c
s32 i2c_smbus_write_block_data(const struct i2c_client *client,
                               u8 command, u8 length, const u8 *values);
```

بتكتب block من الـ data. الـ `length` لازم تكون <= 32 byte.

---

#### `i2c_smbus_read_i2c_block_data`

```c
s32 i2c_smbus_read_i2c_block_data(const struct i2c_client *client,
                                  u8 command, u8 length, u8 *values);
```

بتقرأ عدد محدد من الـ bytes تبدأ من الـ command address. المفرق عن `read_block_data` إن هنا الـ master هو اللي بيحدد الـ length مش الـ slave. بتستخدمها مع EEPROMs والـ sensor arrays.

**Return:** عدد الـ bytes المقروءة، أو negative errno.

---

#### `i2c_smbus_write_i2c_block_data`

```c
s32 i2c_smbus_write_i2c_block_data(const struct i2c_client *client,
                                   u8 command, u8 length, const u8 *values);
```

بتكتب block بـ I2C-style (مش SMBus block format). Max 32 byte.

---

#### `i2c_smbus_read_i2c_block_data_or_emulated`

```c
s32 i2c_smbus_read_i2c_block_data_or_emulated(const struct i2c_client *client,
                                              u8 command, u8 length,
                                              u8 *values);
```

بتحاول تستخدم `read_i2c_block_data` أولاً، لو الـ adapter مش بيدعمها، بتعمل emulation عبر multiple byte reads. مفيدة للـ portability.

---

#### `i2c_smbus_pec`

```c
u8 i2c_smbus_pec(u8 crc, u8 *p, size_t count);
```

بتحسب الـ **Packet Error Code** (CRC-8) لـ SMBus PEC. بتستخدمها الـ core داخلياً لو `I2C_CLIENT_PEC` set في الـ client flags.

---

### المجموعة الثالثة: Adapter Registration & Management

الـ adapter هو الـ hardware controller (مثلاً i2c-designware، i2c-bcm2835). لازم يتسجل مع الـ I2C core قبل ما أي device يشتغل عليه.

---

#### `i2c_add_adapter`

```c
int i2c_add_adapter(struct i2c_adapter *adap);
```

بتسجل الـ adapter مع الـ I2C core. الـ core بيخصص رقم dynamic للـ adapter (في `adap->nr`). بتنشئ الـ sysfs entries والـ debugfs وبتعمل probe لأي devices اتسجلت سابقاً بـ `i2c_register_board_info`.

| Parameter | الوصف |
|---|---|
| `adap` | الـ adapter struct — لازم `adap->algo` يكون set |

**Return:** `0` للنجاح، negative errno للفشل.

**Caller context:** Process context (init أو probe). بتأخذ internal mutex.

**Key details:** قبل الاستدعاء، لازم تضبط: `adap->owner`, `adap->algo`, `adap->dev.parent`.

---

#### `i2c_add_numbered_adapter`

```c
int i2c_add_numbered_adapter(struct i2c_adapter *adap);
```

زي `i2c_add_adapter` بس بتحترم الـ `adap->nr` اللي الـ caller حدده. لو `adap->nr == -1`، بيتصرف زي `i2c_add_adapter`. بيستخدمها الـ board-specific code لضمان ثبات رقم الـ bus (مثلاً i2c-0 دايماً هو الـ main bus).

---

#### `devm_i2c_add_adapter`

```c
int devm_i2c_add_adapter(struct device *dev, struct i2c_adapter *adapter);
```

الـ devres-managed version من `i2c_add_adapter`. بتتعمل undo تلقائياً (`i2c_del_adapter`) لما الـ `dev` يتعمل unbound أو destroy. مثالية للـ platform drivers.

---

#### `i2c_del_adapter`

```c
void i2c_del_adapter(struct i2c_adapter *adap);
```

بتسجل خروج الـ adapter. بتعمل unregister لكل الـ client devices المتصلة بالـ adapter، بتحذف الـ sysfs entries، وبتحرر الـ adapter resources.

**Key details:** بتستدعي `i2c_unregister_device` على كل client متصل. بتنتظر completion (`adap->dev_released`) قبل الإرجاع لضمان عدم وجود references.

---

#### `i2c_get_adapter`

```c
struct i2c_adapter *i2c_get_adapter(int nr);
```

بترجع pointer لـ adapter بالرقم المحدد مع زيادة الـ reference count. **لازم** تستدعي `i2c_put_adapter()` لما تخلص.

**Return:** Pointer للـ adapter، أو `NULL` لو مش موجود.

---

#### `i2c_put_adapter`

```c
void i2c_put_adapter(struct i2c_adapter *adap);
```

بتنزل الـ reference count للـ adapter اللي اتاخد بـ `i2c_get_adapter`. دي الـ counterpart الإلزامية.

---

#### `i2c_adapter_depth`

```c
unsigned int i2c_adapter_depth(struct i2c_adapter *adapter);
```

بتحسب عمق الـ adapter في الـ mux tree. الـ root adapter بيرجع 0، والـ mux اللي فوقيه بيرجع 1، وهكذا. بتستخدمها للـ debugging ولتحديد الـ lock hierarchy في الـ mux scenarios.

---

#### `i2c_for_each_dev`

```c
int i2c_for_each_dev(void *data, int (*fn)(struct device *dev, void *data));
```

بتعمل iterate على كل الـ devices المسجلة في `i2c_bus_type` وبتنفذ الـ callback `fn` على كل منها. بتستخدمها للـ bus-wide operations.

---

#### `i2c_find_adapter_by_fwnode` / `i2c_get_adapter_by_fwnode`

```c
struct i2c_adapter *i2c_find_adapter_by_fwnode(struct fwnode_handle *fwnode);
struct i2c_adapter *i2c_get_adapter_by_fwnode(struct fwnode_handle *fwnode);
```

الفرق الأساسي: `find` بترجع pointer بـ `get_device()` (يستلزم `put_device`)، بينما `get` بترجع pointer بـ `i2c_get_adapter()` (يستلزم `i2c_put_adapter`). الاتنين بيدوروا على الـ adapter المرتبط بـ fwnode محدد من الـ DT أو ACPI.

---

### المجموعة الرابعة: Driver Registration

---

#### `i2c_register_driver`

```c
int i2c_register_driver(struct module *owner, struct i2c_driver *driver);
```

بتسجل الـ I2C driver مع الـ kernel bus infrastructure. بتعمل bind للـ driver مع كل device موجود على أي I2C bus بيطابق الـ `driver->id_table` أو الـ OF/ACPI tables.

| Parameter | الوصف |
|---|---|
| `owner` | عادةً `THIS_MODULE` |
| `driver` | الـ driver struct — لازم `driver->driver.name` وإما `id_table` أو `of_match_table` يكونوا set |

**Return:** `0` للنجاح، negative errno للفشل.

**Key details:** بتستخدم `driver_register()` في الداخل. لأغلب الـ drivers، استخدم `i2c_add_driver` macro بدل الاستدعاء المباشر.

---

#### `i2c_add_driver` (macro)

```c
#define i2c_add_driver(driver) \
    i2c_register_driver(THIS_MODULE, driver)
```

الـ convenience macro اللي بتمرر `THIS_MODULE` تلقائياً. ده اللي بتستخدمه في 99% من الـ drivers.

---

#### `i2c_del_driver`

```c
void i2c_del_driver(struct i2c_driver *driver);
```

بتعمل unregister للـ driver. بتستدعي `driver->remove` على كل device متصل بالـ driver قبل إلغاء التسجيل.

---

#### `module_i2c_driver` (macro)

```c
#define module_i2c_driver(__i2c_driver) \
    module_driver(__i2c_driver, i2c_add_driver, i2c_del_driver)
```

بتولّد `module_init` و`module_exit` تلقائياً عن طريق `module_driver`. للـ drivers اللي مش محتاجة initialization خاصة، ده بيلغي الـ boilerplate كلها. مثال عملي:

```c
/* driver for lm75 temperature sensor */
static struct i2c_driver lm75_driver = {
    .driver = { .name = "lm75", .of_match_table = lm75_of_match },
    .probe = lm75_probe,
    .id_table = lm75_ids,
};
module_i2c_driver(lm75_driver);
```

---

#### `builtin_i2c_driver` (macro)

```c
#define builtin_i2c_driver(__i2c_driver) \
    builtin_driver(__i2c_driver, i2c_add_driver)
```

للـ drivers المـ built-in في الـ kernel (مش modules). بيعمل `device_initcall` تلقائياً.

---

### المجموعة الخامسة: Client Device Management

---

#### `i2c_new_client_device`

```c
struct i2c_client *
i2c_new_client_device(struct i2c_adapter *adap, struct i2c_board_info const *info);
```

بتنشئ `struct i2c_client` جديد وبتسجله مع الـ device model. بتستخدمها للـ devices المعروفة عنوانها مسبقاً (من DT أو ACPI أو board file). الـ core بيعمل probe للـ driver المناسب تلقائياً.

| Parameter | الوصف |
|---|---|
| `adap` | الـ adapter اللي الـ device عليه |
| `info` | board info: اسم الـ chip، العنوان، الـ IRQ، الـ platform_data |

**Return:** Pointer للـ `i2c_client`، أو `ERR_PTR(-errno)` للفشل.

**Key details:** بتتحقق إن العنوان مش مكرر على نفس الـ adapter. بتستدعي `device_register()` في الداخل.

---

#### `i2c_new_scanned_device`

```c
struct i2c_client *
i2c_new_scanned_device(struct i2c_adapter *adap,
                       struct i2c_board_info *info,
                       unsigned short const *addr_list,
                       int (*probe)(struct i2c_adapter *adap, unsigned short addr));
```

بتعمل probe على قائمة عناوين وبتنشئ client للعنوان الأول اللي يستجيب. لو `probe == NULL`، بتستخدم `i2c_probe_func_quick_read` كـ default. مناسبة لـ second-source chips اللي ممكن تكون على أي عنوان.

**Return:** Pointer للـ `i2c_client`، أو `ERR_PTR(-ENODEV)` لو ما لقيتش device.

---

#### `i2c_probe_func_quick_read`

```c
int i2c_probe_func_quick_read(struct i2c_adapter *adap, unsigned short addr);
```

بتعمل **SMBus Quick Read** على عنوان محدد لاكتشاف وجود device. بترجع `1` لو الـ device استجاب، `0` لو لأ.

**Key details:** Quick Read ممكن يأثر على بعض الـ devices (مثلاً يوقفها أو يريّح بعض counters). استخدمها بحذر.

---

#### `i2c_new_dummy_device`

```c
struct i2c_client *
i2c_new_dummy_device(struct i2c_adapter *adapter, u16 address);
```

بتنشئ "dummy" client على عنوان محدد من غير driver حقيقي. بتستخدمها لـ reserve عنوان معين على الـ bus (مثلاً للـ devices multi-address زي بعض الـ codec chips).

**Return:** Pointer للـ `i2c_client`، أو `ERR_PTR(-errno)`.

---

#### `devm_i2c_new_dummy_device`

```c
struct i2c_client *
devm_i2c_new_dummy_device(struct device *dev, struct i2c_adapter *adap, u16 address);
```

الـ devres-managed version من `i2c_new_dummy_device`. بتتعمل cleanup تلقائياً لما الـ `dev` يتعمل unbind.

---

#### `i2c_new_ancillary_device`

```c
struct i2c_client *
i2c_new_ancillary_device(struct i2c_client *client, const char *name, u16 default_addr);
```

بتنشئ client لـ ancillary address لـ device ليها عناوين متعددة. بتقرأ العنوان من الـ firmware (DT/ACPI) باستخدام الـ `name`، ولو مش موجود بتستخدم `default_addr`. مثال: camera sensor ليه عنوان للـ ISP وعنوان تاني للـ AF motor.

---

#### `i2c_unregister_device`

```c
void i2c_unregister_device(struct i2c_client *client);
```

بتعمل unregister وتحرر الـ `i2c_client`. بتستدعي `device_unregister()` اللي بيطلق `driver->remove` وبيحرر الـ resources.

---

#### `i2c_register_board_info`

```c
int i2c_register_board_info(int busnum, struct i2c_board_info const *info, unsigned n);
```

بتسجل static board info قبل ما الـ adapter يكون جاهز. لما الـ adapter رقم `busnum` يتسجل لاحقاً بـ `i2c_add_numbered_adapter`، الـ core بيخلق الـ client devices تلقائياً. بتستخدمها في `arch_initcall` في الـ board files القديمة.

---

### المجموعة السادسة: Bus Locking

الـ I2C bus بيحتاج exclusive access أثناء الـ transfers. في الـ mux scenarios، في مستويات مختلفة من الـ locking.

---

#### `i2c_lock_bus`

```c
static inline void i2c_lock_bus(struct i2c_adapter *adapter, unsigned int flags);
```

بتأخذ الـ exclusive lock على الـ bus segment المحدد. `flags` بيحدد النطاق:

| Flag | الوصف |
|---|---|
| `I2C_LOCK_ROOT_ADAPTER` | بيلوك الـ root adapter (الكل) |
| `I2C_LOCK_SEGMENT` | بيلوك هذا الـ segment فقط في الـ mux tree |

**Implementation:** بتستدعي `adapter->lock_ops->lock_bus()`.

---

#### `i2c_trylock_bus`

```c
static inline int i2c_trylock_bus(struct i2c_adapter *adapter, unsigned int flags);
```

Non-blocking version. **Return:** `1` لو نجح اللوك، `0` لو الـ bus مشغول.

---

#### `i2c_unlock_bus`

```c
static inline void i2c_unlock_bus(struct i2c_adapter *adapter, unsigned int flags);
```

بتطلق الـ lock. لازم دايماً تيجي بعد كل `i2c_lock_bus` ناجح.

---

#### `i2c_mark_adapter_suspended`

```c
static inline void i2c_mark_adapter_suspended(struct i2c_adapter *adap);
```

بتعمل set لـ `I2C_ALF_IS_SUSPENDED` bit في `adap->locked_flags`. بعد كده، أي `i2c_transfer` بترجع `-EACCES`. بتستخدمها في `suspend` callbacks للـ adapter driver.

**Locking:** بتأخذ الـ root adapter lock نفسها عشان تضمن atomicity.

---

#### `i2c_mark_adapter_resumed`

```c
static inline void i2c_mark_adapter_resumed(struct i2c_adapter *adap);
```

عكس `i2c_mark_adapter_suspended`. بتعمل clear للـ bit وتسمح بالـ transfers مجدداً.

---

### المجموعة السابعة: Slave Mode Functions

الـ I2C slave mode (أو target mode) بيخلي الـ adapter يشتغل كـ slave — الـ adapter بيرد على requests من master آخر. بيتطلب `CONFIG_I2C_SLAVE`.

---

#### `i2c_slave_register`

```c
int i2c_slave_register(struct i2c_client *client, i2c_slave_cb_t slave_cb);
```

بتسجل الـ client كـ slave على الـ adapter. الـ adapter بيبدأ يستجيب لعنوان الـ client وبيستدعي الـ `slave_cb` لكل event.

| Parameter | الوصف |
|---|---|
| `client` | الـ client بالعنوان اللي هيتسمع عليه الـ adapter |
| `slave_cb` | الـ callback بـ signature: `int cb(struct i2c_client*, enum i2c_slave_event, u8*)` |

**Return:** `0` للنجاح، negative errno للفشل.

**Key details:** بتستدعي `adap->algo->reg_target()`. الـ client لازم يكون عنده `I2C_CLIENT_SLAVE` set.

---

#### `i2c_slave_unregister`

```c
int i2c_slave_unregister(struct i2c_client *client);
```

بتلغي تسجيل الـ slave. بتستدعي `adap->algo->unreg_target()`.

---

#### `i2c_slave_event`

```c
int i2c_slave_event(struct i2c_client *client,
                    enum i2c_slave_event event, u8 *val);
```

بتستدعيها الـ adapter driver من الـ interrupt handler أو الـ state machine لإخطار الـ slave driver بحدث معين.

| Event | الوصف |
|---|---|
| `I2C_SLAVE_READ_REQUESTED` | الـ master طلب قراءة — الـ slave لازم يحط الـ byte الأول في `*val` |
| `I2C_SLAVE_WRITE_REQUESTED` | الـ master هيكتب — prepare للاستقبال |
| `I2C_SLAVE_READ_PROCESSED` | الـ byte اللي حطيناه اتنقل — جيب الـ byte التالي |
| `I2C_SLAVE_WRITE_RECEIVED` | استلمنا byte في `*val` |
| `I2C_SLAVE_STOP` | الـ master بعت STOP condition |

---

#### `i2c_detect_slave_mode`

```c
bool i2c_detect_slave_mode(struct device *dev);
```

بتتحقق من الـ firmware (DT) لو الـ adapter المفروض يشتغل كـ slave. بتقرأ `linux,slave-24c02` أو ما شابه من الـ device tree.

---

### المجموعة الثامنة: Helper & Utility Functions

---

#### `i2c_get_clientdata` / `i2c_set_clientdata`

```c
static inline void *i2c_get_clientdata(const struct i2c_client *client);
static inline void i2c_set_clientdata(struct i2c_client *client, void *data);
```

الـ accessor/mutator للـ driver-private data المخزنة في `client->dev.driver_data`. الـ driver بيخزن الـ `struct foo_priv *` فيها في الـ `probe()` وبيسترجعها في باقي الـ callbacks.

```c
/* في probe */
priv = devm_kzalloc(&client->dev, sizeof(*priv), GFP_KERNEL);
i2c_set_clientdata(client, priv);

/* في باقي الـ functions */
priv = i2c_get_clientdata(client);
```

---

#### `i2c_get_adapdata` / `i2c_set_adapdata`

```c
static inline void *i2c_get_adapdata(const struct i2c_adapter *adap);
static inline void i2c_set_adapdata(struct i2c_adapter *adap, void *data);
```

نفس المفهوم لكن للـ adapter driver الـ private data. الـ adapter driver بيحتفظ بـ hardware-specific struct.

---

#### `i2c_check_functionality`

```c
static inline int i2c_check_functionality(struct i2c_adapter *adap, u32 func);
```

بتتحقق إن الـ adapter بيدعم الـ features المطلوبة. يُستدعى في `probe()` قبل استخدام أي SMBus functions.

```c
if (!i2c_check_functionality(client->adapter,
                              I2C_FUNC_SMBUS_BYTE_DATA))
    return -EOPNOTSUPP;
```

**Return:** `1` لو بيدعم كل الـ requested flags، `0` لو لأ.

---

#### `i2c_get_functionality`

```c
static inline u32 i2c_get_functionality(struct i2c_adapter *adap);
```

بتستدعي `adap->algo->functionality()` وبترجع الـ bitmask كامل. بتستخدمها لو عايز تعرف إيه اللي الـ adapter بيدعمه بالضبط.

---

#### `i2c_check_quirks`

```c
static inline bool i2c_check_quirks(struct i2c_adapter *adap, u64 quirks);
```

بتتحقق إن الـ adapter عنده الـ quirk flags المحددة. مثلاً لو الـ device عايز تعرف إن الـ adapter مش بيدعم repeated START:

```c
if (i2c_check_quirks(adap, I2C_AQ_NO_REP_START))
    return -EOPNOTSUPP;
```

---

#### `i2c_adapter_id`

```c
static inline int i2c_adapter_id(struct i2c_adapter *adap);
```

بترجع `adap->nr`. بتستخدمها للـ logging أو لتحديد الـ bus رقم كام.

---

#### `i2c_client_has_driver`

```c
static inline bool i2c_client_has_driver(struct i2c_client *client);
```

بتتحقق إن الـ client متعمله bind مع driver. بتستخدمها قبل الاستدعاء على الـ client لو مش متأكد إنه ready.

---

#### `i2c_8bit_addr_from_msg`

```c
static inline u8 i2c_8bit_addr_from_msg(const struct i2c_msg *msg);
```

بتبني الـ 8-bit address byte كما هيتبعت على الـ bus: `(addr << 1) | R/W_bit`. بتستخدمها في الـ adapter drivers أثناء تنفيذ الـ transfer.

---

#### `i2c_10bit_addr_hi_from_msg` / `i2c_10bit_addr_lo_from_msg`

```c
static inline u8 i2c_10bit_addr_hi_from_msg(const struct i2c_msg *msg);
static inline u8 i2c_10bit_addr_lo_from_msg(const struct i2c_msg *msg);
```

لبناء الـ 10-bit address header حسب I2C spec:
- `addr_hi`: `0xF0 | (addr[9:8] >> 7) | R/W`
- `addr_lo`: `addr[7:0]`

---

#### `i2c_get_dma_safe_msg_buf`

```c
u8 *i2c_get_dma_safe_msg_buf(struct i2c_msg *msg, unsigned int threshold);
```

لو `msg->buf` مش DMA-safe (مش `I2C_M_DMA_SAFE` set)، بتعمل `kmalloc` لـ bounce buffer وبترجعه. لو الـ buffer أصغر من الـ `threshold`، مش بتعمل allocation.

**Return:** الـ DMA-safe buffer (ممكن يكون نفس `msg->buf` لو كان safe)، أو `NULL` للفشل.

**Key details:** لازم تستدعي `i2c_put_dma_safe_msg_buf()` بعد الـ transfer.

---

#### `i2c_put_dma_safe_msg_buf`

```c
void i2c_put_dma_safe_msg_buf(u8 *buf, struct i2c_msg *msg, bool xferred);
```

بتحرر الـ bounce buffer. لو `xferred == true` وكان عملية read، بتعمل copy من الـ bounce buffer للـ `msg->buf` الأصلي.

---

#### `i2c_parse_fw_timings`

```c
void i2c_parse_fw_timings(struct device *dev, struct i2c_timings *t, bool use_defaults);
```

بتقرأ الـ I2C timing parameters من الـ firmware (DT properties زي `clock-frequency`, `i2c-scl-rising-time-ns`, إلخ) وبتملا `struct i2c_timings`. لو `use_defaults == true`، بتستخدم قيم standard لأي property مش موجودة.

**بتستخدمها في:** الـ adapter driver `probe()` قبل configure الـ hardware:

```c
i2c_parse_fw_timings(&pdev->dev, &priv->timings, true);
/* بعدين استخدم priv->timings.bus_freq_hz لضبط الـ hardware */
```

---

#### `i2c_freq_mode_string`

```c
const char *i2c_freq_mode_string(u32 bus_freq_hz);
```

بترجع string وصفية للـ frequency mode:
- 100kHz — `"Standard Mode"`
- 400kHz — `"Fast Mode"`
- 1MHz — `"Fast Mode Plus"`
- 3.4MHz — `"High Speed Mode"`
- 5MHz — `"Ultra Fast Mode"`

---

#### `i2c_verify_adapter` / `i2c_verify_client`

```c
struct i2c_adapter *i2c_verify_adapter(struct device *dev);
struct i2c_client *i2c_verify_client(struct device *dev);
```

بتتحققوا إن الـ `struct device` فعلاً adapter/client بالترتيب (عبر `dev->type`). بترجعوا `NULL` لو مش كده. بتستخدمهم في الـ generic device callbacks.

---

#### `i2c_match_id`

```c
const struct i2c_device_id *i2c_match_id(const struct i2c_device_id *id,
                                         const struct i2c_client *client);
```

بتدور في الـ `id_table` على entry تطابق `client->name`. بترجع الـ matching entry أو `NULL`. بتستخدمها في `probe()` للوصول للـ driver-data المرتبطة بـ ID معين.

---

#### `i2c_get_match_data`

```c
const void *i2c_get_match_data(const struct i2c_client *client);
```

بتجيب الـ `driver_data` من أول match في الـ `id_table` أو الـ OF/ACPI match table. اختصار شائع بدلاً من `i2c_match_id()` ثم `.driver_data`.

---

#### `i2c_get_device_id`

```c
int i2c_get_device_id(const struct i2c_client *client,
                      struct i2c_device_identity *id);
```

بتقرأ الـ **Device ID** من الـ I2C device عبر الـ standard I2C Device ID protocol (address `0x7C` reserved). بتملا `struct i2c_device_identity` بالـ manufacturer ID والـ part ID والـ die revision.

**Return:** `0` للنجاح، `-EOPNOTSUPP` لو الـ adapter مش بيدعم الـ protocol.

---

#### `i2c_client_get_device_id`

```c
const struct i2c_device_id *i2c_client_get_device_id(const struct i2c_client *client);
```

بترجع الـ `i2c_device_id` entry المرتبطة بالـ client من الـ driver's `id_table`. بتستخدمها لمعرفة إيه الـ chip variant اللي عملنا probe ليه.

---

#### `kobj_to_i2c_client`

```c
static inline struct i2c_client *kobj_to_i2c_client(struct kobject *kobj);
```

بتحول `kobject` لـ `i2c_client` pointer عبر `kobj -> device -> i2c_client` chain. بتستخدمها في الـ sysfs attribute handlers.

---

#### `i2c_parent_is_i2c_adapter`

```c
static inline struct i2c_adapter *
i2c_parent_is_i2c_adapter(const struct i2c_adapter *adapter);
```

بتتحقق إن الـ adapter هو في الحقيقة mux channel تحت adapter أب. لو أيوه، بترجع الـ parent adapter. لو لأ، بترجع `NULL`. بتستخدمها في الـ mux drivers.

---

### المجموعة التاسعة: Bus Recovery

لما الـ I2C bus يتعلق (stuck SDA أو SCL)، الـ kernel بيوفر آلية recovery.

---

#### `i2c_recover_bus`

```c
int i2c_recover_bus(struct i2c_adapter *adap);
```

بتحاول recovery للـ bus عبر `adap->bus_recovery_info->recover_bus()`. لو المعمول بيه هو `i2c_generic_scl_recovery`، بيعمل clock-toggling على الـ SCL line عشان يفك أي slave عالق في حالة data transmission.

**Return:** `0` للنجاح، negative errno للفشل.

**Caller context:** بتتستدعى من الـ adapter driver لو الـ `xfer` فشل بشكل غريب.

---

#### `i2c_generic_scl_recovery`

```c
int i2c_generic_scl_recovery(struct i2c_adapter *adap);
```

الـ generic implementation للـ bus recovery. بتعمل:
1. Switch SCL/SDA لـ GPIO mode عبر الـ pinctrl
2. Toggle SCL عدد من الـ cycles (عادة 9) مع مراقبة SDA
3. إرسال STOP condition
4. إرجاع SCL/SDA لـ I2C mode

بتتطلب `adap->bus_recovery_info` مملوءة بشكل صحيح مع الـ GPIO descriptors أو الـ get/set_scl/sda callbacks.

---

### المجموعة العاشرة: OF (Device Tree) Functions

---

#### `of_find_i2c_device_by_node`

```c
static inline struct i2c_client *of_find_i2c_device_by_node(struct device_node *node);
```

Wrapper حول `i2c_find_device_by_fwnode`. بتدور على الـ I2C client المرتبط بـ DT node محدد. **لازم** `put_device()` بعد الاستخدام.

---

#### `of_find_i2c_adapter_by_node` / `of_get_i2c_adapter_by_node`

```c
static inline struct i2c_adapter *of_find_i2c_adapter_by_node(struct device_node *node);
static inline struct i2c_adapter *of_get_i2c_adapter_by_node(struct device_node *node);
```

الأولى: `put_device()` بعد الاستخدام. الثانية: `i2c_put_adapter()` بعد الاستخدام. كلاهم بيدوروا على adapter مرتبط بـ DT node.

---

#### `of_i2c_get_board_info`

```c
int of_i2c_get_board_info(struct device *dev, struct device_node *node,
                          struct i2c_board_info *info);
```

بتقرأ الـ I2C device info من DT node وبتملا `struct i2c_board_info`. بتقرأ الـ `reg` property للعنوان والـ `interrupts` للـ IRQ والـ `compatible` للاسم. بتستخدمها الـ I2C core نفسها أثناء enumerate الـ devices من DT.

---

### المجموعة الحادية عشرة: ACPI Functions

---

#### `i2c_acpi_get_i2c_resource`

```c
bool i2c_acpi_get_i2c_resource(struct acpi_resource *ares,
                               struct acpi_resource_i2c_serialbus **i2c);
```

بتتحقق إن الـ ACPI resource هو I2C SerialBus resource وبترجع pointer ليه. بتستخدمها لاستخراج الـ I2C address والـ bus speed من ACPI tables.

---

#### `i2c_acpi_find_bus_speed`

```c
u32 i2c_acpi_find_bus_speed(struct device *dev);
```

بتقرأ الـ bus frequency من الـ ACPI _CRS (Current Resource Settings). بتستخدمها الـ adapter drivers لضبط السرعة الصحيحة.

---

#### `i2c_acpi_new_device_by_fwnode`

```c
struct i2c_client *i2c_acpi_new_device_by_fwnode(struct fwnode_handle *fwnode,
                                                 int index,
                                                 struct i2c_board_info *info);
```

بتنشئ I2C client من ACPI fwnode. الـ `index` بيحدد أنهي I2C resource في الـ _CRS لو الـ device عنده أكتر من resource.

---

#### `i2c_acpi_find_adapter_by_handle`

```c
struct i2c_adapter *i2c_acpi_find_adapter_by_handle(acpi_handle handle);
```

بتدور على الـ adapter المرتبط بـ ACPI handle. بتستخدمها لإنشاء devices جديدة على adapter معين من ACPI context.

---

#### `i2c_acpi_waive_d0_probe`

```c
bool i2c_acpi_waive_d0_probe(struct device *dev);
```

بتتحقق إن الـ driver عنده `I2C_DRV_ACPI_WAIVE_D0_PROBE` flag، ولو أيوه، بيخلي الـ core ما يحطش الـ device في D0 state قبل الـ probe. مفيدة للـ devices اللي بتحتاج initialization خاصة قبل الـ power state transition.

---

#### `i2c_acpi_new_device`

```c
static inline struct i2c_client *i2c_acpi_new_device(struct device *dev,
                                                     int index,
                                                     struct i2c_board_info *info);
```

Wrapper حول `i2c_acpi_new_device_by_fwnode` بيستخرج الـ fwnode من الـ `dev` تلقائياً.

---

#### `i2c_handle_smbus_host_notify`

```c
int i2c_handle_smbus_host_notify(struct i2c_adapter *adap, unsigned short addr);
```

بتستدعيها الـ adapter driver لما تستقبل **SMBus Host Notify** من slave. بتبعت IRQ للـ client المسجل على العنوان `addr` عبر الـ `adap->host_notify_domain`. بتستخدمها مع devices زي بعض الـ touchpads اللي بتبعت host notify.

---

### ملخص الـ Macros المهمة

| Macro | الوصف |
|---|---|
| `to_i2c_client(d)` | من `struct device*` لـ `struct i2c_client*` |
| `to_i2c_adapter(d)` | من `struct device*` لـ `struct i2c_adapter*` |
| `to_i2c_driver(d)` | من `struct device_driver*` لـ `struct i2c_driver*` |
| `I2C_BOARD_INFO(type, addr)` | initializer مختصر لـ `struct i2c_board_info` |
| `I2C_ADDRS(addr, addrs...)` | بيبني array منتهي بـ `I2C_CLIENT_END` |
| `i2c_add_driver(drv)` | = `i2c_register_driver(THIS_MODULE, drv)` |
| `module_i2c_driver(drv)` | بيولّد `module_init/exit` كاملين |
| `builtin_i2c_driver(drv)` | بيولّد `device_initcall` للـ built-in drivers |

---

### نموذج الـ Locking في I2C

```
Thread A (Transfer)            Thread B (PM Suspend)
──────────────────────         ─────────────────────
i2c_transfer()                 i2c_mark_adapter_suspended()
  i2c_lock_bus(SEGMENT)          i2c_lock_bus(ROOT)  <- ينتظر A
    rt_mutex_lock(bus_lock)         set_bit(SUSPENDED)
    __i2c_transfer()                i2c_unlock_bus(ROOT)
    algo->xfer()
  i2c_unlock_bus(SEGMENT)      Thread C (new transfer attempt)
                               i2c_transfer()
                                 checks I2C_ALF_IS_SUSPENDED
                                 returns -EACCES
```

الـ `adap->bus_lock` هو `rt_mutex` (Real-Time mutex) وليس mutex عادي، لأن الـ I2C transfers ممكن تحدث من كود بيشغّل في context بيتأثر بـ priority inversion.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. Debugfs Entries

**الـ i2c core** بيعمل directory في debugfs لكل `i2c_adapter` ولكل `i2c_client` — الـ struct نفسه فيه `struct dentry *debugfs` في كل من `i2c_adapter` و`i2c_client`.

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/i2c-N/` | directory الـ adapter رقم N |
| `/sys/kernel/debug/i2c-N/i2c-dev` | معلومات الـ adapter |
| `/sys/kernel/debug/i2c-0/NAME-ADDR/` | directory الـ client (مثلاً `eeprom-0050`) |

```bash
# اعرض كل الـ debugfs entries الخاصة بـ i2c
ls /sys/kernel/debug/i2c-*

# اقرأ معلومات adapter محدد
cat /sys/kernel/debug/i2c-0/i2c-dev

# شوف كل الـ clients المسجلين على bus 0
ls /sys/kernel/debug/i2c-0/
```

---

#### 2. Sysfs Entries

**الـ sysfs** بيكشف كل adapter وكل client كـ device nodes.

| المسار | المحتوى |
|--------|---------|
| `/sys/bus/i2c/devices/` | كل devices الـ i2c (adapters + clients) |
| `/sys/bus/i2c/devices/i2c-N/` | الـ adapter رقم N |
| `/sys/bus/i2c/devices/N-00AA/` | الـ client على bus N بعنوان 0xAA |
| `/sys/bus/i2c/devices/N-00AA/name` | اسم الـ client |
| `/sys/bus/i2c/devices/N-00AA/driver/` | symlink للـ driver |
| `/sys/bus/i2c/devices/N-00AA/new_device` | إنشاء client يدوياً |
| `/sys/bus/i2c/devices/N-00AA/delete_device` | حذف client يدوياً |
| `/sys/bus/i2c/drivers/` | كل الـ i2c drivers المسجلين |

```bash
# اعرض كل الـ i2c devices
ls /sys/bus/i2c/devices/

# اعرف اسم الـ chip على bus 0 address 0x50
cat /sys/bus/i2c/devices/0-0050/name

# اعرف الـ driver المربوط
ls -la /sys/bus/i2c/devices/0-0050/driver

# scan يدوي: أضف device بدون DT
echo "eeprom 0x50" > /sys/bus/i2c/devices/i2c-0/new_device

# احذف device
echo "0x50" > /sys/bus/i2c/devices/i2c-0/delete_device
```

---

#### 3. Ftrace — Tracepoints والـ Events

**الـ i2c subsystem** عنده tracepoints جاهزة في `drivers/i2c/i2c-core-base.c`.

| Event | ما يرصده |
|-------|---------|
| `i2c:i2c_write` | كل write message |
| `i2c:i2c_read` | كل read message |
| `i2c:i2c_reply` | الـ reply من الـ slave |
| `i2c:i2c_result` | نتيجة الـ transfer (نجاح/فشل) |
| `i2c:smbus_write` | SMBus write transaction |
| `i2c:smbus_read` | SMBus read transaction |
| `i2c:smbus_reply` | SMBus reply |
| `i2c:smbus_result` | نتيجة الـ SMBus transfer |

```bash
# فعّل tracing لكل الـ i2c events
echo 1 > /sys/kernel/debug/tracing/events/i2c/enable

# أو event محدد فقط
echo 1 > /sys/kernel/debug/tracing/events/i2c/i2c_write/enable
echo 1 > /sys/kernel/debug/tracing/events/i2c/i2c_read/enable
echo 1 > /sys/kernel/debug/tracing/events/i2c/i2c_result/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل العملية اللي عايز تتبعها، بعدين اقرأ النتائج
cat /sys/kernel/debug/tracing/trace

# filter على adapter رقم 0 بس
echo 'adapter_nr==0' > /sys/kernel/debug/tracing/events/i2c/i2c_write/filter

# وقف الـ trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo "" > /sys/kernel/debug/tracing/trace  # clear
```

**مثال على output:**

```
i2c_write: i2c-0 #0 a=050 f=0000 l=1 [0e]
i2c_read:  i2c-0 #1 a=050 f=0001 l=4
i2c_reply: i2c-0 #1 a=050 f=0001 l=4 [de ad be ef]
i2c_result: i2c-0 n=2 ret=2
```

التفسير: `a=050` = address 0x50، `f=0000` = write، `f=0001` = read، `l=` = طول البيانات، `ret=2` = اتنين messages اتنفذوا بنجاح.

---

#### 4. Printk والـ Dynamic Debug

**تفعيل dynamic debug** للـ i2c core وللـ drivers:

```bash
# فعّل كل debug messages في i2c-core
echo 'file drivers/i2c/i2c-core-base.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/i2c/i2c-core-smbus.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لـ driver محدد (مثلاً at24 EEPROM driver)
echo 'module at24 +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لكل الـ i2c drivers
echo 'module i2c_* +p' > /sys/kernel/debug/dynamic_debug/control

# شوف اللي فعّال دلوقتي
cat /sys/kernel/debug/dynamic_debug/control | grep i2c

# في kernel cmdline لو عايز permanent
dyndbg="file drivers/i2c/* +p"
```

**في الكود** — لو بتكتب driver:

```c
/* استخدم dev_dbg مش pr_debug عشان تعرف الـ device */
dev_dbg(&client->dev, "reading reg 0x%02x\n", reg);

/* لو عايز تشوف الـ transfer */
dev_dbg(&client->adapter->dev, "xfer: addr=0x%02x len=%d\n",
        client->addr, len);
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_I2C_DEBUG_CORE` | debug messages في i2c-core |
| `CONFIG_I2C_DEBUG_ALGO` | debug في الـ algorithms |
| `CONFIG_I2C_DEBUG_BUS` | debug في bus drivers |
| `CONFIG_I2C_SLAVE` | دعم slave mode (للاختبار) |
| `CONFIG_I2C_CHARDEV` | `/dev/i2c-N` لـ userspace access |
| `CONFIG_I2C_MUX` | دعم الـ mux (يفيد في debug الـ topology) |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في `bus_lock` / `mux_lock` |
| `CONFIG_DEBUG_RT_MUTEXES` | debug الـ rt_mutex المستخدمة في `i2c_adapter` |
| `CONFIG_PROVE_LOCKING` | يثبت صحة الـ locking |
| `CONFIG_FAULT_INJECTION` | حقن أخطاء في الـ transfers |
| `CONFIG_I2C_SMBUS` | SMBus emulation layer |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'I2C_DEBUG|I2C_SLAVE|I2C_CHARDEV'
```

---

#### 6. Tools خاصة بالـ I2C Subsystem

**الـ i2c-tools** هي الأداة الأساسية للـ userspace debugging:

```bash
# اعرض كل الـ buses
i2cdetect -l

# scan bus 0 عشان تلاقي الـ devices (7-bit addresses)
i2cdetect -y 0

# اقرأ byte من register 0x0E على device address 0x50، bus 0
i2cget -y 0 0x50 0x0e b

# اكتب قيمة على register
i2cset -y 0 0x50 0x0e 0xFF b

# dump كل registers (0x00 - 0xFF)
i2cdump -y 0 0x50

# تحقق من الـ functionality flags للـ adapter
i2cdetect -F 0
```

**مثال output من `i2cdetect -y 0`:**

```
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
```

الـ `50` ده يعني في device على address 0x50.

**لو الـ device مش ظاهر** وانت عارف إنه موجود:

```bash
# جرب quick mode (أسرع بس ممكن يعمل damage لبعض الـ devices)
i2cdetect -q -y 0

# أو read mode
i2cdetect -r -y 0
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|----------------------|--------|------|
| `i2c i2c-0: adapter timeout` | الـ slave ما ردش في الوقت (timeout في `i2c_adapter.timeout`) | تحقق من الـ wiring، زود الـ timeout، تحقق من clock stretching |
| `i2c i2c-0: i2c_smbus_xfer_emulated: SMBus emulation unsupported` | الـ adapter مش بيدعم الـ functionality المطلوبة | تحقق من `i2c_check_functionality()` قبل الاستخدام |
| `i2c i2c-0: NAK from device addr 0x50` | الـ slave رفض الـ address أو البيانات | تحقق من الـ address، pull-up resistors، voltage levels |
| `probe of N-00AA failed with error -ENODEV` | الـ driver's `probe()` رجع `-ENODEV` | الـ device مش موجود أو الـ ID ما اتطابقش |
| `i2c i2c-0: bus is busy` | الـ SDA أو SCL محتجزة low | bus recovery مطلوب، `i2c_recover_bus()` |
| `i2c i2c-0: Transfer while suspended` | محاولة transfer والـ adapter suspended (`I2C_ALF_IS_SUSPENDED` set) | تحقق من الـ suspend/resume sequence في الـ driver |
| `i2c i2c-0: Arbitration lost` | multi-master conflict | تحقق من الـ hardware topology، لازم يبقى في master واحد |
| `-ENXIO` من `i2c_transfer()` | NAK على الـ address | الـ device مش موجود على الـ bus أو wrong address |
| `-ETIMEDOUT` | timeout انتهى | تحقق من الـ clock، الـ bus recovery، زود timeout |
| `-EREMOTEIO` | error في الـ transfer level | hardware issue، تحقق بـ logic analyzer |
| `i2c i2c-0: controller timed out` | الـ controller نفسه stuck | reset الـ controller أو reboot |
| `can't use slave address` | العنوان reserved أو conflicting | تحقق من الـ address list في DT أو ACPI |

---

#### 8. Strategic Points لـ dump_stack() / WARN_ON()

```c
/* في xfer function — تحقق إن الـ adapter مش NULL */
int my_driver_read(struct i2c_client *client, u8 reg, u8 *val)
{
    /* WARN لو الـ client مش صح */
    if (WARN_ON(!client || !client->adapter))
        return -EINVAL;

    /* WARN لو الـ adapter suspended بدون إشعار */
    if (WARN_ON(test_bit(I2C_ALF_IS_SUSPENDED,
                         &client->adapter->locked_flags)))
        return -EBUSY;

    /* dump_stack في case الـ unexpected NAK */
    ret = i2c_smbus_read_byte_data(client, reg);
    if (ret == -ENXIO) {
        dev_err(&client->dev, "NAK on reg 0x%02x\n", reg);
        dump_stack();  /* شوف مين طلب القراءة */
        return ret;
    }
    return ret;
}

/* في probe — تحقق من الـ functionality */
static int my_probe(struct i2c_client *client)
{
    if (WARN_ON(!i2c_check_functionality(client->adapter,
                                         I2C_FUNC_SMBUS_BYTE_DATA))) {
        dev_err(&client->dev, "adapter lacks SMBUS_BYTE_DATA\n");
        return -ENODEV;
    }
    /* ... */
}
```

---

### Hardware Level

---

#### 1. التحقق من حالة الـ Hardware مقابل الـ Kernel State

**الـ kernel** بيشوف الـ bus من خلال الـ `i2c_adapter` وبيخزن state في:
- `i2c_adapter.locked_flags` — `I2C_ALF_IS_SUSPENDED`
- `i2c_adapter.nr` — رقم الـ bus
- `i2c_adapter.timeout` — بالـ jiffies
- `i2c_adapter.retries` — عدد المحاولات

```bash
# تحقق من الـ adapter state
cat /sys/bus/i2c/devices/i2c-0/name

# شوف الـ bus frequency من الـ DT أو ACPI
cat /sys/bus/i2c/devices/i2c-0/of_node/clock-frequency 2>/dev/null
# أو
cat /proc/device-tree/i2c@*/clock-frequency | od -A x -t x1z

# تحقق من الـ clients المسجلين على bus
ls /sys/bus/i2c/devices/ | grep '^0-'
```

**مقارنة hardware vs kernel:**

| Hardware State | Kernel State | طريقة التحقق |
|---------------|-------------|--------------|
| Device على address 0x50 | `i2c_client` مسجل على 0-0050 | `ls /sys/bus/i2c/devices/` |
| SCL/SDA high (bus free) | transfers تشتغل | `i2cdetect` ناجح |
| Pull-up resistors صح | لا timeout أو NAK | transfer time معقول |
| Voltage 3.3V/1.8V | driver probe نجح | `dmesg | grep i2c` |

---

#### 2. Register Dump Techniques

**لو الـ controller نفسه فيه memory-mapped registers** (زي i2c controller في SoC):

```bash
# devmem2 لقراءة register معين (المثال لـ i2c controller على عنوان 0xFE804000)
# اقرأ status register
devmem2 0xFE804000 w

# اقرأ بيانات 4 bytes
devmem2 0xFE804004 w

# لو devmem2 مش موجود، استخدم /dev/mem
dd if=/dev/mem bs=4 count=1 skip=$((0xFE804000/4)) 2>/dev/null | od -A x -t x4z

# io utility (من package iotools)
io -4 -r 0xFE804000
```

**لو الـ SoC docs متاحة**، شوف registers زي:
- `I2C_SR` (Status Register) — بيبين busy/NAK/arbitration
- `I2C_CR` (Control Register) — الـ speed والـ enable
- `I2C_DR` (Data Register) — الـ data buffer

```bash
# dump كتلة من registers (16 registers من 0xFE804000)
for i in $(seq 0 15); do
    addr=$(printf "0x%08X" $((0xFE804000 + i*4)))
    val=$(devmem2 $addr w 2>/dev/null | tail -1 | awk '{print $NF}')
    printf "0x%08X: %s\n" $((0xFE804000 + i*4)) "$val"
done
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس الأساسية:**

```
SDA ─────┐     ┌───┐   ┌─┐ ┌─┐       ┌───────
         └─────┘   └───┘ └─┘ └───────┘
SCL ─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─
     └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─
      S  A6 A5 A4 A3 A2 A1 A0 R/W  ACK  ...
      ^                              ^
      START                         ACK من الـ slave
```

**إعداد Logic Analyzer:**

| الإعداد | القيمة |
|---------|--------|
| Sample Rate | >= 4x الـ bus frequency (مثلاً 1.6 MHz لـ 400kHz) |
| Threshold | 1.65V لـ 3.3V logic، 0.9V لـ 1.8V logic |
| Protocol | I2C |
| Trigger | SDA falling edge while SCL high (START condition) |

**ما تدور عليه:**

- **START condition**: SDA ينزل وهو SCL high
- **STOP condition**: SDA يطلع وهو SCL high
- **NAK**: الـ bit 9 high (الـ slave مرفعش SDA low)
- **Clock stretching**: SCL بيفضل low أطول من المتوقع
- **Glitches**: spikes على الـ lines (تحتاج `digital_filter_width_ns` في `i2c_timings`)

**قياسات مهمة من `struct i2c_timings`:**

```
tRISE (scl_rise_ns):  وقت صعود SCL — لازم < 300ns في fast mode
tFALL (scl_fall_ns):  وقت هبوط SCL — لازم < 300ns في fast mode
tHD;DAT (sda_hold_ns): SDA hold time بعد SCL ينزل
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة Hardware | ما تشوفه في الـ Log | الحل |
|-----------------|-------------------|------|
| Pull-up resistors مش صح (قيمة كبيرة أوي) | `timeout`, slow rise time | غير الـ resistors لقيم أصغر (1kΩ-4.7kΩ) |
| Pull-up resistors صغيرة أوي | **overcurrent، bus لا يصل HIGH** | زود القيمة |
| Voltage mismatch (3.3V adapter + 1.8V device) | NAK دايم، probe failed | حط level shifter |
| SDA/SCL مقلوبين في الـ wiring | `bus is busy` دايماً | عكّس الاتنين |
| Device address غلط في DT | `probe of N-XXXX failed with -ENODEV` | صحح `reg` في DT |
| Clock stretch غير مدعوم (`I2C_AQ_NO_CLK_STRETCH`) | `timeout`، `ETIMEDOUT` | شغّل bus recovery أو استخدم adapter تاني |
| Bus حابس (SDA stuck low) | `i2c i2c-0: bus is busy` | شغّل `i2c_generic_scl_recovery()` أو 9 clock pulses |
| Capacitance عالية على الـ line | signal distortion، intermittent | قصّر الـ traces أو استخدم active driver |

---

#### 5. Device Tree Debugging

**التحقق من الـ DT مقابل الـ Hardware:**

```bash
# اعرض الـ DT للـ i2c nodes
find /proc/device-tree -name "i2c*" -type d
ls /proc/device-tree/soc/i2c@fe804000/

# اقرأ الـ clock-frequency
cat /proc/device-tree/soc/i2c@fe804000/clock-frequency | od -A n -t u4

# اقرأ الـ status
cat /proc/device-tree/soc/i2c@fe804000/status

# اقرأ عناوين الـ devices
for node in /proc/device-tree/soc/i2c@*/*/reg; do
    echo -n "$node: "
    od -A n -t x1 < "$node" | tr -d ' \n'
    echo
done

# تحقق من dtc compiled DT
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "i2c@"
```

**مثال DT صحيح لـ EEPROM على 0x50:**

```dts
&i2c0 {
    clock-frequency = <400000>;  /* Fast mode */
    status = "okay";

    eeprom@50 {
        compatible = "atmel,24c32";
        reg = <0x50>;           /* 7-bit address */
        pagesize = <32>;
    };
};
```

**مشاكل DT شائعة:**

| الخطأ في DT | أثره في الـ Kernel |
|------------|------------------|
| `reg = <0x50>` بدل `<0x50>` (أو value غلط) | device مش بيظهر أو wrong address |
| `status = "disabled"` | الـ adapter مش بيتسجل خالص |
| `clock-frequency` غلط | `i2c_timings` بتتحسب غلط |
| compatible string غلط | driver مش بيـ probe |
| Pull-up resistors مش في DT (لـ GPIO bit-bang) | bus مش شغال |

```bash
# لو عندك .dts source، تحقق من overlay
dtoverlay -l  # على Raspberry Pi
fdtdump /boot/dtbs/$(uname -r)/platform.dtb | grep -A20 "i2c"
```

---

### Practical Commands

---

#### أوامر جاهزة للاستخدام الفوري

**1. تشخيص سريع — أول خطوة:**

```bash
# شوف الـ i2c buses المتاحة
i2cdetect -l

# scan bus 0
i2cdetect -y 0

# شوف الـ kernel log المتعلق بـ i2c
dmesg | grep -i i2c | tail -50

# شوف الـ drivers المحملين
lsmod | grep i2c
```

**2. تفعيل full tracing:**

```bash
#!/bin/bash
# i2c_trace.sh — يسجل كل i2c transactions

# وقف أي trace قديم
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo "" > /sys/kernel/debug/tracing/trace

# فعّل كل i2c events
echo 1 > /sys/kernel/debug/tracing/events/i2c/enable

# ابدأ
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "Tracing started. Press Enter to stop..."
read

# وقف واعرض
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep -v "^#" | head -200
```

**مثال output وتفسيره:**

```
# TASK-PID  CPU# |||||  TIMESTAMP  FUNCTION
#              | | ||||     |         |
  kworker/0:1-42    [000] ....  1234.567890: i2c_write: i2c-0 #0 a=050 f=0000 l=1 [0f]
  kworker/0:1-42    [000] ....  1234.567910: i2c_read:  i2c-0 #1 a=050 f=0001 l=2
  kworker/0:1-42    [000] ....  1234.567950: i2c_reply: i2c-0 #1 a=050 f=0001 l=2 [ab cd]
  kworker/0:1-42    [000] ....  1234.567960: i2c_result: i2c-0 n=2 ret=2
```

- `a=050` → address 0x50
- `f=0000` → write (flags=0)
- `f=0001` → read (I2C_M_RD=1)
- `l=1` → 1 byte
- `[0f]` → البيانات المرسلة: register 0x0F
- `[ab cd]` → البيانات المستلمة
- `n=2 ret=2` → 2 messages طُلبوا، 2 اتنجحوا

**3. تشخيص bus hung:**

```bash
# تحقق من البصمة
cat /sys/bus/i2c/devices/i2c-0/name

# جرب recovery يدوي (لو الـ driver بيدعمه)
# عادةً بيتعمل أوتوماتيك من i2c_transfer() لو فيه bus_recovery_info

# شوف state الـ bus
i2cdetect -y 0  # لو فشل، الـ bus محتمل hung

# محاولة 9 clock pulses يدوياً (emergency - hardware)
# الحل البرمجي: unload وreload الـ adapter driver
modprobe -r i2c_designware_platform
modprobe i2c_designware_platform
```

**4. قراءة/كتابة registers مباشرة:**

```bash
# اقرأ register 0x00 من device 0x48 على bus 1
i2cget -y 1 0x48 0x00 b

# اقرأ word (16-bit) من register 0x00
i2cget -y 1 0x48 0x00 w

# اكتب byte
i2cset -y 1 0x48 0x01 0xFF b

# dump كامل (256 bytes)
i2cdump -y 1 0x48 b

# i2ctransfer — أقوى أداة، بتعمل custom transfers
# مثال: اكتب register 0x0E ثم اقرأ 4 bytes (repeated start)
i2ctransfer -y 0 w1@0x50 0x0e r4@0x50
```

**مثال output من `i2cdump -y 0 0x50`:**

```
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f    0123456789abcdef
00: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff    ................
10: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff    ................
20: 41 42 43 44 45 46 47 48 ff ff ff ff ff ff ff ff    ABCDEFGH........
```

التفسير: من offset 0x20 فيه بيانات `ABCDEFGH` متخزنة في الـ EEPROM.

**5. فحص device identity (لو الـ device بيدعم Device ID protocol):**

```bash
# Device ID بيتقرأ من address 0x7C (reserved)
i2ctransfer -y 0 w1@0x7C 0xa1 r3@0x7C
# البايتات التلاتة بتحتوي manufacturer_id, part_id, die_revision
# (مطابق struct i2c_device_identity في i2c.h)
```

**6. تحقق من الـ functionality flags:**

```bash
# اعرض كل الـ capabilities للـ adapter
i2cdetect -F 0
```

**مثال output:**

```
Functionalities implemented by /dev/i2c-0:
I2C                              yes
SMBus Quick Command              yes
SMBus Send Byte                  yes
SMBus Receive Byte               yes
SMBus Write Byte                 yes
SMBus Read Byte                  yes
SMBus Write Word                 yes
SMBus Read Word                  yes
SMBus Process Call               yes
SMBus Block Write                yes
SMBus Block Read                 no   <-- مشكلة لو الـ driver محتاجها
SMBus Block Process Call         no
SMBus PEC                        yes
I2C Block Write                  yes
I2C Block Read                   yes
```

**7. Stress testing:**

```bash
# شغّل 1000 قراءة متتالية وشوف لو في errors
for i in $(seq 1 1000); do
    result=$(i2cget -y 0 0x50 0x00 b 2>&1)
    if echo "$result" | grep -qi "error\|nack\|timeout"; then
        echo "Error at iteration $i: $result"
    fi
done
echo "Done"
```

**8. مراقبة الأخطاء في real-time:**

```bash
# راقب الـ kernel log للـ i2c errors بشكل مباشر
dmesg -w | grep -i "i2c\|nack\|timeout\|error" &

# أو استخدم journalctl
journalctl -k -f | grep -i i2c
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Bus Hang على AM62x — Industrial Gateway

#### العنوان
**الـ I2C bus بيتعلق عشان الـ sensor بيعمل clock stretching والـ controller مش بيدعمه**

#### السياق
Industrial gateway مبني على **Texas Instruments AM62x** بيشتغل كـ Modbus-to-cloud bridge. الـ board فيها temperature/humidity sensor من نوع **SHT31** على `i2c2` بـ `400 kHz`. المنتج وصل للـ production وبعد تقريباً 3 أيام من التشغيل المتواصل الـ I2C bus بيتجمد خالص، ومش بترجع إلا لما تعمل reboot.

#### المشكلة
الـ SHT31 بيعمل **clock stretching** أثناء عملية القياس الداخلية. الـ AM62x I2C controller في بعض الـ silicon revisions عنده quirk: `I2C_AQ_NO_CLK_STRETCH`. لما الـ slave بيشد الـ SCL للأسفل والـ master مش فاهم، الـ controller بيدخل في state غلط ومفيش recover تلقائي.

#### التحليل

**الـ `struct i2c_adapter_quirks`** في `i2c.h`:

```c
struct i2c_adapter_quirks {
    u64 flags;
    int max_num_msgs;
    u16 max_write_len;
    u16 max_read_len;
    u16 max_comb_1st_msg_len;
    u16 max_comb_2nd_msg_len;
};

/* clock stretching is not supported */
#define I2C_AQ_NO_CLK_STRETCH  BIT(4)
```

الـ driver بتاع الـ AM62x i2c بيضع هذا الـ flag في الـ quirks. لما الـ `i2c_transfer()` بتتنفذ، الـ core بيشيك على الـ quirks قبل الـ transfer. لو الـ controller سجّل `I2C_AQ_NO_CLK_STRETCH`، المفروض يتعامل معاه — لكن الـ SHT31 driver مش بيتحقق من ده.

```c
/* التحقق من الـ quirks قبل الإرسال */
static inline bool i2c_check_quirks(struct i2c_adapter *adap, u64 quirks)
{
    if (!adap->quirks)
        return false;
    return (adap->quirks->flags & quirks) == quirks;
}
```

المشكلة إن الـ sensor driver بيستخدم `i2c_transfer()` مباشرة من غير ما يشيك على الـ quirk ده:

```c
/* كود المشكلة في driver الـ SHT31 */
ret = i2c_transfer(client->adapter, msgs, 2);
/* لو الـ adapter عنده I2C_AQ_NO_CLK_STRETCH
   والـ SHT31 بيعمل stretching → HANG */
```

#### الحل

**أولاً:** تفعيل `bus_recovery_info` في الـ AM62x I2C driver:

```c
/* في am62x-i2c.c */
static struct i2c_bus_recovery_info am62x_i2c_recovery = {
    .recover_bus     = i2c_generic_scl_recovery,
    .get_scl         = am62x_i2c_get_scl,
    .set_scl         = am62x_i2c_set_scl,
    .get_sda         = am62x_i2c_get_sda,
};

adap->bus_recovery_info = &am62x_i2c_recovery;
```

**ثانياً:** تخفيض الـ frequency في Device Tree:

```dts
/* arch/arm64/boot/dts/ti/k3-am625-industrial-gw.dts */
&i2c2 {
    status = "okay";
    clock-frequency = <100000>;  /* نزّلنا من 400k لـ 100k */

    sht31: humidity@44 {
        compatible = "sensirion,sht3x";
        reg = <0x44>;
    };
};
```

**ثالثاً:** مراقبة الـ recovery من userspace:

```bash
# شوف لو في recovery بيحصل
cat /sys/kernel/debug/i2c-2/i2c-2
dmesg | grep -i "i2c.*recover"
```

#### الدرس المستفاد
`struct i2c_adapter_quirks` مش مجرد documentation — الـ core بيستخدمها فعلاً. لازم تفهم quirks الـ controller بتاعك وتجهّز `bus_recovery_info` دايماً في أي production design.

---

### السيناريو الثاني: PMIC لمش بيشتغل أثناء Shutdown على STM32MP1

#### العنوان
**الـ PMIC على I2C مش بيستجابش أثناء poweroff بسبب غياب الـ `xfer_atomic`**

#### السياق
Embedded Linux board على **STM32MP157** بتشغّل smart meter. الـ board فيها **STPMIC1** (PMIC من STMicroelectronics) متوصّل على `i2c1`. عند عمل `poweroff`، الكود المفروض يقول للـ PMIC يقطع الكهربا — لكن الـ board مش بتتقفل خالص وبتفضل بتشتغل.

#### المشكلة
أثناء الـ `shutdown` path، الـ kernel بيكون في **atomic context** — interrupts معطّلة أو limited. الـ `i2c_transfer()` العادية بتحتاج sleeping locks (`rt_mutex`) وممكن تتعطّل. المطلوب هو `xfer_atomic` في `struct i2c_algorithm`.

#### التحليل

**الـ `struct i2c_algorithm`** في `i2c.h`:

```c
struct i2c_algorithm {
    union {
        int (*xfer)(struct i2c_adapter *adap,
                    struct i2c_msg *msgs, int num);
        int (*master_xfer)(struct i2c_adapter *adap,
                           struct i2c_msg *msgs, int num);
    };
    union {
        int (*xfer_atomic)(struct i2c_adapter *adap,
                           struct i2c_msg *msgs, int num);
        int (*master_xfer_atomic)(struct i2c_adapter *adap,
                                   struct i2c_msg *msgs, int num);
    };
    /* ... */
};
```

لو `xfer_atomic` بـ `NULL`، الـ I2C core في الـ shutdown path بيفشل أو بيـskip الـ transfer:

```c
/* في i2c-core-base.c — الـ core بيشيك */
if (in_atomic() || irqs_disabled()) {
    if (adap->algo->xfer_atomic)
        ret = adap->algo->xfer_atomic(adap, msgs, num);
    else
        ret = -EAGAIN;  /* الـ transfer اتـskip */
}
```

الـ `i2c_mark_adapter_suspended()`:

```c
static inline void i2c_mark_adapter_suspended(struct i2c_adapter *adap)
{
    i2c_lock_bus(adap, I2C_LOCK_ROOT_ADAPTER);
    set_bit(I2C_ALF_IS_SUSPENDED, &adap->locked_flags);
    i2c_unlock_bus(adap, I2C_LOCK_ROOT_ADAPTER);
}
```

لما الـ adapter اتعلّم إنه suspended قبل الـ PMIC shutdown call، كل الـ transfers اترفضت.

#### الحل

**أولاً:** التأكد إن الـ STM32 I2C driver عنده `xfer_atomic` implemented:

```c
/* في stm32f7_i2c_algo */
static const struct i2c_algorithm stm32f7_i2c_algo = {
    .xfer        = stm32f7_i2c_xfer,
    .xfer_atomic = stm32f7_i2c_xfer_atomic, /* لازم يكون موجود */
    .functionality = stm32f7_i2c_func,
};
```

**ثانياً:** في الـ PMIC driver، استخدم `i2c_transfer()` مع التحقق من timing:

```c
/* في stpmic1 shutdown handler */
static void stpmic1_power_off(void)
{
    struct i2c_client *client = pmic_i2c_client;
    u8 reg = STPMIC1_SWOFF_STDBY_REG;
    u8 val = STPMIC1_RREQ_EN_MASK;

    /*
     * نستخدم i2c_smbus_write_byte_data — الـ core هيختار
     * xfer_atomic تلقائياً لو إحنا في atomic context
     */
    i2c_smbus_write_byte_data(client, reg, val);
}
```

**ثالثاً:** تأخير الـ `i2c_mark_adapter_suspended()` لحد ما الـ PMIC يتعامل معاه:

```c
/* في PM ops بتاع الـ I2C controller */
static int stm32_i2c_suspend(struct device *dev)
{
    /* أول شيل الـ PMIC خلّص شغله */
    /* بعدين علّم الـ adapter suspended */
    i2c_mark_adapter_suspended(i2c_get_adapter(0));
    return 0;
}
```

#### الدرس المستفاد
الـ `xfer_atomic` في `struct i2c_algorithm` مش optional لأي controller بيتحكم في PMIC أو أي device محتاج في آخر لحظات الـ shutdown. غيابه بيخلي الـ board مش بتتقفل صح.

---

### السيناريو الثالث: Address Conflict على i.MX8 — Android TV Box

#### العنوان
**جهازين مختلفين بنفس الـ I2C address على نفس البص في Android TV Box**

#### السياق
Android TV box على **i.MX8MM** بيحتوي على **HDMI transmitter (IT66121)** و **touchpad controller (GT911)**. كلاهما على `i2c1`. بعد تحديث في الـ DT لإضافة الـ touchpad، الـ HDMI بدأ يعمل probe errors والشاشة بتخرج blank.

#### المشكلة
الـ IT66121 HDMI transmitter عنده address `0x4c`. الـ GT911 touchpad بالـ default configuration بتاعته ممكن يكون `0x5d` أو `0x14`. لكن لما اتضافت الـ secondary address للـ GT911 كـ ancillary device باستخدام `i2c_new_ancillary_device()`، أو لما الـ EEPROM على `0x4c` اشتبك مع الـ HDMI، بدأت المشكلة.

#### التحليل

**الـ `struct i2c_client`** في `i2c.h`:

```c
struct i2c_client {
    unsigned short flags;
    unsigned short addr;  /* 7-bit address */
    char name[I2C_NAME_SIZE];
    struct i2c_adapter *adapter;
    struct device dev;
    /* ... */
};
```

الـ kernel بيحتفظ بـ bitmap للعناوين المستخدمة في `struct i2c_adapter`:

```c
struct i2c_adapter {
    /* ... */
    /* 7bit address space */
    DECLARE_BITMAP(addrs_in_instantiation, 1 << 7);
};
```

لما `i2c_new_client_device()` بتتنادى، الـ core بيشيك على الـ bitmap. لو العنوان موجود، بيـreject الـ device جديد. التشخيص:

```bash
# شوف كل الـ devices المتسجّلة على i2c-1
i2cdetect -y 1

# الـ output:
#      0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
# 40: -- -- -- -- -- -- -- -- -- -- -- -- 4c -- -- --
# 50: -- -- -- -- -- -- -- -- -- -- -- -- -- 5d -- --

# شوف في الـ dmesg
dmesg | grep "i2c.*0x4c"
```

#### الحل

**أولاً:** مراجعة الـ DT وتعديل الـ IT66121 address:

```dts
/* arch/arm64/boot/dts/freescale/imx8mm-tv-box.dts */
&i2c1 {
    clock-frequency = <400000>;
    status = "okay";

    /* HDMI transmitter — address 0x4c */
    it66121: hdmi-transmitter@4c {
        compatible = "ite,it66121";
        reg = <0x4c>;
        interrupt-parent = <&gpio3>;
        interrupts = <0 IRQ_TYPE_LEVEL_LOW>;
    };

    /* Touchpad — address 0x5d (ليس 0x4c) */
    gt911: touchpad@5d {
        compatible = "goodix,gt911";
        reg = <0x5d>;
        interrupt-parent = <&gpio1>;
        interrupts = <5 IRQ_TYPE_EDGE_FALLING>;
        reset-gpios = <&gpio1 6 GPIO_ACTIVE_LOW>;
        irq-gpios = <&gpio1 5 GPIO_ACTIVE_FALLING>;
    };
};
```

**ثانياً:** لو الـ GT911 محتاج ancillary address، نستخدم `i2c_new_ancillary_device()` صح:

```c
/* في gt911_probe() */
static int gt911_probe(struct i2c_client *client)
{
    /* الـ GT911 بيدعم عنوانين: 0x5d أو 0x14 */
    /* استخدم ancillary device للعنوان الثاني */
    struct i2c_client *alt_client;

    alt_client = i2c_new_ancillary_device(client,
                                          "alt_addr",
                                          0x14);  /* default_addr */
    if (IS_ERR(alt_client))
        dev_warn(&client->dev, "alt address unavailable\n");

    /* ... */
}
```

**ثالثاً:** تحقق من الـ `addrs_in_instantiation` bitmap:

```bash
# شوف debugfs للـ adapter
ls /sys/kernel/debug/i2c-1/
cat /sys/kernel/debug/i2c-1/i2c-1
```

#### الدرس المستفاد
الـ `DECLARE_BITMAP(addrs_in_instantiation, 1 << 7)` في `struct i2c_adapter` هو الـ guard الأول ضد address conflicts. لازم تعمل `i2cdetect` على كل bus قبل ما تضيف device جديد في الـ DT.

---

### السيناريو الرابع: DMA Corruption على RK3562 — IoT Sensor Hub

#### العنوان
**بيانات الـ sensor جايت corrupted بسبب استخدام stack buffer مع DMA-capable I2C controller**

#### السياق
IoT sensor hub على **Rockchip RK3562** بيجمع بيانات من 8 sensors عبر I2C MUX. أحد الـ sensors هو **BMP388** pressure sensor على `i2c3`. البيانات المقروءة من الـ sensor أحياناً بتكون مش صح — القراءات بتقفز بشكل عشوائي مع إن الـ hardware سليم.

#### المشكلة
الـ RK3562 I2C controller بيستخدم **DMA** للـ transfers. الـ driver بتاع الـ sensor بيحجز الـ buffer على الـ **stack** وبيمرّره لـ `i2c_master_recv()`. الـ DMA مش بيعرف يشتغل صح مع stack buffers لأنها ممكن مش تكون **DMA-safe** (not physically contiguous, cache alignment issues).

#### التحليل

**الـ `i2c_master_recv`** و `i2c_master_recv_dmasafe` في `i2c.h`:

```c
/* الـ function العادية — مش ضامنة DMA safety */
static inline int i2c_master_recv(const struct i2c_client *client,
                                  char *buf, int count)
{
    return i2c_transfer_buffer_flags(client, buf, count, I2C_M_RD);
}

/* الـ function الصح للـ DMA */
static inline int i2c_master_recv_dmasafe(const struct i2c_client *client,
                                          char *buf, int count)
{
    return i2c_transfer_buffer_flags(client, buf, count,
                                     I2C_M_RD | I2C_M_DMA_SAFE);
}
```

الـ `I2C_M_DMA_SAFE` flag بيقول للـ core إن الـ buffer ممكن يتستخدم مع DMA مباشرةً. لو مش موجود، الـ core ممكن يعمل copy لـ DMA-safe buffer — لكن بعض الـ drivers مش بتعمل ده وبتعمل DMA مباشرة للـ buffer الغلط.

كمان في `i2c.h`:

```c
u8 *i2c_get_dma_safe_msg_buf(struct i2c_msg *msg, unsigned int threshold);
void i2c_put_dma_safe_msg_buf(u8 *buf, struct i2c_msg *msg, bool xferred);
```

الـ `i2c_get_dma_safe_msg_buf()` بيعمل allocate لـ bounce buffer لو الـ buffer الأصلي مش DMA-safe.

#### الحل

**أولاً:** تغيير الكود في الـ BMP388 driver:

```c
/* الكود القديم — مشكلة */
static int bmp388_read_raw(struct i2c_client *client, u8 reg, u8 *val)
{
    u8 stack_buf[6];  /* stack buffer — خطر مع DMA */

    i2c_master_send(client, &reg, 1);
    i2c_master_recv(client, stack_buf, 6);  /* مش DMA-safe */
    memcpy(val, stack_buf, 6);
    return 0;
}

/* الكود الصح */
static int bmp388_read_raw(struct i2c_client *client, u8 reg, u8 *val)
{
    /* allocate buffer من الـ heap — DMA-friendly */
    u8 *dma_buf = kzalloc(6, GFP_KERNEL | GFP_DMA);
    int ret;

    if (!dma_buf)
        return -ENOMEM;

    ret = i2c_master_send(client, &reg, 1);
    if (ret < 0)
        goto out;

    /* استخدم الـ DMA-safe variant */
    ret = i2c_master_recv_dmasafe(client, dma_buf, 6);
    if (ret == 6)
        memcpy(val, dma_buf, 6);

out:
    kfree(dma_buf);
    return (ret == 6) ? 0 : -EIO;
}
```

**ثانياً:** أو استخدام `i2c_get_dma_safe_msg_buf()` في الـ controller driver مباشرة:

```c
/* في rk3x_i2c_xfer — الـ RK3562 controller driver */
static int rk3x_i2c_xfer(struct i2c_adapter *adap,
                          struct i2c_msg *msgs, int num)
{
    for (int i = 0; i < num; i++) {
        u8 *dma_buf = i2c_get_dma_safe_msg_buf(&msgs[i], 16);
        /* استخدم dma_buf مع الـ DMA engine */
        /* ... */
        i2c_put_dma_safe_msg_buf(dma_buf, &msgs[i], true);
    }
}
```

**ثالثاً:** debug:

```bash
# فعّل dynamic debug للـ I2C
echo 'module i2c_core +p' > /sys/kernel/debug/dynamic_debug/control

# شوف لو في DMA errors
dmesg | grep -i "dma\|i2c.*error"
```

#### الدرس المستفاد
الـ `I2C_M_DMA_SAFE` flag في `i2c_msg` والفرق بين `i2c_master_recv` و `i2c_master_recv_dmasafe` مش مجرد optimization — غيابهم مع DMA-capable controllers بيسبب silent data corruption أصعب بكتير من crash صريح.

---

### السيناريو الخامس: I2C Slave Mode على Allwinner H616 — Automotive ECU

#### العنوان
**تفعيل I2C slave mode على H616 لاستقبال commands من microcontroller في automotive ECU**

#### السياق
Automotive ECU بتشتغل على **Allwinner H616** كـ main application processor. فيه **STM32G0** microcontroller بيشتغل كـ safety co-processor. الاتصال بينهم عبر I2C — الـ H616 بيكون **slave** والـ STM32 بيكون **master**. الهدف إن الـ STM32 يقدر يرسل commands للـ H616 في أي وقت حتى لو الـ H616 مشغول.

#### المشكلة
الـ engineer حاول يفعّل الـ slave mode لكن الـ `i2c_slave_register()` بيرجع `-ENOTSUPP` — الـ H616 I2C driver مش بيعمل implement الـ `reg_target` callback في `struct i2c_algorithm`.

#### التحليل

**الـ `i2c_slave_event` enum** في `i2c.h`:

```c
enum i2c_slave_event {
    I2C_SLAVE_READ_REQUESTED,   /* master طلب قراءة */
    I2C_SLAVE_WRITE_REQUESTED,  /* master هيكتب */
    I2C_SLAVE_READ_PROCESSED,   /* master قرأ البايت اللي بعثناه */
    I2C_SLAVE_WRITE_RECEIVED,   /* استقبلنا بايت من الـ master */
    I2C_SLAVE_STOP,             /* master بعت STOP condition */
};
```

الـ `i2c_slave_cb_t` callback:

```c
typedef int (*i2c_slave_cb_t)(struct i2c_client *client,
                               enum i2c_slave_event event, u8 *val);
```

في `struct i2c_algorithm`:

```c
#if IS_ENABLED(CONFIG_I2C_SLAVE)
    union {
        int (*reg_target)(struct i2c_client *client);
        int (*reg_slave)(struct i2c_client *client);   /* deprecated */
    };
    union {
        int (*unreg_target)(struct i2c_client *client);
        int (*unreg_slave)(struct i2c_client *client); /* deprecated */
    };
#endif
```

وفي `struct i2c_client`:

```c
struct i2c_client {
    /* ... */
#if IS_ENABLED(CONFIG_I2C_SLAVE)
    i2c_slave_cb_t slave_cb;  /* الـ callback المخصص للـ slave mode */
#endif
    /* ... */
};
```

الـ H616 driver (sun6i-i2c) الـ original مش بيعمل implement `reg_target` — لازم يتضاف.

#### الحل

**أولاً:** إضافة `reg_target` في الـ H616 driver:

```c
/* في sun6i_i2c_algo */
static int sun6i_i2c_reg_target(struct i2c_client *client)
{
    struct sun6i_i2c *i2c = i2c_get_adapdata(client->adapter);

    /* تأكد مفيش slave تاني registered */
    if (i2c->slave)
        return -EBUSY;

    i2c->slave = client;

    /* فعّل الـ slave address في الـ hardware register */
    writel(client->addr << 1, i2c->base + SUN6I_I2C_SADDR_REG);

    /* فعّل الـ slave mode interrupt */
    sun6i_i2c_enable_slave_irq(i2c);

    return 0;
}

static int sun6i_i2c_unreg_target(struct i2c_client *client)
{
    struct sun6i_i2c *i2c = i2c_get_adapdata(client->adapter);

    sun6i_i2c_disable_slave_irq(i2c);
    i2c->slave = NULL;
    return 0;
}

static const struct i2c_algorithm sun6i_i2c_algo = {
    .xfer         = sun6i_i2c_xfer,
    .functionality = sun6i_i2c_func,
    .reg_target   = sun6i_i2c_reg_target,    /* الإضافة الجديدة */
    .unreg_target = sun6i_i2c_unreg_target,  /* الإضافة الجديدة */
};
```

**ثانياً:** كتابة الـ slave driver لاستقبال الـ commands:

```c
/* ecu-slave.c */
static int ecu_slave_cb(struct i2c_client *client,
                        enum i2c_slave_event event, u8 *val)
{
    struct ecu_slave_data *data = i2c_get_clientdata(client);

    switch (event) {
    case I2C_SLAVE_WRITE_REQUESTED:
        data->buf_idx = 0;  /* reset buffer عند بداية transaction */
        break;

    case I2C_SLAVE_WRITE_RECEIVED:
        /* استقبلنا بايت من الـ STM32 */
        if (data->buf_idx < ECU_CMD_MAX_LEN)
            data->cmd_buf[data->buf_idx++] = *val;
        break;

    case I2C_SLAVE_STOP:
        /* الـ STM32 خلّص الـ transaction — نفّذ الأمر */
        ecu_process_command(data->cmd_buf, data->buf_idx);
        break;

    case I2C_SLAVE_READ_REQUESTED:
        /* الـ STM32 بيطلب قراءة — ابعت أول بايت من الـ response */
        *val = data->response[0];
        data->resp_idx = 1;
        break;

    case I2C_SLAVE_READ_PROCESSED:
        /* الـ STM32 قرأ البايت السابق — ابعت الجاي */
        *val = data->response[data->resp_idx++];
        break;
    }
    return 0;
}

static int ecu_slave_probe(struct i2c_client *client)
{
    struct ecu_slave_data *data;

    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    i2c_set_clientdata(client, data);

    /* سجّل الـ slave callback */
    return i2c_slave_register(client, ecu_slave_cb);
}

static void ecu_slave_remove(struct i2c_client *client)
{
    i2c_slave_unregister(client);
}
```

**ثالثاً:** الـ DT:

```dts
/* sun50i-h616-ecu.dts */
&i2c3 {
    status = "okay";
    /* H616 كـ slave — مش بنحدد clock-frequency لأن المـmaster هو اللي بيحدده */

    ecu_slave: ecu-co-processor@42 {
        compatible = "mycompany,ecu-slave";
        reg = <0x42>;
    };
};
```

**تشغيل تجريبي من الـ STM32 side:**

```bash
# اختبار من Linux userspace (H616) قبل الـ STM32 يكون جاهز
# استخدم i2c-slave-eeprom driver كـ placeholder
modprobe i2c-slave-eeprom

# ربط الـ slave driver على address 0x42 على i2c-3
echo slave-24c02 0x1042 > /sys/bus/i2c/devices/i2c-3/new_device

# شوف لو اتسجّل
cat /sys/kernel/debug/i2c-3/i2c-3
```

#### الدرس المستفاد
الـ I2C slave mode في `i2c.h` مكتمل من ناحية الـ API — الـ `i2c_slave_cb_t`، الـ `enum i2c_slave_event`، والـ `i2c_slave_register()` كلها جاهزة. الـ bottleneck دايماً في الـ controller driver نفسه اللي لازم يعمل implement `reg_target` في `struct i2c_algorithm`. لما بتبني multi-processor system، افحص دعم الـ slave mode في driver الـ controller من أول يوم.
## Phase 7: مصادر ومراجع

---

### التوثيق الرسمي للـ Kernel

| المصدر | الرابط |
|--------|--------|
| **I2C/SMBus Subsystem** — الصفحة الرئيسية للـ subsystem في kernel docs | [docs.kernel.org/i2c/index.html](https://docs.kernel.org/i2c/index.html) |
| **I2C and SMBus Subsystem** — واجهة الـ driver API الكاملة | [docs.kernel.org/driver-api/i2c.html](https://docs.kernel.org/driver-api/i2c.html) |
| **Writing I2C Device Drivers** — دليل كتابة الـ client driver | [docs.kernel.org/i2c/writing-clients.html](https://static.lwn.net/kerneldoc/i2c/writing-clients.html) |
| **SMBus Protocol** — تفاصيل بروتوكول الـ SMBus | [docs.kernel.org/i2c/smbus-protocol.html](https://static.lwn.net/kerneldoc/i2c/smbus-protocol.html) |
| **I2C Topology / Muxes** — الـ mux والـ topology المعقدة | [docs.kernel.org/i2c/i2c-topology.html](https://static.lwn.net/kerneldoc/i2c/i2c-topology.html) |
| **Linux I2C Sysfs** — واجهة الـ sysfs للـ I2C | [docs.kernel.org/i2c/i2c-sysfs.html](https://static.lwn.net/kerneldoc/i2c/i2c-sysfs.html) |
| **Implementing I2C drivers in userspace** — الـ /dev/i2c-N interface | [docs.kernel.org/i2c/dev-interface.html](https://docs.kernel.org/i2c/dev-interface.html) |
| **I2C and SMBus — Kernel 4.14** — نسخة قديمة للمقارنة التاريخية | [kernel.org/doc/html/v4.14/driver-api/i2c.html](https://www.kernel.org/doc/html/v4.14/driver-api/i2c.html) |

---

### مقالات LWN.net

الـ [LWN.net](https://lwn.net) هو المرجع الأول لتتبع تطور الـ kernel. أهم المقالات المتعلقة بالـ I2C subsystem:

| المقال | الوصف |
|--------|-------|
| [i2c: slave support framework for Linux devices](https://lwn.net/Articles/611332/) | مقال يشرح إطار عمل الـ I2C slave الذي يُمكّن الـ Linux من العمل كـ slave على الـ bus |
| [i2c: Add i2c-pseudo driver for userspace I2C adapters](https://lwn.net/Articles/820320/) | الـ `i2c-pseudo` driver الذي يسمح بتنفيذ الـ I2C adapter من الـ userspace |
| [i2c_imc: New driver, at long last](https://lwn.net/Articles/635898/) | driver جديد لشرائح Intel LGA2011 |
| [MCTP I2C driver](https://lwn.net/Articles/874560/) | binding driver للـ MCTP protocol فوق الـ I2C bus |
| [i2c: Add SMBus alert support](https://lwn.net/Articles/374391/) | إضافة دعم الـ SMBus Alert Protocol |
| [i2c: Add Intel SCH I2C SMBus support](https://lwn.net/Articles/283120/) | إضافة دعم شرائح Intel SCH |
| [I2C driver for IMX](https://lwn.net/Articles/283121/) | driver الـ I2C لمعالجات Freescale i.MX |
| [Kernel development — I2C overview](https://lwn.net/Articles/625735/) | نظرة عامة على تطور الـ I2C subsystem |

---

### تاريخ الـ Subsystem والـ Maintainer

الـ I2C subsystem في Linux أنشأه **Simon G. Vogl** (1995–2000). الـ maintainer الحالي هو **Wolfram Sang** `<wsa@kernel.org>`.

- **Subsystem History** — تاريخ الـ I2C في الـ kernel (الـ wiki القديم):
  [archive.kernel.org — i2c Subsystem History](https://archive.kernel.org/oldwiki/i2c.wiki.kernel.org/index.php/Subsystem_History.html)

- **Git tree للـ I2C subsystem** (الـ maintainer's tree):
  ```
  git://git.kernel.org/pub/scm/linux/kernel/git/wsa/linux.git
  ```

- **Lore — أرشيف قائمة البريد الإلكتروني للـ I2C:**
  [lore.kernel.org/linux-i2c](https://lore.kernel.org/linux-i2c/)

---

### Commits مهمة في الـ Kernel

| الـ Commit | الأهمية |
|-----------|---------|
| [`9943fa3`](https://github.com/torvalds/linux/commit/9943fa300a5dcd4536e9a4aa5c09cf94e5e7b28c) | إضافة دعم الـ regmap فوق الـ I2C bus |
| [`b90e90f`](https://github.com/torvalds/linux/commit/b90e90f40b4ff23c753126008bf4713a42353af6) | merge branch `i2c/for-current` — يمثّل دورة تطوير طبيعية |

للبحث عن الـ commits الخاصة بالـ I2C subsystem:
```bash
git log --oneline -- drivers/i2c/ include/linux/i2c.h
git log --oneline --author="Wolfram Sang" -- drivers/i2c/
```

---

### مناقشات قائمة البريد الإلكتروني

- **linux-i2c mailing list** (vger.kernel.org):
  [lore.kernel.org/linux-i2c](https://lore.kernel.org/linux-i2c/)

- **LKML Archive** — للبحث العام:
  [lkml.org](https://lkml.org/)

- **Kernel mailing lists index:**
  [kernel.org/lore.html](https://www.kernel.org/lore.html)

للاشتراك في القائمة:
```
linux-i2c@vger.kernel.org
```

---

### مصادر eLinux.org

| الصفحة | الوصف |
|--------|-------|
| [BoardBringUp-i2c](https://elinux.org/BoardBringUp-i2c) | دليل عملي لـ board bring-up باستخدام الـ I2C |
| [Tests:I2C-fault-injection](https://elinux.org/Tests:I2C-fault-injection) | اختبار الـ fault injection في الـ I2C |
| [Tests:RCar-Gen2-I2C-demux](https://elinux.org/Tests:RCar-Gen2-I2C-demux) | اختبار الـ I2C demux على بورد Renesas |
| [Hammer I2C Driver](https://elinux.org/Hammer_I2C_Driver) | مثال تطبيقي على كتابة الـ I2C driver على الـ S3C2410 |
| [The Shiny New I2C Slave Framework (PDF)](https://elinux.org/images/f/f6/ELCE15-WolframSang-ShinyNewI2CSlaveFramework.pdf) | عرض Wolfram Sang في ELCE 2015 عن الـ slave framework |

---

### KernelNewbies.org

| الصفحة | الأهمية |
|--------|---------|
| [HimanshuJha/GSoC18](https://kernelnewbies.org/HimanshuJha/GSoC18) | مشروع GSoC لإضافة الـ I2C/SPI interface للـ bme680 sensor |
| [I2C: kernel & userspace drivers — i2c-stub discussion](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2013-April/008084.html) | نقاش عملي عن الـ i2c-stub لاختبار الـ drivers |
| [Linux_3.8_DriverArch](https://kernelnewbies.org/Linux_3.8_DriverArch) | يتضمن تغييرات الـ I2C في kernel 3.8 |

---

### الكتب الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 8: Allocating Memory** — فهم الـ kernel memory قبل الـ I2C buffers
- **الفصل 12: PCI Drivers** — نمط الـ probe/remove المشابه لـ I2C driver
- **الفصل 14: The Linux Device Model** — الـ `struct device`, bus, driver, الأساس الذي يبني عليه الـ `struct i2c_adapter`
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل 14: The Block I/O Layer** — فهم الـ I/O abstractions
- **الفصل 17: Devices and Modules** — الـ device model الذي يعتمد عليه الـ I2C

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل 8: Device Drivers** — يغطي الـ I2C drivers بشكل عملي على الـ embedded boards
- **الفصل 15: Debugging Embedded Linux** — debugging الـ I2C transactions

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- **الفصل 8: The Inter-Integrated Circuit Protocol** — فصل كامل مخصص للـ I2C في Linux

---

### مصادر إضافية

| المصدر | الوصف |
|--------|-------|
| [Analog Devices — Linux I2C Kernel Docs](https://wiki.analog.com/resources/tools-software/linuxdsp/docs/linux-kernel-and-drivers/i2c/i2c) | شرح تطبيقي من Analog Devices لكتابة الـ I2C drivers |
| [Dealing with I2C devices on Linux](https://hackerbikepacker.com/i2c-on-linux) | مقال عملي للتعامل مع الـ I2C من الـ userspace |
| [i2c-tools on GitHub](https://git.kernel.org/pub/scm/utils/i2c-tools/i2c-tools.git/) | أدوات الـ userspace: `i2cdetect`, `i2cdump`, `i2cget`, `i2cset` |

---

### أوامر البحث الموصى بها

للبحث عن معلومات إضافية عن الـ I2C subsystem في Linux:

```
# LWN.net
site:lwn.net linux i2c adapter driver
site:lwn.net i2c smbus kernel

# Kernel documentation
site:docs.kernel.org i2c
site:kernel.org i2c writing clients

# Lore mailing list
lore.kernel.org/linux-i2c i2c_add_adapter
lore.kernel.org/linux-i2c i2c_transfer

# General
linux kernel i2c_driver probe remove example
linux i2c slave mode eeprom at24
linux i2c-gpio bitbang driver
linux i2c multiplexer pca954x
```

---

### ملفات الـ Source الرئيسية في الـ Kernel

```
include/linux/i2c.h          ← واجهة الـ I2C الرئيسية (الملف الذي ندرسه)
include/uapi/linux/i2c.h     ← الـ uapi definitions
include/uapi/linux/i2c-dev.h ← الـ userspace /dev interface

drivers/i2c/i2c-core-base.c  ← الـ core: i2c_transfer, i2c_add_adapter
drivers/i2c/i2c-core-smbus.c ← الـ SMBus emulation layer
drivers/i2c/i2c-core-slave.c ← الـ slave mode framework
drivers/i2c/i2c-dev.c        ← الـ /dev/i2c-N character device
drivers/i2c/i2c-mux.c        ← الـ mux framework
drivers/i2c/busses/          ← الـ adapter drivers (i2c-i801, i2c-gpio, ...)
drivers/i2c/algos/           ← الـ bit-banging algorithms
```

للتعمق في الكود مباشرة:
```bash
# قراءة الـ core
less drivers/i2c/i2c-core-base.c

# البحث عن كل الـ i2c_driver المسجّلة في الـ kernel
grep -r "\.probe\s*=" drivers/i2c/busses/ --include="*.c" | head -20

# رؤية تاريخ الـ header الذي ندرسه
git log --follow -p include/linux/i2c.h | head -100
```
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على الدالة `i2c_transfer` — وهي الدالة الأساسية اللي بتنفّذ أي I2C message على الـ bus. كل مرة أي driver في النظام يبعت أو يستقبل بيانات I2C، هتتمسك الـ probe وتطبع: اسم الـ adapter، عدد الـ messages، وأول address في القائمة. ده بيخليها أداة ممتازة لفهم من يكلّم من على الـ I2C bus وقت الـ runtime.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * i2c_transfer_spy.c
 *
 * kprobe on i2c_transfer() — logs every I2C transaction:
 *   adapter name, number of messages, first message address & direction.
 */

/* ===== Includes ===== */
#include <linux/kernel.h>      /* pr_info, pr_err */
#include <linux/module.h>      /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/i2c.h>         /* struct i2c_adapter, struct i2c_msg, I2C_M_RD */
#include <linux/uaccess.h>     /* needed transitively by some archs */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe spy on i2c_transfer — logs I2C bus activity");

/* ===== pre-handler: يتشغل قبل ما i2c_transfer تنفّذ ===== */

/*
 * توقيع i2c_transfer:
 *   int i2c_transfer(struct i2c_adapter *adap,
 *                    struct i2c_msg     *msgs,
 *                    int                 num);
 *
 * الـ arguments موجودة في registers حسب ABI:
 *   regs->di = adap, regs->si = msgs, regs->dx = num   (x86-64)
 * بس مش هنعتمد على manual register read —
 * بدل كده نستخدم __kprobes وندخل من خلال pt_regs بشكل portable.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * نجيب الـ arguments من الـ calling convention.
     * على x86-64: RDI=arg0, RSI=arg1, RDX=arg2
     * نعمل cast مباشر — آمن لأننا في pre-handler قبل دخول الدالة.
     */
    struct i2c_adapter *adap = (struct i2c_adapter *)regs->di;
    struct i2c_msg     *msgs = (struct i2c_msg     *)regs->si;
    int                 num  = (int)                 regs->dx;

    /* تأكد إن الـ pointers مش NULL قبل ما نقرأ منهم */
    if (!adap || !msgs || num <= 0)
        return 0;

    pr_info("i2c_spy: adapter=\"%s\" nr=%d  msgs=%d  "
            "first_addr=0x%02x (%s)\n",
            adap->name,          /* اسم الـ adapter زي "i2c-0" أو "SMBus I801" */
            adap->nr,            /* رقم الـ bus */
            num,                 /* عدد الـ messages في الـ transaction */
            msgs[0].addr,        /* address الـ slave device الأول */
            (msgs[0].flags & I2C_M_RD) ? "READ" : "WRITE"); /* اتجاه الـ transfer */

    return 0; /* لازم نرجّع 0 عشان الـ kprobe يكمل تنفيذ الدالة الأصلية */
}

/* ===== تعريف الـ kprobe ===== */

/*
 * بنحدد اسم الدالة كـ symbol_name بدل عنوان raw —
 * ده أسهل وأكثر portability عبر kernel versions.
 */
static struct kprobe kp = {
    .symbol_name = "i2c_transfer",
    .pre_handler = handler_pre,
};

/* ===== module_init ===== */

static int __init i2c_spy_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        /*
         * ممكن يفشل لو:
         *   - الدالة مش exported أو مفيش رمز ليها في kallsyms
         *   - CONFIG_KPROBES مش مفعّل
         *   - الدالة __kprobes (محمية من الـ probe)
         */
        pr_err("i2c_spy: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("i2c_spy: planted kprobe on i2c_transfer @ %p\n", kp.addr);
    return 0;
}

/* ===== module_exit ===== */

static void __exit i2c_spy_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما الـ module يتفرّغ من الذاكرة —
     * لو سبنا الـ kprobe active، أي i2c_transfer call بعد كده
     * هترجّع على كود محذوف وتعمل kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("i2c_spy: removed kprobe from i2c_transfer\n");
}

module_init(i2c_spy_init);
module_exit(i2c_spy_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `<linux/kprobes.h>` | **الـ kprobe API** — `register_kprobe`, `unregister_kprobe`, `struct kprobe` |
| `<linux/i2c.h>` | تعريفات `struct i2c_adapter`, `struct i2c_msg`, `I2C_M_RD` |
| `<linux/module.h>` | الـ macros الأساسية لأي kernel module |
| `<linux/kernel.h>` | `pr_info`, `pr_err` |

---

#### الـ `handler_pre` — قلب الـ module

**الـ pre_handler** بيتشغل مباشرة قبل تنفيذ الدالة المستهدفة. بنقرأ الـ arguments من الـ `pt_regs` اللي بيمثّل state الـ CPU لحظة الـ intercept — على x86-64 الـ arguments بتجي في `RDI, RSI, RDX` بالترتيب.

بنطبع:
- **`adap->name`**: اسم الـ I2C controller زي `"SMBus I801 adapter"` أو `"Synopsys DesignWare I2C adapter"`
- **`adap->nr`**: رقم الـ bus (i2c-0, i2c-1, ...)
- **`num`**: عدد الـ messages في الـ transaction — مثلاً read-after-write بتكون 2
- **`msgs[0].addr`**: عنوان الـ slave device الأول بالـ 7-bit format
- **`I2C_M_RD` flag**: لو موجود يبقى READ، غيره WRITE

---

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "i2c_transfer",
    .pre_handler = handler_pre,
};
```

**الـ `symbol_name`** بيخلي الـ kernel يحوّل الاسم لعنوان تلقائياً من خلال kallsyms — أفضل من hardcoding عنوان. **الـ `pre_handler`** هو الـ callback اللي بيتشغل قبل الدالة.

---

#### الـ `module_init`

بنسجّل الـ kprobe بـ `register_kprobe`. لو نجح، الـ kernel بيكتب **breakpoint instruction** في بداية `i2c_transfer` ويحفظ العنوان الفعلي في `kp.addr`. أي call لـ `i2c_transfer` بعد كده هيمرّ على الـ handler قبل ما ينفّذ.

---

#### الـ `module_exit`

**إلزامي** نعمل `unregister_kprobe` هنا. لو الـ module اتحذف من الذاكرة والـ kprobe لسه active، أي I2C transaction هتشغّل كود في عنوان محذوف ← kernel panic. الـ unregister بيشيل الـ breakpoint ويرجّع الـ original instruction.

---

### Build & Test

```bash
# Makefile بسيط
obj-m += i2c_transfer_spy.o

# Build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# Load
sudo insmod i2c_transfer_spy.ko

# شوف الـ output — أي I2C activity في النظام هتظهر
sudo dmesg -w | grep i2c_spy

# Unload
sudo rmmod i2c_transfer_spy
```

**مثال على الـ output:**

```
[  142.331204] i2c_spy: planted kprobe on i2c_transfer @ ffffffffc0a1b230
[  142.889451] i2c_spy: adapter="SMBus I801 adapter at efa0" nr=0  msgs=2  first_addr=0x50 (WRITE)
[  142.889523] i2c_spy: adapter="SMBus I801 adapter at efa0" nr=0  msgs=1  first_addr=0x50 (READ)
[  143.002100] i2c_spy: adapter="i2c-1" nr=1  msgs=1  first_addr=0x1a (WRITE)
```

الـ address `0x50` ده EEPROM كلاسيكي، و`0x1a` ممكن يكون audio codec — بالظبط اللي بيحصل على الـ bus في الـ production system.
