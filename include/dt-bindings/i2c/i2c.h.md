## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من **I2C Subsystem** في Linux kernel. موجود في MAINTAINERS تحت اسم:
- `I2C SUBSYSTEM` — المشرف: Wolfram Sang (نفس الشخص اللي عمل الملف ده)
- `I2C SUBSYSTEM HOST DRIVERS` — كمان بيغطي مجلد `include/dt-bindings/i2c/`

---

### ما هو الـ I2C؟ — قصة قبل الكود

تخيل عندك PCB (لوحة إلكترونية) فيها chip رئيسية (زي microprocessor) وعلى نفس اللوحة في sensor للحرارة، وشاشة صغيرة، وساعة real-time، وذاكرة EEPROM. كل دي أجهزة مختلفة من شركات مختلفة.

المشكلة: إزاي الـ processor يكلم كل الأجهزة دي؟ لو عمل wire منفصل لكل جهاز — هيحتاج عشرات الـ pins. الـ engineers اخترعوا **I2C (Inter-Integrated Circuit)**، بروتوكول بسيط اخترعته Philips سنة 1982. فكرته:

- **سلكين بس**: SCL (Clock) و SDA (Data).
- كل جهاز على الـ bus عنده **address** فريد — زي أرقام الشقق في عمارة.
- الـ processor (الـ **master**) بيبعت: "أنا عايز أكلم العنوان 0x48" — والجهاز صاحب العنوان ده بيرد.

---

### العناوين في I2C — المشكلة اللي الملف بيحلها

#### عناوين 7-bit (الحالة العادية)

العنوان العادي في I2C هو **7 bits** — يعني ممكن يوصل لـ 128 جهاز على نفس الـ bus. كل جهاز عنوانه رقم من 0x00 لـ 0x7F.

في الـ Device Tree (DTS)، لما بتعرّف جهاز I2C بتكتب:

```c
/* جهاز عادي عنوانه 0x50 */
eeprom@50 {
    compatible = "atmel,24c02";
    reg = <0x50>;
};
```

الـ `reg` ده هو العنوان — رقم بسيط في 7 bits الأولى.

#### المشكلة الأولى — الـ 10-bit Address

بعض الأجهزة الحديثة بتحتاج أكتر من 128 عنوان — خصوصاً في الأنظمة الكبيرة. I2C spec أضافت دعم **10-bit addressing**. لكن هنا السؤال: في الـ Device Tree، إزاي تفرق بين جهاز عنوانه `0x05` في 7-bit وجهاز تاني عنوانه `0x05` في 10-bit؟

**الحل**: بيستخدموا **bit عالي جداً في الـ reg** كـ flag. الـ `I2C_TEN_BIT_ADDRESS` هو `(1 << 31)` — يعني أعلى bit في الـ 32-bit integer. لما بتشوف البت ده مضروب، يعني العنوان ده 10-bit مش 7-bit:

```c
#define I2C_TEN_BIT_ADDRESS   (1 << 31)  /* bit 31 = علامة إن العنوان 10-bit */
```

مثال في DTS:
```c
sensor@205 {
    reg = <(I2C_TEN_BIT_ADDRESS | 0x205)>;
    /* يعني عنوان 10-bit قيمته 0x205 */
};
```

#### المشكلة التانية — الـ Own Slave Address

في الحالة العادية، الـ Linux هو الـ **master** — هو اللي بيبدأ الكلام وبيطلب من الأجهزة. لكن في بعض الأنظمة المعقدة (زي server boards من IBM بتستخدم ASPEED BMC)، الـ Linux نفسه محتاج يكون **slave** — يعني جهاز تاني (master تاني) يقدر يكلمه على الـ I2C bus.

مثال واقعي: في IBM Power Server، في chip اسمها **OP Panel (Operation Panel)** — شاشة صغيرة على الـ server. الـ BMC (Board Management Controller) بيشتغل Linux وعنده I2C controller. الـ OP Panel محتاج يكلم الـ BMC كـ slave على العنوان `0x62`.

في الـ DTS بتكتب:
```c
/* من aspeed-bmc-ibm-balcones.dts */
lcd-controller@62 {
    compatible = "ibm,op-panel";
    reg = <(0x62 | I2C_OWN_SLAVE_ADDRESS)>;
    /* يعني: الـ BMC نفسه بيستمع على العنوان 0x62 كـ slave */
};
```

الـ `I2C_OWN_SLAVE_ADDRESS` هو `(1 << 30)` — bit 30:

```c
#define I2C_OWN_SLAVE_ADDRESS (1 << 30)  /* bit 30 = علامة إن ده عنوان slave للـ controller نفسه */
```

---

### الـ Device Tree Bindings — ليه في `dt-bindings/`؟

الـ **Device Tree** هو ملف نصي (`.dts`) بيوصف الـ hardware للـ kernel. لما تكتب فيه أرقام، ممكن تكتب الأرقام raw أو تستخدم constants من header files.

لكن في مشكلة: الـ DTS بيتحول لـ **DTB (Device Tree Blob)** بواسطة `dtc` compiler، والـ DTS نفسه ممكن يتضمن C headers. الـ headers الموجودة في `include/dt-bindings/` هي بالضبط headers مصممة عشان تتضمن في DTS وفي C code في نفس الوقت — من غير dependency على Linux-specific types.

ولهذا مجلد `dt-bindings` منفصل عن `include/linux/`:
- **`include/dt-bindings/i2c/i2c.h`**: مجرد `#define` constants — لا structs، لا types، لا kernel APIs. يعني يقدر يتضمن في أي DTS file بسهولة.
- **`include/linux/i2c.h`**: الـ full I2C API للـ kernel — structs كبيرة، function prototypes، وغيره.

---

### الملف نفسه — بساطة متعمدة

```c
/* SPDX-License-Identifier: GPL-2.0-only */
#ifndef _DT_BINDINGS_I2C_I2C_H
#define _DT_BINDINGS_I2C_I2C_H

#define I2C_TEN_BIT_ADDRESS   (1 << 31)  /* flag: العنوان 10-bit مش 7-bit */
#define I2C_OWN_SLAVE_ADDRESS (1 << 30)  /* flag: ده عنوان slave للـ controller نفسه */

#endif
```

سطرين بس. لكن وراهم:
- اتفاقية بين كل مستخدمين الـ DTS في الـ kernel
- يضمن إن الـ DTB compiler والـ C compiler بيفهموا نفس الـ encoding
- الـ parser في `drivers/i2c/i2c-core-of.c` بيشيل الـ flags ويحول الـ reg لعنوان حقيقي

---

### إزاي بيشتغل في الـ Kernel

في `drivers/i2c/i2c-core-of.c` الـ function `of_i2c_get_board_info()` بتقرأ الـ `reg` property من الـ DT وبتعمل:

```c
if (addr & I2C_TEN_BIT_ADDRESS) {
    addr &= ~I2C_TEN_BIT_ADDRESS;    /* شيل الـ flag */
    info->flags |= I2C_CLIENT_TEN;   /* قول للـ driver إنه 10-bit */
}

if (addr & I2C_OWN_SLAVE_ADDRESS) {
    addr &= ~I2C_OWN_SLAVE_ADDRESS;  /* شيل الـ flag */
    info->flags |= I2C_CLIENT_SLAVE; /* قول للـ driver إنه slave mode */
}

info->addr = addr;  /* العنوان الحقيقي بدون الـ flags */
```

---

### الملفات المرتبطة اللي المفروض تعرفها

| الملف | الدور |
|---|---|
| `include/dt-bindings/i2c/i2c.h` | **الملف ده** — constants للـ DT |
| `include/linux/i2c.h` | Full I2C API: structs, flags, functions |
| `drivers/i2c/i2c-core-of.c` | بيقرأ الـ DT ويستخدم الـ constants دي |
| `drivers/i2c/i2c-core-slave.c` | دعم الـ slave mode في الـ kernel |
| `drivers/i2c/i2c-core-base.c` | Core I2C bus logic |
| `Documentation/i2c/slave-interface.rst` | شرح الـ slave mode |
| `Documentation/i2c/i2c-protocol.rst` | شرح الـ I2C protocol |
| `arch/arm/boot/dts/aspeed/*.dts` | أمثلة واقعية بتستخدم الـ constants دي |

### ملفات الـ Subsystem بالكامل (من MAINTAINERS)

- **Core**: `drivers/i2c/i2c-core-*.c`, `drivers/i2c/i2c-core.h`
- **Headers**: `include/linux/i2c.h`, `include/linux/i2c-smbus.h`, `include/linux/i2c-dev.h`
- **DT Bindings**: `include/dt-bindings/i2c/i2c.h` ← الملف ده
- **UAPI**: `include/uapi/linux/i2c.h`, `include/uapi/linux/i2c-*.h`
- **Bus Drivers**: `drivers/i2c/busses/` (المئات من الـ hardware drivers)
- **Docs**: `Documentation/i2c/`
## Phase 2: شرح الـ I2C Framework

### المشكلة — ليه الـ I2C Framework موجود أصلاً؟

في أي SoC حديث بتلاقي عشرات الـ peripherals اللي بتحتاج تتكلم مع الـ CPU: sensors، EEPROMs، PMICs، codecs، touchscreen controllers. كل chip من دول ليه register map جواه، وبتوصله عبر I2C bus — سلكين بس (SDA و SCL).

المشكلة من غير framework:
- كل driver لازم يتعامل مع الـ hardware controller (I2C master) مباشرة.
- لو غيرت الـ SoC من STM32 لـ iMX8 — كل الـ device drivers بتتكسر.
- ملوش طريقة standard لـ enumeration ولا لـ address conflict detection.
- Concurrency على الـ bus (أكتر من driver بيعمل transfer في نفس الوقت) — كارثة.

الـ kernel لازم يعمل **abstraction layer** يفصل الـ device driver (اللي بيتكلم مع شيبة EEPROM مثلاً) عن الـ bus controller driver (اللي بيتحكم في الـ hardware master).

---

### الحل — الـ I2C Framework بيعمل إيه؟

الـ I2C framework بيقدم:

1. **Bus abstraction** — الـ `i2c_adapter` يمثل أي I2C controller بغض النظر عن الـ SoC.
2. **Device model integration** — كل device على الـ bus بيتعامل كـ `i2c_client` مرتبط بـ `struct device`، يعني بيستفيد من كل الـ driver model infrastructure (power management، sysfs، devres).
3. **Transfer API موحد** — `i2c_transfer()` و `i2c_smbus_*` بيشتغلوا على أي adapter.
4. **Locking على مستوى الـ bus** — rt_mutex يمنع race conditions.
5. **Device Tree / ACPI enumeration** — بدل ما تكتب board files يدوياً.
6. **Address flags standardization** — ومنها اللي إحنا شايفينه في الملف: `I2C_TEN_BIT_ADDRESS` و `I2C_OWN_SLAVE_ADDRESS`.

---

### الـ dt-bindings/i2c/i2c.h — دور الملف ده في الـ Framework

الملف ده صغير جداً — سطرين macro بس — لكنه محوري:

```c
#define I2C_TEN_BIT_ADDRESS   (1 << 31)  /* bit 31 = 10-bit address mode */
#define I2C_OWN_SLAVE_ADDRESS (1 << 30)  /* bit 30 = adapter acts as slave */
```

**الـ dt-bindings headers** هي headers مشتركة بين:
- الـ **kernel C code** — بيستخدمها الـ drivers والـ core.
- الـ **Device Tree source files** (`.dts` / `.dtsi`) — بتحدد الـ node properties.

الهدف: لو عندك node في الـ DTS بتقول إن الـ device ده عنده 10-bit address، الـ flag المستخدم في الـ DTS **نفسه** المستخدم في الـ C code. ده بيمنع desynchronization بين الـ firmware description والـ kernel code.

```
/* في الـ DTS */
eeprom@200 {
    compatible = "atmel,24c32";
    reg = <(0x200 | I2C_TEN_BIT_ADDRESS)>;  /* = 0x80000200 */
};
```

الـ I2C core بيشوف الـ bit 31 مضبوط في الـ `reg` property، فيعرف إن العنوان ده 10-bit مش 7-bit.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────┐
  │                   User Space / sysfs                    │
  └──────────────────────────┬──────────────────────────────┘
                             │  /dev/i2c-N, /sys/bus/i2c/
  ┌──────────────────────────▼──────────────────────────────┐
  │              I2C Core  (drivers/i2c/i2c-core-*.c)       │
  │                                                         │
  │   i2c_bus_type   ←→   driver model (struct bus_type)    │
  │   i2c_transfer()        i2c_smbus_xfer()                │
  │   i2c_register_adapter()  i2c_new_client_device()       │
  │   Locking: rt_mutex (bus_lock, mux_lock)                │
  └────────┬─────────────────────────────────────┬──────────┘
           │                                     │
           ▼                                     ▼
  ┌─────────────────┐                  ┌──────────────────────┐
  │  i2c_adapter    │                  │  i2c_client          │
  │  (Provider)     │                  │  (Consumer)          │
  │                 │                  │                      │
  │  struct         │  owns/manages    │  struct              │
  │  i2c_algorithm  │ ───────────────► │  i2c_driver          │
  │  .xfer()        │                  │  .probe()            │
  │  .smbus_xfer()  │                  │  .remove()           │
  │  .functionality │                  │  .alert()            │
  └────────┬────────┘                  └──────────────────────┘
           │
           ▼
  ┌─────────────────┐
  │  SoC Hardware   │
  │  I2C Controller │
  │  (e.g. i.MX8,   │
  │   STM32, RPi)   │
  └────────┬────────┘
           │  SDA / SCL (physical wires)
           ▼
  ┌─────────────────────────────────────────────────────────┐
  │  I2C Bus                                                │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
  │  │ 0x50     │  │ 0x68     │  │ 0x1A (10b)│             │
  │  │ EEPROM   │  │ RTC      │  │ CODEC    │             │
  │  │ (client) │  │ (client) │  │ (client) │             │
  │  └──────────┘  └──────────┘  └──────────┘             │
  └─────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstractions — الـ Structs الأساسية

#### 1. `struct i2c_adapter` — يمثل الـ Bus Controller

```c
struct i2c_adapter {
    struct module *owner;
    unsigned int class;
    const struct i2c_algorithm *algo;  /* HOW to talk on the bus */
    void *algo_data;

    const struct i2c_lock_operations *lock_ops;
    struct rt_mutex bus_lock;   /* protects transfers */
    struct rt_mutex mux_lock;   /* for i2c-mux trees */

    int timeout;    /* transfer timeout in jiffies */
    int retries;
    struct device dev;
    int nr;         /* bus number: i2c-0, i2c-1, ... */
    char name[48];

    struct i2c_bus_recovery_info *bus_recovery_info;
    const struct i2c_adapter_quirks *quirks;

    struct irq_domain *host_notify_domain;
    struct regulator *bus_regulator;

    DECLARE_BITMAP(addrs_in_instantiation, 1 << 7); /* 7-bit address tracking */
};
```

الـ `i2c_adapter` هو التجريد للـ physical bus. الـ SoC driver بيسجله مع الـ core عبر `i2c_add_adapter()`. الـ `bus_lock` هو `rt_mutex` (مش mutex عادي) عشان الـ I2C بيتاخد من contexts بتدعم priority inheritance زي الـ PREEMPT_RT.

#### 2. `struct i2c_algorithm` — الـ Virtual Function Table للـ Adapter

```c
struct i2c_algorithm {
    /* raw I2C transfers */
    union {
        int (*xfer)(struct i2c_adapter *adap,
                    struct i2c_msg *msgs, int num);
        int (*master_xfer)(...); /* deprecated alias */
    };

    /* atomic variant for PMIC access at shutdown */
    union {
        int (*xfer_atomic)(...);
        int (*master_xfer_atomic)(...);
    };

    /* SMBus-native transfers (optional) */
    int (*smbus_xfer)(struct i2c_adapter *adap, u16 addr,
                      unsigned short flags, char read_write,
                      u8 command, int size, union i2c_smbus_data *data);

    /* what does this adapter support? */
    u32 (*functionality)(struct i2c_adapter *adap);

    /* slave mode support */
    int (*reg_target)(struct i2c_client *client);
    int (*unreg_target)(struct i2c_client *client);
};
```

ملحوظة مهمة: لو الـ adapter مش بيدعم `smbus_xfer`، الـ I2C core بيعمل **SMBus emulation** تلقائياً — بيحول الـ SMBus protocol لـ I2C messages عادية. ده بيخلي كل الـ SMBus devices تشتغل على أي I2C adapter.

#### 3. `struct i2c_client` — يمثل Device على الـ Bus

```c
struct i2c_client {
    unsigned short flags;   /* I2C_CLIENT_TEN, I2C_CLIENT_PEC, ... */
    unsigned short addr;    /* 7-bit stored in lower 7 bits */
    char name[I2C_NAME_SIZE];
    struct i2c_adapter *adapter;  /* which bus? */
    struct device dev;            /* embedded device */
    int irq;
    struct list_head detected;
    i2c_slave_cb_t slave_cb;      /* if adapter used in slave mode */
};
```

الـ `I2C_CLIENT_TEN = 0x10` — اللي بيتطابق مع `I2C_M_TEN` في مستوى الـ message. لما الـ core يشوف هذا الـ flag، بيبعت الـ 10-bit address في أول message بصيغة خاصة حسب الـ I2C spec.

#### 4. `struct i2c_driver` — الـ Device Driver نفسه

```c
struct i2c_driver {
    unsigned int class;
    int (*probe)(struct i2c_client *client);
    void (*remove)(struct i2c_client *client);
    void (*shutdown)(struct i2c_client *client);
    void (*alert)(struct i2c_client *client,
                  enum i2c_alert_protocol protocol,
                  unsigned int data);
    struct device_driver driver;  /* embedded driver */
    const struct i2c_device_id *id_table;
    int (*detect)(struct i2c_client *client,
                  struct i2c_board_info *info);
    const unsigned short *address_list;
    struct list_head clients;
    u32 flags;
};
```

---

### العلاقة بين الـ Structs

```
  i2c_driver
  ┌──────────────────────────┐
  │ driver (device_driver)   │
  │ probe() ─────────────────┼──► called by i2c-core when client matches
  │ remove()                 │
  │ id_table[]               │◄── matching against i2c_client.name / OF compat
  └──────────────────────────┘
            │
            │ manages (1:N)
            ▼
  i2c_client
  ┌──────────────────────────┐
  │ addr = 0x50              │
  │ name = "24c32"           │
  │ flags = I2C_CLIENT_TEN?  │◄── populated from DT: I2C_TEN_BIT_ADDRESS
  │ adapter ─────────────────┼──► points to i2c_adapter
  │ dev (struct device)      │◄── registered in driver model
  └──────────────────────────┘
            │
            │ .adapter points to
            ▼
  i2c_adapter
  ┌──────────────────────────┐
  │ nr = 1 (i2c-1)           │
  │ algo ─────────────────── ┼──► i2c_algorithm vtable
  │ bus_lock (rt_mutex)      │
  │ quirks                   │
  │ bus_recovery_info        │
  │ dev (struct device)      │
  └──────────────────────────┘
            │
            │ .algo points to
            ▼
  i2c_algorithm
  ┌──────────────────────────┐
  │ xfer() ◄─────────────────┼── implemented by SoC driver
  │ functionality()          │      e.g. imx_i2c_xfer()
  │ smbus_xfer() (optional)  │
  └──────────────────────────┘
```

---

### الـ Address Flags — التفاصيل الكاملة

#### `I2C_TEN_BIT_ADDRESS` (bit 31)

الـ I2C standard الأصلي بيعمل 7-bit addressing — يعني 128 عنوان بس. لما الـ devices كترت، الـ I2C spec أضاف **10-bit addressing mode**. البرتوكول بيبعت:
- أول byte: `11110XX` حيث XX هما الـ bits 8-9 من العنوان + write bit.
- تاني byte: الـ 8 bits الباقيين من العنوان.

في الـ DTS:
```
sensor@280 {
    reg = <(0x280 | I2C_TEN_BIT_ADDRESS)>;
    /* = <0x80000280> */
};
```

الـ `I2C_TEN_BIT_ADDRESS = BIT(31)` بيتخزن في الـ upper bits من الـ `reg` property عشان يفرق العنوان عن مجرد عنوان 7-bit طويل. الـ i2c-core بيشيل الـ flag ده ويحط `I2C_CLIENT_TEN` في الـ `i2c_client.flags`.

#### `I2C_OWN_SLAVE_ADDRESS` (bit 30)

ده بيقول إن الـ I2C controller نفسه ممكن يتعامل كـ **I2C slave** — مش بس master. ده مهم في:
- **PMBus** — الـ voltage regulators بتبعت events.
- **IPMI** — management controllers بتتكلم مع بعض.
- بعض الـ multi-master setups.

لما تشوف الـ bit ده في الـ `reg` property، الـ kernel بيسجل الـ adapter كـ slave على العنوان ده، وبيستخدم الـ `slave_cb` في الـ `i2c_client` لاستقبال الـ events.

---

### الـ Device Tree — الوصلة بين الـ Firmware والـ Framework

**قبل الـ Device Tree** كان لازم تكتب:

```c
/* board file — arch/arm/mach-xxx/board-xxx.c */
static struct i2c_board_info board_i2c[] = {
    { I2C_BOARD_INFO("ds1307", 0x68) },
    { I2C_BOARD_INFO("at24", 0x50) },
};
```

**مع الـ Device Tree**:

```dts
/* في الـ .dts */
&i2c1 {
    clock-frequency = <400000>;
    status = "okay";

    rtc@68 {
        compatible = "dallas,ds1307";
        reg = <0x68>;
    };

    eeprom@50 {
        compatible = "atmel,24c32";
        reg = <0x50>;
    };

    /* 10-bit address device */
    adc@200 {
        compatible = "ti,ads1115";
        reg = <(0x200 | I2C_TEN_BIT_ADDRESS)>;
    };
};
```

الـ `dt-bindings/i2c/i2c.h` بيتضمن في الـ DTS preprocessor phase عشان الـ `I2C_TEN_BIT_ADDRESS` macro يكون متاح في الـ DTS نفسه — نفس القيمة اللي بيفهمها الـ kernel.

---

### الـ Transfer Flow — من الـ Driver لـ الـ Hardware

```
  Driver calls i2c_master_send(client, buf, count)
        │
        ▼
  i2c_transfer_buffer_flags()
        │
        ▼ builds struct i2c_msg[]
  i2c_transfer(adapter, msgs, num)
        │
        ├── lock: rt_mutex_lock(&adap->bus_lock)
        ├── check: adap->quirks  (max_num_msgs, max_write_len, ...)
        │
        ▼
  __i2c_transfer()
        │
        ▼
  adap->algo->xfer(adap, msgs, num)   ← SoC-specific implementation
        │
        ▼ hardware registers / DMA
  [physical I2C transaction on SDA/SCL]
        │
        └── unlock: rt_mutex_unlock(&adap->bus_lock)
```

---

### التشبيه الواقعي — الـ Post Office

تخيل الـ I2C framework زي **نظام البريد المحلي** في مدينة:

| مفهوم I2C | تشبيه البريد |
|---|---|
| `i2c_adapter` | مكتب البريد (يمثل الـ bus نفسه) |
| `i2c_algorithm` | طريقة التوصيل (سيارة، دراجة، طائرة) |
| `i2c_client` | مستلم بريد (عنده address محدد) |
| `i2c_driver` | موظف المكتب المسؤول عن نوع معين من الطرود |
| `i2c_msg` | الطرد / الرسالة نفسها |
| `i2c_transfer()` | عملية الإرسال |
| `bus_lock` (rt_mutex) | شباك الخدمة — بس واحد يعمل transaction في لحظة واحدة |
| `I2C_TEN_BIT_ADDRESS` | كود بريدي مكون من 10 أرقام بدل 7 |
| `I2C_OWN_SLAVE_ADDRESS` | مكتب البريد نفسه له عنوان — ممكن يستلم طرود هو كمان |
| `i2c_adapter_quirks` | قيود المكتب: "مش بنشيل طرود أثقل من 5 كيلو" |
| SMBus emulation | لو الموظف مش عارف بروتوكول الـ express، يحوله لـ بريد عادي |

التشبيه ده بيتعمق أكتر:
- **مكاتب بريد متعددة** (أكتر من `i2c_adapter`) = أكتر من bus في نفس الـ SoC.
- **مدير مكتب البريد** (الـ I2C core) بيشوف كل المكاتب ومش مهتم بالـ hardware تفاصيلها.
- **لو المكتب اتعطل** (bus stuck low) — في **Bus Recovery** procedure يحاول يصلحه عبر `i2c_bus_recovery_info` بدل ما تحتاج تعيد تشغيل النظام.

---

### ما الذي يمتلكه الـ I2C Framework مقابل ما يفوضه للـ Drivers

#### الـ Framework بيمتلك:
- **Bus registration & numbering** — `i2c_add_adapter()` بتدي الـ adapter رقم تلقائي (`/dev/i2c-0`, `i2c-1`, ...)
- **Device matching** — يربط الـ `i2c_client` بالـ `i2c_driver` المناسب (by name, DT compatible, ACPI ID).
- **Locking policy** — `bus_lock` و `mux_lock` والـ `I2C_LOCK_ROOT_ADAPTER` / `I2C_LOCK_SEGMENT` flags.
- **SMBus emulation** — لو الـ adapter مش بيدعم SMBus natively.
- **Address conflict detection** — الـ bitmap `addrs_in_instantiation` بيمنع تسجيل نفس العنوان مرتين.
- **Bus Recovery orchestration** — بيحدد متى يستدعي `bus_recovery_info->recover_bus()`.
- **DT/ACPI/board_info enumeration** — بيمشي على الـ device nodes وبينشئ `i2c_client` لكل واحد.

#### الـ Framework بيفوض للـ Drivers:
- **SoC controller driver** — تنفيذ `i2c_algorithm.xfer()` — ده الـ hardware-specific code.
- **Device driver** — تنفيذ `i2c_driver.probe()` — ده يفهم معنى الـ registers في الـ chip.
- **Quirks declaration** — الـ controller driver بيعلن عن قيوده عبر `i2c_adapter_quirks`.
- **Bus Recovery implementation** — الـ controller driver بيدي الـ callbacks لـ SCL/SDA toggling.
- **Timing configuration** — `i2c_timings` بيتملى من الـ DT وبيتدي للـ controller driver.

---

### الـ `i2c_adapter_quirks` — التعامل مع الـ Hardware Limitations

بعض الـ I2C controllers محدودة hardware:

```c
struct i2c_adapter_quirks {
    u64 flags;
    int max_num_msgs;       /* بعض controllers بس يعرفوا يبعتوا message واحدة */
    u16 max_write_len;      /* حد أقصى للـ write payload */
    u16 max_read_len;       /* حد أقصى للـ read payload */
    u16 max_comb_1st_msg_len;
    u16 max_comb_2nd_msg_len;
};

/* مثال: controller بيدعم write-then-read بس */
static const struct i2c_adapter_quirks my_quirks = {
    .flags = I2C_AQ_COMB_WRITE_THEN_READ,
    .max_comb_1st_msg_len = 4,
    .max_comb_2nd_msg_len = 32,
};
```

الـ I2C core بيشيك على الـ quirks قبل ما ينفذ أي transfer — لو الـ driver طلب حاجة مش مدعومة، بيرجع error قبل ما يمس الـ hardware.

---

### ملخص — الـ Central Idea

**الـ I2C framework هو bus abstraction يفصل "ماذا" عن "كيف":**
- "**ماذا**" — الـ device driver عارف الـ register map للـ chip.
- "**كيف**" — الـ adapter driver عارف الـ hardware controller.

الـ framework نفسه بيعمل الـ **matching، locking، enumeration، و error handling** — بيخلي الـ device driver pure logic بدون hardware dependencies، وبيخلي الـ controller driver pure electrical/timing code بدون معرفة بالـ devices فوقيه.

الـ `I2C_TEN_BIT_ADDRESS` و `I2C_OWN_SLAVE_ADDRESS` في `dt-bindings/i2c/i2c.h` هما **contract بين الـ firmware description والـ kernel** — بيضمنوا إن الـ Device Tree والـ i2c-core بيفهموا نفس الـ address flags بدون أي drift.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### ملاحظة

**الـ** `include/dt-bindings/i2c/i2c.h` هو header بسيط جداً — بيحتوي على macro تعريفين بس ومفيش أي structs أو enums أو locking. الهدف منه إنه يُستخدم في كل من **Device Tree source files** (`.dts`) و **kernel C code** بدون تعارض.

---

### 0. الـ Macros — Cheatsheet

| الـ Macro | القيمة | الـ Bit | الاستخدام |
|---|---|---|---|
| `I2C_TEN_BIT_ADDRESS` | `0x80000000` | Bit 31 | بيحدد إن العنوان 10-bit وميس 7-bit |
| `I2C_OWN_SLAVE_ADDRESS` | `0x40000000` | Bit 30 | بيحدد إن العنوان هو عنوان الـ controller نفسه كـ slave |

```c
#define I2C_TEN_BIT_ADDRESS   (1 << 31)  /* flag: 10-bit addressing mode */
#define I2C_OWN_SLAVE_ADDRESS (1 << 30)  /* flag: controller acts as slave at this addr */
```

---

### 1. الـ Structs

**مفيش structs** في الملف ده. هو header خاص بـ **Device Tree bindings constants** فقط — بيُعرَّف في `dt-bindings/` عشان يكون مشترك بين:

- **Device Tree Compiler** (`dtc`) اللي بيقرا ملفات `.dts`
- **Kernel drivers** اللي بيستخدموا `#include <dt-bindings/i2c/i2c.h>`

---

### 2. Struct Relationship Diagram

```
لا يوجد structs — فقط constants مشتركة

  .dts file                        kernel driver / i2c core
  ─────────────────────            ──────────────────────────────
  #include <dt-bindings/           #include <dt-bindings/
            i2c/i2c.h>                       i2c/i2c.h>
       │                                          │
       ▼                                          ▼
  reg = <I2C_TEN_BIT_ADDRESS      if (addr & I2C_TEN_BIT_ADDRESS)
          | 0x123>;                   flags |= I2C_CLIENT_TEN;
```

---

### 3. Lifecycle Diagram

```
[Device Tree Source (.dts)]
        │
        │  reg = <I2C_TEN_BIT_ADDRESS | 0x050>;
        ▼
[DTC compiles → .dtb]
        │
        │  binary blob يُمرَّر للـ kernel أثناء boot
        ▼
[of_i2c_register_devices()]          ← kernel parses DT at boot
        │
        │  بيقرا الـ reg property
        │  بيشوف لو Bit 31 مضروب → 10-bit mode
        ▼
[i2c_client allocated]
        │  .addr  = 0x050
        │  .flags = I2C_CLIENT_TEN   ← مترجم من I2C_TEN_BIT_ADDRESS
        ▼
[driver probe() called]
        │
        ▼
[normal I2C transfers]
        │
        ▼
[device removed → i2c_unregister_device()]
```

---

### 4. Call Flow Diagram

```
DT property: reg = <(1<<31) | 0x123>
        │
        ▼
of_i2c_get_board_info()
        │  بيقرا الـ reg u32
        │  بيعمل check على Bit 31
        │    → info->flags |= I2C_CLIENT_TEN
        │  بيعمل check على Bit 30
        │    → info->flags |= I2C_CLIENT_SLAVE
        ▼
i2c_new_client_device(adap, &info)
        │
        ▼
[i2c_client created with correct flags]
        │
        ▼
driver->probe(client, id)
```

---

### 5. Locking Strategy

**مفيش أي locking** في الملف ده — هو preprocessor constants بس.

الـ locking بتاع الـ I2C subsystem نفسه (مش جزء من الملف ده) بيتم في:

| المورد | الـ Lock |
|---|---|
| `i2c_adapter.userspace_clients_lock` | mutex بيحمي قائمة الـ clients |
| `i2c_adapter.bus_lock` | mutex بيحمي الـ bus أثناء transfer |
| `i2c_adapter.mux_lock` | mutex خاص بالـ mux adapters |

---

### ملخص

**الـ** `dt-bindings/i2c/i2c.h` هو أبسط نوع من kernel headers — بيحتوي على **macro تعريفين فقط** يمثلان flag bits في الـ I2C address field الخاص بالـ Device Tree. قيمته الحقيقية في إنه **single source of truth** بين DT source files والـ kernel code، فبيمنع hardcoded magic numbers في الاتنين.
## Phase 4: شرح الـ Functions

> الملف `include/dt-bindings/i2c/i2c.h` مش بيحتوي على functions — هو **header خالص للـ macro constants** بس. الـ macros دي بتتستخدم في الـ Device Tree source files (`.dts`) وفي الـ kernel driver code بالتساوي. في الحالة دي، الـ "Phase 4" بيتحول لشرح شامل للـ constants وطريقة استخدامها لأنها هي الـ API الوحيد اللي الملف بيوفره.

---

### ملخص الـ Constants (Cheatsheet)

| الـ Macro | القيمة | الاستخدام |
|---|---|---|
| `I2C_TEN_BIT_ADDRESS` | `(1 << 31)` = `0x80000000` | يشير إن الـ I2C address هو 10-bit وليس 7-bit |
| `I2C_OWN_SLAVE_ADDRESS` | `(1 << 30)` = `0x40000000` | يشير إن الـ address هو للـ controller نفسه كـ slave |

---

### التصنيف: Address Flag Constants

الملف بيوفر **flag bits** بيتم OR-ing بيها مع الـ I2C address الأصلي عشان تحدد نوع الـ address في الـ Device Tree. الفكرة إن الـ `reg` property في الـ DT بتاخد `u32`، وعشان تميز بين الـ addressing modes من غير ما تكسر الـ ABI، بيتم استخدام الـ high bits اللي مش ممكن تبقى جزء من address حقيقي.

---

### `I2C_TEN_BIT_ADDRESS`

```c
#define I2C_TEN_BIT_ADDRESS   (1 << 31)
```

الـ macro ده بيمثل **bit 31** في الـ 32-bit address value. لما بتحط الـ flag ده في الـ `reg` property جنب الـ slave address، بتقول للـ I2C subsystem إن الـ device بيستخدم **10-bit addressing mode** بدل الـ standard 7-bit mode.

**الـ I2C standard بيدعم نوعين من الـ addresses:**
- **7-bit**: النطاق `0x00 – 0x7F` — الأشيع
- **10-bit**: النطاق `0x000 – 0x3FF` — للأجهزة اللي محتاجة address space أكبر

**مثال DTS:**

```dts
/* Device بـ 10-bit address 0x123 */
some_device: sensor@80000123 {
    compatible = "vendor,some-sensor";
    reg = <(I2C_TEN_BIT_ADDRESS | 0x123)>;
    /* يساوي reg = <0x80000123> */
};
```

**مثال kernel driver:**

```c
#include <dt-bindings/i2c/i2c.h>

/* لما بتقرأ الـ reg property وتحتاج تعرف الـ addressing mode */
u32 addr;
of_property_read_u32(node, "reg", &addr);

if (addr & I2C_TEN_BIT_ADDRESS) {
    /* 10-bit addressing */
    real_addr = addr & ~I2C_TEN_BIT_ADDRESS; /* mask out the flag */
    flags |= I2C_CLIENT_TEN;
}
```

**التفاصيل الداخلية:**
- الـ I2C core في `drivers/i2c/i2c-core-of.c` بيقرأ الـ flag ده من الـ DT وبيترجمه لـ `I2C_CLIENT_TEN` flag في الـ `i2c_board_info.flags`.
- الـ bit 31 اتاختار عمداً لأنه خارج نطاق أي I2C address حقيقي (حتى 10-bit addresses أقصاها `0x3FF` = 10 bits).
- الـ flag ده **read-only من ناحية الـ hardware** — مش بيتبعت على الـ bus، بيتستخدم فقط لتكوين الـ software.

---

### `I2C_OWN_SLAVE_ADDRESS`

```c
#define I2C_OWN_SLAVE_ADDRESS (1 << 30)
```

الـ macro ده بيمثل **bit 30**. بيستخدم لما الـ I2C controller نفسه بيشتغل كـ **slave** على الـ bus، وبتحتاج تحدد الـ address اللي هيرد عليه. ده مختلف عن addresses الـ devices المتوصلين بيه.

**السيناريو العملي:**

في بعض الأنظمة المدمجة (multi-master أو SMBus environments)، الـ SoC I2C controller ممكن يحتاج يبقى slave ويرد على address محدد — مثلاً في الـ ACPI `_DSM` methods أو في الـ embedded controller communication.

**مثال DTS:**

```dts
i2c_controller: i2c@40005400 {
    compatible = "vendor,i2c-controller";
    reg = <0x40005400 0x400>;
    #address-cells = <1>;
    #size-cells = <0>;

    /* الـ controller نفسه بيرد على slave address 0x08 */
    own-address = <(I2C_OWN_SLAVE_ADDRESS | 0x08)>;
    /* يساوي <0x40000008> */
};
```

**التفاصيل الداخلية:**
- الـ bit 30 اتاختار عشان يبقى distinct من `I2C_TEN_BIT_ADDRESS` (bit 31) ومن أي I2C address حقيقي.
- الـ kernel بيستخدمه في سياق الـ `i2c_slave_register()` API عشان يسجل الـ controller كـ slave.
- مش كل الـ I2C controllers بتدعم الـ slave mode — بيعتمد على الـ hardware capabilities والـ driver implementation.

---

### تفاصيل التصميم: ليه High Bits؟

```
 31  30  29 ... 10  9  8 ... 0
 |   |              |
 |   |              +-- 10-bit address space (0x000 to 0x3FF)
 |   +-- I2C_OWN_SLAVE_ADDRESS flag
 +-- I2C_TEN_BIT_ADDRESS flag

7-bit addresses: bits [6:0] فقط (0x00 to 0x7F)
10-bit addresses: bits [9:0] فقط (0x000 to 0x3FF)
```

الـ design decision ده بيخلي الـ DT source code **self-documenting** — لما حد يشوف:

```dts
reg = <0x80000055>;
```

هيعرف فوراً إنه 10-bit address `0x55`، بدل ما يحتاج documentation خارجي.

---

### الـ Header Guards

```c
#ifndef _DT_BINDINGS_I2C_I2C_H
#define _DT_BINDINGS_I2C_I2C_H
/* ... */
#endif
```

الـ include guard standard بيمنع double inclusion. الاسم بيتبع الـ convention الخاص بـ `dt-bindings` headers: `_DT_BINDINGS_<SUBSYSTEM>_<FILE>_H`.

---

### الـ Callers والـ Context

| السياق | الاستخدام |
|---|---|
| `.dts` / `.dtsi` files | مباشرة في الـ `reg` property |
| `drivers/i2c/i2c-core-of.c` | `of_i2c_get_board_info()` بتقرأ وتفسر الـ flags |
| `drivers/i2c/busses/*.c` | بعض الـ bus drivers بتقرأها لتكوين الـ slave mode |
| Userspace DTC compiler | الـ `dtc` tool بيفهم الـ macros عبر `-i` include path |
## Phase 5: دليل الـ Debugging الشامل

الـ file ده `include/dt-bindings/i2c/i2c.h` بيعرّف constant واحد بس اتنين بالظبط:

```c
#define I2C_TEN_BIT_ADDRESS   (1 << 31)  /* flag: device uses 10-bit I2C address */
#define I2C_OWN_SLAVE_ADDRESS (1 << 30)  /* flag: controller acts as slave on this addr */
```

الـ constants دول بيتعملوا OR مع الـ I2C address في الـ Device Tree عشان يحددوا نوع العنوان. الـ debugging هنا بيبقى على مستويين: التأكد إن الـ DT binding اتفسّر صح، والتأكد إن الـ I2C controller/device بيشتغل صح على المستوى الأدنى.

---

### Software Level

#### 1. debugfs Entries

الـ I2C subsystem بيكشف entries في `/sys/kernel/debug/i2c-*/`:

```bash
# list all I2C buses exposed via debugfs
ls /sys/kernel/debug/i2c-*/

# show all devices registered on bus 0
cat /sys/kernel/debug/i2c-0/i2c-dev
```

| Entry | المعنى |
|---|---|
| `/sys/kernel/debug/i2c-0/i2c-dev` | الـ devices المسجّلة على البص |
| `/sys/kernel/debug/regmap/i2c-*` | لو الـ device بيستخدم regmap، بيظهر register map |
| `/sys/kernel/debug/i2c-0/timings` | على بعض الـ controllers بتظهر timing parameters |

```bash
# regmap dump for a specific I2C device (e.g., codec on i2c-1 address 0x1a)
cat /sys/kernel/debug/regmap/1-001a/registers
```

#### 2. sysfs Entries

```bash
# list all I2C adapters
ls /sys/bus/i2c/devices/

# show device address (includes the flag bits from DT)
cat /sys/bus/i2c/devices/i2c-0/0-0050/name

# check if device is bound to a driver
ls /sys/bus/i2c/devices/0-0050/driver

# show 10-bit address devices — address will appear with 4 hex digits
ls /sys/bus/i2c/devices/ | grep -E '^[0-9]+-[0-9]{4}$'
```

**الـ I2C_TEN_BIT_ADDRESS flag** لو اتضبط في الـ DT، الـ kernel بيعمل mask عليه ويسجّل الـ device بـ 10-bit addressing mode — ظاهر في sysfs كـ `i2c-0/0-03ff` (4 أرقام بدل 2).

```bash
# verify 10-bit device appears correctly
cat /sys/bus/i2c/devices/0-0155/name
```

#### 3. ftrace — Tracepoints والـ Events

```bash
# enable I2C tracepoints
echo 1 > /sys/kernel/debug/tracing/events/i2c/enable

# trace only i2c_write and i2c_read events
echo 1 > /sys/kernel/debug/tracing/events/i2c/i2c_write/enable
echo 1 > /sys/kernel/debug/tracing/events/i2c/i2c_read/enable

# trace i2c_reply (ACK/NACK from slave)
echo 1 > /sys/kernel/debug/tracing/events/i2c/i2c_reply/enable

# filter on specific bus (adapter_nr == 0)
echo 'adapter_nr == 0' > /sys/kernel/debug/tracing/events/i2c/i2c_write/filter

# read the trace
cat /sys/kernel/debug/tracing/trace
```

مثال على الـ output:

```
i2c_write: i2c-0 #0 a=050 f=0000 l=1 [0x10]
i2c_reply: i2c-0 #0 a=050 f=0000 l=1 [0x10]
i2c_read:  i2c-0 #0 a=050 f=0001 l=4 [de ad be ef]
```

- `a=050` = الـ 7-bit address
- `a=155` = الـ 10-bit address (لو `I2C_TEN_BIT_ADDRESS` متضبط)
- `f=0000` = no flags / `f=0400` = I2C_M_TEN
- `l=` = طول الـ message

#### 4. printk و Dynamic Debug

```bash
# enable dynamic debug for i2c core
echo 'file drivers/i2c/i2c-core-base.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/i2c/i2c-core-of.c +p'   > /sys/kernel/debug/dynamic_debug/control

# enable for specific controller driver (مثلاً i2c-designware)
echo 'module i2c_designware_core +p' > /sys/kernel/debug/dynamic_debug/control

# enable ALL i2c dynamic debug messages
echo 'module i2c* +p' > /sys/kernel/debug/dynamic_debug/control

# watch kernel log in real time
dmesg -w | grep -i i2c
```

لو محتاج تشوف إزاي الـ `I2C_TEN_BIT_ADDRESS` flag بيتحول لـ `I2C_M_TEN` message flag:

```bash
echo 'file drivers/i2c/i2c-core-base.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
```

#### 5. Kernel Config Options للـ Debugging

| Config | الغرض |
|---|---|
| `CONFIG_I2C_DEBUG_CORE` | verbose logging في الـ i2c core |
| `CONFIG_I2C_DEBUG_BUS` | verbose logging في الـ bus drivers |
| `CONFIG_I2C_DEBUG_ALGO` | verbose في bit-banging algorithms |
| `CONFIG_DEBUG_I2C_STUB` | يفعّل stub adapter للـ unit testing |
| `CONFIG_I2C_CHARDEV` | يكشف `/dev/i2c-*` للـ userspace tools |
| `CONFIG_I2C_MUX` | لو بتستخدم I2C multiplexers |
| `CONFIG_OF` | لازم يبقى enabled عشان DT parsing يشتغل |
| `CONFIG_OF_DYNAMIC` | لـ runtime DT overlays debugging |
| `CONFIG_REGMAP_I2C` | لو الـ driver بيستخدم regmap فوق I2C |
| `CONFIG_TRACING` + `CONFIG_I2C_TRACE` | لتفعيل i2c tracepoints |

#### 6. i2c-tools — أدوات الـ Subsystem

```bash
# detect all I2C buses
i2cdetect -l

# scan bus 0 for all 7-bit devices (UU = used by driver)
i2cdetect -y 0

# scan for 10-bit addresses on bus 0
i2cdetect -y -a 0

# read register 0x00 from device 0x50 on bus 0
i2cget -y 0 0x50 0x00

# write value 0xFF to register 0x01
i2cset -y 0 0x50 0x01 0xFF

# dump all registers from 0x00 to 0xFF
i2cdump -y 0 0x50

# for 10-bit address device (address 0x155)
i2cget -y 0 0x155 0x00 b  # (بعض الـ tools بتدعمها)
```

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ kernel | المعنى | الحل |
|---|---|---|
| `i2c i2c-0: of_i2c: invalid reg on /soc/i2c@...` | الـ reg property في DT غلط | افحص إن الـ address في DT مش بيشمل الـ flag bits بغلط |
| `i2c i2c-0: Failed to register i2c client` | فشل تسجيل الـ device | افحص duplicate address أو خطأ في الـ flags |
| `i2c i2c-0: 0-0050: can't claim addr` | address محجوز بالفعل | conflict في الـ DT أو driver تاني حجز العنوان |
| `i2c i2c-0: of_i2c: modalias failure on /...` | فشل توليد الـ modalias | افحص `compatible` string في الـ DT node |
| `i2c i2c-0: adapter quirk prevented` | الـ adapter مش بيدعم 10-bit | controller مش بيدعم `I2C_FUNC_10BIT_ADDR`، افحص `i2c_check_functionality()` |
| `probe of i2c failed with error -5 (EIO)` | الـ device مش راد على البص | افحص الـ hardware — power, pull-ups, address |
| `i2c i2c-0: 0-0050: probe failed` | driver probe فشل | افحص الـ driver logs بـ dynamic debug |
| `of_i2c: unknown I2C address` | address مش valid | الـ address في DT أكبر من 0x3ff أو 0x7f |

#### 8. Strategic Points لـ dump_stack() و WARN_ON()

النقاط الاستراتيجية لو بتكتب driver يستخدم الـ constants دي:

```c
/* في دالة probe — تحقق إن الـ address اتحمل صح من DT */
static int my_probe(struct i2c_client *client)
{
    /* الـ address الفعلي موجود في client->addr */
    /* الـ flags موجودة في client->flags */

    WARN_ON(client->addr == 0);

    /* لو بتتوقع 10-bit device */
    if (!(client->flags & I2C_M_TEN)) {
        dev_warn(&client->dev, "expected 10-bit addressing, got 7-bit\n");
        WARN_ON(1);
    }

    /* تحقق إن الـ adapter بيدعم 10-bit لو محتاجه */
    if (!i2c_check_functionality(client->adapter, I2C_FUNC_10BIT_ADDR)) {
        dev_err(&client->dev, "adapter doesn't support 10-bit addresses\n");
        return -EINVAL;
    }
}

/* لو بتبني messages يدويًا */
static int my_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num)
{
    int i;
    for (i = 0; i < num; i++) {
        /* تحقق إن 10-bit flag متضبط صح */
        if (msgs[i].addr > 0x7f)
            WARN_ON(!(msgs[i].flags & I2C_M_TEN));
    }
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

```bash
# تحقق إن الـ device موجود على البص فعلاً
i2cdetect -y 0

# لو الـ device مش ظاهر في scan لكن الـ driver مسجّله
# ممكن يكون address مش صح في DT
# افحص الـ address المسجّل
cat /sys/bus/i2c/devices/0-0050/name

# تحقق إن الـ flags اتحطت صح (I2C_M_TEN = 0x0010)
# لو الـ device 10-bit، المفروض يظهر كـ 4-digit address في sysfs
ls /sys/bus/i2c/devices/ | grep '^0-'
```

**الـ I2C_TEN_BIT_ADDRESS** في الـ DT بيبقى `(1 << 31)` = `0x80000000`. الـ kernel في `i2c-core-of.c` بيعمل:
```c
if (addr & I2C_TEN_BIT_ADDRESS) {
    addr &= ~I2C_TEN_BIT_ADDRESS;
    info.flags |= I2C_CLIENT_TEN;  /* يتحول لـ I2C_M_TEN في الـ message */
}
```
يعني لو شفت في sysfs address بـ 4 أرقام — ده تأكيد الـ flag اتفسّر صح.

#### 2. Register Dump — devmem2 و /dev/mem

```bash
# لو عايز تقرأ registers الـ I2C controller نفسه (مثلاً i2c-designware على عنوان 0xFF020000)
# اعرف عنوان الـ controller من DT أو datasheet
devmem2 0xFF020000 w   # قراءة IC_CON register
devmem2 0xFF020004 w   # قراءة IC_TAR (target address register)
devmem2 0xFF020024 w   # قراءة IC_STATUS

# أو عن طريق /dev/mem (لازم CONFIG_STRICT_DEVMEM=n)
dd if=/dev/mem bs=4 count=1 skip=$((0xFF020000/4)) 2>/dev/null | xxd
```

**الـ IC_TAR register** (في i2c-designware مثلاً) بيحتوي على:
- bits [9:0] = الـ target address
- bit [12] = `IC_10BITADDR_MASTER` — لازم يتضبط لو `I2C_TEN_BIT_ADDRESS` موجود في DT

```bash
# طريقة أسهل: io utility
io -4 -r 0xFF020004  # قراءة IC_TAR
```

#### 3. Logic Analyzer و Oscilloscope

**نقاط القياس:**
- **SCL** (Serial Clock Line) — عادةً pin مكتوب عليه SCL في الـ schematic
- **SDA** (Serial Data Line) — عادةً pin مكتوب عليه SDA

**ما تبص عليه:**

```
7-bit START sequence:
S [A6 A5 A4 A3 A2 A1 A0 R/W] ACK ...

10-bit START sequence:
S [1 1 1 1 0 A9 A8 0] ACK [A7 A6 A5 A4 A3 A2 A1 A0] ACK ...
```

لو `I2C_TEN_BIT_ADDRESS` متضبط، المفروض تشوف الـ 10-bit address format ده:
- أول byte = `0xF0 | (addr >> 8)` — الـ 2 MSBs من الـ address
- تاني byte = الـ 8 LSBs من الـ address

**إعدادات الـ Logic Analyzer:**
- Sample rate: مينيمم 4× أعلى I2C speed (400kHz → 2MHz sampling)
- Trigger: falling edge على SDA مع SCL = HIGH (START condition)
- Protocol decoder: I2C مع تفعيل "10-bit address" mode

**إعدادات الـ Oscilloscope:**
- Rise time للـ SCL و SDA لازم تكون < 300ns في Standard Mode
- لو Rise time بطيئة → pull-up resistors كبيرة أوي أو capacitance عالية
- VOL لازم < 0.4V، VOH لازم > 0.7 × VCC

#### 4. Hardware Issues الشائعة وـ Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log |
|---|---|
| Pull-up resistors مش موجودة أو كبيرة | `i2c_smbus_xfer: read failed (timeout)` — الـ bus مش بيرجع HIGH |
| الـ Device مش موجود أو مش powered | `0-0050: probe: no such device (ENXIO)` |
| Address خاطئ في DT (10-bit بدل 7-bit) | Device بيظهر في sysfs بـ address غلط، لا يرد |
| الـ controller مش بيدعم 10-bit | `adapter quirk prevented read` + `-EOPNOTSUPP` |
| Bus stuck LOW (SDA أو SCL عالق) | `i2c_designware: i2c timeout, ipd: 0x00000001` |
| Clock stretching timeout | `i2c i2c-0: controller timed out` |
| Wrong voltage level | ACK بيجي ناقص أو مش واضح على الـ scope |

#### 5. Device Tree Debugging

```bash
# تحقق إن الـ DT node اتحمل صح
cat /proc/device-tree/soc/i2c@ff020000/device@50/reg | xxd
# المفروض: 00 00 00 50  (7-bit)
# أو:      80 00 01 55  (10-bit: I2C_TEN_BIT_ADDRESS | 0x155)

# قراءة كل الـ properties للـ I2C node
ls /proc/device-tree/soc/i2c@ff020000/
cat /proc/device-tree/soc/i2c@ff020000/device@155/reg | xxd

# تحقق باستخدام dtc (device tree compiler)
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 10 "i2c@ff020000"

# فحص compiled DT binary
dtc -I dtb -O dts /boot/dtb/your-board.dtb | grep -A 5 "i2c"

# لو بتستخدم DT overlay، تحقق إنه اتطبق
cat /proc/device-tree/soc/i2c@ff020000/__overlay__/reg | xxd
```

**مثال DT صح لـ 10-bit device:**

```dts
&i2c0 {
    /* device with 10-bit address 0x155 */
    sensor@80000155 {   /* reg = I2C_TEN_BIT_ADDRESS | 0x155 */
        compatible = "vendor,my-sensor";
        reg = <(0x80000000 | 0x155)>;  /* I2C_TEN_BIT_ADDRESS = bit 31 */
    };
};
```

في `/proc/device-tree`، الـ reg المفروض تبقى `80 00 01 55`.

**مثال DT صح لـ own slave address:**

```dts
&i2c0 {
    /* controller acts as slave on address 0x10 */
    reg = <(0x40000000 | 0x10)>;  /* I2C_OWN_SLAVE_ADDRESS = bit 30 */
};
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

```bash
#!/bin/bash
# === I2C DT Bindings Debug Script ===

BUS=0
ADDR=0x50  # غيّر للـ address بتاعتك

echo "=== I2C Bus Scan ==="
i2cdetect -y $BUS

echo "=== I2C Devices in sysfs ==="
ls /sys/bus/i2c/devices/ | grep "^${BUS}-"

echo "=== Check for 10-bit devices ==="
ls /sys/bus/i2c/devices/ | grep -E "^${BUS}-[0-9a-f]{4}$" || echo "No 10-bit devices found"

echo "=== Enable I2C Tracing ==="
echo 1 > /sys/kernel/debug/tracing/events/i2c/enable
echo "" > /sys/kernel/debug/tracing/trace  # clear trace

echo "=== Enable Dynamic Debug ==="
echo 'file drivers/i2c/i2c-core-of.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/i2c/i2c-core-base.c +p' > /sys/kernel/debug/dynamic_debug/control

echo "=== Trigger some I2C activity ==="
i2cdetect -y $BUS > /dev/null

echo "=== Trace Output ==="
cat /sys/kernel/debug/tracing/trace | head -50

echo "=== DT reg property for device ==="
# استبدل المسار بمسار الـ device الفعلي
find /proc/device-tree -name "reg" -path "*i2c*" -exec sh -c \
    'echo "File: $1"; xxd $1' _ {} \;

echo "=== I2C functionality check ==="
# يوضح الـ capabilities للـ adapter
python3 -c "
import fcntl, struct
I2C_FUNCS = 0x0705
with open('/dev/i2c-$BUS', 'rb') as f:
    funcs = struct.unpack('I', fcntl.ioctl(f, I2C_FUNCS, b'\x00'*4))[0]
print(f'I2C_FUNC_10BIT_ADDR: {bool(funcs & 0x0002)}')
print(f'I2C_FUNC_SMBUS_READ_BYTE: {bool(funcs & 0x0800)}')
print(f'Capabilities: 0x{funcs:08x}')
"
```

```bash
# تشغيل لحظي مع مراقبة الـ log
watch -n 1 'dmesg | tail -20 | grep -i i2c'

# تتبع كل I2C transactions مع timestamps
echo 'global' > /sys/kernel/debug/tracing/trace_clock
cat /sys/kernel/debug/tracing/trace_pipe &
TRACE_PID=$!
i2cdump -y 0 0x50
kill $TRACE_PID

# فحص الـ address parsing في kernel للـ 10-bit
# لو DT فيه reg = <0x80000155>، الـ kernel لازم يسجّل device بـ address 0x155
dmesg | grep -E "i2c.*155|10.bit|TEN"
```

#### تفسير الـ Output

```
# مثال ftrace output لـ 10-bit device:
i2c_write: i2c-0 #0 a=155 f=0010 l=1 [0x00]
#                       ^^^  ^^^^
#                       |    f=0010 = I2C_M_TEN flag — ده تأكيد إن I2C_TEN_BIT_ADDRESS اتفسّر صح
#                       address = 0x155 (10-bit)

# مثال i2cdetect output:
#      0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
# 00:          -- -- -- -- -- -- -- -- -- -- -- -- --
# 50: UU  ...
# UU = device موجود وـ driver بيستخدمه
# -- = no device
# 50 = device موجود بس مفيش driver
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على AM62x — جهاز I2C بـ 10-bit address مش بيتعرف

#### العنوان
**الـ EEPROM مش شغال على AM62x Industrial Gateway**

#### السياق
شركة بتعمل industrial gateway بيستخدم TI AM62x SoC. الـ gateway عنده EEPROM خارجي نوعه AT24C512 بيتستخدم لتخزين إعدادات الشبكة. الـ EEPROM ده عنده **10-bit I2C address** وهو `0x158`. المهندس عامل الـ Device Tree بس الـ driver مش بيلاقي الجهاز.

#### المشكلة
الـ DTS فيه:
```dts
&i2c1 {
    eeprom@158 {
        compatible = "atmel,24c512";
        reg = <0x158>;   /* 10-bit address لكن من غير flag */
    };
};
```
الـ kernel بيطلع في dmesg:
```
i2c i2c-1: Failed to register I2C device: -EINVAL
```
الجهاز مش اتسجل خالص.

#### التحليل
الـ I2C subsystem في الـ kernel بيشوف الـ `reg` property. لو القيمة أكبر من `0x77` (127) من غير الـ flag `I2C_TEN_BIT_ADDRESS`، الـ kernel بيعتبرها invalid لأن الـ standard 7-bit addressing بتاخد قيم من `0x00` لـ `0x77` بس.

الـ flag المعرف في الملف:
```c
#define I2C_TEN_BIT_ADDRESS   (1 << 31)  /* bit 31 = 0x80000000 */
```

الـ kernel بيعمل bitwise OR للـ flag مع الـ address عشان يميّز الـ 10-bit addresses. لما المهندس كتب `<0x158>` من غير الـ flag، الـ core فضل بيتعامل معاه كـ 7-bit address وقيمته أكبر من المسموح.

#### الحل
**تعديل الـ DTS:**
```dts
#include <dt-bindings/i2c/i2c.h>

&i2c1 {
    eeprom@158 {
        compatible = "atmel,24c512";
        /* I2C_TEN_BIT_ADDRESS | 0x158 = 0x80000158 */
        reg = <(I2C_TEN_BIT_ADDRESS | 0x158)>;
    };
};
```

**التحقق بعد الـ compile:**
```bash
# نشوف الـ node اتسجل صح
ls /sys/bus/i2c/devices/ | grep 1-0158

# نقرأ من الـ EEPROM
i2ctransfer -y 1 w2@0x158 0x00 0x00 r8
```

#### الدرس المستفاد
الـ `I2C_TEN_BIT_ADDRESS` flag مش optional — ده **required** لأي جهاز بـ address أكبر من `0x77`. من غيره الـ kernel core يرفض الـ registration تماماً.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — تعارض في الـ slave address

#### العنوان
**الـ HDMI audio codec بيتعارض مع الـ touch controller**

#### السياق
مهندس بيعمل Android TV box على Allwinner H616. الـ board عندها:
- HDMI audio codec: ES8316 بـ address `0x11`
- Capacitive touch controller: FT5426 بـ address `0x38`

بعد إضافة feature جديدة، المهندس لاحظ إن الـ ES8316 بدأ يطلع أخطاء I2C errors بشكل متقطع.

#### المشكلة
بعد مراجعة الـ DTS، المهندس لقى إن زميله حاول يضيف second I2C slave interface للـ ES8316 باستخدام `I2C_OWN_SLAVE_ADDRESS` بطريقة غلط:

```dts
/* غلط — المهندس الثاني حاول يعمل كده */
&i2c2 {
    codec@11 {
        compatible = "everest,es8316";
        reg = <0x11>;
    };

    codec-slave@11 {
        compatible = "everest,es8316";
        /* I2C_OWN_SLAVE_ADDRESS بدون تغيير في الـ address */
        reg = <(I2C_OWN_SLAVE_ADDRESS | 0x11)>;
    };
};
```

الـ kernel حاول يسجل جهازين على نفس الـ physical address، وده أدى لـ bus contention.

#### التحليل
الـ `I2C_OWN_SLAVE_ADDRESS` flag:
```c
#define I2C_OWN_SLAVE_ADDRESS  (1 << 30)  /* bit 30 = 0x40000000 */
```

الـ flag ده مش بيغير الـ physical address على الـ bus — هو بس بيقول للـ I2C controller "اعتبر نفسك slave بـ address ده". الـ use case الصح هو لما الـ SoC نفسه بيعمل كـ I2C slave (مش master) على bus خارجي، مش لتسجيل جهازين على نفس الـ address.

الـ ES8316 على `0x11` والـ node التاني بـ `(0x40000000 | 0x11)` = `0x40000011` — الـ kernel سجلهم كـ منفصلين، بس في الحقيقة الـ physical wire واحدة وفي كل scan بيبعت transactions للـ address الغلط.

#### الحل
**حذف الـ node الغلط والرجوع للـ DTS الأصلي:**
```dts
#include <dt-bindings/i2c/i2c.h>

&i2c2 {
    codec@11 {
        compatible = "everest,es8316";
        reg = <0x11>;  /* 7-bit address عادي */
        /* الـ slave functionality بتتحكم فيها من الـ driver مش من DT */
    };
};
```

**لو الـ SoC نفسه محتاج يبقى slave:**
```dts
/* ده الاستخدام الصح لـ I2C_OWN_SLAVE_ADDRESS */
&i2c0 {
    /* الـ SoC نفسه هيجاوب على address 0x50 كـ slave */
    reg = <(I2C_OWN_SLAVE_ADDRESS | 0x50)>;
};
```

#### الدرس المستفاد
الـ `I2C_OWN_SLAVE_ADDRESS` مش للـ external devices — ده للـ controller نفسه لما يشتغل كـ slave. غلط استخدامه مع external device بيعمل phantom registrations وبيكسر الـ bus.

---

### السيناريو الثالث: IoT Sensor Board على STM32MP1 — bring-up جديد والـ sensors مش بتظهر

#### العنوان
**3 sensors على I2C bus وواحد بس شغال على STM32MP1**

#### السياق
مهندس IoT بيعمل custom sensor board على STM32MP157. الـ board عندها:
- BME280 (temp/humidity): address `0x76`
- MPU6050 (IMU): address `0x68`
- MLX90614 (IR thermometer): address `0x5A`

بعد الـ boot، بس الـ BME280 ظهر في `/sys/bus/i2c/devices/`.

#### المشكلة
```bash
$ i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- 5b -- -- -- --
60: -- -- -- -- -- -- -- -- 68 -- -- -- -- -- -- --
70: -- -- -- -- -- -- 76 --
```
الـ hardware كله موجود، بس الـ DT nodes مش اتسجلوا.

#### التحليل
المهندس راجع الـ DTS ولقى:
```dts
/* النسخة الغلط */
&i2c1 {
    bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;
    };
    /* المهندس نسي يضيف الـ nodes دي */
};
```

المشكلة مش في الـ `i2c.h` constants نفسها — الـ addresses كلها standard 7-bit من غير flags. المشكلة إن الـ MPU6050 والـ MLX90614 محتاجين nodes في الـ DTS.

المهندس عمل `i2cdetect` ولقى إن الـ MLX90614 بيظهر على `0x5B` مش `0x5A` — ده لأن الـ pull-up resistor على الـ address pin غلط (hardware bug).

#### الحل
**تصحيح الـ DTS مع الـ addresses الصح:**
```dts
#include <dt-bindings/i2c/i2c.h>

&i2c1 {
    clock-frequency = <400000>;  /* Fast mode */

    bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;
    };

    mpu6050@68 {
        compatible = "invensense,mpu6050";
        reg = <0x68>;
        interrupt-parent = <&gpio4>;
        interrupts = <12 IRQ_TYPE_EDGE_RISING>;
    };

    /* address 0x5b بعد تصحيح الـ hardware */
    mlx90614@5b {
        compatible = "melexis,mlx90614";
        reg = <0x5b>;
    };
};
```

**بعد الـ fix:**
```bash
# التحقق من الـ registration
ls /sys/bus/i2c/devices/
# 1-0068  1-005b  1-0076

# قراءة temperature من MLX90614
cat /sys/bus/iio/devices/iio\:device0/in_temp_object_raw
```

#### الدرس المستفاد
الـ `i2c.h` constants (الـ flags) بتتحرك بس لما تكون addresses غير standard. للـ 7-bit addresses العادية، الـ DT node نفسه لازم يكون موجود بالـ address الفعلي من `i2cdetect` — مش من الـ datasheet بس لأن hardware strapping ممكن يغير الـ address.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — الـ ADAS camera I2C مش بيتعرف

#### العنوان
**Camera serializer بـ 10-bit I2C address على i.MX8QM ECU**

#### السياق
شركة automotive بتعمل ADAS ECU على NXP i.MX8QM. الـ camera system بيستخدم MAX9295A camera serializer بيتكلم مع الـ SoC عبر I2C. الـ MAX9295A بيستخدم **10-bit I2C address** = `0x200` (مش standard). الـ production DTS اتعمل من غير الـ flag وكل الـ camera modules فشلوا في الـ field.

#### المشكلة
```
[    2.341582] max9295 2-0200: probe failed: -ENODEV
[    2.341601] i2c i2c-2: Failed to register device: Invalid argument
```

الـ kernel يرفض يسجل الـ device لأن `0x200` أكبر من `0x3FF` مش... لا، في الحقيقة `0x200` = 512 وده valid 10-bit address (range 0x000 - 0x3FF) بس الـ kernel core شايفه كـ invalid 7-bit address.

#### التحليل
```c
/* من include/dt-bindings/i2c/i2c.h */
#define I2C_TEN_BIT_ADDRESS   (1 << 31)
```

الـ I2C core في الـ kernel عند parse الـ `reg` property بيعمل:
```c
/* من drivers/i2c/i2c-core-of.c (مبسط) */
if (addr & I2C_TEN_BIT_ADDRESS) {
    addr &= ~I2C_TEN_BIT_ADDRESS;
    /* validate: 0x000 - 0x3FF */
    if (addr > 0x3FF)
        return -EINVAL;
    info->flags |= I2C_CLIENT_TEN;
} else {
    /* validate: 0x08 - 0x77 */
    if (addr < 0x08 || addr > 0x77)
        return -EINVAL;
}
```

من غير الـ flag، الـ core حاول يـ validate الـ `0x200` كـ 7-bit address وفشل.

#### الحل
**الـ DTS المصحح:**
```dts
#include <dt-bindings/i2c/i2c.h>

&i2c2 {
    clock-frequency = <100000>;

    /* MAX9295A camera serializer — 10-bit address */
    max9295a@200 {
        compatible = "maxim,max9295a";
        /* الـ flag ضروري: 0x80000200 */
        reg = <(I2C_TEN_BIT_ADDRESS | 0x200)>;

        /* camera port config */
        port {
            max9295a_out: endpoint {
                remote-endpoint = <&mipi_csi0_in>;
            };
        };
    };
};
```

**التحقق:**
```bash
# نشوف الـ device اتسجل
ls /sys/bus/i2c/devices/ | grep "2-0200"

# نقرأ device ID register
i2ctransfer -y 2 w1@0x200 0x0D r1
# المفروض يرجع 0x91 لـ MAX9295A
```

#### الدرس المستفاد
في الـ automotive وأي system بيستخدم camera serializers/deserializers (كـ MAX9295/MAX9296, DS90UB953)، الـ 10-bit I2C addresses شائعة جداً. **دايماً** راجع الـ datasheet للـ address width قبل كتابة الـ DT، وخصوصاً في production systems لأن الـ bug ممكن يوصل لـ field failures.

---

### السيناريو الخامس: Custom Board Bring-up على RK3562 — الـ PMIC مش بيشتغل كـ I2C slave

#### العنوان
**RK809 PMIC على RK3562 — الـ OWN_SLAVE_ADDRESS مفهوم غلط**

#### السياق
مهندس جديد بيعمل bring-up لـ custom tablet board على Rockchip RK3562. الـ board عندها RK809 PMIC متوصل بـ I2C. المهندس قرأ في datasheet إن الـ RK809 بيدعم "slave mode للـ power sequencing" وفهم غلط إن ده معناه إنه لازم يستخدم `I2C_OWN_SLAVE_ADDRESS` في الـ DT.

#### المشكلة
المهندس كتب:
```dts
/* غلط تماماً */
&i2c0 {
    rk809@20 {
        compatible = "rockchip,rk809";
        /* المهندس فهم غلط — استخدم I2C_OWN_SLAVE_ADDRESS */
        reg = <(I2C_OWN_SLAVE_ADDRESS | 0x20)>;
        /* ... */
    };
};
```

النتيجة: الـ PMIC driver فشل في الـ probe ومفيش power rails اتعمل، والـ board مفضلش يـ boot.

```
[    0.892341] rk808 0-0020: Failed to get irq chip data
[    1.023451] rk808: probe of 0-0020 failed with error -22
```

#### التحليل
الـ flag:
```c
#define I2C_OWN_SLAVE_ADDRESS  (1 << 30)  /* 0x40000000 */
```

الـ `I2C_OWN_SLAVE_ADDRESS` معناه: "الـ I2C **controller** (الـ SoC نفسه) هيشتغل كـ slave على الـ address ده". ده بيُستخدم في scenarios زي:
- الـ SoC بيتكلم مع host processor تاني عبر I2C
- الـ SoC بيستقبل أوامر من external master

لكن الـ RK809 ده external PMIC والـ SoC هو الـ **master**. الـ "slave mode" في الـ datasheet ده feature في الـ PMIC نفسه (بيستقبل commands)، مش في الـ DT binding.

الـ `(I2C_OWN_SLAVE_ADDRESS | 0x20)` = `0x40000020` — الـ kernel حاول يسجل الـ I2C controller نفسه كـ slave على address `0x20`، مش يـ probe الـ PMIC.

#### الحل
**الـ DTS الصح:**
```dts
#include <dt-bindings/i2c/i2c.h>
/* ملاحظة: مش محتاجين الـ header هنا لأن مش بنستخدم أي flag */

&i2c0 {
    clock-frequency = <400000>;

    rk809: pmic@20 {
        compatible = "rockchip,rk809";
        reg = <0x20>;  /* 7-bit address عادي — مفيش flags */
        interrupt-parent = <&gpio0>;
        interrupts = <3 IRQ_TYPE_LEVEL_LOW>;
        #clock-cells = <1>;
        /* power domains config */
        regulators {
            vdd_cpu: DCDC_REG1 {
                regulator-min-microvolt = <712500>;
                regulator-max-microvolt = <1390000>;
                regulator-always-on;
            };
        };
    };
};
```

**Debug commands عشان نتأكد:**
```bash
# نشوف الـ PMIC اتسجل صح
ls /sys/bus/i2c/devices/0-0020/

# نشوف الـ regulators
ls /sys/class/regulator/ | grep rk809

# نقرأ voltage
cat /sys/class/regulator/regulator.0/microvolts
```

#### الدرس المستفاد
الـ `I2C_OWN_SLAVE_ADDRESS` flag نادر الاستخدام في الـ DT. الـ 99% من الـ devices (PMICs, codecs, sensors) محتاجة 7-bit address عادي أو 10-bit مع `I2C_TEN_BIT_ADDRESS`. لو قرأت "slave" في الـ datasheet، يعني الـ device بيستجيب لـ I2C commands (يعني هو slave على الـ bus) — وده الوضع الطبيعي لكل I2C device ومش محتاج flag.
## Phase 7: مصادر ومراجع

### توثيق الـ kernel الرسمي

| المستند | الوصف |
|---------|-------|
| `Documentation/devicetree/bindings/i2c/i2c.txt` | الـ binding الرئيسي للـ I2C في Device Tree، بيشرح `reg` flags زي `I2C_TEN_BIT_ADDRESS` |
| `Documentation/i2c/` | مجلد التوثيق الكامل للـ I2C subsystem |
| `Documentation/i2c/ten-bit-addresses.rst` | شرح تفصيلي لعناوين الـ 10-bit وازاي بيتعمل mapping ليها |
| `Documentation/i2c/slave-interface.rst` | توثيق الـ I2C slave mode وإزاي Linux ممكن يشتغل كـ slave |
| `Documentation/i2c/instantiating-devices.rst` | طرق إنشاء الـ I2C devices، بما فيها الـ Device Tree |
| `Documentation/i2c/writing-clients.rst` | دليل كتابة الـ I2C device drivers |

**الـ source file** الأساسي اللي اتكلمنا عنه:

```
include/dt-bindings/i2c/i2c.h
```

بيعرّف constant واحدين بس لكنهم محوريين:

```c
#define I2C_TEN_BIT_ADDRESS   (1 << 31)  /* 10-bit address flag */
#define I2C_OWN_SLAVE_ADDRESS (1 << 30)  /* slave mode flag     */
```

---

### مقالات LWN.net

دي أهم المقالات على [lwn.net](https://lwn.net) المتعلقة بالموضوع ده:

1. **[i2c: slave support framework for Linux devices](https://lwn.net/Articles/611332/)**
   - مقال بيشرح أول implementation للـ I2C slave support في Linux
   - ذو صلة مباشرة بـ `I2C_OWN_SLAVE_ADDRESS`

2. **[Documentation/i2c/slave-interface (v4.1)](https://lwn.net/Articles/640346/)**
   - توثيق الـ slave interface بعد دخوله الـ kernel في v4.1
   - بيشرح الـ events والـ callbacks الخاصة بالـ slave mode

3. **[mux controller abstraction and iio/i2c muxes](https://lwn.net/Articles/713971/)**
   - نقاش حول الـ I2C mux abstraction وتأثيره على الـ DT bindings

4. **[i2c-atr and FPDLink](https://lwn.net/Articles/920347/)**
   - الـ I2C Address Translator — مثال عملي على تعقيدات عناوين الـ I2C في الـ Device Tree

5. **[Documentation/devicetree/bindings/i2c/i2c-demux-pinctrl.txt (v4.6)](https://lwn.net/Articles/681046/)**
   - binding لـ I2C bus demultiplexer عن طريق pin multiplexing

6. **[iProc I2C slave mode and NIC mode](https://lwn.net/Articles/784812/)**
   - مثال تطبيقي على I2C slave mode في controller حقيقي

7. **[i2c: designware: add I2C SLAVE support](https://lwn.net/Articles/722129/)**
   - إضافة الـ slave support لأشهر I2C controller IP في Linux

---

### توثيق LWN.net للـ kernel documentation

- **[I2C and SMBus Subsystem — Driver API](https://static.lwn.net/kerneldoc/driver-api/i2c.html)**
- **[I2C/SMBus Subsystem Index](https://static.lwn.net/kerneldoc/i2c/index.html)**
- **[Implementing I2C device drivers](https://static.lwn.net/kerneldoc/i2c/writing-clients.html)**
- **[I2C muxes and complex topologies](https://static.lwn.net/kerneldoc/i2c/i2c-topology.html)**

---

### التوثيق الرسمي على docs.kernel.org

- **[I2C/SMBus Subsystem](https://docs.kernel.org/i2c/index.html)** — نقطة البداية
- **[I2C Ten-bit Addresses](https://docs.kernel.org/i2c/ten-bit-addresses.html)** — شرح `I2C_TEN_BIT_ADDRESS` بالتفصيل
- **[How to instantiate I2C devices](https://docs.kernel.org/i2c/instantiating-devices.html)** — إزاي الـ Device Tree بيعمل instantiate للـ I2C clients
- **[I2C slave testunit backend](https://docs.kernel.org/i2c/slave-testunit-backend.html)** — اختبار الـ slave mode

---

### نقاشات الـ mailing list

- **[PATCH: i2c: regroup documentation of bindings](https://www.spinics.net/lists/devicetree/msg343770.html)**
  — Wolfram Sang بيعيد تنظيم الـ I2C DT binding docs، ذو صلة مباشرة بالـ header ده

- **[PATCH v2: check: Add 10bit/slave i2c reg flags support](https://lkml.iu.edu/hypermail/linux/kernel/2005.3/04249.html)**
  — patch بيضيف دعم لفحص الـ `I2C_TEN_BIT_ADDRESS` و`I2C_OWN_SLAVE_ADDRESS` في الـ DT compiler

- **[check: Add 10bit/slave i2c reg flags support — Patchwork](https://patches.linaro.org/project/linux-i2c/patch/20200527122525.6929-1-Sergey.Semin@baikalelectronics.ru/)**
  — نفس الـ patch على Linaro Patchwork

---

### مصادر eLinux.org

- **[Tests: I2C-fault-injection](https://elinux.org/Tests:I2C-fault-injection)**
  — اختبار سلوك الـ I2C core عند حدوث errors، مفيد لفهم الـ error handling

- **[Tests: I2C-core-DMA](https://elinux.org/Tests:I2C-core-DMA)**
  — اختبار الـ DMA buffer handling في الـ I2C core

- **[Tests: RCar-Gen2-I2C-demux](https://elinux.org/Tests:RCar-Gen2-I2C-demux)**
  — اختبار الـ I2C demultiplexer على R-Car Gen2، مثال على استخدام الـ DT bindings

- **[EBC I2C](https://elinux.org/EBC_I2C)**
  — مدخل عملي للـ I2C على BeagleBone

---

### كومِتات الـ kernel المهمة

**الـ header ده اتعمل في Linux 3.19** كجزء من جهد Wolfram Sang لتوحيد الـ I2C DT bindings.

ممكن تتتبع التاريخ كامل من:

```bash
# تاريخ الـ file
git log --follow -- include/dt-bindings/i2c/i2c.h

# من وِيْن جه الـ I2C_OWN_SLAVE_ADDRESS
git log --all --oneline --grep="I2C_OWN_SLAVE_ADDRESS"

# الـ commit الأصلي للـ slave framework
git log --all --oneline --grep="i2c slave"
```

**الـ core file** اللي بيستخدم الـ constants دي:

```
drivers/i2c/i2c-core-of.c
```

```c
/* من i2c-core-of.c — إزاي بيتعمل parse للـ flags */
if (addr & I2C_TEN_BIT_ADDRESS) {
    addr &= ~I2C_TEN_BIT_ADDRESS;
    info.flags |= I2C_CLIENT_TEN;
}
if (addr & I2C_OWN_SLAVE_ADDRESS) {
    addr &= ~I2C_OWN_SLAVE_ADDRESS;
    info.flags |= I2C_CLIENT_SLAVE;
}
```

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)

- **الفصل 8**: I2C/SMBus subsystem — شرح الـ `i2c_driver`, `i2c_client`, `i2c_adapter`
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)

- **الفصل 14**: The Block I/O Layer — ومنه تفهم الـ bus abstraction model
- **الفصل 17**: Devices and Modules — الـ device model اللي I2C بيبني عليه

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- **الفصل 15**: Device Drivers — شرح الـ I2C من منظور embedded
- أمثلة عملية على الـ Device Tree وربطه بالـ I2C drivers

#### Professional Linux Kernel Architecture — Wolfgang Mauerer

- بيغطي الـ driver model بعمق أكبر من LDD3

---

### kernelnewbies.org

- **[Linux 3.19 — DriverArch](https://kernelnewbies.org/Linux_3.19-DriversArch)**
  — بيذكر إضافة `dt-bindings/i2c/i2c.h` ضمن تغييرات الـ I2C في الـ release ده

- **[Linux 3.8 — DriverArch](https://kernelnewbies.org/Linux_3.8_DriverArch)**
  — مراجعة لتغييرات الـ I2C driver architecture في تلك المرحلة

- **[Linux 6.8](https://kernelnewbies.org/Linux_6.8)**
  — آخر التغييرات في الـ I2C subsystem

---

### search terms للبحث عن مزيد

```
# للبحث في الـ mailing lists
"I2C_TEN_BIT_ADDRESS" site:lore.kernel.org
"I2C_OWN_SLAVE_ADDRESS" site:lore.kernel.org
"dt-bindings i2c" site:lore.kernel.org

# للبحث في الـ kernel source
"i2c.h" "dt-bindings" linux kernel
"i2c-core-of.c" I2C_TEN_BIT_ADDRESS

# موضوعات مرتبطة
"I2C slave mode linux"
"ten bit i2c address device tree"
"i2c reg property flags device tree"
"Wolfram Sang i2c dt-bindings"
```

---

### ملخص المصادر الأساسية

| الأولوية | المصدر | الرابط |
|----------|--------|--------|
| *** | I2C Ten-bit Addresses (kernel docs) | [docs.kernel.org](https://docs.kernel.org/i2c/ten-bit-addresses.html) |
| *** | I2C Subsystem Index (kernel docs) | [docs.kernel.org](https://docs.kernel.org/i2c/index.html) |
| *** | I2C slave support framework (LWN) | [lwn.net/611332](https://lwn.net/Articles/611332/) |
| ** | DT bindings i2c.txt (kernel.org) | [kernel.org](https://www.kernel.org/doc/Documentation/devicetree/bindings/i2c/i2c.txt) |
| ** | PATCH: 10bit/slave i2c reg flags | [lkml.iu.edu](https://lkml.iu.edu/hypermail/linux/kernel/2005.3/04249.html) |
| ** | Writing I2C clients (LWN) | [static.lwn.net](https://static.lwn.net/kerneldoc/i2c/writing-clients.html) |
| * | eLinux I2C fault injection | [elinux.org](https://elinux.org/Tests:I2C-fault-injection) |
| * | LDD3 (free) | [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/) |
## Phase 8: Writing simple module

الـ file `dt-bindings/i2c/i2c.h` بتعرّف constant واحد بس هو `I2C_TEN_BIT_ADDRESS` و `I2C_OWN_SLAVE_ADDRESS` — مش فيها functions أو structs. عشان كده هنختار نعمل **kprobe** على `i2c_transfer()` اللي هي أهم function في subsystem الـ I2C وبتستخدم هذه الـ constants لما بتشتغل مع ten-bit addresses.

### الـ Hook المختار: `i2c_transfer` via kprobe

الـ `i2c_transfer()` هي entry point الـ I2C في الكيرنل — كل message بيتبعت على أي bus I2C بيمشي من هنا. بنعمل kprobe عليها عشان نشوف مين بيبعت، على أنهي adapter، وإيه الـ flags اللي بتتشيك منها `I2C_TEN_BIT_ADDRESS`.

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * i2c_transfer kprobe demo
 * Hooks i2c_transfer() and prints adapter name + message flags,
 * highlighting I2C_TEN_BIT_ADDRESS usage from dt-bindings/i2c/i2c.h.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe    */
#include <linux/i2c.h>          /* struct i2c_adapter, struct i2c_msg */
#include <linux/kernel.h>       /* pr_info                            */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Demo <demo@example.com>");
MODULE_DESCRIPTION("kprobe on i2c_transfer to trace I2C messages");

/*
 * I2C_TEN_BIT_ADDRESS is bit 31 of the msg->addr field
 * as defined in dt-bindings/i2c/i2c.h — same bit the core
 * checks to switch to 10-bit addressing mode.
 */
#define I2C_TEN_BIT_ADDRESS  (1 << 31)
#define I2C_OWN_SLAVE_ADDRESS (1 << 30)

/* ------------------------------------------------------------------
 * pre-handler: runs just before i2c_transfer() executes
 *
 * signature of i2c_transfer:
 *   int i2c_transfer(struct i2c_adapter *adap,
 *                    struct i2c_msg     *msgs,
 *                    int                 num);
 * ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * نجيب الـ arguments من الـ registers.
     * على x86-64: rdi=adap, rsi=msgs, rdx=num
     * على arm64:  x0=adap,  x1=msgs,  x2=num
     */
#if defined(CONFIG_X86_64)
    struct i2c_adapter *adap = (struct i2c_adapter *)regs->di;
    struct i2c_msg     *msgs = (struct i2c_msg *)regs->si;
    int                 num  = (int)regs->dx;
#elif defined(CONFIG_ARM64)
    struct i2c_adapter *adap = (struct i2c_adapter *)regs->regs[0];
    struct i2c_msg     *msgs = (struct i2c_msg *)regs->regs[1];
    int                 num  = (int)regs->regs[2];
#else
    /* unsupported arch — skip silently */
    return 0;
#endif

    int i;

    if (!adap || !msgs || num <= 0)
        return 0;

    for (i = 0; i < num; i++) {
        u16 raw_addr = msgs[i].addr;
        int is_ten   = !!(msgs[i].flags & I2C_M_TEN); /* kernel flag */
        int is_rd    = !!(msgs[i].flags & I2C_M_RD);

        pr_info("i2c_probe: adapter=\"%s\" msg[%d] addr=0x%03x "
                "flags=0x%04x len=%u %s %s\n",
                adap->name,
                i,
                raw_addr,
                msgs[i].flags,
                msgs[i].len,
                is_ten ? "[10-bit]" : "[7-bit]",
                is_rd  ? "READ"     : "WRITE");
    }

    return 0; /* 0 = let i2c_transfer continue normally */
}

/* ------------------------------------------------------------------
 * kprobe descriptor — نحدد اسم الـ function اللي هنعمل probe عليها
 * ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "i2c_transfer",  /* kernel resolves address at runtime */
    .pre_handler = handler_pre,     /* our callback before the function   */
};

/* ------------------------------------------------------------------
 * module_init: بنسجّل الـ kprobe لما الـ module يتحمّل
 * ------------------------------------------------------------------ */
static int __init i2c_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("i2c_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("i2c_probe: planted on i2c_transfer at %p\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------
 * module_exit: لازم نشيل الـ kprobe قبل ما الـ module يتفكّ من الذاكرة
 * عشان لو فضل مسجّل وحصل call هيعمل crash لأن الـ handler اتشال
 * ------------------------------------------------------------------ */
static void __exit i2c_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("i2c_probe: removed from i2c_transfer\n");
}

module_init(i2c_probe_init);
module_exit(i2c_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | ماكروهات `module_init` / `module_exit` و `MODULE_LICENSE` |
| `linux/kprobes.h` | `struct kprobe` و `register_kprobe` / `unregister_kprobe` |
| `linux/i2c.h` | تعريف `struct i2c_adapter` و `struct i2c_msg` و flags زي `I2C_M_TEN` |
| `linux/kernel.h` | `pr_info` / `pr_err` |

الـ `linux/i2c.h` هو اللي بيربط كل حاجة — فيه `I2C_M_TEN` اللي هو الـ runtime flag اللي بيقابل `I2C_TEN_BIT_ADDRESS` في الـ dt-bindings على مستوى الـ devicetree.

#### الـ `handler_pre` — الـ Callback

بيشتغل **قبل** ما `i2c_transfer()` تنفّذ. بنجيب الـ arguments من الـ registers عشان الـ kprobe بيشتغل على مستوى الـ machine code مش على مستوى الـ C. بنطبع اسم الـ adapter والـ address ونوع العملية (read/write) وإذا كان الـ address ده 10-bit — ده هو الـ connection المباشر مع `I2C_TEN_BIT_ADDRESS` اللي في الملف الأصلي.

القيمة `0` في الـ return معناها "اكمل تنفيذ الـ function الأصلية" — لو رجّعنا قيمة تانية ممكن نوقف التنفيذ بس مش هنا محتاجينها.

#### الـ `struct kprobe`

الـ field الوحيد المهم هنا هو `symbol_name` — الكيرنل بيحوّله لـ address وقت تسجيل الـ probe بدل ما نحدد عنوان ثابت. ده بيخلي الـ module يشتغل على أي kernel version طالما الـ symbol موجود.

#### الـ `module_init` / `module_exit`

- **`register_kprobe`**: بتحجز الـ probe وتزرع breakpoint افتراضي في الكود. لو فشلت (مثلاً الـ symbol مش exported أو الكيرنل مش compiled مع `CONFIG_KPROBES`) بترجع error ومش بنكمل.
- **`unregister_kprobe`**: **ضرورية** في الـ exit — لو الـ module اتشال من الذاكرة والـ probe فضلت مسجلة، أي call لـ `i2c_transfer` هيجمّد الـ system لأن الـ handler_pre pointer بقى invalid.

---

### Makefile للـ Build

```makefile
obj-m += i2c_kprobe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

### تشغيل وإخراج متوقع

```bash
# تحميل الـ module
sudo insmod i2c_kprobe.ko

# نفذ أي أمر I2C مثلاً scan
sudo i2cdetect -y 1

# شوف الـ output
dmesg | grep i2c_probe
```

```
[  123.456] i2c_probe: planted on i2c_transfer at ffffffffc0123456
[  124.789] i2c_probe: adapter="i2c-1" msg[0] addr=0x050 flags=0x0000 len=1 [7-bit] WRITE
[  124.791] i2c_probe: adapter="i2c-1" msg[0] addr=0x050 flags=0x0001 len=1 [7-bit] READ
...

# إزالة الـ module
sudo rmmod i2c_kprobe
```
