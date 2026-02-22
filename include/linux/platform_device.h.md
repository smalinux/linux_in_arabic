## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

الـ `include/linux/platform_device.h` هو القلب النابض لنظام **Platform Driver Model** في Linux — الإطار الذي يربط الأجهزة المدمجة مباشرة في الـ SoC (System-on-Chip) بالـ drivers الخاصة بها، دون الحاجة لـ bus قابل للاكتشاف التلقائي مثل PCI أو USB.

ينتمي هذا الملف إلى subsystem اسمه **DRIVER CORE, KOBJECTS, DEBUGFS AND SYSFS**، وهو من أهم subsystems في النواة بأكملها.

---

### القصة: لماذا يوجد هذا أصلاً؟

تخيل أنك تصنع هاتف Android أو راوتر أو كاميرا مدمجة. داخل الـ chip الواحد (SoC) يوجد:
- وحدة UART (منفذ تسلسلي)
- وحدة I2C و SPI
- وحدة USB controller
- وحدة GPU
- وحدة watchdog timer

كل هذه الأجهزة **مدمجة داخل الـ chip مباشرة** — لا يمكن خلعها ولا اكتشافها تلقائياً مثل بطاقات PCI. عناوينها في الذاكرة وأرقام الـ IRQ الخاصة بها **ثابتة ومعروفة مسبقاً** من datasheet الـ chip.

المشكلة: Linux يحتاج طريقة موحدة لـ:
1. **تسجيل** هذه الأجهزة مع الـ kernel
2. **ربطها** بالـ drivers الصحيحة
3. **إدارة مواردها** (memory regions, IRQs, DMA)
4. **دعم Power Management** (suspend/resume)

الحل هو **Platform Device / Platform Driver** framework.

---

### التشبيه البسيط

فكر في المطعم:
- **`platform_device`** = الطبق (يحمل الطعام = الموارد: عنوان الذاكرة، رقم IRQ)
- **`platform_driver`** = الطاهي (يعرف كيف يتعامل مع هذا النوع من الطبق)
- **`platform_bus_type`** = مطعم يربط الطاهي بالطبق المناسب عبر تطابق الأسماء

حين يُسجَّل الجهاز والـ driver معاً، يقوم الـ bus بمطابقتهما ويستدعي `probe()` — مثل "هذا الطاهي يعرف هذا الطبق، دعه يطبخ".

---

### ماذا يعرّف هذا الملف؟

#### 1. الـ `struct platform_device` — تمثيل الجهاز

```c
struct platform_device {
    const char      *name;           /* اسم الجهاز، يستخدم للمطابقة مع الـ driver */
    int             id;              /* رقم المثيل، مثل uart.0 أو uart.1 */
    bool            id_auto;         /* هل خُصص الـ id تلقائياً؟ */
    struct device   dev;             /* الجهاز الأساسي في نموذج الـ driver */
    u64             platform_dma_mask;
    struct device_dma_parameters dma_parms;
    u32             num_resources;   /* عدد الموارد */
    struct resource *resource;       /* مصفوفة الموارد: IRQ، MMIO، DMA */

    const struct platform_device_id *id_entry; /* مدخل المطابقة مع الـ driver */
    const char *driver_override;     /* إجبار driver بعينه */
    struct mfd_cell *mfd_cell;       /* للأجهزة متعددة الوظائف MFD */
    struct pdev_archdata archdata;   /* بيانات خاصة بالمعمارية */
};
```

الـ `struct resource` (من `<linux/ioport.h>`) هو الذي يصف كل مورد:
```c
struct resource {
    resource_size_t start;   /* بداية النطاق: عنوان ذاكرة أو رقم IRQ */
    resource_size_t end;     /* نهاية النطاق */
    const char *name;
    unsigned long flags;     /* نوع المورد: IORESOURCE_MEM, IORESOURCE_IRQ... */
};
```

#### 2. الـ `struct platform_driver` — تمثيل الـ driver

```c
struct platform_driver {
    int  (*probe)(struct platform_device *);   /* يُستدعى حين يجد الـ driver جهازه */
    void (*remove)(struct platform_device *);  /* يُستدعى حين يُزال الجهاز */
    void (*shutdown)(struct platform_device *);
    int  (*suspend)(struct platform_device *, pm_message_t state);
    int  (*resume)(struct platform_device *);
    struct device_driver driver;               /* الـ driver الأساسي */
    const struct platform_device_id *id_table; /* جدول الأجهزة المدعومة */
    bool prevent_deferred_probe;
    bool driver_managed_dma;                   /* للـ VFIO وما شابه */
};
```

#### 3. الـ `struct platform_device_info` — طريقة آمنة لتسجيل الجهاز

```c
struct platform_device_info {
    struct device       *parent;
    struct fwnode_handle *fwnode;  /* يربطه بـ Device Tree أو ACPI */
    const char          *name;
    int                 id;
    const struct resource *res;
    unsigned int        num_res;
    const void          *data;     /* بيانات خاصة بالـ platform */
    size_t              size_data;
    u64                 dma_mask;
    const struct property_entry *properties;
};
```

---

### دورة حياة الجهاز: من التسجيل إلى الـ probe

```
  Board File / Device Tree / ACPI
           |
           v
  platform_device_register()
  platform_device_register_full()
           |
           v
  [platform_bus_type يبحث عن driver مناسب]
           |
           v
  platform_driver_register()  <-- من الـ driver module
           |
           v
  [المطابقة: name matching أو id_table أو of_match_table]
           |
           v
  driver->probe(pdev)  <-- الـ driver يبدأ عمله
```

---

### الـ macros المساعدة — تبسيط الحياة

| Macro | ما يفعله |
|-------|----------|
| `module_platform_driver(drv)` | يولّد `module_init` و `module_exit` تلقائياً |
| `builtin_platform_driver(drv)` | للـ drivers المدمجة في الـ kernel مباشرة |
| `platform_driver_probe(drv, probe)` | للأجهزة غير القابلة للـ hotplug، يحفظ الذاكرة |
| `platform_create_bundle(...)` | يسجّل الجهاز والـ driver معاً دفعة واحدة |
| `platform_register_drivers(...)` | يسجل مجموعة drivers دفعة واحدة |

---

### كيف يحصل الـ driver على موارده؟

```c
/* الحصول على عنوان الذاكرة MMIO */
struct resource *res = platform_get_resource(pdev, IORESOURCE_MEM, 0);

/* الحصول على رقم الـ IRQ */
int irq = platform_get_irq(pdev, 0);

/* map الذاكرة مباشرة مع إدارة تلقائية للتحرير (devres) */
void __iomem *base = devm_platform_ioremap_resource(pdev, 0);
```

الـ `devm_` prefix يعني **device-managed**: حين يُفصل الجهاز، يحرر الـ kernel الموارد تلقائياً دون تدخل الـ driver.

---

### مثال واقعي: driver لـ UART مدمج

```c
static int my_uart_probe(struct platform_device *pdev)
{
    /* احصل على عنوان الـ registers في الذاكرة */
    void __iomem *base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(base))
        return PTR_ERR(base);

    /* احصل على رقم الـ interrupt */
    int irq = platform_get_irq(pdev, 0);
    if (irq < 0)
        return irq;

    /* احفظ البيانات الخاصة بالـ driver */
    struct my_uart *uart = devm_kzalloc(&pdev->dev, sizeof(*uart), GFP_KERNEL);
    platform_set_drvdata(pdev, uart);

    /* ... بقية الإعداد ... */
    return 0;
}

static struct platform_driver my_uart_driver = {
    .probe  = my_uart_probe,
    .driver = {
        .name = "my-uart",
        .of_match_table = my_uart_of_match, /* للـ Device Tree */
    },
};
module_platform_driver(my_uart_driver); /* يغني عن module_init/module_exit */
```

---

### الـ Power Management

الملف يوفر عمليات PM جاهزة:

```c
/* في تعريف الـ driver */
static const struct dev_pm_ops my_pm_ops = {
    USE_PLATFORM_PM_SLEEP_OPS  /* يُضمّن: suspend, resume, freeze, thaw, poweroff, restore */
};
```

هذه الـ macros تُدمج تلقائياً حسب `CONFIG_SUSPEND` و `CONFIG_HIBERNATE_CALLBACKS`.

---

### الملفات المرتبطة التي يجب معرفتها

| الملف | الدور |
|-------|-------|
| `drivers/base/platform.c` | التنفيذ الفعلي لكل الدوال المُعلنة هنا |
| `include/linux/device.h` | تعريف `struct device` و `struct device_driver` الأساسيين |
| `include/linux/ioport.h` | تعريف `struct resource` وثوابت `IORESOURCE_*` |
| `include/linux/mod_devicetable.h` | تعريف `struct platform_device_id` لجداول المطابقة |
| `drivers/base/dd.c` | منطق ربط الـ device بالـ driver (device/driver binding) |
| `include/linux/of_device.h` | ربط الـ platform_device بالـ Device Tree |
| `include/linux/acpi.h` | ربط الـ platform_device بالـ ACPI |
| `include/linux/mfd/core.h` | الأجهزة متعددة الوظائف التي تولّد platform_devices |
| `drivers/base/platform-msi.c` | دعم الـ MSI interrupts للـ platform devices |

---

### ملفات الـ subsystem (DRIVER CORE)

**الأساسية:**
- `drivers/base/platform.c` — تنفيذ الـ platform bus
- `drivers/base/core.c` — نواة نموذج الـ driver
- `drivers/base/dd.c` — ربط الأجهزة بالـ drivers
- `drivers/base/bus.c` — إدارة الـ buses
- `drivers/base/driver.c` — إدارة الـ drivers
- `drivers/base/devres.c` — إدارة موارد الـ device تلقائياً (devm_*)

**الـ headers:**
- `include/linux/platform_device.h` — هذا الملف
- `include/linux/device.h` — الواجهة الرئيسية
- `include/linux/device/bus.h` — تعريف `struct bus_type`
- `include/linux/device/driver.h` — تعريف `struct device_driver`
- `include/linux/device/devres.h` — واجهة الـ devres

**الـ hardware drivers** (أمثلة تستخدم هذا الـ framework):
- `drivers/tty/serial/` — UART drivers
- `drivers/i2c/busses/` — I2C controllers
- `drivers/spi/` — SPI controllers
- `drivers/gpio/` — GPIO controllers
- `drivers/watchdog/` — Watchdog timers
## Phase 2: شرح الـ Platform Device Framework

### المشكلة — لماذا وُجد هذا الـ Subsystem؟

في عالم الـ embedded Linux وبالذات على معالجات ARM، توجد عشرات الـ peripherals مدمجة داخل الـ SoC نفسه: UART، I2C، SPI، GPIO، timers، watchdog، DMA controllers. هذه الأجهزة **لا تدعم الـ auto-discovery** — فالـ CPU لا يستطيع أن يسأل الـ bus "من هناك؟" كما يحدث مع PCI أو USB. عنوان الـ register الـ base address وأرقام الـ IRQs مكتوبة في المخطط الكهربائي للـ board، ليست قابلة للاكتشاف بالبرمجيات.

قبل ظهور هذا الـ framework كان المطور يكتب عناوين الـ hardware مباشرةً داخل كود الـ driver:

```c
/* الأسلوب القديم — كارثي لأنه يربط الـ driver بـ board بعينها */
#define MY_UART_BASE  0x101F1000
#define MY_UART_IRQ   12

void my_uart_init(void) {
    void __iomem *regs = ioremap(MY_UART_BASE, 0x1000);
    request_irq(MY_UART_IRQ, my_handler, 0, "uart", NULL);
}
```

النتيجة: driver لا يعمل إلا على board واحدة، وأي تغيير في الـ hardware يعني إعادة كتابة كود الـ kernel.

---

### الحل — ماذا قدّم الـ Kernel؟

الـ **Platform Device Framework** هو طبقة تجريد تفصل بين ثلاثة أطراف:

| الطرف | دوره |
|---|---|
| **Platform Device** | يصف الـ hardware: عناوينه، IRQs، بياناته |
| **Platform Driver** | يحتوي منطق التشغيل (probe, remove, suspend, resume) |
| **Platform Bus** | يُجري الـ matching بين الـ device والـ driver |

الـ board-specific data (عناوين الـ registers، أرقام الـ IRQs) تُعرَّف مرة واحدة في ملفات الـ board setup أو Device Tree، ثم يستلمها الـ driver ديناميكياً عبر `platform_get_resource()` و`platform_get_irq()`.

---

### المقارنة الواقعية — مستودع الطرود

تخيل **مستودع توزيع طرود**:

```
┌─────────────────────────────────────────────────────────┐
│                   مستودع الطرود                          │
│                  (Platform Bus)                          │
│                                                          │
│  ┌─────────────────┐      ┌──────────────────────┐      │
│  │   طرد مُعنوَن    │      │   موظف متخصص         │      │
│  │ (Platform Device)│ ───► │  (Platform Driver)   │      │
│  │                  │      │                      │      │
│  │ الاسم: "uart"    │      │ يعرف كيف يفتح "uart" │      │
│  │ عنوان: 0x101F1000│      │ probe() → يبدأ العمل │      │
│  │ IRQ: 12          │      │ remove() → يوقف      │      │
│  └─────────────────┘      └──────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

**الـ mapping الكامل للمقارنة:**

| عنصر المستودع | المقابل في الـ kernel |
|---|---|
| الطرد ونفسه | `struct platform_device` |
| اللاصقة على الطرد (الاسم + المعلومات) | `name`, `resource[]`, `platform_data` |
| الموظف المتخصص | `struct platform_driver` |
| قاعدة بيانات المستودع التي تُطابق الطرد بالموظف | `platform_bus_type` و matching logic |
| رف المستودع (التسلسل الهرمي) | `device.parent` tree |
| وصل الطرد بالموظف | `driver_probe_device()` → `platform_driver.probe()` |
| سحب الطرد من الموظف | `platform_driver.remove()` |

---

### البنية الكبيرة — أين يجلس هذا الـ Subsystem؟

```
┌──────────────────────────────────────────────────────────────────────┐
│                        User Space                                     │
│              /sys/bus/platform/devices/   /sys/bus/platform/drivers/  │
└──────────────────────────┬───────────────────────────────────────────┘
                           │ sysfs
┌──────────────────────────▼───────────────────────────────────────────┐
│                     Kernel Space                                      │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                    Driver Model Core (device.h)                  │ │
│  │    struct device   struct device_driver   struct bus_type        │ │
│  └────────────────────────────┬────────────────────────────────────┘ │
│                               │ يرث منه                               │
│  ┌────────────────────────────▼────────────────────────────────────┐ │
│  │              Platform Bus Layer (platform_device.h)              │ │
│  │                                                                  │ │
│  │   platform_bus_type ←→ platform_device ←→ platform_driver       │ │
│  └──────┬──────────────────────────────────────────┬───────────────┘ │
│         │                                          │                  │
│  ┌──────▼──────────┐                    ┌──────────▼──────────────┐  │
│  │  Device Sources  │                    │   Driver Examples       │  │
│  │                  │                    │                         │  │
│  │ • Device Tree    │                    │ • drivers/tty/serial/   │  │
│  │   (of_platform)  │                    │   amba-pl011.c          │  │
│  │ • ACPI tables    │                    │ • drivers/i2c/          │  │
│  │ • Board files    │                    │   i2c-gpio.c            │  │
│  │   (arch/arm/     │                    │ • drivers/watchdog/     │  │
│  │    mach-*/       │                    │   bcm2835_wdt.c         │  │
│  │    board-*.c)    │                    │ • drivers/mfd/          │  │
│  │ • MFD children   │                    │   (MFD subsystem)       │  │
│  └──────────────────┘                    └─────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────────┐
│                        Hardware (SoC)                                 │
│  UART @ 0x101F1000  I2C @ 0x40002000  GPIO @ 0xE0200000  ...        │
└──────────────────────────────────────────────────────────────────────┘
```

---

### التجريد المركزي — الـ Core Abstraction

الفكرة المحورية: **فصل وصف الـ hardware عن منطق التشغيل**، وربطهما عبر الـ name matching أو OF matching.

#### `struct platform_device` — البطاقة التعريفية للجهاز

```c
struct platform_device {
    const char    *name;           /* اسم الجهاز — مفتاح الـ matching */
    int            id;             /* رقم النسخة: uart.0، uart.1، ... */
    bool           id_auto;        /* الكيرنل يختار الـ id تلقائياً */
    struct device  dev;            /* الـ generic device — مدفون هنا */
    u64            platform_dma_mask;
    struct device_dma_parameters dma_parms; /* قيود الـ DMA للـ SoC */
    u32            num_resources;  /* عدد الـ resources (MEM + IRQ + ...) */
    struct resource *resource;     /* مصفوفة resources: عناوين + IRQs */

    const struct platform_device_id *id_entry; /* يُعبأ بعد الـ matching */
    const char *driver_override;   /* إجبار driver بعينه */
    struct mfd_cell *mfd_cell;     /* إذا كان child لـ MFD device */
    struct pdev_archdata archdata;  /* إضافات architecture-specific */
};
```

**ملاحظة:** `struct device dev` مدفونة داخل `platform_device` — وهذا النمط الأساسي في الـ Linux Driver Model: الـ generic object يُضمَّن داخل الـ specific object، ثم تُستخدم `container_of()` للتنقل بينهما.

```c
/* كيف تصل من struct device* إلى struct platform_device* */
#define to_platform_device(x) container_of((x), struct platform_device, dev)

/* مثال داخل الـ bus code */
struct platform_device *pdev = to_platform_device(dev);
```

#### `struct resource` — قلب معلومات الـ hardware

**الـ Resource subsystem** (ليس Platform Device) هو من يمتلك `struct resource` — وهو نظام شجري لإدارة مناطق الذاكرة والـ I/O. الـ Platform Device يستخدمه لتسجيل موارده.

```c
struct resource {
    resource_size_t start;   /* بداية المنطقة: 0x101F1000 */
    resource_size_t end;     /* نهاية المنطقة: 0x101F1FFF */
    const char *name;        /* وصف اختياري: "uart0-regs" */
    unsigned long flags;     /* النوع: IORESOURCE_MEM أو IORESOURCE_IRQ */
    unsigned long desc;      /* وصف تفصيلي */
    struct resource *parent, *sibling, *child; /* شجرة الـ resources */
};
```

أنواع الـ resources الشائعة:

| الـ Flag | المعنى | مثال |
|---|---|---|
| `IORESOURCE_MEM` | منطقة ذاكرة mapped | `0x101F1000..0x101F1FFF` |
| `IORESOURCE_IRQ` | رقم interrupt | `IRQ 12` |
| `IORESOURCE_IO` | I/O port (x86) | نادر في ARM |
| `IORESOURCE_DMA` | قناة DMA | DMA channel 3 |

#### `struct platform_driver` — بطاقة الـ driver

```c
struct platform_driver {
    int  (*probe) (struct platform_device *);   /* الـ driver يبدأ هنا */
    void (*remove)(struct platform_device *);   /* التنظيف عند الإزالة */
    void (*shutdown)(struct platform_device *); /* إيقاف تشغيل النظام */
    int  (*suspend)(struct platform_device *, pm_message_t state);
    int  (*resume) (struct platform_device *);
    struct device_driver driver;     /* الـ generic driver مدفون هنا */
    const struct platform_device_id *id_table; /* جدول الأسماء المدعومة */
    bool prevent_deferred_probe;     /* لا تؤجل الـ probe */
    bool driver_managed_dma;         /* الـ driver يدير الـ DMA بنفسه */
};
```

**العلاقة بين `platform_driver.driver` و `device_driver`:**

الـ `platform_driver` يحتوي `struct device_driver driver` وهو نفسه يحتوي `of_match_table` (لمطابقة Device Tree) و `acpi_match_table`. أي أن الـ matching يمكن أن يتم بثلاث طرق:

```
1. Name matching:       platform_device.name == platform_driver.driver.name
2. ID table matching:   platform_device.name in platform_driver.id_table[]
3. OF matching:         device_tree_compatible in platform_driver.driver.of_match_table
```

---

### رسم العلاقات بين الـ Structs

```
struct platform_device
├── name: "pl011"                      ←─── مفتاح الـ matching
├── id: 0
├── dev: struct device ◄───────────────────────────────────────────┐
│   ├── kobj: struct kobject           ←─── يظهر في /sys/          │
│   ├── parent: *device               ←─── الـ parent في الشجرة   │
│   ├── bus: &platform_bus_type        ←─── ينتمي للـ platform bus  │
│   ├── driver: *device_driver        ←─── يُعبأ بعد الـ matching  │
│   ├── platform_data: void*          ←─── board-specific data     │
│   ├── driver_data: void*            ←─── private driver data     │
│   ├── of_node: *device_node         ←─── DT node إذا وُجد        │
│   └── power: dev_pm_info            ←─── PM state                │
├── num_resources: 2
├── resource[0]: struct resource      ←─── MEM: 0x101F1000
│   ├── start: 0x101F1000
│   ├── end:   0x101F1FFF
│   └── flags: IORESOURCE_MEM
└── resource[1]: struct resource      ←─── IRQ: 12
    ├── start: 12
    ├── end:   12
    └── flags: IORESOURCE_IRQ

struct platform_driver
├── probe:  pl011_probe()
├── remove: pl011_remove()
├── driver: struct device_driver
│   ├── name: "pl011"                 ←─── مفتاح الـ matching
│   ├── bus:  &platform_bus_type
│   ├── of_match_table: pl011_of_match[]  ←─── DT compatible strings
│   └── pm: &pl011_dev_pm_ops
└── id_table: amba_ids[]

platform_bus_type
├── name: "platform"
├── match():   platform_match()       ←─── يُقارن الأسماء / OF nodes
├── probe():   platform_drv_probe()   ←─── يستدعي platform_driver.probe()
└── remove():  platform_drv_remove()
                     │
                     │ عند التطابق
                     ▼
        platform_driver.probe(pdev)
                     │
                     ▼
        pdev->dev.driver = &pdrv->driver  /* ربط الـ driver بالـ device */
```

---

### كيف يعمل الـ Matching — خطوة بخطوة

```
1. platform_device_register(pdev)
        │
        ▼
   يُسجَّل الـ device في platform_bus
   يُنشأ /sys/bus/platform/devices/pl011.0
        │
        ▼
   bus->match() يُجرى مع كل driver مسجل
        │
        ▼
   platform_match(dev, drv):
     أولاً: driver_override إذا وُجد
     ثانياً: of_driver_match_device() ←── Device Tree matching
     ثالثاً: acpi_driver_match_device()
     رابعاً: id_table matching
     أخيراً: strcmp(pdev->name, drv->name)
        │
        │ تطابق!
        ▼
   platform_drv_probe(dev):
     pdev = to_platform_device(dev)
     pdrv = to_platform_driver(dev->driver)
     pdrv->probe(pdev)  ←── الـ driver يستلم pdev ويبدأ العمل
```

---

### ما يمتلكه الـ Framework وما يُفوّضه للـ Driver

#### ما يمتلكه الـ Platform Framework:

| المسؤولية | الآلية |
|---|---|
| تسجيل الـ device في الـ bus | `platform_device_register()` |
| إجراء الـ name/OF matching | `platform_match()` داخل `platform_bus_type` |
| استدعاء `probe()` عند التطابق | `platform_drv_probe()` |
| إدارة sysfs entries | Driver Model Core |
| PM callbacks العامة | `platform_pm_suspend/resume/freeze/restore` |
| إدارة الـ deferred probe | Core يعيد المحاولة لاحقاً إذا أرجع probe `EPROBE_DEFER` |
| devres cleanup عند remove | Core يحرر كل `devm_*` allocations |

#### ما يُفوّضه إلى الـ Driver:

| المسؤولية | الآلية |
|---|---|
| فهم محتوى الـ resources | `platform_get_resource()`, `platform_get_irq()` |
| الـ ioremap للـ registers | `devm_platform_ioremap_resource()` |
| تهيئة الـ hardware | `probe()` callback |
| تسجيل الـ hardware مع subsystems أعلى | مثلاً `uart_add_one_port()` أو `i2c_add_adapter()` |
| التنظيف عند الإزالة | `remove()` callback |
| Power management الخاص بالـ hardware | `suspend()` و`resume()` callbacks |
| قراءة الـ platform_data | `pdev->dev.platform_data` |

---

### مثال واقعي — UART Driver على ARM SoC

#### 1. تعريف الـ Device (board file أو Device Tree)

```c
/* arch/arm/mach-versatile/board-versatile.c — الأسلوب القديم */
static struct resource pl011_resources[] = {
    {
        .start = 0x101F1000,    /* base address من datasheet */
        .end   = 0x101F1FFF,
        .flags = IORESOURCE_MEM,
    },
    {
        .start = 12,            /* IRQ number */
        .end   = 12,
        .flags = IORESOURCE_IRQ,
    },
};

static struct platform_device pl011_device = {
    .name          = "uart-pl011",
    .id            = 0,
    .num_resources = ARRAY_SIZE(pl011_resources),
    .resource      = pl011_resources,
};
```

أو عبر Device Tree (الأسلوب الحديث):
```dts
/* arch/arm/boot/dts/versatile-pb.dts */
uart0: serial@101f1000 {
    compatible = "arm,pl011", "arm,primecell";
    reg = <0x101f1000 0x1000>;
    interrupts = <12>;
};
```

#### 2. تعريف الـ Driver

```c
/* drivers/tty/serial/amba-pl011.c — مبسط */
static int pl011_probe(struct platform_device *pdev)
{
    struct resource *mem;
    int irq;
    void __iomem *base;

    /* استلام الـ resources من الـ platform framework */
    base = devm_platform_ioremap_resource(pdev, 0); /* MEM resource[0] */
    if (IS_ERR(base))
        return PTR_ERR(base);

    irq = platform_get_irq(pdev, 0); /* IRQ resource[0] */
    if (irq < 0)
        return irq;

    /* الآن pdev->dev.platform_data أو DT properties للـ board data */
    /* تهيئة الـ UART hardware ... */
    /* تسجيل مع الـ serial subsystem ... */
    return 0;
}

static const struct of_device_id pl011_of_match[] = {
    { .compatible = "arm,pl011" },
    {},
};

static struct platform_driver pl011_driver = {
    .probe  = pl011_probe,
    .remove = pl011_remove,
    .driver = {
        .name          = "uart-pl011",
        .of_match_table = pl011_of_match,
        .pm            = &pl011_dev_pm_ops,
    },
};

/* يولّد module_init() و module_exit() تلقائياً */
module_platform_driver(pl011_driver);
```

#### 3. المسار الكامل من التسجيل إلى الـ probe

```
Boot time:
    device_tree_parse()
          │
          ▼
    of_platform_populate()
          │
          ▼
    platform_device_register(pl011_device)   ← pdev.name = "uart-pl011"
          │
          └──► /sys/bus/platform/devices/uart-pl011.0

    module_init() → platform_driver_register(pl011_driver)
          │
          ▼
    platform_bus.match(pl011_device, pl011_driver)
          │  strcmp("uart-pl011", "uart-pl011") == 0  ✓
          ▼
    platform_drv_probe(pl011_device)
          │
          ▼
    pl011_probe(pdev)
          │
          ├── devm_platform_ioremap_resource(pdev, 0)
          │     └── platform_get_resource(pdev, IORESOURCE_MEM, 0)
          │           └── resource[0]: 0x101F1000..0x101F1FFF → ioremap()
          │
          ├── platform_get_irq(pdev, 0)
          │     └── resource[1].start = 12
          │
          └── uart_add_one_port() → /dev/ttyAMA0 جاهز
```

---

### الـ `platform_device_info` — الطريق الأحدث للتسجيل

بدلاً من ملء `struct platform_device` يدوياً، الـ `platform_device_info` يُستخدم مع `platform_device_register_full()` لبناء الـ platform_device داخلياً:

```c
struct platform_device_info {
    struct device       *parent;      /* الـ parent device */
    struct fwnode_handle *fwnode;     /* firmware node (DT أو ACPI) */
    bool                of_node_reused; /* مشاركة الـ DT node مع الأب */
    const char          *name;
    int                  id;
    const struct resource *res;
    unsigned int         num_res;
    const void          *data;        /* platform_data */
    size_t               size_data;
    u64                  dma_mask;
    const struct property_entry *properties; /* software-defined properties */
};
```

هذا النمط شائع في الـ **MFD (Multi-Function Device) subsystem** حيث device واحد (مثل PMIC) يسجّل عدة platform_devices كـ children له، كل منها يمثل وظيفة مختلفة (regulator، RTC، battery).

---

### الـ `devm_*` APIs — إدارة الموارد التلقائية

**الـ devres subsystem** (يتطلب فهم `device.h` و`devres.c`) هو آلية تُسجّل الـ resources المخصصة مقرونةً بـ `struct device`، وتُحرَّر تلقائياً عند `device_unregister()` أو عند فشل الـ probe. الـ Platform Framework يوفر:

```c
/* يُنفذ: platform_get_resource() + ioremap() + devres_add() كلها */
void __iomem *devm_platform_ioremap_resource(struct platform_device *pdev,
                                              unsigned int index);

/* يُنفذ: platform_get_resource() + ioremap() + يُعيد الـ resource أيضاً */
void __iomem *devm_platform_get_and_ioremap_resource(struct platform_device *pdev,
                                                      unsigned int index,
                                                      struct resource **res);

/* بالاسم بدل الفهرس — عندما يكون للـ resource اسم نصي */
void __iomem *devm_platform_ioremap_resource_byname(struct platform_device *pdev,
                                                     const char *name);
```

---

### الـ PM Integration — Power Management

الـ Platform Framework يوفر wrapper callbacks مرتبطة بالـ `dev_pm_ops`:

```c
#ifdef CONFIG_PM_SLEEP
#define USE_PLATFORM_PM_SLEEP_OPS       \
    .suspend  = platform_pm_suspend,    \
    .resume   = platform_pm_resume,     \
    .freeze   = platform_pm_freeze,     \
    .thaw     = platform_pm_thaw,       \
    .poweroff = platform_pm_poweroff,   \
    .restore  = platform_pm_restore,
#endif
```

الـ driver يضع `USE_PLATFORM_PM_SLEEP_OPS` داخل الـ `dev_pm_ops` الخاص به، فيستدعي الـ framework تلقائياً `platform_driver.suspend/resume` عند أحداث الـ system sleep.

---

### الماكروز المساعدة — تقليل الـ Boilerplate

```c
/* يولّد module_init() و module_exit() لـ driver لا يحتاج شيئاً خاصاً */
module_platform_driver(my_driver);

/* مكافئ يدوي لـ module_platform_driver */
static int __init my_driver_init(void) {
    return platform_driver_register(&my_driver);
}
static void __exit my_driver_exit(void) {
    platform_driver_unregister(&my_driver);
}
module_init(my_driver_init);
module_exit(my_driver_exit);

/* للـ drivers المدمجة في الـ kernel (built-in) */
builtin_platform_driver(my_driver);

/* للـ drivers غير القابلة للـ hotplug — probe() في __init section */
#define platform_driver_probe(drv, probe) \
    __platform_driver_probe(drv, probe, THIS_MODULE)
/* بعد الـ probe تُحرَّر ذاكرة الـ __init section */
```

---

### ملخص — ما يمتلكه الـ Framework مقابل ما يُفوّضه

```
┌─────────────────────────────────────────────────────────────────┐
│                   Platform Framework يمتلك:                      │
│                                                                   │
│  ✓ platform_bus_type — الـ bus الوهمي                            │
│  ✓ Matching logic (name / OF / ACPI / id_table)                  │
│  ✓ probe/remove lifecycle orchestration                          │
│  ✓ sysfs entries تحت /sys/bus/platform/                          │
│  ✓ devm_* resource management integration                        │
│  ✓ PM callbacks wrappers                                         │
│  ✓ Deferred probe retry                                           │
│  ✓ Driver override mechanism                                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│               Platform Framework يُفوّض للـ Driver:              │
│                                                                   │
│  ✗ تفسير محتوى resources (أي عنوان يفعل ماذا)                  │
│  ✗ تهيئة الـ hardware registers                                  │
│  ✗ التسجيل مع الـ subsystems العليا (serial, i2c, spi, ...)     │
│  ✗ interrupt handling logic                                      │
│  ✗ DMA setup (إلا إذا driver_managed_dma = true)                │
│  ✗ منطق الـ suspend/resume الخاص بالـ hardware                   │
│  ✗ قراءة الـ platform_data وتفسيرها                              │
└─────────────────────────────────────────────────────────────────┘
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Macros والـ Config Options المهمة

#### أولاً: معرّفات الـ Device ID

| Macro | القيمة | المعنى |
|---|---|---|
| `PLATFORM_DEVID_NONE` | `-1` | الجهاز لا يحتاج ID رقمي |
| `PLATFORM_DEVID_AUTO` | `-2` | الـ kernel يُعيّن ID تلقائياً |

#### ثانياً: أنواع الـ Resource (`IORESOURCE_TYPE_BITS`)

| Flag | القيمة | النوع |
|---|---|---|
| `IORESOURCE_IO` | `0x00000100` | منافذ I/O (PCI/ISA) |
| `IORESOURCE_MEM` | `0x00000200` | نطاق ذاكرة مُعيَّن |
| `IORESOURCE_REG` | `0x00000300` | إزاحات سجلات registers |
| `IORESOURCE_IRQ` | `0x00000400` | خط مقاطعة |
| `IORESOURCE_DMA` | `0x00000800` | قناة DMA |
| `IORESOURCE_BUS` | `0x00001000` | رقم bus |

#### ثالثاً: خصائص الـ Resource الإضافية

| Flag | الوصف |
|---|---|
| `IORESOURCE_PREFETCH` | يمكن الـ prefetch — لا آثار جانبية |
| `IORESOURCE_READONLY` | للقراءة فقط |
| `IORESOURCE_EXCLUSIVE` | لا يُسمح لـ userland بـ mmap |
| `IORESOURCE_DISABLED` | مُعطَّل حالياً |
| `IORESOURCE_UNSET` | لم يُخصَّص له عنوان بعد |
| `IORESOURCE_BUSY` | الـ driver حجزه |
| `IORESOURCE_MEM_64` | ذاكرة 64-bit |
| `IORESOURCE_MUXED` | مورد يُشترك برمجياً |

#### رابعاً: خيارات الـ Config المؤثرة في السلوك

| Config | التأثير |
|---|---|
| `CONFIG_HAS_IOMEM` | يُفعّل دوال `devm_platform_ioremap_*` |
| `CONFIG_SUSPEND` | يُفعّل `platform_pm_suspend/resume` |
| `CONFIG_HIBERNATE_CALLBACKS` | يُفعّل freeze/thaw/poweroff/restore |
| `CONFIG_PM_SLEEP` | يُفعّل `USE_PLATFORM_PM_SLEEP_OPS` |
| `CONFIG_SUPERH` | تفعيل `is_sh_early_platform_device` |

#### خامساً: Device Link Flags (من `device.h`)

| Flag | المعنى |
|---|---|
| `DL_FLAG_STATELESS` | الـ core لا يحذف الـ link تلقائياً |
| `DL_FLAG_PM_RUNTIME` | يؤثر على runtime PM |
| `DL_FLAG_AUTOREMOVE_CONSUMER` | يحذف الـ link عند unbind الـ consumer |
| `DL_FLAG_AUTOREMOVE_SUPPLIER` | يحذف الـ link عند unbind الـ supplier |
| `DL_FLAG_AUTOPROBE_CONSUMER` | probe الـ consumer تلقائياً بعد الـ supplier |

---

### الـ Structs المهمة

#### 1. `struct platform_device`

**الغرض:** التمثيل الكامل لجهاز platform في نموذج driver الـ Linux. كل جهاز على platform bus (UART، SPI controller، GPIO، إلخ) هو instance من هذه البنية.

```c
struct platform_device {
    const char  *name;           /* اسم الجهاز — يُستخدم للمطابقة مع الـ driver */
    int          id;             /* رقم النسخة: -1 (NONE) أو -2 (AUTO) أو رقم */
    bool         id_auto;        /* true إذا خصّص الـ kernel الـ id تلقائياً */
    struct device dev;           /* الـ struct الأساسي — مُضمَّن بالكامل (not pointer) */
    u64          platform_dma_mask; /* قناع DMA خاص بهذا الجهاز */
    struct device_dma_parameters dma_parms; /* حدود DMA (segment size, alignment) */
    u32          num_resources;  /* عدد entries في مصفوفة الـ resources */
    struct resource *resource;   /* مصفوفة resources: IRQ, MEM, IO, DMA */
    const struct platform_device_id *id_entry; /* مُعبَّأ عند المطابقة مع id_table */
    const char  *driver_override; /* إجبار مطابقة driver بالاسم */
    struct mfd_cell *mfd_cell;   /* للأجهزة الفرعية من MFD (Multi-Function Device) */
    struct pdev_archdata archdata; /* إضافات معمارية خاصة (arm/x86/mips) */
};
```

**الحقول الأساسية:**

| الحقل | النوع | الدور |
|---|---|---|
| `name` | `const char *` | أساس المطابقة مع `platform_driver` |
| `id` | `int` | يُميّز النسخ: `uart.0`, `uart.1` |
| `dev` | `struct device` | مُضمَّن — يحمل kobject, bus, driver, pm |
| `resource` | `struct resource *` | مصفوفة موارد: IRQ, MEM, IO |
| `num_resources` | `u32` | حجم مصفوفة الـ resources |
| `id_entry` | `platform_device_id *` | مُعبَّأ تلقائياً عند المطابقة |
| `driver_override` | `const char *` | تجاوز آلية المطابقة التلقائية |
| `mfd_cell` | `struct mfd_cell *` | ربط بـ MFD parent |

---

#### 2. `struct platform_driver`

**الغرض:** تعريف driver يعمل على platform bus. يُوفّر callbacks دورة الحياة ومصفوفة ID للمطابقة.

```c
struct platform_driver {
    int  (*probe)(struct platform_device *);    /* تهيئة الـ hardware عند الربط */
    void (*remove)(struct platform_device *);   /* تنظيف عند الفصل */
    void (*shutdown)(struct platform_device *); /* عند إيقاف النظام */
    int  (*suspend)(struct platform_device *, pm_message_t state); /* نوم */
    int  (*resume)(struct platform_device *);   /* استيقاظ */
    struct device_driver driver;                /* مُضمَّن — يحمل name, bus, owner */
    const struct platform_device_id *id_table;  /* مصفوفة IDs للمطابقة */
    bool prevent_deferred_probe;                /* لا تؤخّر الـ probe */
    bool driver_managed_dma;                    /* الـ driver يدير DMA بنفسه (VFIO) */
};
```

**الحقول الأساسية:**

| الحقل | الدور |
|---|---|
| `probe` | **أهم callback** — يُستدعى عند مطابقة driver+device |
| `remove` | تنظيف الموارد، استدعاء `platform_set_drvdata(NULL)` |
| `shutdown` | إيقاف الـ hardware قبل reboot |
| `driver.name` | يُستخدم للمطابقة مع `platform_device.name` |
| `id_table` | مطابقة متعددة الأجهزة بـ ID |
| `driver_managed_dma` | لـ VFIO وما شابه |

---

#### 3. `struct platform_device_info`

**الغرض:** بنية مؤقتة تُمرَّر إلى `platform_device_register_full()` لإنشاء platform device كامل في خطوة واحدة. لا تبقى بعد الإنشاء.

```c
struct platform_device_info {
    struct device       *parent;      /* الجهاز الأب في شجرة الأجهزة */
    struct fwnode_handle *fwnode;     /* عقدة firmware (ACPI أو DT) */
    bool                 of_node_reused; /* مشاركة عقدة DT مع الأب */
    const char          *name;        /* اسم الجهاز الجديد */
    int                  id;          /* رقم النسخة */
    const struct resource *res;       /* مصفوفة الـ resources */
    unsigned int         num_res;     /* عدد الـ resources */
    const void          *data;        /* platform_data خاصة بالـ board */
    size_t               size_data;   /* حجم platform_data */
    u64                  dma_mask;    /* قناع DMA */
    const struct property_entry *properties; /* خصائص software nodes */
};
```

---

#### 4. `struct resource`

**الغرض:** وصف مورد هاردوير (ذاكرة، IRQ، I/O port، DMA). تُشكّل شجرة هرمية.

```c
struct resource {
    resource_size_t start;   /* عنوان البداية أو رقم IRQ */
    resource_size_t end;     /* عنوان النهاية (شامل) */
    const char     *name;    /* اسم وصفي للـ sysfs والـ /proc/iomem */
    unsigned long   flags;   /* نوع المورد + خصائصه (IORESOURCE_*) */
    unsigned long   desc;    /* وصف ممتد (IORES_DESC_*) */
    struct resource *parent, *sibling, *child; /* شجرة الموارد */
};
```

---

#### 5. `struct device` (من `device.h` — مُضمَّن في `platform_device`)

**الغرض:** البنية الجوهرية لكل جهاز في Linux. تحمل kobject، bus، driver، power management، DMA، devres.

| الحقل المهم | الدور |
|---|---|
| `kobj` | ربط بـ sysfs وإدارة المرجعية (refcount) |
| `parent` | الجهاز الأب في الشجرة |
| `bus` | يُشير إلى `platform_bus_type` |
| `driver` | الـ driver المرتبط حالياً |
| `platform_data` | بيانات board-specific (مثل pin configs) |
| `driver_data` | private data للـ driver (يُستخدم عبر `platform_get/set_drvdata`) |
| `mutex` | يحمي calls للـ driver |
| `devres_head` | قائمة devres (managed resources) |
| `devres_lock` | spinlock لحماية قائمة devres |
| `power` | بيانات runtime PM وsystem PM |
| `of_node` | عقدة Device Tree المقابلة |
| `fwnode` | عقدة firmware (DT أو ACPI) |

---

### مخططات العلاقات بين الـ Structs

#### الشكل العام للتضمين والتأشير

```
┌─────────────────────────────────────────────────────────┐
│                  struct platform_device                  │
│                                                         │
│  name ──────────────────────────► "serial8250"          │
│  id   ──────────────────────────► 0                     │
│                                                         │
│  ┌─────────────────────────────┐                        │
│  │       struct device (dev)   │  ◄── مُضمَّن (embedded) │
│  │  kobj ──► kobject           │                        │
│  │  bus  ──► platform_bus_type │                        │
│  │  driver─► platform_driver   │                        │
│  │  parent─► parent device     │                        │
│  │  of_node─► device_node      │                        │
│  │  fwnode ─► fwnode_handle    │                        │
│  │  power ──► dev_pm_info      │                        │
│  └─────────────────────────────┘                        │
│                                                         │
│  resource ──────────────────────► struct resource[n]    │
│  num_resources: n                                       │
│                                                         │
│  id_entry ──────────────────────► platform_device_id   │
│  driver_override ───────────────► "my_driver"          │
│  mfd_cell ──────────────────────► struct mfd_cell       │
│  archdata ──────────────────────► (embedded, arch only) │
└─────────────────────────────────────────────────────────┘

         ▲  to_platform_device(dev)
         │  container_of(dev, struct platform_device, dev)
         │

┌─────────────────────────────────────────────────────────┐
│                  struct platform_driver                  │
│                                                         │
│  probe  ──────────────────────► my_probe()              │
│  remove ──────────────────────► my_remove()             │
│                                                         │
│  ┌─────────────────────────────┐                        │
│  │    struct device_driver     │  ◄── مُضمَّن (embedded) │
│  │  name ──► "serial8250"      │                        │
│  │  bus  ──► platform_bus_type │                        │
│  │  owner──► THIS_MODULE       │                        │
│  └─────────────────────────────┘                        │
│                                                         │
│  id_table ──────────────────────► platform_device_id[] │
└─────────────────────────────────────────────────────────┘
```

#### شجرة الـ Resource

```
struct resource (iomem_resource — root)
│
├── struct resource (0x10000000 - 0x10000FFF, IORESOURCE_MEM)
│   └── name: "serial8250"
│       flags: IORESOURCE_MEM | IORESOURCE_BUSY
│
└── struct resource (IRQ 42, IORESOURCE_IRQ)
        name: "serial8250"
        flags: IORESOURCE_IRQ | IORESOURCE_IRQ_HIGHLEVEL

platform_device.resource ──► resource[0] (MEM)
                              resource[1] (IRQ)
platform_device.num_resources = 2
```

#### العلاقة بين `platform_device_info` و `platform_device`

```
platform_device_info        platform_device_register_full()
┌──────────────┐  ─────────────────────────────────►  ┌──────────────┐
│ name         │                                       │ name         │
│ id           │    يُنشئ ويُهيّئ                      │ id           │
│ res/num_res  │    platform_device جديد               │ resource[]   │
│ data/size    │    ويُسجّله تلقائياً                   │ dev.platform  │
│ parent       │                                       │  _data       │
│ fwnode       │                                       │ dev.parent   │
│ dma_mask     │                                       │ dev.fwnode   │
└──────────────┘                                       └──────────────┘
  (stack/temp)                                          (heap — يبقى)
```

---

### مخططات دورة الحياة

#### أولاً: دورة حياة `platform_device`

```
ALLOCATION
──────────
platform_device_alloc(name, id)
  └─► kzalloc(struct platform_device)
  └─► kobject_init(&pdev->dev.kobj)
  └─► pdev->dev.release = platform_device_release
  └─► device_initialize(&pdev->dev)

        │
        ▼

RESOURCE & DATA ATTACHMENT
───────────────────────────
platform_device_add_resources(pdev, res, n)
  └─► كopying مصفوفة resources إلى pdev->resource

platform_device_add_data(pdev, data, size)
  └─► kmemdup(data) → pdev->dev.platform_data

        │
        ▼

REGISTRATION
─────────────
platform_device_add(pdev)
  └─► تحديد id (DEVID_AUTO → ida_alloc)
  └─► dev_set_name(&pdev->dev, "%s.%d", name, id)
  └─► insert resources في شجرة iomem/ioport
  └─► device_add(&pdev->dev)
        └─► kobject_add → sysfs entry
        └─► bus_probe_device → matching

        │
        ▼

IN USE
───────
driver probe() يُستدعى
platform_set_drvdata(pdev, priv_data)
platform_get_irq / platform_get_resource
devm_platform_ioremap_resource

        │
        ▼

TEARDOWN
─────────
platform_device_unregister(pdev)
  └─► platform_device_del(pdev)
        └─► device_del(&pdev->dev)
              └─► driver remove()
              └─► release resources من شجرة iomem
              └─► kobject_del → sysfs removal
  └─► platform_device_put(pdev)
        └─► put_device(&pdev->dev)
              └─► refcount → 0 → release()
                    └─► kfree(pdev->dev.platform_data)
                    └─► kfree(pdev->resource)
                    └─► kfree(pdev)
```

#### ثانياً: دورة حياة `platform_driver`

```
DEFINE
───────
static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = { .name = "my_device", .owner = THIS_MODULE },
};

        │
        ▼

REGISTRATION
─────────────
module_platform_driver(my_driver)
  └─► module_init → platform_driver_register(&my_driver)
        └─► __platform_driver_register(drv, THIS_MODULE)
              └─► drv->driver.bus = &platform_bus_type
              └─► driver_register(&drv->driver)
                    └─► bus_add_driver
                    └─► driver_attach → scan existing devices
                          └─► platform_match() → probe()

        │
        ▼

MATCHING & PROBE
─────────────────
platform_match(dev, drv)
  1. driver_override → strcmp
  2. ACPI matching
  3. OF (Device Tree) matching
  4. id_table matching
  5. name matching

إذا نجح المطابقة:
platform_drv_probe(dev)
  └─► my_driver.probe(pdev)

        │
        ▼

UNLOAD
───────
module_exit → platform_driver_unregister(&my_driver)
  └─► driver_unregister
        └─► لكل device مرتبط: my_driver.remove(pdev)
        └─► driver_detach
        └─► bus_remove_driver
```

---

### مخططات تدفق الاستدعاء (Call Flow)

#### تدفق 1: `devm_platform_ioremap_resource`

```
driver probe() يحتاج map ذاكرة:
─────────────────────────────────

devm_platform_ioremap_resource(pdev, index)
  │
  ├─► platform_get_resource(pdev, IORESOURCE_MEM, index)
  │     └─► يبحث في pdev->resource[]
  │         عن العنصر رقم index من نوع IORESOURCE_MEM
  │         يُعيد pointer على struct resource
  │
  └─► devm_platform_get_and_ioremap_resource(pdev, index, &res)
        │
        ├─► devm_request_mem_region(dev, res->start, size, name)
        │     └─► __request_region(&iomem_resource, ...)
        │           └─► يُسجّل المنطقة في شجرة iomem
        │               (يحجبها من الآخرين)
        │
        └─► devm_ioremap(dev, res->start, size)
              └─► ioremap(phys_addr, size)
                    └─► يُنشئ virtual mapping في kernel space
                    └─► يُضيف إلى devres list
                        (تحرير تلقائي عند remove)

النتيجة: void __iomem *base ← يستخدمه الـ driver للكتابة/القراءة
```

#### تدفق 2: `platform_get_irq`

```
driver probe() يحتاج IRQ:
──────────────────────────

platform_get_irq(pdev, index)
  │
  ├─► platform_get_resource(pdev, IORESOURCE_IRQ, index)
  │     └─► يبحث في pdev->resource[]
  │         يجد entry بـ flags = IORESOURCE_IRQ
  │
  ├─► إذا وُجد: يُعيد res->start (رقم IRQ الـ Linux)
  │
  └─► إذا لم يوجد في resource[]:
        └─► يُجرّب fwnode_irq_get(pdev->dev.fwnode, index)
              └─► يقرأ من ACPI أو Device Tree
                    └─► يُعيد IRQ number

ثم في الـ driver:
devm_request_irq(dev, irq, handler, flags, name, dev_id)
  └─► request_irq(irq, ...)
  └─► يُسجّل في devres → تحرير تلقائي
```

#### تدفق 3: تسجيل driver كامل بـ `platform_device_register_full`

```
board code أو MFD driver:
──────────────────────────

platform_device_register_full(&pdevinfo)
  │
  ├─► platform_device_alloc(pdevinfo->name, pdevinfo->id)
  │     └─► kzalloc
  │     └─► device_initialize
  │
  ├─► pdev->dev.parent = pdevinfo->parent
  ├─► pdev->dev.fwnode = pdevinfo->fwnode
  │
  ├─► platform_device_add_resources(pdev, res, num_res)
  │
  ├─► platform_device_add_data(pdev, data, size_data)
  │     └─► kmemdup → pdev->dev.platform_data
  │
  ├─► pdev->dev.coherent_dma_mask = pdevinfo->dma_mask
  │
  └─► platform_device_add(pdev)
        ├─► if id == DEVID_AUTO: ida_alloc → id
        ├─► dev_set_name: "name.id"
        ├─► insert resources in iomem/ioport tree
        └─► device_add(&pdev->dev)
              ├─► kobject_add → /sys/devices/platform/name.id/
              ├─► uevent → udev notification
              └─► bus_probe_device → platform_match → probe()
```

#### تدفق 4: `module_platform_driver` — الاختصار الكامل

```c
/* يتمدد وقت الترجمة إلى: */
module_platform_driver(my_drv)
  ↓
static int __init my_drv_init(void) {
    return platform_driver_register(&my_drv);
}
static void __exit my_drv_exit(void) {
    platform_driver_unregister(&my_drv);
}
module_init(my_drv_init);
module_exit(my_drv_exit);
```

```
insmod my_driver.ko
  └─► my_drv_init()
        └─► platform_driver_register(&my_drv)
              └─► driver.bus = &platform_bus_type
              └─► driver_register
                    └─► bus_add_driver
                    └─► driver_attach
                          └─► لكل device على platform bus:
                                platform_match(dev, &my_drv.driver)
                                  ├─► driver_override؟
                                  ├─► ACPI match؟
                                  ├─► OF match؟
                                  ├─► id_table match؟
                                  └─► name match؟
                                إذا نجح:
                                really_probe(dev, drv)
                                  └─► platform_drv_probe(dev)
                                        └─► my_drv.probe(pdev)
                                              └─► الـ driver يبدأ عمله
```

---

### استراتيجية الـ Locking

#### 1. حماية دورة حياة الجهاز — `device.mutex`

**الـ lock:** `struct mutex device.mutex`

**يحمي:** جميع calls للـ driver (probe، remove، suspend، resume). يمنع race بين unbind وioctls.

```c
/* الـ core يمسك هذا الـ mutex قبل استدعاء probe/remove */
device_lock(dev);         /* mutex_lock(&dev->mutex) */
    driver->probe(dev);   /* أو driver->remove(dev) */
device_unlock(dev);       /* mutex_unlock(&dev->mutex) */
```

#### 2. حماية قائمة devres — `device.devres_lock`

**الـ lock:** `spinlock_t device.devres_lock`

**يحمي:** قائمة `devres_head` — قائمة الموارد المُدارة (managed resources).

```c
spin_lock_irqsave(&dev->devres_lock, flags);
    /* إضافة أو حذف من devres_head */
spin_unlock_irqrestore(&dev->devres_lock, flags);
```

**لماذا spinlock وليس mutex؟** لأن devres يُستدعى من سياق interrupt-safe (مثل devm cleanup).

#### 3. حماية شجرة الـ Resources — `resource_lock`

**الـ lock:** `rwlock_t resource_lock` (داخلي في `kernel/resource.c`)

**يحمي:** شجرة `iomem_resource` و`ioport_resource` بالكامل.

```c
/* عند platform_device_add: */
insert_resource(&iomem_resource, res)
  └─► write_lock(&resource_lock)
        /* إدراج في الشجرة */
      write_unlock(&resource_lock)

/* عند platform_get_resource: */
/* لا يحتاج lock — القراءة من pdev->resource[] المحلية */
```

#### 4. حماية platform bus — `platform_bus_type.p->mutex`

**الـ lock:** mutex داخلي في `subsys_private`

**يحمي:** قائمة الأجهزة والـ drivers على platform bus.

#### 5. ترتيب الـ Locks (Lock Ordering) — قاعدة منع الـ Deadlock

```
الترتيب الصحيح (من الأعلى إلى الأسفل):

1. device_lock(parent)        ← أولاً الأب
2.   device_lock(child)       ← ثم الأبناء
3.     spin_lock(devres_lock) ← ثم devres
4.       write_lock(resource_lock) ← أخيراً resources

قاعدة: لا تمسك resource_lock ثم تطلب device_lock
        لا تمسك devres_lock ثم تطلب device_lock
```

#### 6. حماية الـ DMA في SWIOTLB — `dma_io_tlb_lock`

**الـ lock:** `spinlock_t device.dma_io_tlb_lock` (عند تفعيل `CONFIG_SWIOTLB_DYNAMIC`)

**يحمي:** قائمة `dma_io_tlb_pools` — pools الـ software IO TLB المؤقتة.

---

### ملخص سريع: من يمتلك من؟

```
platform_bus_type
    │
    ├── platform_device (device على الـ bus)
    │       │
    │       ├── struct device (embedded)
    │       │       ├── kobject → sysfs
    │       │       ├── bus → &platform_bus_type
    │       │       ├── driver → platform_driver (بعد المطابقة)
    │       │       ├── devres_head → list of managed resources
    │       │       └── power → PM state
    │       │
    │       └── resource[] → struct resource (IRQ/MEM/IO)
    │                               └── يُدرج في iomem_resource tree
    │
    └── platform_driver (driver على الـ bus)
            │
            ├── struct device_driver (embedded)
            │       └── bus → &platform_bus_type
            │
            ├── probe/remove/suspend/resume callbacks
            └── id_table → platform_device_id[]
```
## Phase 4: شرح الـ Functions

---

### جدول ملخص الـ API (Cheatsheet)

#### تسجيل وحذف الـ Platform Devices

| الدالة / الـ Macro | النوع | الغرض المختصر |
|---|---|---|
| `platform_device_register` | extern | تسجيل device موجود في bus |
| `platform_device_unregister` | extern | إلغاء التسجيل وتحرير الموارد |
| `platform_device_alloc` | extern | تخصيص device جديد في الذاكرة |
| `platform_device_add` | extern | إضافة device مخصص مسبقاً إلى النظام |
| `platform_device_del` | extern | حذف device من bus دون تحرير الذاكرة |
| `platform_device_put` | extern | تخفيض refcount، تحرير عند الصفر |
| `platform_device_register_full` | extern | تسجيل كامل عبر `platform_device_info` |
| `platform_device_register_resndata` | inline | تسجيل مع resources وdata معاً |
| `platform_device_register_simple` | inline | تسجيل مبسط بدون parent/data |
| `platform_device_register_data` | inline | تسجيل مع data بدون resources |
| `platform_add_devices` | extern | تسجيل مصفوفة devices دفعة واحدة |
| `platform_device_add_resources` | extern | إضافة resources لـ device مخصص |
| `platform_device_add_data` | extern | إضافة platform_data لـ device مخصص |

#### تسجيل وحذف الـ Platform Drivers

| الدالة / الـ Macro | النوع | الغرض المختصر |
|---|---|---|
| `platform_driver_register` | macro | تسجيل driver مع THIS_MODULE |
| `__platform_driver_register` | extern | النسخة الداخلية مع module صريح |
| `platform_driver_unregister` | extern | إلغاء تسجيل driver |
| `platform_driver_probe` | macro | تسجيل driver بـ probe في `__init` |
| `__platform_driver_probe` | extern | النسخة الداخلية لـ probe |
| `platform_register_drivers` | macro | تسجيل مصفوفة drivers |
| `platform_unregister_drivers` | extern | إلغاء تسجيل مصفوفة drivers |
| `module_platform_driver` | macro | boilerplate كامل لـ module |
| `builtin_platform_driver` | macro | boilerplate لـ builtin driver |
| `module_platform_driver_probe` | macro | boilerplate لـ probe-only driver |
| `builtin_platform_driver_probe` | macro | boilerplate لـ builtin probe-only |
| `platform_create_bundle` | macro | إنشاء device+driver معاً |

#### الوصول إلى الـ Resources

| الدالة | النوع | الغرض المختصر |
|---|---|---|
| `platform_get_resource` | extern | الحصول على resource بالنوع والرقم |
| `platform_get_resource_byname` | extern | الحصول على resource بالاسم |
| `platform_get_mem_or_io` | extern | أول MEM أو IO بالترتيب |
| `platform_get_irq` | extern | رقم IRQ الـ virtual |
| `platform_get_irq_optional` | extern | IRQ اختياري (no error إن غاب) |
| `platform_get_irq_byname` | extern | IRQ بالاسم |
| `platform_get_irq_byname_optional` | extern | IRQ بالاسم، اختياري |
| `platform_irq_count` | extern | عدد IRQs المتاحة |
| `platform_get_irq_affinity` | extern | IRQ مع cpumask |
| `devm_platform_get_irqs_affinity` | extern | IRQs متعددة مع affinity، مُدارة |

#### الـ MMIO (CONFIG_HAS_IOMEM)

| الدالة | الغرض المختصر |
|---|---|
| `devm_platform_get_and_ioremap_resource` | جلب resource وعمل ioremap معاً |
| `devm_platform_ioremap_resource` | ioremap لـ resource بالرقم |
| `devm_platform_ioremap_resource_byname` | ioremap لـ resource بالاسم |

#### الـ Driver Data والـ Helpers

| الدالة / الـ Macro | الغرض المختصر |
|---|---|
| `platform_get_drvdata` | قراءة driver private data |
| `platform_set_drvdata` | كتابة driver private data |
| `platform_get_device_id` | الـ id_entry المطابق |
| `to_platform_device` | تحويل من `struct device *` إلى `platform_device *` |
| `to_platform_driver` | تحويل من `device_driver *` إلى `platform_driver *` |
| `dev_is_platform` | فحص إن كان device ينتمي لـ platform bus |

#### الـ Power Management

| الدالة | الغرض المختصر |
|---|---|
| `platform_pm_suspend` | callback الـ suspend (CONFIG_SUSPEND) |
| `platform_pm_resume` | callback الـ resume |
| `platform_pm_freeze` | callback الـ hibernate freeze |
| `platform_pm_thaw` | callback الـ thaw |
| `platform_pm_poweroff` | callback الـ poweroff |
| `platform_pm_restore` | callback الـ restore |

---

### المجموعة الأولى: تخصيص وبناء الـ Platform Device

هذه المجموعة تغطي دورة حياة الـ `platform_device` من التخصيص حتى الإضافة للـ bus، وتستخدم عادةً عندما تحتاج بناء device خطوة بخطوة (مثلاً في كود الـ board init أو MFD).

---

#### `platform_device_alloc`

```c
struct platform_device *platform_device_alloc(const char *name, int id);
```

تخصص `platform_device` جديد من الـ heap وتهيئ الـ `struct device` الداخلية بـ refcount يبدأ من 1. لا تضيفه للـ bus. يجب أن يتبعها `platform_device_add` أو `platform_device_put` عند الإلغاء.

- **`name`**: اسم الـ device (يُنسخ داخلياً).
- **`id`**: رقم الـ instance؛ يقبل `PLATFORM_DEVID_NONE` أو `PLATFORM_DEVID_AUTO`.
- **القيمة المرجعة**: مؤشر للـ device عند النجاح، `NULL` عند فشل الـ allocation.
- **تفاصيل مهمة**: الـ `dev.release` يُضبط تلقائياً لتحرير الذاكرة عند الـ `put`. استخدم `DEFINE_FREE(platform_device_put, ...)` للـ cleanup التلقائي.

---

#### `platform_device_add_resources`

```c
int platform_device_add_resources(struct platform_device *pdev,
                                  const struct resource *res,
                                  unsigned int num);
```

تنسخ مصفوفة resources للـ device المخصص مسبقاً بـ `platform_device_alloc`. تعمل قبل `platform_device_add`.

- **`pdev`**: الـ device المستهدف (لم يُضف للـ bus بعد).
- **`res`**: مصفوفة `struct resource`؛ تُنسخ داخلياً.
- **`num`**: عدد العناصر.
- **القيمة المرجعة**: `0` عند النجاح، `-ENOMEM` عند فشل الـ allocation.
- **تفاصيل مهمة**: تستبدل resources القديمة إن كانت موجودة. الكود لا يحتفظ بمؤشر `res` الأصلي.

---

#### `platform_device_add_data`

```c
int platform_device_add_data(struct platform_device *pdev,
                             const void *data, size_t size);
```

تنسخ `platform_data` للـ device، تُخزن في `pdev->dev.platform_data`. ضرورية قبل `platform_device_add` إن أردت تمرير board-specific data للـ driver.

- **`pdev`**: الـ device المستهدف.
- **`data`**: pointer للبيانات؛ تُنسخ بـ `kmemdup`.
- **`size`**: حجم البيانات بالبايت.
- **القيمة المرجعة**: `0` أو `-ENOMEM`.

---

#### `platform_device_add`

```c
int platform_device_add(struct platform_device *pdev);
```

تضيف الـ device المهيأ بـ `platform_device_alloc` إلى الـ platform bus وتسجله في الـ driver model. تطلب الـ resources وتعمل device_add الكاملة (uevent، sysfs، probe matching).

**تدفق العمل (Pseudocode):**

```c
// 1. ضبط device parent إن لم يُحدد
if (!pdev->dev.parent)
    pdev->dev.parent = &platform_bus;

// 2. ضبط bus
pdev->dev.bus = &platform_bus_type;

// 3. بناء اسم sysfs ("name.id" أو "name")
if (id == PLATFORM_DEVID_AUTO)
    assign_auto_id(pdev);
dev_set_name(&pdev->dev, "%s.%d", pdev->name, pdev->id);

// 4. طلب الـ resources من kernel resource tree
for each resource in pdev->resource:
    insert_resource(parent_resource, res);

// 5. إضافة الـ device لنظام الـ driver model
device_add(&pdev->dev);  // يطلق probe matching
```

- **القيمة المرجعة**: `0` أو كود خطأ سلبي.
- **تفاصيل مهمة**: عند الفشل، يحرر الـ resources التي طلبها. لا تحتاج قفلاً خاصاً؛ الـ device_add يدير الـ locking داخلياً.

---

#### `platform_device_del`

```c
void platform_device_del(struct platform_device *pdev);
```

تعكس `platform_device_add`: تحذف الـ device من الـ bus وتحرر resources، لكن لا تحرر ذاكرة الـ `pdev` نفسه. تستخدم مع `platform_device_put` لاحقاً.

---

#### `platform_device_put`

```c
void platform_device_put(struct platform_device *pdev);
```

تخفض الـ refcount للـ device. عند الوصول لصفر، تُستدعى الـ `release` التي تحرر الذاكرة. مكافئة لـ `put_device`.

- الـ `DEFINE_FREE(platform_device_put, ...)` يتيح استخدامها مع `__free()` للـ cleanup التلقائي في C cleanup attribute.

---

### المجموعة الثانية: التسجيل المباشر والمساعدات

هذه الدوال تجمع الخطوات السابقة في واجهة واحدة وهي الأكثر استخداماً في الـ board files والـ MFD drivers.

---

#### `platform_device_register_full`

```c
struct platform_device *platform_device_register_full(
    const struct platform_device_info *pdevinfo);
```

الدالة الأساسية التي تعتمد عليها جميع `platform_device_register_*`. تنشئ device كامل من `platform_device_info` وتسجله فوراً.

**تدفق العمل:**

```c
pdev = platform_device_alloc(pdevinfo->name, pdevinfo->id);

// ضبط parent وfwnode
pdev->dev.parent   = pdevinfo->parent;
pdev->dev.fwnode   = pdevinfo->fwnode;

// ضبط DMA mask
if (pdevinfo->dma_mask)
    setup_dma_mask(pdev, pdevinfo->dma_mask);

// نسخ resources وdata
platform_device_add_resources(pdev, pdevinfo->res, pdevinfo->num_res);
platform_device_add_data(pdev, pdevinfo->data, pdevinfo->size_data);

// إضافة properties (software nodes)
if (pdevinfo->properties)
    device_create_managed_software_node(&pdev->dev, pdevinfo->properties, NULL);

// تسجيل
platform_device_add(pdev);
return pdev;
```

- **القيمة المرجعة**: مؤشر الـ device أو `ERR_PTR(-errno)`.
- **تفاصيل مهمة**: `pdevinfo->of_node_reused` يمنع تحرير الـ OF node عند الـ unregister. مفيد في MFD حيث يشترك الـ child في نفس DT node الـ parent.

---

#### `platform_device_register_resndata` (inline)

```c
static inline struct platform_device *platform_device_register_resndata(
    struct device *parent, const char *name, int id,
    const struct resource *res, unsigned int num,
    const void *data, size_t size);
```

مجرد wrapper تبني `platform_device_info` وتستدعي `platform_device_register_full`. تستخدم عندما تريد تحديد parent وresources وdata في سطر واحد.

---

#### `platform_device_register_simple` (inline)

```c
static inline struct platform_device *platform_device_register_simple(
    const char *name, int id,
    const struct resource *res, unsigned int num);
```

النسخة الأبسط: بدون parent، بدون platform_data. مخصصة للـ legacy drivers التي تسجل hardware مباشرةً. تستدعي `platform_device_register_resndata` بـ `NULL` لكل من parent وdata.

- **تحذير**: الـ drivers التي تستخدمها لا تدعم hotplug بشكل صحيح.

---

#### `platform_device_register_data` (inline)

```c
static inline struct platform_device *platform_device_register_data(
    struct device *parent, const char *name, int id,
    const void *data, size_t size);
```

تسجيل مع parent وdata فقط، بدون resources. شائعة في MFD children التي لا تحتاج memory-mapped regions.

---

#### `platform_device_register` / `platform_device_unregister`

```c
int  platform_device_register(struct platform_device *pdev);
void platform_device_unregister(struct platform_device *pdev);
```

**الـ register**: تستدعي `device_initialize` ثم `platform_device_add` لـ device مُعرَّف statically. تستخدم للـ devices المُعرَّفة كـ static global في الـ board code.

**الـ unregister**: تستدعي `platform_device_del` ثم `platform_device_put`.

---

#### `platform_add_devices`

```c
int platform_add_devices(struct platform_device **devs, int num);
```

تسجل مصفوفة من الـ platform devices دفعة واحدة. إن فشل أي device، تتوقف ولا ترجع لتلغي السابقين.

- **`devs`**: مصفوفة مؤشرات.
- **`num`**: العدد.
- **القيمة المرجعة**: `0` أو كود خطأ أول device فاشل.

---

### المجموعة الثالثة: تسجيل وإلغاء الـ Platform Drivers

---

#### `platform_driver_register` / `__platform_driver_register`

```c
#define platform_driver_register(drv) \
    __platform_driver_register(drv, THIS_MODULE)

int __platform_driver_register(struct platform_driver *drv,
                                struct module *owner);
```

تسجل الـ driver في الـ platform bus ويُنفذ فوراً match ضد جميع الـ devices المسجلة. الـ macro يمرر `THIS_MODULE` تلقائياً لضمان صحة الـ module ownership.

- **`drv`**: يجب أن يحتوي على `driver.name` على الأقل.
- **القيمة المرجعة**: `0` أو `-errno`.
- **تفاصيل مهمة**: يضبط `drv->driver.bus = &platform_bus_type` و`drv->driver.probe = platform_drv_probe`. الـ driver لا يملك الـ device، بل يُقرن به.

---

#### `platform_driver_unregister`

```c
void platform_driver_unregister(struct platform_driver *drv);
```

تفصل الـ driver عن جميع الـ devices المقترنة (يستدعي `remove` لكل منها) ثم تحذفه من الـ bus. synchronous: لا ترجع حتى تنتهي جميع `remove` callbacks.

---

#### `platform_driver_probe` / `__platform_driver_probe`

```c
#define platform_driver_probe(drv, probe) \
    __platform_driver_probe(drv, probe, THIS_MODULE)

int __platform_driver_probe(struct platform_driver *driver,
    int (*probe)(struct platform_device *), struct module *module);
```

مخصصة للـ non-hotpluggable devices. تسجل الـ driver مع probe في `__init` section لتوفير الذاكرة بعد الـ boot. بعد نجاح الـ probe، يُزال الـ probe pointer فلا يمكن إضافة devices جديدة لاحقاً.

- **تفاصيل مهمة**: لا تستخدم إن كان الـ device قابلاً للـ hotplug. الـ `prevent_deferred_probe` يُضبط تلقائياً.

---

#### `platform_register_drivers` / `platform_unregister_drivers`

```c
#define platform_register_drivers(drivers, count) \
    __platform_register_drivers(drivers, count, THIS_MODULE)

int __platform_register_drivers(struct platform_driver * const *drivers,
                                unsigned int count, struct module *owner);
void platform_unregister_drivers(struct platform_driver * const *drivers,
                                 unsigned int count);
```

تسجل/تلغي مصفوفة من الـ drivers. `platform_register_drivers` تتوقف عند أول فشل وتلغي السابقين تلقائياً. `platform_unregister_drivers` تعمل بالترتيب العكسي.

**مثال:**

```c
static struct platform_driver * const mydrv_list[] = {
    &mydrv_uart,
    &mydrv_spi,
    &mydrv_i2c,
};

module_init:
    return platform_register_drivers(mydrv_list, ARRAY_SIZE(mydrv_list));

module_exit:
    platform_unregister_drivers(mydrv_list, ARRAY_SIZE(mydrv_list));
```

---

### المجموعة الرابعة: الـ Macros للـ Module Boilerplate

---

#### `module_platform_driver`

```c
#define module_platform_driver(__platform_driver) \
    module_driver(__platform_driver, platform_driver_register, \
                  platform_driver_unregister)
```

يولد `module_init` و`module_exit` تلقائياً. الأكثر شيوعاً في كود الـ drivers. يكافئ:

```c
static int __init mydrv_init(void) {
    return platform_driver_register(&mydrv);
}
module_init(mydrv_init);

static void __exit mydrv_exit(void) {
    platform_driver_unregister(&mydrv);
}
module_exit(mydrv_exit);
```

**مثال حقيقي (i2c-gpio driver):**
```c
static struct platform_driver i2c_gpio_driver = {
    .driver = { .name = "i2c-gpio", .of_match_table = i2c_gpio_dt_ids },
    .probe  = i2c_gpio_probe,
    .remove = i2c_gpio_remove,
};
module_platform_driver(i2c_gpio_driver);
```

---

#### `builtin_platform_driver`

```c
#define builtin_platform_driver(__platform_driver) \
    builtin_driver(__platform_driver, platform_driver_register)
```

مثل `module_platform_driver` لكن للـ drivers المدمجة في الـ kernel (لا `module_exit`). يستخدم `device_initcall` بدلاً من `module_init`.

---

#### `module_platform_driver_probe`

```c
#define module_platform_driver_probe(__platform_driver, __platform_probe)
```

يجمع `module_platform_driver` مع `platform_driver_probe` في macro واحد. للـ drivers التي تضع الـ probe في `__init` لتوفير ذاكرة الـ runtime.

---

#### `platform_create_bundle`

```c
#define platform_create_bundle(driver, probe, res, n_res, data, size) \
    __platform_create_bundle(driver, probe, res, n_res, data, size, THIS_MODULE)

struct platform_device *__platform_create_bundle(
    struct platform_driver *driver,
    int (*probe)(struct platform_device *),
    struct resource *res, unsigned int n_res,
    const void *data, size_t size,
    struct module *module);
```

تسجل device وdriver معاً دفعة واحدة (bundle). مفيدة للـ legacy hardware حيث الـ device وDriver مرتبطان بشكل ثابت.

**تدفق العمل:**
```c
pdev = platform_device_register_resndata(NULL, driver->driver.name,
                                          PLATFORM_DEVID_NONE,
                                          res, n_res, data, size);
__platform_driver_probe(driver, probe, module);
return pdev;
```

- **القيمة المرجعة**: مؤشر الـ device أو `ERR_PTR`.
- **تحذير**: الـ probe يُنفذ في `__init`، لذا لا يدعم hotplug.

---

### المجموعة الخامسة: الوصول إلى الـ Resources

---

#### `platform_get_resource`

```c
struct resource *platform_get_resource(struct platform_device *dev,
                                        unsigned int type,
                                        unsigned int num);
```

تبحث في مصفوفة resources الخاصة بالـ device عن resource بنوع `type` ورقم تسلسلي `num`. تُستخدم داخلياً بواسطة دوال الـ IRQ والـ MMIO.

- **`type`**: `IORESOURCE_MEM`, `IORESOURCE_IRQ`, `IORESOURCE_IO`, إلخ.
- **`num`**: الرقم التسلسلي بين resources من نفس النوع (يبدأ من 0).
- **القيمة المرجعة**: مؤشر لـ `struct resource` أو `NULL` إن لم يُوجد.
- **تفاصيل مهمة**: المؤشر المُرجع يشير مباشرة للـ resource في مصفوفة الـ device، لا تحرره.

**مثال:**
```c
struct resource *mem = platform_get_resource(pdev, IORESOURCE_MEM, 0);
/* الحصول على أول memory region */
```

---

#### `platform_get_resource_byname`

```c
struct resource *platform_get_resource_byname(struct platform_device *dev,
                                               unsigned int type,
                                               const char *name);
```

مثل `platform_get_resource` لكن تبحث بالاسم بدلاً من الرقم. تحتاج أن تكون الـ resources مُسماة في board data أو DT.

```c
struct resource *res = platform_get_resource_byname(pdev, IORESOURCE_MEM, "regs");
```

---

#### `platform_get_mem_or_io`

```c
struct resource *platform_get_mem_or_io(struct platform_device *dev,
                                         unsigned int num);
```

تجلب resource بالرقم التسلسلي بغض النظر عن النوع (MEM أو IO). مفيدة للـ drivers التي تدعم كلا النوعين.

---

#### `platform_get_irq`

```c
int platform_get_irq(struct platform_device *dev, unsigned int num);
```

تجلب الـ virtual IRQ number للـ interrupt رقم `num`. تتعامل مع كل من:
- Resources مُضافة مباشرةً (`IORESOURCE_IRQ`)
- الـ DT `interrupts` property عبر `irq_of_parse_and_map`
- الـ ACPI interrupt resources

- **`num`**: الرقم التسلسلي للـ interrupt (يبدأ من 0).
- **القيمة المرجعة**: virtual IRQ number موجب عند النجاح. قيم سالبة: `-ENXIO` (غير موجود)، `-EPROBE_DEFER` (يحتاج إعادة محاولة لاحقاً).
- **تفاصيل مهمة**: لا تستخدم في `IRQF_SHARED` scenarios بدون فحص مسبق. الـ `-EPROBE_DEFER` يجب أن يُعاد من `probe` مباشرةً.

**مثال:**
```c
int irq = platform_get_irq(pdev, 0);
if (irq < 0)
    return irq;  /* يشمل EPROBE_DEFER */
ret = devm_request_irq(&pdev->dev, irq, my_handler, 0, "mydev", priv);
```

---

#### `platform_get_irq_optional`

```c
int platform_get_irq_optional(struct platform_device *dev, unsigned int num);
```

مثل `platform_get_irq` لكن لا تطبع error message إن لم يُوجد الـ IRQ. تُستخدم للـ IRQs الاختيارية التي قد تكون غائبة في بعض board variants.

- **القيمة المرجعة**: virtual IRQ أو `0` إن غاب، أو قيمة سالبة لأخطاء أخرى.

---

#### `platform_get_irq_byname`

```c
int platform_get_irq_byname(struct platform_device *dev, const char *name);
```

تجلب IRQ بالاسم من DT `interrupt-names` property أو resource مُسماة. تطبع error إن غاب.

```c
int irq = platform_get_irq_byname(pdev, "tx");
/* من DT: interrupts = <...>; interrupt-names = "rx", "tx"; */
```

---

#### `platform_get_irq_byname_optional`

```c
int platform_get_irq_byname_optional(struct platform_device *dev,
                                      const char *name);
```

نفس `platform_get_irq_byname` لكن بدون error log إن غاب الاسم.

---

#### `platform_irq_count`

```c
int platform_irq_count(struct platform_device *dev);
```

تُرجع إجمالي عدد الـ IRQs المعرفة للـ device (من resources أو DT). تستخدم عند الحاجة لمعالجة عدد متغير من الـ interrupts.

---

#### `platform_get_irq_affinity`

```c
int platform_get_irq_affinity(struct platform_device *dev,
                               unsigned int num,
                               const struct cpumask **affinity);
```

تجلب الـ IRQ مع الـ CPU affinity mask المرتبطة به. تستخدم في الـ multi-queue drivers (مثل NVMe، network) التي تريد توزيع الـ interrupts على الـ cores.

- **`affinity`**: مؤشر لـ `cpumask`، يُملأ بالـ CPUs المخصصة لهذا الـ IRQ.

---

#### `devm_platform_get_irqs_affinity`

```c
int devm_platform_get_irqs_affinity(struct platform_device *dev,
                                     struct irq_affinity *affd,
                                     unsigned int minvec,
                                     unsigned int maxvec,
                                     int **irqs);
```

تخصص وتجلب مصفوفة IRQs مع affinity للـ multi-queue scenarios. المصفوفة مُدارة بـ devm.

- **`affd`**: وصف الـ affinity المطلوب.
- **`minvec`** / **`maxvec`**: الحد الأدنى والأقصى للـ vectors.
- **`irqs`**: مؤشر مزدوج يُملأ بمصفوفة الـ IRQ numbers.
- **القيمة المرجعة**: عدد الـ IRQs المخصصة أو كود خطأ سالب.

---

### المجموعة السادسة: الـ MMIO (Memory-Mapped I/O)

تعمل هذه الدوال فقط عند تفعيل `CONFIG_HAS_IOMEM`، وإلا ترجع `IOMEM_ERR_PTR(-EINVAL)`.

---

#### `devm_platform_get_and_ioremap_resource`

```c
void __iomem *devm_platform_get_and_ioremap_resource(
    struct platform_device *pdev,
    unsigned int index,
    struct resource **res);
```

تجلب الـ memory resource رقم `index` وتعمل `ioremap` عليها في خطوة واحدة. الـ mapping مُدار بـ devm ويُحرر تلقائياً عند `remove`. تُملأ `*res` بمؤشر الـ resource إن أردت حجمها أو عنوانها الفيزيائي.

- **القيمة المرجعة**: مؤشر `__iomem` أو `IOMEM_ERR_PTR(-errno)`.
- **تفاصيل مهمة**: تتحقق من أن الـ resource من نوع `IORESOURCE_MEM`. تستدعي `devm_ioremap_resource` داخلياً.

**مثال:**
```c
struct resource *res;
void __iomem *base = devm_platform_get_and_ioremap_resource(pdev, 0, &res);
if (IS_ERR(base))
    return PTR_ERR(base);
priv->regsize = resource_size(res);  /* استخدام res للحجم */
```

---

#### `devm_platform_ioremap_resource`

```c
void __iomem *devm_platform_ioremap_resource(struct platform_device *pdev,
                                              unsigned int index);
```

مثل السابقة لكن لا تُرجع الـ resource pointer. تستخدم حين لا تحتاج بيانات الـ resource (الحجم أو العنوان الفيزيائي). الأكثر استخداماً في الـ drivers.

```c
priv->base = devm_platform_ioremap_resource(pdev, 0);
if (IS_ERR(priv->base))
    return PTR_ERR(priv->base);
```

---

#### `devm_platform_ioremap_resource_byname`

```c
void __iomem *devm_platform_ioremap_resource_byname(
    struct platform_device *pdev,
    const char *name);
```

نفس `devm_platform_ioremap_resource` لكن تحدد الـ resource بالاسم. تستخدم مع DT nodes التي تحدد `reg-names`.

```c
/* من DT: reg = <0x1000 0x100>, <0x2000 0x100>;
          reg-names = "ctrl", "data"; */
priv->ctrl = devm_platform_ioremap_resource_byname(pdev, "ctrl");
priv->data = devm_platform_ioremap_resource_byname(pdev, "data");
```

---

### المجموعة السابعة: الـ Driver Data والـ Helpers

---

#### `platform_get_drvdata` / `platform_set_drvdata`

```c
static inline void *platform_get_drvdata(const struct platform_device *pdev)
{
    return dev_get_drvdata(&pdev->dev);
}

static inline void platform_set_drvdata(struct platform_device *pdev,
                                         void *data)
{
    dev_set_drvdata(&pdev->dev, data);
}
```

الدالتان الأساسيتان لتخزين واسترجاع الـ driver-private data. **`platform_set_drvdata`** تستخدم في بداية `probe` بعد تخصيص الـ private struct. **`platform_get_drvdata`** تستخدم في كل الـ callbacks (remove، IRQ handler، pm).

**مثال نمطي:**
```c
static int mydrv_probe(struct platform_device *pdev)
{
    struct mydrv_priv *priv;
    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    platform_set_drvdata(pdev, priv);  /* حفظ pointer */
    /* ... */
}

static void mydrv_remove(struct platform_device *pdev)
{
    struct mydrv_priv *priv = platform_get_drvdata(pdev);  /* استرجاعه */
    /* تحرير الموارد */
}
```

---

#### `platform_find_device_by_driver`

```c
struct device *platform_find_device_by_driver(struct device *start,
                                               const struct device_driver *drv);
```

تبحث في platform bus عن device مقترن بـ driver محدد. مفيدة في سيناريوهات الـ cross-driver communication حيث driver يحتاج device مسجل بـ driver آخر.

- **`start`**: البداية (بعد هذا الـ device)؛ `NULL` للبدء من الأول.
- **القيمة المرجعة**: مؤشر device مع refcount مرفوع، أو `NULL`. يجب استدعاء `put_device` بعد الاستخدام.

---

#### `to_platform_device` / `to_platform_driver`

```c
#define to_platform_device(x) container_of((x), struct platform_device, dev)
#define to_platform_driver(drv) \
    container_of((drv), struct platform_driver, driver)
```

**الـ `to_platform_device`** يستخدم في callbacks الـ bus (مثل `dev_pm_ops`) التي تستقبل `struct device *` وتحتاج الـ `platform_device`.

**الـ `to_platform_driver`** يستخدم في bus internals للوصول للـ `platform_driver` من `device_driver`.

---

#### `dev_is_platform`

```c
#define dev_is_platform(dev) ((dev)->bus == &platform_bus_type)
```

فحص بسيط للتأكد من أن الـ device ينتمي لـ platform bus. يستخدم في كود generic يتعامل مع أنواع buses مختلفة.

---

#### `platform_get_device_id`

```c
#define platform_get_device_id(pdev) ((pdev)->id_entry)
```

يُرجع مؤشر لـ `platform_device_id` التي طابقت الـ device عند الـ probe. يستخدم حين driver يدعم أجهزة متعددة بـ `id_table` ويحتاج معرفة النوع الدقيق في الـ probe.

```c
static int mydrv_probe(struct platform_device *pdev)
{
    const struct platform_device_id *id = platform_get_device_id(pdev);
    if (id)
        priv->variant = id->driver_data;  /* بيانات الـ variant */
}
```

---

### المجموعة الثامنة: الـ Power Management

هذه الدوال تعمل كـ bridge بين الـ generic PM framework وـ platform driver callbacks.

---

#### `platform_pm_suspend` / `platform_pm_resume`

```c
int platform_pm_suspend(struct device *dev);
int platform_pm_resume(struct device *dev);
```

(تتوفر فقط مع `CONFIG_SUSPEND`)

تستدعيان `suspend`/`resume` callbacks في الـ `platform_driver`. الـ `platform_pm_suspend` تستدعي `driver->pm->suspend` أولاً إن وُجدت، وإلا تلجأ لـ `platform_driver->suspend` للتوافق مع الـ legacy API.

**تفاصيل مهمة**: الـ `USE_PLATFORM_PM_SLEEP_OPS` macro يُضاف لـ `dev_pm_ops` لتوصيل هذه الدوال:

```c
static const struct dev_pm_ops mydrv_pm = {
    USE_PLATFORM_PM_SLEEP_OPS
    /* أو SET_SYSTEM_SLEEP_PM_OPS() للـ runtime pm أيضاً */
};
```

---

#### `platform_pm_freeze` / `platform_pm_thaw` / `platform_pm_poweroff` / `platform_pm_restore`

```c
int platform_pm_freeze(struct device *dev);
int platform_pm_thaw(struct device *dev);
int platform_pm_poweroff(struct device *dev);
int platform_pm_restore(struct device *dev);
```

(تتوفر فقط مع `CONFIG_HIBERNATE_CALLBACKS`)

تغطي دورة الـ hibernation الكاملة:
- **`freeze`**: تجميد الـ device قبل حفظ image الذاكرة.
- **`thaw`**: إعادة التشغيل بعد فشل الـ hibernation أو عند resume من الـ image.
- **`poweroff`**: إطفاء الـ device بعد حفظ الـ image.
- **`restore`**: استعادة حالة الـ device بعد تحميل الـ image.

---

### مخطط دورة حياة الـ Platform Device

```
          ┌─────────────────────────────────────────────────┐
          │              طريقتا الإنشاء                     │
          │                                                 │
          │  Static (board code)        Dynamic (MFD/etc)  │
          │  ────────────────────        ──────────────────  │
          │  platform_device_register   platform_device_alloc│
          │         │                        │              │
          │         │               platform_device_add_res │
          │         │               platform_device_add_data│
          │         │                        │              │
          │         └──────────┬─────────────┘              │
          │                    ▼                            │
          │          platform_device_add                    │
          │          (إضافة للـ bus + probe matching)        │
          │                    │                            │
          │                    ▼                            │
          │         ┌──────────────────┐                   │
          │         │  Device مسجل في │                   │
          │         │  platform bus    │                   │
          │         └──────────────────┘                   │
          │                    │                            │
          │         ┌──────────┴──────────┐                │
          │         │ platform_driver     │                 │
          │         │ register            │                 │
          │         │ ──► probe()         │                 │
          │         └─────────────────────┘                │
          │                    │                            │
          │          عند الـ cleanup:                       │
          │          platform_device_del                    │
          │          platform_device_put                    │
          │          ──► release() ──► kfree                │
          └─────────────────────────────────────────────────┘
```

---

### مخطط الـ Resource Access في الـ probe

```
probe(pdev)
    │
    ├─► devm_platform_ioremap_resource(pdev, 0)
    │       │
    │       ├─ platform_get_resource(pdev, IORESOURCE_MEM, 0)
    │       └─ devm_ioremap_resource(dev, res)
    │               └─► __iomem * (مُحرر تلقائياً عند remove)
    │
    ├─► platform_get_irq(pdev, 0)
    │       │
    │       ├─ [DT] irq_of_parse_and_map()
    │       ├─ [ACPI] acpi_irq_get()
    │       └─ [resource] res->start  ──► virtual IRQ number
    │
    └─► platform_get_drvdata / platform_set_drvdata
            └─ dev->driver_data (في struct device)
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بـ platform_device

**الـ debugfs** يُوفّر رؤية مباشرة في هياكل الـ driver model الداخلية.

```bash
# تركيب الـ debugfs إن لم يكن مركّباً
mount -t debugfs none /sys/kernel/debug

# عرض شجرة الـ devices المرتبطة بـ platform bus
ls /sys/kernel/debug/devices/

# عرض معلومات الـ devm (managed resources) لجهاز معيّن
# اسم الجهاز يكون على صيغة: <name>.<id>  مثل  serial8250.0
cat /sys/kernel/debug/devices/<name>.<id>/resources

# عرض الـ driver_override المضبوط على الجهاز
cat /sys/kernel/debug/devices/<name>.<id>/driver_override

# متابعة الـ deferred probe
cat /sys/kernel/debug/driver_deferred_probe_pending_list
cat /sys/kernel/debug/driver_deferred_probe_trigger
```

| الملف | ما يُظهره |
|---|---|
| `/sys/kernel/debug/driver_deferred_probe_pending_list` | الأجهزة التي لم تجد driver بعد (deferred probe) |
| `/sys/kernel/debug/driver_deferred_probe_trigger` | كتابة أي شيء فيه يُعيد تشغيل دورة الـ probe |
| `/sys/kernel/debug/devices/<dev>/resources` | الـ resources المحجوزة لكل جهاز |
| `/sys/kernel/debug/regulator/` | إن كان الجهاز يستخدم regulator |
| `/sys/kernel/debug/clk/` | إن كان الجهاز يستخدم clock framework |

---

#### 2. مدخلات الـ sysfs المتعلقة بـ platform_device

**الـ sysfs** يعكس بنية `struct platform_device` مباشرةً.

```bash
# عرض جميع platform devices
ls /sys/bus/platform/devices/

# معلومات جهاز معيّن (مثال: serial8250.0)
PDEV=/sys/bus/platform/devices/serial8250.0

cat $PDEV/driver_override        # الـ driver المُجبَر
ls  $PDEV/driver                 # symlink للـ driver الحالي (أو غائب إن لم يُربط)
ls  $PDEV/of_node                # symlink لعقدة الـ Device Tree
cat $PDEV/modalias               # صيغة platform:<name> تُستخدم للـ hotplug
cat $PDEV/uevent                 # متغيرات البيئة عند الـ hotplug

# عرض الـ resources المخصصة
cat $PDEV/resource               # (PCI-style, غير موجود على كل الأجهزة)

# عرض المعلومات الهيكلية للـ device
cat $PDEV/subsystem_vendor
cat $PDEV/subsystem_device

# عرض جميع drivers على platform bus
ls /sys/bus/platform/drivers/

# ربط/فك ربط driver يدوياً
echo "serial8250.0" > /sys/bus/platform/drivers/serial8250/bind
echo "serial8250.0" > /sys/bus/platform/drivers/serial8250/unbind

# إجبار driver معيّن
echo "serial8250" > /sys/bus/platform/devices/serial8250.0/driver_override
echo "serial8250.0" > /sys/bus/platform/drivers_probe
```

| المسار في sysfs | الوصف |
|---|---|
| `/sys/bus/platform/devices/` | جميع platform devices المسجّلة |
| `/sys/bus/platform/drivers/` | جميع platform drivers المسجّلة |
| `<dev>/driver` | symlink للـ driver المرتبط حالياً |
| `<dev>/driver_override` | إجبار ربط driver محدد |
| `<dev>/of_node` | symlink لعقدة DT المقابلة |
| `<dev>/modalias` | `platform:<name>` للـ module autoloading |
| `<dev>/power/` | حالة الـ PM (runtime_status, wakeup, ...) |
| `/sys/bus/platform/drivers_probe` | كتابة اسم device يُعيد تشغيل probe |
| `/sys/bus/platform/drivers_autoprobe` | تمكين/تعطيل الـ autoprobe |

---

#### 3. استخدام الـ ftrace مع platform_device

**الـ ftrace** يسمح بتتبع مسار الـ probe ودورة حياة الـ platform device.

```bash
# تفعيل tracing filesystem
mount -t tracefs nodev /sys/kernel/tracing

TRACE=/sys/kernel/tracing

# --- تفعيل أحداث driver model ---
echo 1 > $TRACE/events/device/enable
echo 1 > $TRACE/events/devres/enable

# تتبع أحداث platform driver بالتفصيل
echo 1 > $TRACE/events/platform/enable  2>/dev/null || \
  echo "platform tracepoint غير متاح، استخدم function tracer"

# --- استخدام function_graph لتتبع دالة probe ---
echo function_graph > $TRACE/current_tracer
echo "platform_drv_probe" > $TRACE/set_graph_function
echo 1 > $TRACE/tracing_on
# ... تشغيل modprobe أو rebind ...
echo 0 > $TRACE/tracing_on
cat $TRACE/trace | head -100

# --- تتبع دوال platform_device الأساسية ---
echo function > $TRACE/current_tracer
echo "platform_device_register
platform_device_unregister
platform_drv_probe
platform_drv_remove
devm_platform_ioremap_resource
platform_get_irq
platform_get_resource" > $TRACE/set_ftrace_filter

echo 1 > $TRACE/tracing_on
# ... العملية المطلوبة ...
echo 0 > $TRACE/tracing_on
cat $TRACE/trace

# --- أحداث IRQ المرتبطة بالجهاز ---
echo 1 > $TRACE/events/irq/irq_handler_entry/enable
echo 1 > $TRACE/events/irq/irq_handler_exit/enable

# --- أحداث الـ PM ---
echo 1 > $TRACE/events/power/device_pm_callback_start/enable
echo 1 > $TRACE/events/power/device_pm_callback_end/enable
```

مثال على ناتج `function_graph` عند تسجيل جهاز:

```
 1)               |  platform_device_register() {
 1)               |    platform_device_add() {
 1)               |      device_add() {
 1)   0.520 us    |        bus_probe_device();
 1)   0.110 us    |        kobject_uevent();
 1)   1.340 us    |      }
 1)   1.890 us    |    }
 1)   2.100 us    |  }
```

---

#### 4. تفعيل الـ printk / dynamic debug

```bash
# --- تفعيل dynamic debug لـ platform bus كاملاً ---
# الملف المصدري الرئيسي في الكرنل: drivers/base/platform.c
echo "file drivers/base/platform.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل رسائل drivers/base/
echo "file drivers/base/* +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل رسائل دالة probe فقط
echo "func platform_drv_probe +p" > /sys/kernel/debug/dynamic_debug/control

# عرض القواعد الفعّالة حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep platform

# --- ضبط مستوى الـ printk لرؤية رسائل KERN_DEBUG ---
echo 8 > /proc/sys/kernel/printk
# أو
dmesg -n 8

# --- مشاهدة الرسائل فور صدورها ---
dmesg -w | grep -i platform

# --- تضمين الـ timestamp في رسائل الكرنل ---
echo Y > /sys/module/printk/parameters/time
```

الـ flags في dynamic_debug:
| Flag | المعنى |
|---|---|
| `p` | طباعة الرسالة |
| `f` | إضافة اسم الدالة |
| `l` | إضافة رقم السطر |
| `m` | إضافة اسم الـ module |
| `t` | إضافة الـ thread id |

---

#### 5. خيارات الـ Kconfig لتصحيح أخطاء platform_device

```
CONFIG_DEBUG_DRIVER=y          # رسائل debug مفصّلة من driver core
CONFIG_DEBUG_DEVRES=y          # تتبع عمليات devm_* (managed resources)
CONFIG_DEBUG_KOBJECT=y         # تتبع حياة الـ kobject
CONFIG_DEBUG_KOBJECT_RELEASE=y # اكتشاف use-after-free في kobject
CONFIG_PROVE_LOCKING=y         # lockdep لاكتشاف deadlocks
CONFIG_DEBUG_LOCK_ALLOC=y      # تتبع تخصيص الـ locks
CONFIG_KASAN=y                 # اكتشاف memory bugs (heap)
CONFIG_KMSAN=y                 # اكتشاف uninit memory
CONFIG_DYNAMIC_DEBUG=y         # تفعيل dynamic debug
CONFIG_FTRACE=y                # تفعيل function tracer
CONFIG_FUNCTION_TRACER=y       # tracing الدوال
CONFIG_FUNCTION_GRAPH_TRACER=y # رسم شجرة استدعاء الدوال
CONFIG_TRACEPOINTS=y           # tracepoints في الكرنل
CONFIG_DEBUG_SHIRQ=y           # اختبار shared IRQs عند الـ shutdown
CONFIG_GENERIC_IRQ_DEBUGFS=y   # مدخلات debugfs للـ IRQ subsystem
CONFIG_OF_DEBUG=y              # رسائل debug لـ Device Tree
CONFIG_PM_DEBUG=y              # رسائل debug لـ Power Management
CONFIG_PM_ADVANCED_DEBUG=y     # معلومات PM إضافية في sysfs
CONFIG_DEFERRED_STRUCT_PAGE_INIT=n # لتجنب تأخير الـ init
CONFIG_COMPILE_TEST=y          # للبناء بدون hardware حقيقي
```

---

#### 6. أدوات خاصة بالـ subsystem

```bash
# --- عرض شجرة devices كاملة ---
lsplatform() { find /sys/bus/platform/devices -maxdepth 1 -type l \
  -exec basename {} \; | sort; }

# --- أداة udevadm لتحليل أحداث الـ uevent ---
udevadm monitor --kernel --udev &
# ثم تسجيل الجهاز أو إزالته

# --- معلومات تفصيلية لجهاز ---
udevadm info --attribute-walk --path=/sys/bus/platform/devices/serial8250.0

# --- اختبار الـ modalias وتحميل الـ module تلقائياً ---
cat /sys/bus/platform/devices/my_device.0/modalias
# ناتج: platform:my_device
modprobe $(cat /sys/bus/platform/devices/my_device.0/modalias | sed 's/platform://')

# --- عرض الـ resources من /proc ---
cat /proc/iomem          # ذاكرة I/O المحجوزة
cat /proc/ioports        # منافذ I/O المحجوزة
cat /proc/interrupts     # IRQs الفعّالة وعداداتها

# --- أداة devmem2 (خارج الكرنل) لقراءة السجلات مباشرة ---
# devmem2 <address> [type] [data]
devmem2 0xFF200000 w        # قراءة 32-bit من العنوان

# --- فحص الـ DMA mask ---
cat /sys/bus/platform/devices/<dev>/dma_mask
```

---

#### 7. جدول رسائل الأخطاء الشائعة ومعناها وطريقة الإصلاح

| رسالة الكرنل | السبب | الإصلاح |
|---|---|---|
| `platform <name>: Driver <drv> requests probe deferral` | resource غير جاهز (clock, regulator, GPIO) | انتظار تسجيل الـ dependency أولاً؛ تحقق من `deferred_probe_pending_list` |
| `platform <name>: probe with driver <drv> failed with error -EPROBE_DEFER` | نفس السبب أعلاه، الكرنل أعاد الجدولة | طبيعي إن حُلّ لاحقاً؛ إن استمر: تحقق من ترتيب initcall |
| `platform <name>: probe with driver <drv> failed with error -ENOMEM` | فشل تخصيص الذاكرة في probe | راجع كميات الـ devm allocations؛ فعّل `CONFIG_KASAN` |
| `platform <name>: probe with driver <drv> failed with error -ENXIO` | resource غير موجود (IRQ/MEM) | تحقق من DT أو board file؛ تحقق من `platform_get_resource` |
| `platform <name>: probe with driver <drv> failed with error -ENODEV` | الجهاز غير مدعوم أو ID لا يتطابق | تحقق من `id_table` في الـ driver |
| `Driver <drv> requests no_deferred_probe` | الـ driver ضبط `prevent_deferred_probe=true` | يفشل فوراً بدل الانتظار؛ تحقق من صحة الـ resources في DT |
| `platform <name>: IRQ index 0 not found` | `platform_get_irq()` فشل، لا يوجد IRQ في resources | أضف `interrupts` property في DT أو board file |
| `devm_ioremap_resource: resource [mem ...] busy` | عنوان الذاكرة محجوز لجهاز آخر | تحقق من `/proc/iomem`؛ لا تتداخل resources بين أجهزة |
| `<name> <name>.N: Unable to handle kernel paging request at <addr>` | استخدام مؤشر __iomem بدون ioremap | استخدم `devm_platform_ioremap_resource()` دائماً |
| `platform <name>: DMA mask not set` | لم يُضبط `dma_mask` أو `coherent_dma_mask` | اضبط `platform_device_info.dma_mask` أو استدعِ `dma_set_mask_and_coherent()` |
| `kobject: '<name>': fill_kobj_path: bad kobj` | تسجيل جهاز بعد تحرير كائنه | تتبع دورة حياة `platform_device_alloc/put` |
| `platform: failed to claim resource 1` | تعارض resources عند `platform_device_add()` | تحقق من عدم تسجيل جهازين بنفس المنطقة |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

يُنصح بإضافة هذه النقاط عند تطوير driver جديد أو تصحيح خطأ عميق:

```c
/* في probe() — التحقق من أن drvdata يُضبط صحيحاً */
static int my_driver_probe(struct platform_device *pdev)
{
    struct my_priv *priv;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    WARN_ON(!priv);  /* يجب ألا يحدث؛ إن حدث: مشكلة ذاكرة */

    platform_set_drvdata(pdev, priv);

    /* التحقق من أن resource موجود قبل ioremap */
    priv->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(priv->base)) {
        /* dump_stack() هنا يُظهر من استدعى probe */
        dump_stack();
        return PTR_ERR(priv->base);
    }

    return 0;
}

/* في remove() — التحقق من عدم وجود استخدام جارٍ */
static void my_driver_remove(struct platform_device *pdev)
{
    struct my_priv *priv = platform_get_drvdata(pdev);

    /* يجب ألا يكون NULL هنا أبداً */
    WARN_ON(!priv);

    /* التحقق من إيقاف العمليات قبل الإزالة */
    WARN_ON(atomic_read(&priv->active_transfers) != 0);
}

/* في interrupt handler — التحقق من تزامن الحالة */
static irqreturn_t my_isr(int irq, void *dev_id)
{
    struct my_priv *priv = dev_id;

    /* WARN_ON_ONCE لعدم إغراق الـ log */
    WARN_ON_ONCE(!priv->hw_initialized);

    return IRQ_HANDLED;
}

/* عند الوصول لحالة مستحيلة في state machine */
    default:
        WARN(1, "unexpected state %d in %s\n", state, __func__);
        dump_stack();
        break;
```

**نقاط `WARN_ON()` الاستراتيجية في دورة حياة platform_device:**

```
platform_device_alloc()
       │
       ├─ WARN_ON(id == PLATFORM_DEVID_AUTO && !name)
       │
platform_device_add_resources()
       │
       ├─ WARN_ON(pdev->resource != NULL)  /* تأكد عدم إضافة مرتين */
       │
platform_device_register()
       │
       ├─ WARN_ON(in_interrupt())  /* يجب ألا يُستدعى من interrupt context */
       │
probe()
       │
       ├─ WARN_ON(!platform_get_drvdata(pdev) == NULL بعد الإعداد)
       │
remove()
       │
       └─ WARN_ON(platform_get_drvdata(pdev) == NULL)
```

---

### Hardware Level

---

#### 1. التحقق من تطابق حالة الـ Hardware مع حالة الكرنل

```bash
# مقارنة العناوين في DT مع /proc/iomem
# مثال: جهاز UART عند 0xFF200000
grep -i uart /proc/iomem
# المتوقع: ff200000-ff2000ff : /soc/serial@ff200000

# التحقق من IRQ
grep -i "serial\|uart" /proc/interrupts
# العمود الأول هو رقم الـ IRQ؛ يجب أن يتطابق مع DT

# التحقق من أن الـ clock يعمل (مثال: clk framework)
cat /sys/kernel/debug/clk/uart_clk/clk_rate
cat /sys/kernel/debug/clk/uart_clk/clk_enable_count

# التحقق من حالة الـ regulator إن وُجد
cat /sys/kernel/debug/regulator/vcc_uart/state
cat /sys/kernel/debug/regulator/vcc_uart/voltage

# قراءة رقم سجل يُعكس الـ hardware reset state
devmem2 0xFF200000 w     # يجب أن يطابق reset value في datasheet
```

---

#### 2. تقنيات قراءة السجلات (Register Dump)

```bash
# --- devmem2 (أداة userspace شائعة) ---
# تثبيت: apt install devmem2  أو  buildroot
devmem2 0xFF200000 b    # قراءة byte
devmem2 0xFF200000 h    # قراءة halfword (16-bit)
devmem2 0xFF200000 w    # قراءة word (32-bit)
devmem2 0xFF200000 w 0xDEADBEEF  # كتابة word

# --- /dev/mem مباشرة بـ Python ---
python3 - <<'EOF'
import mmap, struct, os

PHYS_ADDR = 0xFF200000
SIZE       = 0x100

fd = os.open("/dev/mem", os.O_RDWR | os.O_SYNC)
mem = mmap.mmap(fd, SIZE, offset=PHYS_ADDR)

for offset in range(0, SIZE, 4):
    val = struct.unpack_from("<I", mem, offset)[0]
    print(f"  0x{PHYS_ADDR + offset:08X}: 0x{val:08X}")

mem.close()
os.close(fd)
EOF

# --- io utility (من busybox) ---
io -4 -r 0xFF200000      # قراءة 32-bit
io -4 -w 0xFF200000 0x1  # كتابة 32-bit

# --- من داخل kernel module (للتصحيح فقط) ---
# في kernel code:
void __iomem *base = ioremap(0xFF200000, 0x100);
pr_info("REG0: 0x%08x\n", readl(base + 0x00));
pr_info("REG4: 0x%08x\n", readl(base + 0x04));
iounmap(base);
```

**تنبيه:** `/dev/mem` يتطلب `CONFIG_DEVMEM=y` وقد يكون محدوداً بـ `CONFIG_STRICT_DEVMEM=y`. استخدم `CONFIG_DEVMEM=y` و `CONFIG_STRICT_DEVMEM=n` في بيئة التطوير فقط.

---

#### 3. نصائح استخدام Logic Analyzer / Oscilloscope

**عند تصحيح أخطاء بروتوكول I2C/SPI/UART الذي يُدار عبر platform_device:**

```
Logic Analyzer Connections:
─────────────────────────────────────────────────────
Signal          | Pin              | الهدف
─────────────────────────────────────────────────────
IRQ line        | GPIO_IRQ_PIN     | تحقق من توقيت الـ interrupt
CS/CE           | SPI_CS           | تحقق من deassert timing
RESET           | RESET_GPIO       | طول نبضة الـ reset (≥ datasheet min)
POWER_GOOD      | PMIC_PG          | هل يكون HIGH قبل probe؟
─────────────────────────────────────────────────────

Oscilloscope Tips:
- قِس الـ rise time لخط الـ IRQ: يجب أن يكون < 1µs لمعظم MCU peripherals
- تحقق من الـ voltage levels: 1.8V vs 3.3V vs 5V — mismatch يُسبب صمت الجهاز
- راقب الـ power rail أثناء probe(): انهيار الجهد يدل على overcurrent
- trigger على RESET falling edge لقياس setup time قبل أول register access
```

**ربط زمن الـ ftrace مع الـ logic analyzer:**

```bash
# أضف GPIO toggle في بداية probe() ونهايتها (للتزامن مع LA)
# في الكود:
gpiod_set_value(priv->debug_gpio, 1);  /* probe start */
/* ... probe body ... */
gpiod_set_value(priv->debug_gpio, 0);  /* probe end */
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في kernel log

| المشكلة | النمط في dmesg | التشخيص |
|---|---|---|
| الـ resource غير متاح (clock gate) | `probe with driver X failed with error -ENODEV` أو سكوت تام | تحقق من `clk_enable_count` في debugfs |
| خطأ في عنوان MEM في DT | `devm_ioremap_resource: resource [mem A-B] busy` | قارن DT مع `/proc/iomem` |
| الـ IRQ معطوب أو غلط | لا استجابة من الجهاز، عداد IRQ صفر في `/proc/interrupts` | تحقق من IRQ routing في GIC/PIC |
| انقطاع الـ power rail | `Unable to handle kernel paging request` بعد probe مباشرة | راقب PMIC/regulator أثناء probe |
| عدم تطابق الـ polarity للـ reset | الجهاز يبقى في reset، لا يستجيب للـ probe | تحقق من `gpio-reset` active-high vs active-low في DT |
| الـ DMA يكتب خارج الـ buffer | corruption عشوائي + `KASAN: slab-out-of-bounds` | تحقق من `dma_mask` وحجم الـ buffer |
| الـ clock frequency خاطئة | بيانات مشوّهة (UART gibberish, I2C NACK) | قِس الـ baud rate بـ oscilloscope، قارن بـ `clk_rate` |

---

#### 5. تصحيح أخطاء الـ Device Tree

```bash
# --- التحقق من تحليل الـ DT ---
# عرض الـ DT كما يراه الكرنل (compiled + overlays)
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null | less

# البحث عن جهاز معيّن
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null | grep -A 20 "serial@ff200000"

# --- مقارنة DT source مع /sys/firmware/devicetree ---
# عرض خاصية معيّنة مباشرة
hexdump -C /sys/firmware/devicetree/base/soc/serial@ff200000/reg
# يجب أن يُظهر: ff 20 00 00 00 00 01 00  (start=0xFF200000, size=0x100)

# --- فحص compatible string ---
cat /sys/firmware/devicetree/base/soc/serial@ff200000/compatible
# يجب أن يتطابق مع أحد المدخلات في driver's of_match_table

# --- فحص الـ interrupts ---
hexdump -C /sys/firmware/devicetree/base/soc/serial@ff200000/interrupts

# --- الـ of_node في sysfs ---
ls -la /sys/bus/platform/devices/ff200000.serial/of_node
# symlink يشير لـ /sys/firmware/devicetree/base/soc/serial@ff200000

# --- أداة fdtdump لـ DTB مباشرة ---
fdtdump /boot/dtb.dtb | grep -A 30 "serial@ff200000"

# --- فحص التعارضات ---
dmesg | grep "OF: fdt:"         # أخطاء تحليل الـ DT عند الـ boot
dmesg | grep "dtb:"
dmesg | grep "platform.*ff200000"

# --- اختبار DT overlay ---
mkdir -p /sys/kernel/config/device-tree/overlays/test
cp my_overlay.dtbo /sys/kernel/config/device-tree/overlays/test/dtbo
echo 1 > /sys/kernel/config/device-tree/overlays/test/status
dmesg | tail -20   # هل تمّ التعرف على الجهاز الجديد؟
```

**قائمة تحقق DT للـ platform_device:**

```
[ ] compatible = "vendor,device" يتطابق مع of_match_table في driver
[ ] reg = <start size> بالـ big-endian صحيح
[ ] interrupts = <GIC_SPI N IRQ_TYPE_LEVEL_HIGH>
[ ] clocks تشير لعقد clock صحيحة
[ ] resets موجودة إن كان الجهاز يحتاج reset controller
[ ] pinctrl-0 و pinctrl-names = "default" مضبوطان
[ ] status = "okay" (ليس "disabled")
[ ] power-domains إن كانت الـ SoC تدعمه
```

---

### Practical Commands

---

#### أوامر جاهزة للنسخ

```bash
#!/bin/bash
# =============================================================
# platform_device Debug Script
# Usage: ./pdev_debug.sh [device_name]  e.g.: serial8250.0
# =============================================================

PDEV_NAME=${1:-""}
TRACE=/sys/kernel/tracing
DEBUG=/sys/kernel/debug

# 1. قائمة جميع platform devices
echo "=== Platform Devices ==="
ls /sys/bus/platform/devices/ | column

# 2. معلومات جهاز محدد
if [ -n "$PDEV_NAME" ]; then
  echo "=== Device: $PDEV_NAME ==="
  PDEV_PATH=/sys/bus/platform/devices/$PDEV_NAME
  [ -f $PDEV_PATH/modalias ]        && echo "modalias:         $(cat $PDEV_PATH/modalias)"
  [ -L $PDEV_PATH/driver ]          && echo "driver:           $(basename $(readlink $PDEV_PATH/driver))"
  [ -f $PDEV_PATH/driver_override ] && echo "driver_override:  $(cat $PDEV_PATH/driver_override)"
  [ -L $PDEV_PATH/of_node ]         && echo "of_node:          $(readlink $PDEV_PATH/of_node)"
fi

# 3. Deferred probe list
echo "=== Deferred Probe ==="
cat $DEBUG/driver_deferred_probe_pending_list 2>/dev/null || echo "(فارغة)"

# 4. IRQs
echo "=== /proc/interrupts (top 20) ==="
head -21 /proc/interrupts

# 5. iomem
echo "=== /proc/iomem ==="
cat /proc/iomem

# 6. آخر 50 رسالة مرتبطة بـ platform
echo "=== dmesg (platform related) ==="
dmesg | grep -i "platform\|probe\|driver" | tail -50
```

---

```bash
# تفعيل function tracing لـ probe كاملاً
TRACE=/sys/kernel/tracing
echo 0           > $TRACE/tracing_on
echo function_graph > $TRACE/current_tracer
echo "platform_drv_probe
platform_device_register
platform_device_add
devm_platform_ioremap_resource
platform_get_irq
platform_get_resource" > $TRACE/set_graph_function
echo ""          > $TRACE/trace
echo 1           > $TRACE/tracing_on

# الآن: modprobe <driver>  أو  echo <dev> > /sys/bus/platform/drivers/<drv>/bind

sleep 2
echo 0           > $TRACE/tracing_on
cat $TRACE/trace | head -200
```

---

```bash
# فحص dynamic debug للـ platform bus
echo "file drivers/base/platform.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/core.c     +pflmt" >> /sys/kernel/debug/dynamic_debug/control
dmesg -w &
# الآن: modprobe أو rmmod
```

---

```bash
# dump كامل لـ sysfs attributes لجهاز
find /sys/bus/platform/devices/serial8250.0/ -maxdepth 2 -type f \
  -exec sh -c 'echo "--- $1 ---"; cat "$1" 2>/dev/null' _ {} \;
```

---

#### تفسير النواتج الشائعة

**ناتج `/proc/interrupts` لـ platform device:**

```
           CPU0       CPU1
 45:       1234       5678   GICv3  45 Level     ff200000.serial
```
- `45`: رقم IRQ المُخصَّص في Linux
- `GICv3 45 Level`: نوع الـ interrupt controller والـ hardware IRQ
- `ff200000.serial`: اسم الجهاز (من DT: `serial@ff200000`)
- إن كان العداد يزيد: الجهاز يُولّد interrupts → الـ firmware يعمل
- إن كان صفراً دائماً: الـ IRQ لا يصل للكرنل → مشكلة hardware أو DT

---

**ناتج `/proc/iomem` لـ platform device:**

```
ff200000-ff2000ff : /soc/serial@ff200000
  ff200000-ff2000ff : ff200000.serial
```
- السطر الأول: حجز من الـ DT parser
- السطر الثاني المُدمَج: حجز من الـ driver عبر `devm_platform_ioremap_resource()`
- إن ظهر السطر الأول فقط بدون الثاني: الـ driver لم يُكمل probe أو لم يستدعِ ioremap
- إن ظهرت رسالة `busy`: resource محجوز لجهاز آخر ← تعارض في DT

---

**ناتج `deferred_probe_pending_list`:**

```
my_device.0
my_device.1
```
- يعني هذان الجهازان ينتظران resource (clock, regulator, GPIO, ...) لم يُسجَّل بعد
- الحل: تحقق من أن الـ provider device (مثل clock driver) مُسجَّل وناجح في probe
- أداة التشخيص: `dmesg | grep "registered\|probe"` لمتابعة ترتيب التسجيل
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: بوابة صناعية على RK3562 — درايفر UART لا يُحمَّل

#### العنوان
فشل تسجيل `platform_driver` لـ UART على بوابة صناعية مبنية على RK3562

#### السياق
شركة تصنّع بوابات Modbus/RTU صناعية باستخدام Rockchip RK3562. البوابة تعتمد على أربعة منافذ UART للتواصل مع أجهزة الاستشعار. بعد ترقية kernel من 5.15 إلى 6.6، توقف UART2 عن العمل كلياً.

#### المشكلة
**الـ** `platform_device` الخاص بـ `serial@fe660000` يظهر في `/sys/bus/platform/devices/` لكن لا يوجد driver مرتبط به. الـ `dmesg` يُظهر:

```
serial fe660000.serial: probe deferral
```

#### التحليل

الـ `platform_driver` يُسجَّل عبر:

```c
/* في driver/tty/serial/8250/8250_dw.c */
module_platform_driver(dw8250_platform_driver);
/* يتوسّع إلى: */
static int __init dw8250_platform_driver_init(void) {
    return platform_driver_register(&dw8250_platform_driver);
}
```

**الـ** `platform_driver_register` يستدعي `__platform_driver_register(drv, THIS_MODULE)` الذي يُضيف الـ driver إلى `platform_bus_type`. عند المطابقة، يُنفَّذ `probe()`.

المشكلة تكمن في أن الـ DT node يحتوي على `clocks = <&cru CLK_UART2>` ولكن الـ clock driver لم يُسجَّل بعد. الـ kernel يضع الـ `platform_device` في قائمة الانتظار بسبب `prevent_deferred_probe = false` في الـ `platform_driver`:

```c
struct platform_driver {
    int (*probe)(struct platform_device *);
    /* ... */
    bool prevent_deferred_probe;  /* false = يقبل deferred probe */
    /* ... */
};
```

الـ `probe()` يستدعي `devm_platform_ioremap_resource()`:

```c
void __iomem *devm_platform_ioremap_resource(struct platform_device *pdev,
                                              unsigned int index)
```

هذه الدالة تستدعي `platform_get_resource(pdev, IORESOURCE_MEM, index)` للحصول على عنوان base من `pdev->resource[]`، ثم تُنفِّذ `ioremap`. لكن الـ probe لم يصل أصلاً لهذه النقطة.

الجذر الحقيقي: الـ clock provider يعتمد على `platform_device` آخر لم يُكمل `probe()` بعد، مما يُنشئ سلسلة deferred probe.

#### الحل

```bash
# تشخيص سلسلة deferred probe
cat /sys/kernel/debug/devices_deferred

# التحقق من الـ clock
cat /sys/kernel/debug/clk/clk_summary | grep uart

# فرض ترتيب التحميل عبر DT
# في rk3562.dtsi — نقل cru قبل serial في initcalls
```

```c
/* إضافة log في probe لتتبع سبب الـ deferral */
static int dw8250_probe(struct platform_device *pdev)
{
    struct resource *regs;

    /* platform_get_resource تُعيد NULL إن لم يكن index صحيحاً */
    regs = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    if (!regs) {
        dev_err(&pdev->dev, "no mem resource\n");
        return -EINVAL;
    }
    /* ... */
}
```

الحل الفعلي: إضافة `initcall_sync` في Kconfig أو ضبط `CONFIG_SERIAL_8250_DW=y` بدلاً من `=m` لضمان التحميل المبكر.

#### الدرس المستفاد
**الـ** `platform_driver.prevent_deferred_probe` يتحكم في سلوك الانتظار. عند تعقّد سلاسل الاعتماد بين الـ drivers، يجب فحص `/sys/kernel/debug/devices_deferred` أولاً قبل البحث في الكود.

---

### السيناريو 2: STM32MP1 IoT Sensor — خطأ في عنوان I2C من `platform_get_resource`

#### العنوان
قراءة عنوان خاطئ لـ I2C controller على لوحة IoT مخصصة بـ STM32MP1

#### السياق
فريق يطوّر عقدة استشعار IoT باستخدام STM32MP157 لقياس درجة الحرارة عبر حساسات متعددة على I2C1 وI2C2. عند bring-up اللوحة المخصصة، يفشل درايفر I2C2 مع خطأ `-ENXIO`.

#### المشكلة
الـ `dmesg` يُظهر:

```
i2c i2c-1: stm32f7-i2c: probe failed: -ENXIO
i2c i2c-1: stm32f7-i2c: can't get resource
```

#### التحليل

الـ driver يستدعي:

```c
/* في drivers/i2c/busses/i2c-stm32f7.c */
static int stm32f7_i2c_probe(struct platform_device *pdev)
{
    struct resource *res;

    /* تُبحث عن resource رقم 0 من نوع MEM */
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    if (!res)
        return -ENXIO;

    base = devm_ioremap_resource(&pdev->dev, res);
    /* ... */
}
```

الـ `platform_get_resource` تبحث في `pdev->resource[]` عن أول عنصر من نوع `IORESOURCE_MEM`:

```c
struct platform_device {
    /* ... */
    u32             num_resources;
    struct resource *resource;   /* مصفوفة الـ resources */
    /* ... */
};
```

فحص الـ DT كشف الخطأ:

```dts
/* خطأ: المبرمج كتب عنوان I2C1 بدلاً من I2C2 */
&i2c2 {
    reg = <0x40012000 0x400>;  /* هذا عنوان I2C1 ! */
    /* العنوان الصحيح لـ I2C2 هو 0x40013000 */
};
```

عند تحميل الـ kernel، يُحوِّل الـ OF layer الـ DT إلى `struct resource` ويُدرجها في `pdev->resource`. الـ `platform_get_resource` تُعيد العنوان الخاطئ `0x40012000`، فيحاول `ioremap` تعيين عنوان لـ I2C1 وهو مُستخدَم بالفعل، فيُعيد `-ENXIO`.

#### الحل

```bash
# فحص الـ resources المُسجَّلة للجهاز
cat /sys/bus/platform/devices/40013000.i2c/resources

# أو عبر iomem
cat /proc/iomem | grep i2c
```

```dts
/* تصحيح Device Tree */
&i2c2 {
    reg = <0x40013000 0x400>;  /* عنوان I2C2 الصحيح */
    interrupts = <GIC_SPI 74 IRQ_TYPE_LEVEL_HIGH>;
    status = "okay";
};
```

```bash
# التحقق بعد التصحيح
dtc -I dtb -O dts /boot/stm32mp157-custom.dtb | grep -A5 "i2c@40013000"
```

#### الدرس المستفاد
**الـ** `pdev->resource[]` تُبنى مباشرة من الـ DT. خطأ واحد في `reg =` يُنتج عنواناً خاطئاً داخل `platform_get_resource`، والـ driver لا يملك وسيلة للتحقق من صحة العنوان — هذا دور المهندس في مراجعة الـ DT.

---

### السيناريو 3: i.MX8M Plus Android TV Box — تعارض `platform_dma_mask` مع HDMI DMA

#### العنوان
تعطّل عرض HDMI على صندوق Android TV بسبب إعداد خاطئ لـ DMA mask

#### السياق
شركة تصنّع Android TV boxes مبنية على NXP i.MX8M Plus. بعد تفعيل ميزة 4K HDMI output، يتعطل الشاشة عشوائياً مع رسائل kernel تشير إلى فشل DMA allocation.

#### المشكلة

```
dma_alloc_coherent: hwaddr 0x100000000 is too large for mask 0xffffffff
lcdif: Failed to allocate framebuffer
```

#### التحليل

الـ `struct platform_device` يحتوي على:

```c
struct platform_device {
    /* ... */
    u64     platform_dma_mask;        /* DMA mask خاص بالـ platform */
    struct device_dma_parameters dma_parms;
    /* ... */
    struct device dev;  /* يحتوي على dev->dma_mask */
};
```

عند تسجيل الـ `platform_device`، يُستخدم `platform_dma_mask` لتهيئة `dev->dma_mask`. درايفر LCDIF (HDMI display engine) يُسجَّل كـ:

```c
/* في drivers/gpu/drm/imx/imx-lcdif.c */
static int imx_lcdif_probe(struct platform_device *pdev)
{
    /* الـ DMA mask يُضبط من DT عبر of_dma_configure */
    /* لكن بعض kernel versions تتجاهل dma-ranges */
}
```

المشكلة: الـ LCDIF على i.MX8M Plus يستطيع الوصول إلى عناوين 64-bit (فوق 4GB RAM). لكن `platform_dma_mask` في الـ DT node لا يحتوي على `dma-ranges` صحيح، فيبقى الـ mask على `0xFFFFFFFF` (32-bit فقط). عند محاولة تخصيص framebuffer 4K (حوالي 32MB)، يقع العنوان فوق 4GB في بعض configurations.

```dts
/* قبل التصحيح — بدون dma-ranges */
lcdif@32e80000 {
    compatible = "fsl,imx8mp-lcdif1";
    reg = <0x0 0x32e80000 0x0 0x10000>;
    /* لا يوجد dma-ranges */
};
```

#### الحل

```dts
/* بعد التصحيح */
lcdif@32e80000 {
    compatible = "fsl,imx8mp-lcdif1";
    reg = <0x0 0x32e80000 0x0 0x10000>;
    /* تحديد DMA mask صريح */
    dma-ranges = <0x0 0x0 0x0 0x0 0xffffffff 0xffffffff>;
};
```

```c
/* أو في board file إن كان الـ DT غير قابل للتعديل */
static int imx_lcdif_probe(struct platform_device *pdev)
{
    int ret;

    /* ضبط الـ DMA mask يدوياً */
    ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
    if (ret) {
        ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
        if (ret)
            return ret;
    }
    /* ... */
}
```

```bash
# التحقق من الـ DMA mask الحالي
cat /sys/bus/platform/devices/32e80000.lcdif/dma_mask
```

#### الدرس المستفاد
**الـ** `platform_dma_mask` في `struct platform_device` هو نقطة تحكم حرجة لعمليات DMA. أي peripheral يتعامل مع بيانات كبيرة (HDMI, USB3, PCIe) على منصات 64-bit يحتاج ضبط الـ mask صراحةً، سواء في DT أو في الـ `probe()`.

---

### السيناريو 4: AM62x Automotive ECU — استخدام `driver_override` لإجبار تحميل driver محدد

#### العنوان
ربط درايفر CAN خاص بـ ECU مع `platform_device` معين على TI AM62x

#### السياق
شركة تصنّع وحدات تحكم ECU للسيارات على TI AM62x SoC. تحتاج إلى استخدام نسخة مخصصة من درايفر MCAN بإضافات خاصة للسلامة (ISO 26262)، مع منع الـ driver العام من الاستحواذ على الـ device.

#### المشكلة
عند تحميل كلا الـ drivers (الرسمي وذلك الخاص)، يتسابقان على نفس الـ `platform_device`. الـ driver العام يفوز دائماً لأنه يُحمَّل أبكر.

#### التحليل

الـ `struct platform_device` يحتوي على:

```c
struct platform_device {
    /* ... */
    /*
     * Driver name to force a match. Do not set directly, because core
     * frees it. Use driver_set_override() to set or clear it.
     */
    const char *driver_override;  /* اسم الـ driver المُجبَر */
    /* ... */
};
```

عند عملية المطابقة في `platform_bus_type`, الـ `platform_match()` يتحقق أولاً من `driver_override`:

```c
/* في drivers/base/platform.c */
static int platform_match(struct device *dev, const struct device_driver *drv)
{
    struct platform_device *pdev = to_platform_device(dev);
    struct platform_driver *pdrv = to_platform_driver(drv);

    /* أولوية قصوى: driver_override */
    if (pdev->driver_override)
        return !strcmp(pdev->driver_override, drv->name);

    /* ثم: OF matching */
    /* ثم: ACPI matching */
    /* أخيراً: id_table matching */
}
```

الـ `to_platform_device(x)` هو:
```c
#define to_platform_device(x) container_of((x), struct platform_device, dev)
```

#### الحل

```c
/* في كود الـ board init أو عبر sysfs */

/* الطريقة 1: من الكود (board file) */
static int __init ecn_can_force_driver(void)
{
    struct device *dev;
    struct platform_device *pdev;

    dev = bus_find_device_by_name(&platform_bus_type, NULL,
                                   "20700000.can");
    if (!dev)
        return -ENODEV;

    pdev = to_platform_device(dev);

    /* driver_set_override تُخزِّن نسخة من الاسم ليُحرَّر لاحقاً */
    return driver_set_override(dev, &pdev->driver_override, "can-iso26262");
}
late_initcall(ecn_can_force_driver);
```

```bash
# الطريقة 2: من userspace عبر sysfs (بعد boot)
echo "can-iso26262" > /sys/bus/platform/devices/20700000.can/driver_override

# إجبار إعادة probe
echo "20700000.can" > /sys/bus/platform/drivers_probe
```

```bash
# التحقق
cat /sys/bus/platform/devices/20700000.can/driver_override
ls -la /sys/bus/platform/devices/20700000.can/driver
```

#### الدرس المستفاد
**الـ** `driver_override` في `struct platform_device` هو آلية قوية لإجبار المطابقة. يجب دائماً استخدام `driver_set_override()` وليس الكتابة المباشرة في `pdev->driver_override` لأن الـ kernel core يتولى إدارة الذاكرة.

---

### السيناريو 5: Allwinner H616 — فشل `devm_platform_ioremap_resource_byname` في درايفر SPI لـ Display Controller

#### العنوان
crash في `probe()` لـ SPI display controller على لوحة Orange Pi Zero 2 (H616) بسبب اسم resource خاطئ

#### السياق
مطوّر يُنفِّذ درايفر SPI display controller مخصص لشاشة TFT مرتبطة بـ Orange Pi Zero 2 المبني على Allwinner H616. الـ driver يُكمَّل ويُحمَّل، لكن يحدث NULL pointer dereference فور تشغيله.

#### المشكلة

```
Unable to handle kernel NULL pointer dereference at virtual address 0000000000000010
PC is at my_spi_display_probe+0x48/0x200
```

#### التحليل

الـ driver الجديد يستخدم `devm_platform_ioremap_resource_byname`:

```c
/* في my-spi-display.c */
static int my_spi_display_probe(struct platform_device *pdev)
{
    void __iomem *base;

    /* يبحث عن resource باسم "regs" */
    base = devm_platform_ioremap_resource_byname(pdev, "regs");
    if (IS_ERR(base))
        return PTR_ERR(base);

    /* crash هنا — base ليس NULL لكن الـ IS_ERR لم يلتقطه */
    writel(0x1, base + 0x10);
}
```

الدالة `devm_platform_ioremap_resource_byname` تستدعي:

```c
void __iomem *
devm_platform_ioremap_resource_byname(struct platform_device *pdev,
                                       const char *name)
```

هذه بدورها تستدعي `platform_get_resource_byname(pdev, IORESOURCE_MEM, name)` التي تبحث في `pdev->resource[]` عن عنصر من نوع `IORESOURCE_MEM` مع مطابقة `resource->name` للـ `name` المُعطى.

فحص الـ DT:

```dts
/* الـ DT الخاطئ */
spi_display@0 {
    compatible = "my,spi-display";
    reg = <0x02049000 0x1000>;  /* بدون reg-names */
};
```

بما أن الـ DT لا يحتوي على `reg-names`, فاسم الـ resource يكون NULL. `platform_get_resource_byname` تُجري مقارنة `strcmp(r->name, name)` حيث `r->name == NULL`، ما يُسبب crash.

لكن لماذا `IS_ERR(base)` لم يلتقط الخطأ؟ لأن الـ crash يحدث داخل الدالة نفسها قبل إعادة القيمة.

#### الحل

```dts
/* تصحيح الـ DT بإضافة reg-names */
spi_display@0 {
    compatible = "my,spi-display";
    reg = <0x02049000 0x1000>;
    reg-names = "regs";  /* يُطابق اسم الـ resource في الكود */
    spi-max-frequency = <50000000>;
    status = "okay";
};
```

```c
/* أو تغيير الـ driver لاستخدام index بدلاً من الاسم */
static int my_spi_display_probe(struct platform_device *pdev)
{
    void __iomem *base;

    /* استخدام index (أكثر أماناً حين لا يوجد reg-names) */
    base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(base)) {
        dev_err(&pdev->dev, "ioremap failed: %ld\n", PTR_ERR(base));
        return PTR_ERR(base);
    }

    writel(0x1, base + 0x10);
    return 0;
}
```

```bash
# التحقق من أسماء الـ resources في sysfs
ls /sys/bus/platform/devices/2049000.spi_display/
cat /proc/iomem | grep 2049000

# التحقق من الـ DT المُجمَّع
dtc -I dtb -O dts /sys/firmware/fdt | grep -A 10 "spi_display"
```

```c
/* دفاعي: التحقق من وجود الاسم قبل الاستخدام */
static int my_spi_display_probe(struct platform_device *pdev)
{
    struct resource *res;
    void __iomem *base;

    /* محاولة بالاسم أولاً */
    res = platform_get_resource_byname(pdev, IORESOURCE_MEM, "regs");
    if (!res) {
        /* fallback إلى index */
        dev_warn(&pdev->dev,
                 "no 'regs' resource name, falling back to index 0\n");
        res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    }
    if (!res) {
        dev_err(&pdev->dev, "no MEM resource found\n");
        return -EINVAL;
    }

    base = devm_ioremap_resource(&pdev->dev, res);
    return PTR_ERR_OR_ZERO(base);
}
```

#### الدرس المستفاد
**الـ** `devm_platform_ioremap_resource_byname` تفترض وجود `reg-names` في الـ DT. عند غيابه، السلوك غير محدد وقد يُسبب crash. دائماً تزامن بين أسماء الـ resources في الكود وقيم `reg-names` في الـ DT، أو استخدم `platform_get_resource` بالـ index كـ fallback آمن.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ** LWN.net هو المرجع الأول لمتابعة تطور kernel Linux، وفيما يلي أهم المقالات المتعلقة بـ `platform_device`:

| المقال | الوصف |
|--------|-------|
| [The platform device API](https://lwn.net/Articles/448499/) | شرح شامل لـ `struct platform_device` وآلية تسجيل الأجهزة وربطها بالـ drivers — نقطة البداية الأمثل |
| [Platform devices and device trees](https://lwn.net/Articles/448502/) | كيفية تهيئة platform devices من device tree، واستخراج resources تلقائياً من وصف الـ DT |
| [Deferred driver probing](https://lwn.net/Articles/450460/) | آلية `EPROBE_DEFER` وكيف يؤجّل الـ driver تهيئته حتى تتوفر الموارد المطلوبة |
| [drivercore: Add driver probe deferral mechanism](https://lwn.net/Articles/485194/) | الـ patch الذي أضاف آلية deferred probe إلى driver core |
| [Device dependencies and deferred probing](https://lwn.net/Articles/662820/) | نقاش متقدم حول تبعيات الأجهزة وتسلسل الـ probe في الأنظمة المعقدة |
| [A tour of /sys/devices](https://lwn.net/Articles/646617/) | جولة في sysfs وكيف تظهر platform devices في شجرة `/sys/devices/platform/` |
| [Improved dynamically allocated platform_device interface](https://lwn.net/Articles/158747/) | الـ patch الذي أضاف `platform_device_alloc()` والـ API الديناميكية الحديثة |
| [platform_device_probe() to conserve memory](https://lwn.net/Articles/198166/) | مناقشة `platform_driver_probe()` ووضع دوال `probe` في قسم `__init` لتوفير الذاكرة |

---

### التوثيق الرسمي للـ Kernel

**الـ** kernel documentation المضمّن في المصدر هو المرجع الدقيق لأي تغيير:

```bash
# المسارات داخل شجرة مصدر الـ kernel
Documentation/driver-api/driver-model/platform.rst
Documentation/driver-api/driver-model/overview.rst
Documentation/driver-api/driver-model/driver.rst
Documentation/driver-api/driver-model/devres.rst
```

**الروابط المباشرة:**

| الوثيقة | الرابط |
|---------|--------|
| Platform Devices and Drivers (رسمي) | [docs.kernel.org/driver-api/driver-model/platform.html](https://docs.kernel.org/driver-api/driver-model/platform.html) |
| Linux Device Driver Model (نظرة عامة) | [docs.kernel.org/driver-api/driver-model/overview.html](https://docs.kernel.org/driver-api/driver-model/overview.html) |
| Devres — Managed Device Resource | [docs.kernel.org/driver-api/driver-model/devres.html](https://docs.kernel.org/driver-api/driver-model/devres.html) |
| Device Drivers (driver ops) | [docs.kernel.org/driver-api/driver-model/driver.html](https://docs.kernel.org/driver-api/driver-model/driver.html) |
| Linux and the Devicetree | [kernel.org/doc/html/latest/devicetree/usage-model.html](https://www.kernel.org/doc/html/latest/devicetree/usage-model.html) |

---

### الـ Header الأصلي

الملف المُوثَّق في هذه السلسلة:

```
include/linux/platform_device.h
```

**الـ** implementation الأساسي موجود في:

```
drivers/base/platform.c
```

---

### Commits بارزة في تاريخ platform_device

يمكن تتبع التغييرات عبر:

```bash
# عرض تاريخ الملف كاملاً
git log --oneline -- include/linux/platform_device.h

# عرض التغييرات التفصيلية
git log -p -- include/linux/platform_device.h
```

**أبرز commits تاريخية:**

| التغيير | الدلالة |
|---------|---------|
| إضافة `struct platform_driver` (kernel 2.6.15) | فصل driver عن device بدلاً من الـ `driver_register` المباشر |
| إضافة `platform_device_alloc()` | السماح بإنشاء devices ديناميكياً بدلاً من static arrays فقط |
| إضافة `module_platform_driver()` macro | تبسيط كتابة الـ drivers وإلغاء boilerplate في `module_init/exit` |
| إضافة `devm_platform_ioremap_resource()` | ربط memory mapping بـ device lifetime تلقائياً |
| إضافة `driver_override` field | السماح بتحديد driver بعينه عبر sysfs بدون تعديل الـ kernel |
| إضافة `platform_device_register_full()` + `platform_device_info` | واجهة موحّدة لتسجيل الأجهزة مع كل الخصائص |

**للبحث في lore.kernel.org (أرشيف الـ mailing list):**

```
https://lore.kernel.org/linux-kernel/?q=platform_device
https://lore.kernel.org/all/20180524175024.19874-8-robh@kernel.org/T/
```

---

### نقاشات Mailing List

**الـ** mailing list هو المكان الذي تُناقَش فيه القرارات التصميمية قبل دمجها:

| الموضوع | الرابط |
|---------|--------|
| Make deferring probe forever optional (Rob Herring, 2018) | [lore.kernel.org](https://lore.kernel.org/all/20180524175024.19874-8-robh@kernel.org/T/) |
| Disable driver deferred probe timeout by default | [lore.kernel.org/lkml](https://lore.kernel.org/lkml/2b8c40ba-0068-99ca-6dc8-64d075e9112c@kali.org/T/) |
| Extend deferred probe timeout on driver registration | [lore.kernel.org/lkml](https://lore.kernel.org/lkml/Yo3WvGnNk3LvLb7R@linutronix.de/t/) |
| When "probe" is called? (kernelnewbies list, 2011) | [lists.kernelnewbies.org](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2011-February/000815.html) |
| Platform driver probe() not called with DT overlay (2019) | [lists.kernelnewbies.org](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2019-February/019811.html) |

---

### Kernelnewbies.org

**الـ** kernelnewbies wiki يوثّق التغييرات عبر إصدارات الـ kernel المختلفة:

| الصفحة | الوصف |
|--------|-------|
| [Linux_2_6_15](https://kernelnewbies.org/Linux_2_6_15) | الإصدار الذي أضاف `struct platform_driver` بشكل منفصل عن `device_driver` |
| [Linux_3.0_DriverArch](https://kernelnewbies.org/Linux_3.0_DriverArch) | تغييرات معمارية في driver model في kernel 3.0 |
| [Linux_3.4_DriverArch](https://kernelnewbies.org/Linux_3.4_DriverArch) | تحسينات platform drivers في kernel 3.4 |
| [Linux_3.8_DriverArch](https://kernelnewbies.org/Linux_3.8_DriverArch) | إضافة generic platform drivers لـ EHCI/OHCI |
| [Linux_4.0-DriversArch](https://kernelnewbies.org/Linux_4.0-DriversArch) | تغييرات driver architecture في kernel 4.0 |

---

### eLinux.org

**الـ** eLinux wiki يركّز على embedded Linux والأجهزة الحقيقية:

| الصفحة | الوصف |
|--------|-------|
| [Hammer Button Driver](https://www.elinux.org/Hammer_Button_Driver) | مثال عملي كامل لكتابة platform device driver لزر GPIO |
| [BeagleBoard/Mikrobus Support](https://elinux.org/BeagleBoard/Mikrobus_Support) | استخدام platform data وDT overlays مع BeagleBone |

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الكتاب المجاني** المتاح على: https://lwn.net/Kernel/LDD3/
- **الفصول ذات الصلة:**
  - Chapter 14: The Linux Device Model — يشرح `kobject`, `bus_type`, `device`, `device_driver`
  - Chapter 3: Char Drivers — أساس فهم كيف يتفاعل driver مع device
  - Chapter 9: Communicating with Hardware — `ioremap`, `request_mem_region`

> **ملاحظة:** LDD3 مكتوب لـ kernel 2.6 — بعض الـ API تغيّرت، لكن المفاهيم الأساسية راسخة.

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول ذات الصلة:**
  - Chapter 17: Devices and Modules — نظرة عامة على device model وـ kobject hierarchy
  - Chapter 14: The Block I/O Layer — لفهم كيف تعمل resources

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصول ذات الصلة:**
  - Chapter 8: Device Drivers — platform drivers في سياق embedded SoC
  - Chapter 16: Kernel Debugging Techniques — تشخيص مشاكل probe وresource conflicts

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- يتناول device model بعمق أكبر من LDD3 مع رسوم بيانية للـ kobject hierarchy

---

### مصادر الكود الأصلي للمراجعة

```bash
# الملفات الأساسية في kernel source
include/linux/platform_device.h   # تعريفات الـ API
drivers/base/platform.c           # التنفيذ الكامل
drivers/base/bus.c                # منطق bus_type وربط device بـ driver
include/linux/device.h            # struct device الأساسية
include/linux/ioport.h            # struct resource
Documentation/driver-api/driver-model/platform.rst
```

**للبحث عن examples حقيقية في الـ kernel:**

```bash
# عثور على drivers تستخدم module_platform_driver
grep -r "module_platform_driver" drivers/ --include="*.c" | head -20

# عثور على drivers تستخدم devm_platform_ioremap_resource
grep -r "devm_platform_ioremap_resource" drivers/ --include="*.c" | head -20
```

---

### كلمات البحث المقترحة

للبحث عن معلومات إضافية، استخدم هذه المصطلحات:

```
platform_device Linux kernel
platform_driver probe remove
struct platform_device_id id_table
module_platform_driver macro
devm_platform_ioremap_resource
PLATFORM_DEVID_AUTO
platform_device_register_full
platform_device_info fwnode
driver_override sysfs
deferred probe EPROBE_DEFER
platform bus pseudo-bus
non-discoverable hardware Linux
SoC device registration Linux
device tree platform device instantiation
```

---

### ملخص المصادر حسب الأولوية

| الأولوية | المصدر | لماذا؟ |
|----------|--------|--------|
| 1 | [The platform device API — LWN](https://lwn.net/Articles/448499/) | الشرح الأشمل والأوضح للـ API |
| 2 | [Platform Devices and Drivers — kernel.org](https://docs.kernel.org/driver-api/driver-model/platform.html) | التوثيق الرسمي المحدَّث |
| 3 | [Platform devices and device trees — LWN](https://lwn.net/Articles/448502/) | ضروري لفهم DT integration |
| 4 | `drivers/base/platform.c` في kernel source | الحقيقة الوحيدة المطلقة |
| 5 | LDD3 Chapter 14 | الأساس النظري للـ device model |
| 6 | [Deferred probing — LWN](https://lwn.net/Articles/450460/) | لفهم مشاكل الـ probe الحديثة |
| 7 | [Devres documentation](https://docs.kernel.org/driver-api/driver-model/devres.html) | لفهم `devm_*` functions |
## Phase 8: Writing simple module

### الهدف

سنقوم بعمل **kprobe** على الدالة `platform_device_register` — وهي نقطة دخول مركزية تُستدعى في كل مرة يُسجَّل فيها **platform device** جديد في النظام. هذا يجعلها مثالية لمراقبة جميع الأجهزة التي يكتشفها الـ kernel وقت التشغيل أو عند تحميل driver.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * pdev_register_spy.c
 *
 * Hooks platform_device_register() via kprobe and logs
 * the device name, id, and number of resources each time
 * a new platform device is registered.
 */

#include <linux/kernel.h>       /* pr_info, pr_err                        */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit        */
#include <linux/kprobes.h>      /* kprobe, register_kprobe, etc.           */
#include <linux/platform_device.h> /* struct platform_device              */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("Spy on platform_device_register() using kprobe");

/* ------------------------------------------------------------------ */
/*  kprobe pre-handler: called just BEFORE platform_device_register() */
/* ------------------------------------------------------------------ */
static int pdev_register_pre(struct kprobe *kp, struct pt_regs *regs)
{
    struct platform_device *pdev;

#ifdef CONFIG_X86_64
    /* On x86_64, first argument is in RDI register */
    pdev = (struct platform_device *)regs->di;
#elif defined(CONFIG_ARM64)
    /* On ARM64, first argument is in X0 register */
    pdev = (struct platform_device *)regs->regs[0];
#elif defined(CONFIG_ARM)
    /* On ARM32, first argument is in R0 register */
    pdev = (struct platform_device *)regs->ARM_r0;
#else
    /* Fallback: skip logging on unknown arch */
    return 0;
#endif

    if (!pdev || !pdev->name)
        return 0;

    pr_info("pdev_spy: registering platform_device name='%s' id=%d num_resources=%u\n",
            pdev->name,
            pdev->id,
            pdev->num_resources);

    return 0; /* returning 0 means: let the original function execute normally */
}

/* ------------------------------------------------------------------ */
/*  kprobe definition                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe pdev_kp = {
    .symbol_name = "platform_device_register", /* hook by symbol name  */
    .pre_handler = pdev_register_pre,           /* our callback         */
};

/* ------------------------------------------------------------------ */
/*  module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init pdev_spy_init(void)
{
    int ret;

    ret = register_kprobe(&pdev_kp);
    if (ret < 0) {
        pr_err("pdev_spy: register_kprobe failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("pdev_spy: kprobe planted at %s (%px)\n",
            pdev_kp.symbol_name, pdev_kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit pdev_spy_exit(void)
{
    unregister_kprobe(&pdev_kp);
    pr_info("pdev_spy: kprobe removed from %s\n", pdev_kp.symbol_name);
}

module_init(pdev_spy_init);
module_exit(pdev_spy_exit);
```

---

### Makefile للبناء

```makefile
obj-m += pdev_register_spy.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل الـ module
make
sudo insmod pdev_register_spy.ko
dmesg | tail -20        # مشاهدة النتائج فور تسجيل أي platform device
sudo rmmod pdev_register_spy
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|--------|
| `linux/kprobes.h` | يوفر `struct kprobe` وكل دوال التسجيل/الإلغاء |
| `linux/platform_device.h` | تعريف `struct platform_device` لقراءة حقل `name` و`id` و`num_resources` |
| `linux/module.h` | الماكروهات الأساسية لأي kernel module مثل `MODULE_LICENSE` و`module_init` |

#### الـ `pdev_register_pre` callback

الـ **pre_handler** يُستدعى مباشرةً قبل تنفيذ الدالة المُراقَبة، مما يتيح قراءة المعاملات قبل أي تعديل. نقرأ المعامل الأول من السجلات المعمارية (`regs->di` على x86_64) لأن الـ kprobe يعطينا `pt_regs` — snapshot كامل لحالة المعالج — لا مؤشرًا مباشرًا للمعاملات.

الـ **`return 0`** ضروري: أي قيمة أخرى غير صفر ستجعل الـ kprobe يتجاهل تنفيذ الدالة الأصلية أو يُعيد توجيه التنفيذ، وهذا غير مرغوب هنا لأننا نريد المراقبة فقط دون تدخل.

#### الـ `struct kprobe`

**`symbol_name`** يُخبر الـ kernel بالبحث عن رمز `platform_device_register` في جدول الرموز تلقائيًا وحساب العنوان وقت `register_kprobe` — أفضل من كتابة عنوان ثابت الذي يتغير بين إصدارات الـ kernel.

#### الـ `module_init`

نستدعي `register_kprobe` لزرع الـ breakpoint البرمجي على أول تعليمة في `platform_device_register`. الفشل هنا (مثل `CONFIG_KPROBES` غير مفعّل أو الدالة مُدرجة في قائمة الحظر `kprobes_blacklist`) يُعاد كـ error code سلبي فنُبلّغ عنه ونرجع فورًا.

#### الـ `module_exit`

**`unregister_kprobe`** ضرورية حتمًا: بدونها، إذا أُزيل الـ module من الذاكرة وبقي الـ breakpoint البرمجي مزروعًا، فسيؤدي أي استدعاء لاحق لـ `platform_device_register` إلى كارثة لأن الـ handler يشير إلى كود تم تحريره — **kernel panic** مضمون.

---

### مثال على الـ output

```
[  123.456789] pdev_spy: kprobe planted at platform_device_register (ffffffff81a3b2c0)
[  124.001234] pdev_spy: registering platform_device name='serial8250' id=0 num_resources=4
[  124.002567] pdev_spy: registering platform_device name='rtc-cmos' id=-1 num_resources=2
[  124.100000] pdev_spy: registering platform_device name='i2c-gpio' id=0 num_resources=0
```

الـ `id=-1` يعني `PLATFORM_DEVID_NONE` — جهاز لا يحتاج رقم instance لأنه وحيد من نوعه. الـ `id=-2` يعني `PLATFORM_DEVID_AUTO` حيث يختار الـ kernel رقمًا تلقائيًا.
