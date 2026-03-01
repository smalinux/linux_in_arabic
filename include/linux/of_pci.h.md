## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ `of_pci.h` وليه موجود؟

تخيّل إنك بتبني جهاز embedded — زي gateway صناعي أو TV box — وجواه chip بيحتوي على PCI Express controller. الـ CPU مش بيعرف أوتوماتيك إن في PCI controller موجود وإيه العناوين والـ interrupts بتاعته. محتاج حاجة "تعرّفهم لبعض".

الـ **Device Tree** (المعروف بـ OF — Open Firmware) هو الـ mechanism اللي بنستخدمه على embedded systems (ARM، RISC-V، MIPS، PowerPC) بدل من الـ BIOS/ACPI. هو ملف نصي (`.dts`) بيوصف الـ hardware: "في PCI controller على العنوان ده، بيستخدم الـ interrupt اللي رقمه كذا."

الـ **`include/linux/of_pci.h`** هو الـ header اللي بيوفر الـ glue بين عالمَين:
- عالم **OF/Device Tree**: الـ `device_node`، الـ properties، الـ interrupt-map.
- عالم **PCI**: الـ `pci_dev`، الـ `devfn`، الـ IRQ assignment.

### القصة ببساطة

```
[Bootloader/Firmware]
        |
        | يحمّل الـ Device Tree Blob (DTB)
        v
[Linux Kernel - OF subsystem]
        |
        | يقرأ الـ DT ويبني شجرة device_node
        v
[PCI Host Controller Driver]
        |
        | يسأل: "إيه الـ IRQ للـ PCI device ده؟"
        | يسأل: "إيه الـ devfn للـ node ده؟"
        v
[of_pci.h functions]
        |
        | of_irq_parse_and_map_pci()
        | of_pci_find_child_device()
        | of_pci_get_devfn()
        v
[PCI device ready with correct IRQ + DT node]
```

### الفائدة العملية

بدون `of_pci.h`، كل driver لازم يعرف بنفسه إزاي يقرأ الـ interrupt-map من الـ DT. الـ header ده بيوحّد الكود ويوفر API مشترك لكل platform.

### الملفات المرتبطة

| الملف | الدور |
|-------|-------|
| `include/linux/of_pci.h` | الـ header (واجهة الـ API) |
| `drivers/pci/of.c` | الـ implementation الكامل |
| `include/linux/of_irq.h` | IRQ parsing من الـ DT |
| `include/linux/of.h` | الـ OF core structs (`device_node`) |
| `include/linux/pci.h` | الـ PCI core structs (`pci_dev`) |
| `drivers/pci/pci-host-common.c` | host bridge يستخدم `of_pci_check_probe_only` |
| `Documentation/devicetree/bindings/pci/` | توثيق الـ DT bindings |
| `Documentation/PCI/` | توثيق الـ PCI subsystem |

### الـ Subsystem

الـ file ده تحت مسؤولية **PCI SUBSYSTEM** في `MAINTAINERS`:
- **Maintainer**: Bjorn Helgaas `<bhelgaas@google.com>`
- **Mailing List**: `linux-pci@vger.kernel.org`
- **Git Tree**: `git://git.kernel.org/pub/scm/linux/kernel/git/pci/pci.git`

### الملفات الأساسية للـ Subsystem

```
drivers/pci/
├── of.c              ← implementation of of_pci.h
├── pci.c             ← PCI core
├── probe.c           ← device discovery
├── controller/
│   ├── pci-host-common.c   ← generic host bridge
│   ├── pcie-mt7621.c       ← MediaTek PCI
│   ├── pci-aardvark.c      ← Marvell Armada
│   └── pci-mvebu.c         ← Marvell EBU
include/linux/
├── of_pci.h          ← هذا الملف
├── of_irq.h
├── of.h
└── pci.h
```
## Phase 2: شرح الـ OF/PCI Framework

### المشكلة اللي بيحلها الـ Subsystem

على الـ x86، الـ BIOS/UEFI بيعمل enumerate لأجهزة PCI ويرتب الـ interrupts. لكن على الـ embedded SoCs (ARM/RISC-V):
1. مافيش BIOS.
2. الـ interrupt routing مش standard.
3. كل SoC ليه interrupt controller مختلف.
4. الـ bootloader (U-Boot/GRUB) بيمرر الـ hardware description كـ **Device Tree Blob (DTB)**.

المشكلة تحديدًا في نقطتين:
- **Device identification**: إزاي الكيرنل يعرف إن الـ PCI device على الـ bus بالـ `devfn` كذا يقابل الـ DT node كذا؟
- **IRQ mapping**: إزاي يترجم الـ `interrupt-map` property من الـ DT لـ virtual IRQ number؟

### الحل

الـ kernel يبني جسر بين:
- **OF/DT world**: الـ `struct device_node` اللي بيمثل كل node في الـ DT.
- **PCI world**: الـ `struct pci_dev` اللي بيمثل كل PCI device.

الـ `of_pci.h` بيوفر الـ API لعمل الربط ده.

### ASCII Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Device Tree (DTB)                     │
│  pcie@fe8b0000 {                                         │
│    #address-cells = <3>;                                 │
│    #interrupt-cells = <1>;                               │
│    interrupt-map = <...>;                                │
│    ranges = <0x02000000 ...>;                            │
│    device@0,0 { reg = <0x0000 0 0 0 0>; }               │
│  }                                                       │
└────────────────────┬────────────────────────────────────┘
                     │  of_pci.h API
                     │
         ┌───────────▼────────────┐
         │   OF/PCI Glue Layer    │
         │   (drivers/pci/of.c)   │
         │                        │
         │ of_pci_find_child_device()   ← device lookup
         │ of_pci_get_devfn()           ← devfn extraction
         │ of_irq_parse_and_map_pci()   ← IRQ mapping
         │ of_pci_check_probe_only()    ← probe mode
         └───────────┬────────────┘
                     │
         ┌───────────▼────────────┐
         │   PCI Core Subsystem   │
         │   (drivers/pci/)       │
         │                        │
         │  pci_scan_bus()        │
         │  pci_assign_irq()      │
         │  pci_set_of_node()     │
         └───────────┬────────────┘
                     │
    ┌────────────────▼───────────────────┐
    │           PCI Drivers              │
    │  (WiFi, GPU, NVMe, USB3, etc.)     │
    └────────────────────────────────────┘
```

### المصطلحات والمفاهيم الأساسية

| المصطلح | المعنى |
|---------|--------|
| **OF** | Open Firmware — نظام قديم من Apple/Sun أصبح standard |
| **DT / DTB** | Device Tree / Device Tree Blob — binary representation |
| **device_node** | struct تمثل node واحدة في الـ DT |
| **devfn** | Device Function number في الـ PCI — 8 bits: [7:3]=slot, [2:0]=function |
| **interrupt-map** | DT property تربط PCI IRQ pins بـ interrupt controller lines |
| **VIRQ** | Virtual IRQ — رقم منطقي بيوفره الـ kernel |
| **irq_domain** | تجريد الـ kernel لإدارة الـ IRQ controllers المختلفة |
| **probe-only** | وضع بيمنع الـ kernel من إعادة ترتيب BARs اللي الـ firmware رتبها |

### الـ Real-world Analogy

الـ interrupt-map في الـ DT زي "دليل التليفونات":
- الـ PCI device بيقول "عايز أتكلم على رقم IRQ#A".
- الـ interrupt-map بيقول "رقم IRQ#A عند الـ parent interrupt controller هو رقم 42".
- الـ `of_irq_parse_and_map_pci` بيعمل الترجمة دي ويرجع الـ VIRQ.

### ملكية الـ Subsystem

| ما يملكه الـ of/pci glue | ما يفوّضه للـ drivers |
|--------------------------|----------------------|
| قراءة `reg` property وتحويلها لـ devfn | إدارة الـ PCI config space |
| تفسير `interrupt-map` وعمل الـ IRQ mapping | تفعيل/تعطيل الـ device |
| ربط الـ DT node بالـ `pci_dev` | إدارة الـ DMA |
| كشف `linux,pci-probe-only` property | تسجيل الـ PCI driver |

### الـ CONFIG Options

```
CONFIG_OF          ← تفعيل دعم Device Tree
CONFIG_PCI         ← تفعيل دعم PCI
CONFIG_OF_IRQ      ← تفعيل IRQ parsing من DT
```

الـ `of_pci.h` يعمل stub functions (ترجع NULL أو -EINVAL) لو الـ configs دي مش مفعّلة — يضمن إن الكود بيكمبايل حتى بدون الـ OF support.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### الـ CONFIG Options (Cheatsheet)

| الـ Config | الوظيفة | تأثيره على of_pci.h |
|-----------|---------|---------------------|
| `CONFIG_OF` | تفعيل دعم Open Firmware/DT | يتيح الـ real implementations |
| `CONFIG_PCI` | تفعيل دعم PCI subsystem | يتيح الـ real implementations |
| `CONFIG_OF_IRQ` | دعم IRQ parsing من DT | يتيح `of_irq_parse_and_map_pci` |
| `CONFIG_IRQ_DOMAIN` | إدارة IRQ domains | لازم لـ MSI support |
| `CONFIG_PCI_PROBE_ONLY` | منع الكيرنل من re-assigning BARs | يتحكم فيه `of_pci_check_probe_only` |

### الـ Structs الأساسية

#### `struct device_node` (من `include/linux/of.h`)

```c
struct device_node {
    const char *name;           /* اسم الـ node زي "pcie" */
    phandle phandle;            /* معرف فريد للـ node في الـ DT */
    const char *full_name;      /* المسار الكامل "/soc/pcie@fe8b0000" */
    struct fwnode_handle fwnode; /* ربط مع الـ firmware node abstraction */

    struct property *properties; /* قائمة الـ properties */
    struct device_node *parent;  /* الـ node الأب */
    struct device_node *child;   /* أول child node */
    struct device_node *sibling; /* الـ node التالي على نفس المستوى */
    unsigned long _flags;        /* flags داخلية */
    void *data;                  /* بيانات إضافية */
};
```

**الغرض**: تمثيل node واحدة في الـ Device Tree في الذاكرة.

#### `struct property` (من `include/linux/of.h`)

```c
struct property {
    char  *name;        /* اسم الـ property مثل "reg", "interrupts" */
    int    length;      /* الحجم بالـ bytes */
    void  *value;       /* القيمة الخام (big-endian) */
    struct property *next; /* الـ property التالية */
};
```

#### `struct of_phandle_args` (من `include/linux/of.h`)

```c
struct of_phandle_args {
    struct device_node *np;  /* الـ node المشار إليه بالـ phandle */
    int args_count;          /* عدد الـ arguments */
    uint32_t args[MAX_PHANDLE_ARGS]; /* قيم الـ arguments */
};
```

**الغرض**: يُستخدم في parsing الـ `interrupt-map` — بيحمل `device_node` الـ interrupt controller مع الـ specifier arguments.

#### `struct pci_dev` (من `include/linux/pci.h`) — الأجزاء ذات الصلة

```c
struct pci_dev {
    struct pci_bus *bus;      /* الـ PCI bus اللي عليه */
    unsigned int devfn;       /* device + function number */
    struct device dev;        /* الـ generic device struct */
    /* dev.of_node ← ده اللي بيتسيته of_pci_find_child_device */
    u8 pin;                   /* IRQ pin (INTA=1..INTD=4) */
    unsigned int irq;         /* الـ VIRQ المخصص */
    /* ... */
};
```

### علاقات الـ Structs

```
device_node (DT root)
    │
    ├── device_node "pcie@fe8b0000"
    │       properties:
    │         - "reg" → base address
    │         - "ranges" → memory windows
    │         - "interrupt-map" → IRQ routing
    │         - "bus-range" → PCI bus numbers
    │
    │   ├── device_node "device@0,0"   ← child PCI device
    │           properties:
    │             - "reg" = <0x0000 0 0 0 0>
    │                         ↑
    │                    [phys.hi phys.mid phys.lo size.hi size.lo]
    │                    phys.hi bits[15:8] = devfn
    │
    └── (linked via of_pci_find_child_device)
            ↓
        pci_dev.dev.of_node ← pointer to device_node
```

### تفسير الـ `reg` Property في PCI DT Nodes

```
reg = <phys.hi phys.mid phys.lo size.hi size.lo>

phys.hi (32-bit):
  bits[31:30] = space (00=config, 01=I/O, 10=32-bit MEM, 11=64-bit MEM)
  bits[25:16] = bus number
  bits[15:11] = device number (slot)
  bits[10:8]  = function number
  bits[7:0]   = register number

مثال: <0x00000800 0 0 0 0>
  → bus=0, device=1, function=0 → devfn = (1<<3)|0 = 8
```

**`of_pci_get_devfn()`** يقرأ الـ 5 cells ويستخرج bits [15:8] من phys.hi.

```c
return (reg[0] >> 8) & 0xff;  /* يستخرج [device:function] */
```

### Lifecycle Diagram

```
Boot:
  Bootloader → loads DTB into RAM
  Kernel OF subsystem → unflatten_device_tree()
                       → builds device_node tree in memory

PCI Host Bridge Init:
  platform_driver probe()
    → of_pci_check_probe_only()      ← هل نحترم config الـ firmware؟
    → pci_scan_bus()
        → for each device found:
            → pci_set_of_node()
                → of_pci_find_child_device()  ← ابحث عن DT node
                → device_set_node()           ← ربط pci_dev بـ DT node
            → pci_assign_irq()
                → bridge->map_irq(dev, slot, pin)
                    → of_irq_parse_and_map_pci()  ← حوّل DT→VIRQ
                        → of_irq_parse_pci()
                            → of_irq_parse_one()    ← لو في DT node
                            → walk up PCI tree + swizzle ← لو مافيش
                        → irq_create_of_mapping()  ← ثبّت الـ VIRQ

Device Removal:
  pci_release_of_node()  ← of_node_put() → decrement refcount
```

### استراتيجية الـ Locking

- الـ `device_node` محمية بـ reference counting (`of_node_get/put`).
- الـ `of_pci_find_child_device` تستخدم `for_each_child_of_node` اللي بيعمل `of_node_get` تلقائي على كل child — لازم المستدعي يعمل `of_node_put` على الـ node المرجوع.
- الـ `pci_dev->dev.of_node` protected by the device model's reference counting.
- مافيش mutex خاص — الـ PCI enumeration بيحصل تحت `pci_lock_rescan_remove()`.

### الـ interrupt-map Parsing Flow

```
of_irq_parse_and_map_pci(pdev, slot, pin)
    │
    ├─ of_irq_parse_pci(pdev, &oirq)
    │       │
    │       ├─ pci_device_to_OF_node(pdev)  → dn (DT node للـ device)
    │       │
    │       ├─ [لو dn موجود]:
    │       │    of_irq_parse_one(dn, 0, &oirq)
    │       │
    │       └─ [لو dn مش موجود]:
    │            pci_read_config_byte(PCI_INTERRUPT_PIN, &pin)
    │            while (!ppnode):
    │                ppdev = pdev->bus->self
    │                if ppdev==NULL: ppnode = pci_bus_to_OF_node(bus)
    │                else: ppnode = pci_device_to_OF_node(ppdev)
    │                if !ppnode: pin = pci_swizzle_interrupt_pin(pin)
    │            of_irq_parse_raw(laddr, &oirq)
    │
    └─ irq_create_of_mapping(&oirq)  → VIRQ number
```

### الـ Probe-Only Flag

```
of_pci_check_probe_only()
    │
    ├─ of_pci_preserve_config(of_chosen)
    │       │
    │       ├─ reads "linux,pci-probe-only" from DT
    │       │   (falls back to /chosen node if not in PCI node)
    │       │
    │       └─ returns true/false
    │
    ├─ [true]:  pci_add_flags(PCI_PROBE_ONLY)
    └─ [false]: pci_clear_flags(PCI_PROBE_ONLY)
```

**`PCI_PROBE_ONLY`** يمنع الكيرنل من إعادة تخصيص BARs و bridge windows — مهم لـ systems بيعمل فيها الـ firmware assigned memory layout لازم يتحافظ عليه.
## Phase 4: شرح الـ Functions

### جدول سريع — كل الـ Functions

| الدالة | الـ Guard | الغرض |
|--------|-----------|--------|
| `of_pci_find_child_device()` | `CONFIG_OF && CONFIG_PCI` | ابحث عن DT child node بالـ devfn |
| `of_pci_get_devfn()` | `CONFIG_OF && CONFIG_PCI` | استخرج devfn من DT node |
| `of_pci_check_probe_only()` | `CONFIG_OF && CONFIG_PCI` | اضبط probe-only mode من DT |
| `of_irq_parse_and_map_pci()` | `CONFIG_OF_IRQ` | حوّل PCI IRQ من DT لـ VIRQ |

### الـ Stubs (عند غياب الـ CONFIG)

| الدالة | القيمة المُرجعة |
|--------|----------------|
| `of_pci_find_child_device()` stub | `NULL` |
| `of_pci_get_devfn()` stub | `-EINVAL` |
| `of_pci_check_probe_only()` stub | void (لا تفعل شيء) |
| `of_irq_parse_and_map_pci()` stub | `0` (= NO_IRQ) |

---

### تفاصيل كل دالة

#### 1. `of_pci_find_child_device()`

```c
struct device_node *of_pci_find_child_device(
    struct device_node *parent,  /* DT node للـ PCI bus */
    unsigned int devfn           /* device+function number */
);
```

**ما تفعله**: تبحث في children الـ DT node عن child له نفس الـ devfn. بتفحص كمان الـ nodes اللي اسمها `multifunc-device` (firmware قديمة بتعملها كـ wrapper لـ multi-function devices).

**المعاملات**:
- `parent`: الـ `device_node` للـ PCI bus أو host bridge.
- `devfn`: الـ 8-bit value — الـ bits [7:3] الـ slot، الـ bits [2:0] الـ function.

**القيمة المُرجعة**: مؤشر للـ `device_node` لو لقاه، أو `NULL`. المُستدعي مسؤول عن `of_node_put()`.

**الـ Locking**: لا mutex خاص — بيستخدم `for_each_child_of_node` اللي thread-safe مع OF RCU.

**من يستدعيها**: `pci_set_of_node()` في `drivers/pci/of.c` — بتتستدعى أثناء الـ PCI enumeration لكل `pci_dev`.

**Pseudocode**:
```
for each child of parent:
    get devfn from child via of_pci_get_devfn()
    if devfn matches: return child

    if child name == "multifunc-device":
        for each sub-child:
            if devfn matches: put(child); return sub-child

return NULL
```

---

#### 2. `of_pci_get_devfn()`

```c
int of_pci_get_devfn(struct device_node *np);
```

**ما تفعله**: تقرأ الـ `reg` property من الـ DT node وتستخرج منها الـ `devfn` (device+function number). الـ `reg` property في PCI DT nodes بتكون 5 cells (PCI address format).

**المعاملات**:
- `np`: الـ `device_node` للـ PCI device.

**القيمة المُرجعة**: الـ `devfn` كـ int موجب (0-255)، أو error code سالب لو `reg` property مش موجودة أو ناقصة.

**التفصيل التقني**:
```c
u32 reg[5];
of_property_read_u32_array(np, "reg", reg, 5);
return (reg[0] >> 8) & 0xff;
/* reg[0] = phys.hi في PCI address format
   bits[15:8] = devfn = (slot<<3)|function */
```

**الأخطاء الممكنة**:
- `-EINVAL`: لو الـ node مافيهاش `reg` property.
- قيمة سالبة أخرى: من `of_property_read_u32_array`.

**من يستدعيها**: `of_pci_find_child_device()` (داخليًا)، و `pcie-mt7621.c`، و `pci-mvebu.c` مباشرة.

---

#### 3. `of_pci_check_probe_only()`

```c
void of_pci_check_probe_only(void);
```

**ما تفعله**: بتشوف لو الـ DT عنده property اسمه `linux,pci-probe-only` في `/chosen` أو في الـ PCI controller node. لو موجود وقيمته 1، بتضيف `PCI_PROBE_ONLY` flag للـ PCI subsystem.

**`PCI_PROBE_ONLY`**: بيمنع الكيرنل من:
- إعادة تخصيص الـ BARs.
- إعادة ترتيب الـ bridge windows.
- الفرض يكون الـ firmware رتب كل حاجة صح.

**متى تستخدمها**: في `pci-host-common.c` أثناء تهيئة الـ host bridge:
```c
of_pci_check_probe_only();  /* قبل ما تبدأ scan الـ PCI bus */
```

**لا تأخذ parameters، لا ترجع قيمة**.

**Flow**:
```
of_pci_check_probe_only()
    → of_pci_preserve_config(of_chosen)
        → of_property_read_u32(node, "linux,pci-probe-only", &val)
        → if not found: retry with of_chosen node
        → return val != 0
    → if true:  pci_add_flags(PCI_PROBE_ONLY)
    → if false: pci_clear_flags(PCI_PROBE_ONLY)
```

---

#### 4. `of_irq_parse_and_map_pci()`

```c
int of_irq_parse_and_map_pci(
    const struct pci_dev *dev,  /* الـ PCI device محتاج IRQ */
    u8 slot,                    /* غير مستخدم (للتوافق مع callback signature) */
    u8 pin                      /* غير مستخدم (للتوافق مع callback signature) */
);
```

**ما تفعله**: الدالة الرئيسية لـ IRQ mapping. بتحول الـ PCI device لـ virtual IRQ number عن طريق الـ Device Tree.

**المعاملات**: `slot` و `pin` موجودين علشان الـ signature تتطابق مع `pci_host_bridge.map_irq` callback pointer. قيمهم بتتجاهل — الـ IRQ pin بيتقرأ من الـ PCI config space.

**القيمة المُرجعة**: الـ VIRQ number عند النجاح، أو `0` عند الفشل (0 = NO_IRQ في الـ PCI دنيا).

**الأخطاء**:
- لو مافيش `interrupt-map` في الـ DT: يطبع warning ويرجع 0.
- لو الـ PCI device مش عنده interrupt pin: يرجع 0 بصمت.

**من يستدعيها**:
- `pci_host_bridge.map_irq` pointer في `drivers/pci/of.c:645`.
- `pci-loongson.c:265` مباشرة.
- `pci-aardvark.c:1675` مباشرة.
- `pci-mvebu.c:1125` مباشرة.

**الـ Internal Flow التفصيلي**:
```
of_irq_parse_and_map_pci(pdev, _, _)
    │
    └─ of_irq_parse_pci(pdev, &oirq)    [static function]
            │
            ├─ [لو للـ device DT node]:
            │    of_irq_parse_one(dn, 0, &oirq)
            │    → reads "interrupts" or "interrupt-parent"
            │
            └─ [لو مافيش DT node]:
                 pci_read_config_byte(PCI_INTERRUPT_PIN, &pin)
                 if pin == 0: return -ENODEV

                 walk up PCI bus tree:
                   if P2P bridge with DT node: stop
                   else: pci_swizzle_interrupt_pin(pin) ← INTA→INTB→INTC→INTD
                   until reach host bridge DT node

                 build laddr[3]:
                   laddr[0] = (bus<<16)|(devfn<<8) in big-endian

                 of_irq_parse_raw(laddr, &oirq)
                   → walks interrupt-map property
                   → finds matching entry
                   → fills oirq.np (parent IRQ controller)
                   → fills oirq.args[] (IRQ specifier)

    └─ irq_create_of_mapping(&oirq)
         → finds/creates irq_domain for oirq.np
         → maps hardware IRQ to VIRQ
         → returns VIRQ number
```

### استخدام الـ `map_irq` Callback

```c
/* في drivers/pci/of.c — تسجيل الـ callback */
bridge->map_irq = of_irq_parse_and_map_pci;

/* الـ PCI core يستدعيها هكذا: */
irq = bridge->map_irq(dev, PCI_SLOT(dev->devfn), PCI_FUNC(dev->devfn));
dev->irq = irq;
```
## Phase 5: دليل الـ Debugging الشامل

### الـ Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_OF` | لازم مفعّل علشان الكود يشتغل |
| `CONFIG_OF_IRQ` | لازم لـ IRQ mapping |
| `CONFIG_PCI_DEBUG` | يفعّل `dev_dbg` messages في PCI subsystem |
| `CONFIG_DEBUG_SHIRQ` | يتحقق من shared IRQ handlers |
| `CONFIG_IRQ_DOMAIN_DEBUG` | debugfs entries لـ IRQ domains |
| `CONFIG_OF_DYNAMIC` | دعم تعديل الـ DT runtime |

### الـ Dynamic Debug

```bash
# تفعيل كل messages الـ PCI/OF في drivers/pci/of.c
echo 'file drivers/pci/of.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو تفعيل كل رسائل الـ PCI
echo 'module pci +p' > /sys/kernel/debug/dynamic_debug/control

# مشاهدة الرسائل
dmesg | grep "PCI: OF:"
```

### الـ sysfs — فحص الـ DT Node المرتبط بالـ PCI Device

```bash
# عرض الـ DT node للـ PCI device
ls -la /sys/bus/pci/devices/0000:00:00.0/of_node

# مثال على الـ output:
# lrwxrwxrwx /sys/bus/pci/devices/0000:00:00.0/of_node -> ../../../firmware/devicetree/base/soc/pcie@fe8b0000/device@0,0

# قراءة الـ device tree properties
ls /sys/firmware/devicetree/base/soc/pcie@fe8b0000/
# device@0,0/  #address-cells  #interrupt-cells  bus-range  compatible  interrupt-map  ranges  reg

cat /sys/firmware/devicetree/base/soc/pcie@fe8b0000/bus-range | xxd
# 00000000: 0000 0000 0000 00ff  ........ (bus 0 to 255)

# تحقق من interrupt-map
xxd /sys/firmware/devicetree/base/soc/pcie@fe8b0000/interrupt-map
```

### الـ debugfs — IRQ Domain

```bash
# عرض كل IRQ domains (يساعد في فهم الـ IRQ mapping)
cat /sys/kernel/debug/irq/domains/*/name 2>/dev/null

# عرض الـ mappings لـ domain معين
cat /sys/kernel/debug/irq/domains/pcie-irq-domain/

# عرض كل الـ IRQs المخصصة
cat /proc/interrupts | grep -i pci
```

### الـ PCI Device Scan Debugging

```bash
# عرض كل PCI devices مع IRQs
lspci -v | grep -E "^[0-9]|IRQ"

# مثال output:
# 0000:00:00.0 PCI bridge: ...
#   Interrupt: pin A routed to IRQ 32

# فحص الـ DT bindings لـ PCI controller
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "pcie@"
```

### الـ ftrace — Tracing الـ OF/PCI Functions

```bash
# trace مباشر للدوال باستخدام function tracer
echo 'of_pci_find_child_device' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'of_pci_get_devfn' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'of_pci_check_probe_only' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'of_irq_parse_and_map_pci' >> /sys/kernel/debug/tracing/set_ftrace_filter

echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# trigger PCI rescan
echo 1 > /sys/bus/pci/rescan

cat /sys/kernel/debug/tracing/trace
echo 0 > /sys/kernel/debug/tracing/tracing_on

# مثال output:
#  kworker/0:2-95  [000] .... of_pci_check_probe_only+0x0/0x3c
#  kworker/0:2-95  [000] .... of_pci_find_child_device+0x0/0x70
#  kworker/0:2-95  [000] .... of_pci_get_devfn+0x0/0x30
```

### الـ printk/dmesg — رسائل مهمة

```bash
# فحص رسائل الـ PCI OF أثناء boot
dmesg | grep "PCI: OF:"

# مثال رسائل طبيعية:
# [    2.341] PCI: OF: host bridge /soc/pcie@fe8b0000 ranges:
# [    2.342] PCI: OF:   MEM 0x000000f0000000..0x000000f3ffffff -> 0x00000000f0000000
# [    2.343] PCI: OF:   IO  0x000000fbe00000..0x000000fbeffff -> 0x0000000000000000

# رسائل خطأ:
# [    2.400] PCI: OF: of_irq_parse_and_map_pci: no interrupt-map found
# [    2.401] PCI: OF: possibly some PCI slots don't have level triggered interrupts
```

### جدول الأخطاء الشائعة

| رسالة الخطأ | المعنى | الحل |
|-------------|--------|-------|
| `no interrupt-map found` | الـ DT مش فيه `interrupt-map` property في الـ PCI controller node | أضف `interrupt-map` و`interrupt-map-mask` للـ DTS |
| `Incorrect value for linux,pci-probe-only` | الـ property موجودة بس قيمتها مش صح | اضبط القيمة لـ 0 أو 1 فقط |
| `More than one I/O resource converted` | في أكثر من IO range في الـ `ranges` | راجع الـ `ranges` property في الـ DTS |
| `I/O range found ... Please provide io_base` | الـ driver ما مرّرش io_base pointer | راجع الـ driver code |
| `Invalid end bus number` | `bus-range` end > 0xff | صحّح `bus-range` في الـ DTS |
| `failed with rc=-EINVAL` | فشل parsing الـ interrupt-map | افحص صحة الـ interrupt-map في الـ DTS |

### نقاط استراتيجية لـ `WARN_ON()` / `dump_stack()`

```c
/* في of_pci_get_devfn — لو الـ reg property مش موجودة */
int of_pci_get_devfn(struct device_node *np)
{
    u32 reg[5];
    int error = of_property_read_u32_array(np, "reg", reg, ARRAY_SIZE(reg));
    if (error) {
        /* نقطة جيدة: WARN_ONCE لأن ده غالبًا DTS bug */
        WARN_ONCE(1, "PCI: OF: missing 'reg' in %pOF\n", np);
        return error;
    }
    return (reg[0] >> 8) & 0xff;
}
```

### فحص الـ Hardware Level — DT Debugging

```bash
# تحقق من الـ DTB اللي حمّله الـ bootloader
ls /sys/firmware/fdt   # الـ raw DTB
dtc -I dtb -O dts /sys/firmware/fdt > /tmp/board.dts 2>/dev/null

# ابحث عن PCI nodes
grep -A 30 "pcie\|pci" /tmp/board.dts | head -60

# تحقق من الـ interrupt-map format
# يجب أن يكون:
# interrupt-map = <child_unit_addr child_irq_specifier parent_phandle parent_unit_addr parent_irq_specifier>;
# interrupt-map-mask = <0xf800 0 0 7>; ← [slot bits, 0, 0, pin bits]

# فحص حالة PCI controller registers (مثال Rockchip RK3562)
devmem2 0xfe8b0000 w   # PCI_STATUS register
devmem2 0xfe8b0004 w   # PCI_COMMAND register

# فحص الـ IRQ assignment
cat /proc/interrupts | grep -E "Edge|Level" | head -20
```

### Shell Commands الجاهزة للنسخ

```bash
#!/bin/bash
# script لفحص شامل لـ OF/PCI setup

echo "=== PCI Devices with OF nodes ==="
for dev in /sys/bus/pci/devices/*/; do
    devid=$(basename $dev)
    if [ -L "$dev/of_node" ]; then
        node=$(readlink $dev/of_node)
        irq=$(cat $dev/irq 2>/dev/null)
        echo "$devid → DT: $node | IRQ: $irq"
    fi
done

echo ""
echo "=== IRQ Domain Info ==="
cat /sys/kernel/debug/irq/irqs/* 2>/dev/null | grep -E "irq|domain" | head -20

echo ""
echo "=== PCI IRQs from /proc/interrupts ==="
cat /proc/interrupts | grep -i "pci\|pcie"

echo ""
echo "=== DT PCI Properties ==="
for pci_node in /sys/firmware/devicetree/base/*/pcie* \
                /sys/firmware/devicetree/base/soc/*/pcie*; do
    [ -d "$pci_node" ] || continue
    echo "Node: $pci_node"
    ls "$pci_node/"
done
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — IRQ لا يشتغل

**العنوان**: PCI-to-Ethernet card على Gateway صناعي لا تستقبل interrupts

**السياق**: Industrial gateway بيشغّل Linux على Rockchip RK3562. في PCIe x1 slot فيه Intel I211 NIC. الكارت بيتعرف عليه الـ `lspci` بس الـ interface مش شغّالة.

**المشكلة**:
```
dmesg:
[    3.201] PCI: OF: of_irq_parse_and_map_pci: no interrupt-map found, INTx interrupts not available
[    3.202] igb 0000:01:00.0: Failed to initialize interrupt allocation
```

**التحليل**:
`of_irq_parse_and_map_pci()` استدعت `of_irq_parse_pci()`. الـ I211 مش عنده DT node (الـ device_node = NULL)، فبدأت تمشي فوق في الـ PCI tree. لقت الـ host bridge node في الـ DT بس مالقتش فيها `interrupt-map` property. رجعت -ENOENT.

```c
/* في of_irq_parse_pci — الكود الحرفي */
if (rc == -ENOENT) {
    dev_warn(&pdev->dev,
        "%s: no interrupt-map found, INTx interrupts not available\n",
        __func__);
}
```

**الحل**:
أضف الـ `interrupt-map` للـ DTS:

```dts
/* في rk3562-gateway.dts */
&pcie2x1 {
    status = "okay";
    pinctrl-names = "default";
    reset-gpios = <&gpio1 RK_PB2 GPIO_ACTIVE_HIGH>;

    /* هذا اللي كان ناقص */
    #interrupt-cells = <1>;
    interrupt-map-mask = <0 0 0 7>;
    interrupt-map = <0 0 0 1 &pcie_intc 0>,  /* INTA */
                   <0 0 0 2 &pcie_intc 1>,  /* INTB */
                   <0 0 0 3 &pcie_intc 2>,  /* INTC */
                   <0 0 0 4 &pcie_intc 3>;  /* INTD */

    pcie_intc: interrupt-controller {
        #interrupt-cells = <1>;
        interrupt-controller;
    };
};
```

**الدرس**: كل PCIe host bridge node في الـ DT لازم يحتوي على `interrupt-map` حتى لو الـ device بيستخدم MSI — لأن الـ legacy INTx fallback محتاجه.

---

### السيناريو 2: STM32MP1 — Board Bring-up وإشكالية الـ devfn

**العنوان**: custom board بـ STM32MP1 — PCI WiFi module يظهر في الـ bus الغلط

**السياق**: Custom industrial board بـ STM32MP157 SoC وـ PCIe WiFi module (QCA6174). الـ `lspci` بيشوف الـ device على `0000:00:00.0` بدل `0000:01:00.0`.

**المشكلة**:
```
dmesg:
[    2.891] of_pci_find_child_device: comparing devfn=0x00 vs node devfn=0x08
[    2.892] pci 0000:00:00.0: [168c:003e] type 00 class 0x028000
```

الـ WiFi module على الـ slot 1 (devfn=0x08) بس الـ DT node بيقول `reg = <0x0000 0 0 0 0>` اللي معناه devfn=0.

**التحليل**:
`of_pci_get_devfn()` بيقرأ:
```c
return (reg[0] >> 8) & 0xff;
/* reg[0] = 0x0000 → (0x0000 >> 8) & 0xff = 0x00 */
```

المفروض الـ `reg` تكون `<0x0800 0 0 0 0>` — لأن device number=1 معناه bits[15:11] = 1 → value = 0x0800.

**الحل**:
```dts
/* غلط */
pcie-wifi@0,0 {
    reg = <0x0000 0 0 0 0>;   /* device 0, function 0 */
};

/* صح */
pcie-wifi@1,0 {
    reg = <0x0800 0 0 0 0>;   /* device 1 (bit 11 set), function 0 */
    /* phys.hi: bits[15:11]=device=1 → 0x0800 */
};
```

**الدرس**: الـ PCI DT `reg` format مش زي الـ reg العادية. الـ device number بيبدأ من bit 11 في الـ phys.hi cell. راجع ePAPR spec.

---

### السيناريو 3: i.MX8MQ — Probe-Only Bug مع Android Auto

**العنوان**: NVMe SSD يختفي بعد الـ reboot على Android Automotive ECU

**السياق**: Automotive ECU بـ NXP i.MX8MQ يشغّل Android. PCIe NVMe SSD يشتغل تمام أول boot، لكن بعد warm reboot يختفي أو يعطي corruption.

**المشكلة**:
الـ Android bootloader بيحمّل الـ NVMe driver وبيعمل له init على عناوين معينة. بعد الـ kernel boot، الـ PCI subsystem بيعيد تخصيص الـ BARs وبيغير العناوين، فالـ DMA setup القديم بيبقى غلط.

```bash
# dmesg يظهر:
[    2.100] pcieport 0000:00:00.0: BAR 0: assigned [mem 0x18000000-0x18ffffff]
# لكن الـ bootloader حطّه على 0x40000000 !
```

**التحليل**:
`of_pci_check_probe_only()` في `pci-host-common.c` استدعت `of_pci_preserve_config(of_chosen)`. الـ `linux,pci-probe-only` property مش موجودة في الـ DT، فـ `pci_clear_flags(PCI_PROBE_ONLY)` اتستدعت وسمحت للكيرنل يعيد تخصيص كل حاجة.

**الحل**:
```dts
/* في /chosen node في الـ DTS */
chosen {
    stdout-path = &uart1;
    linux,pci-probe-only = <1>;   /* امنع الكيرنل من إعادة تخصيص BARs */
};
```

أو أضفها في الـ PCIe controller node مباشرة:
```dts
&pcie0 {
    linux,pci-probe-only = <1>;
    status = "okay";
};
```

**الدرس**: على الـ Automotive systems حيث الـ bootloader بيعمل PCI init كامل، `linux,pci-probe-only` ضروري لمنع الـ kernel من مسح config الـ firmware.

---

### السيناريو 4: AM62x — IoT Gateway وإشكالية الـ multifunc-device

**العنوان**: PCI multi-function device على TI AM62x لا يتعرف عليه بشكل صحيح

**السياق**: IoT Gateway بـ Texas Instruments AM62x SoC وـ PCIe multi-function card (Intel I350 — 4-port Ethernet). الكيرنل بيشوف فقط function 0 ومش بيربط functions 1-3 بالـ DT nodes.

**المشكلة**:
الـ DTS مكتوب كالتالي:
```dts
pcie@0 {
    i350@0,0 { reg = <0x0000 0 0 0 0>; };  /* function 0 */
    i350@0,1 { reg = <0x0001 0 0 0 0>; };  /* function 1 */
    i350@0,2 { reg = <0x0002 0 0 0 0>; };  /* function 2 */
    i350@0,3 { reg = <0x0003 0 0 0 0>; };  /* function 3 */
};
```

بس الـ firmware القديم على بعض الـ boards بيولّد:
```dts
pcie@0 {
    multifunc-device {
        i350@0,0 { reg = <0x0000 0 0 0 0>; };
        i350@0,1 { reg = <0x0001 0 0 0 0>; };
    };
};
```

**التحليل**:
`of_pci_find_child_device()` عنده handling خاص لـ `multifunc-device`:

```c
if (of_node_name_eq(node, "multifunc-device")) {
    for_each_child_of_node(node, node2) {
        if (__of_pci_pci_compare(node2, devfn)) {
            of_node_put(node);
            return node2;  /* يرجع الـ sub-child مباشرة */
        }
    }
}
```

الكود كان بيتجاهل الـ `multifunc-device` wrapper، لكن بعد الـ fix صار يدخل جواها.

**الحل**: الـ kernel بيعالج ده تلقائيًا. المطلوب فقط إن الـ DTS يستخدم `multifunc-device` كاسم صريح:

```dts
pcie@0 {
    multifunc-device {           /* الاسم لازم يكون بالضبط هكذا */
        #address-cells = <1>;
        #size-cells = <0>;
        ethernet@0,0 { reg = <0x0000 0 0 0 0>; };
        ethernet@0,1 { reg = <0x0001 0 0 0 0>; };
    };
};
```

**الدرس**: الـ `of_pci_find_child_device()` عارف بـ `multifunc-device` wrapper. استخدم الاسم الصريح ده لو محتاج تغلّف multiple functions.

---

### السيناريو 5: Allwinner H616 — Android TV Box وـ PCIe WiFi IRQ Swizzling

**العنوان**: WiFi intermittent على Android TV Box بـ Allwinner H616

**السياق**: Android TV Box شعبي بـ Allwinner H616 SoC وـ PCIe WiFi chip (RTL8822CE). الـ WiFi بيشتغل بس فيه packet loss عالي وأحيانًا kernel warnings عن missed interrupts.

**المشكلة**:
```
dmesg:
[   45.231] rtw_8822ce 0000:01:00.0: firmware failed to leave power save mode
[   45.232] irq: type mismatch, failed to map hwirq-3 for /soc/pcie/interrupt-controller!
```

**التحليل**:
الـ RTL8822CE مش عنده DT node (شايع في الـ vendor kernels). `of_irq_parse_pci()` بتمشي فوق في الـ PCI tree:

```c
/* walk up: device → bus → host bridge */
pin = pci_swizzle_interrupt_pin(pdev, pin);
/* INTA (pin=1) على slot 0 → swizzle → INTA
   INTB (pin=2) على slot 0 → swizzle → INTB
   لكن الـ interrupt-map فيه entries بس لـ INTA */
```

الـ DTS `interrupt-map` فيها entry واحدة فقط لـ INTA، بس الـ WiFi chip بيستخدم INTB:

```dts
/* الـ DTS الخاطئ */
interrupt-map = <0 0 0 1 &plic 135 4>;  /* INTA فقط */
interrupt-map-mask = <0 0 0 7>;
```

**الحل**:
```dts
/* الـ DTS الصح — أضف كل الـ 4 pins */
interrupt-map = <0 0 0 1 &plic 135 IRQ_TYPE_LEVEL_HIGH>,  /* INTA */
               <0 0 0 2 &plic 136 IRQ_TYPE_LEVEL_HIGH>,  /* INTB */
               <0 0 0 3 &plic 137 IRQ_TYPE_LEVEL_HIGH>,  /* INTC */
               <0 0 0 4 &plic 138 IRQ_TYPE_LEVEL_HIGH>;  /* INTD */
interrupt-map-mask = <0xf800 0 0 7>;
```

ثم تحقق من الـ IRQ type — PCIe legacy INTx لازم تكون `IRQ_TYPE_LEVEL_HIGH`:
```bash
cat /proc/interrupts | grep 135  # تحقق من الـ IRQ type
```

**الدرس**: الـ `interrupt-map` لازم تشمل كل الـ 4 INTx pins (INTA-INTD) وتحدد `IRQ_TYPE_LEVEL_HIGH`. الـ swizzling بيغير الـ pin على طول الـ PCI tree — قد تكون محتاج entry لكل pin حتى لو device واحد موجود.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الرابط |
|--------|--------|
| Open Firmware device tree virtual filesystem | [lwn.net/Articles/215853](https://lwn.net/Articles/215853/) |
| Generate device tree node for PCI devices | [lwn.net/Articles/917999](https://lwn.net/Articles/917999/) |
| Device trees I: Are we having fun yet? | [lwn.net/Articles/572692](https://lwn.net/Articles/572692/) |
| Device Tree on ARM platform (patch discussion) | [lwn.net/Articles/334826](https://lwn.net/Articles/334826/) |
| Open Firmware and Devicetree — kernel docs | [static.lwn.net/kerneldoc/devicetree/index.html](https://static.lwn.net/kerneldoc/devicetree/index.html) |
| Linux and the Devicetree — usage model | [static.lwn.net/kerneldoc/devicetree/usage-model.html](https://static.lwn.net/kerneldoc/devicetree/usage-model.html) |

### توثيق الـ Kernel الرسمي

```
Documentation/PCI/
├── pci.rst                   ← دليل PCI للـ driver writers
├── pciebus-howto.rst         ← PCIe bus operations
└── endpoint/                 ← PCIe endpoint framework

Documentation/devicetree/bindings/pci/
├── pci-bus.yaml              ← الـ DT bindings للـ PCI bus
├── rockchip-pcie.yaml        ← Rockchip PCIe bindings
├── snps,dw-pcie.yaml         ← DesignWare PCIe (STM32, i.MX8, etc.)
└── ti,j721e-pcie-host.yaml   ← TI PCIe (AM62x)

Documentation/driver-api/
└── pci/                      ← PCI driver API reference
```

### الملفات الأساسية في الـ Kernel Source

```
include/linux/of_pci.h        ← هذا الملف (الـ API)
drivers/pci/of.c              ← الـ implementation
drivers/pci/pci.h             ← internal PCI definitions
drivers/pci/probe.c           ← PCI device discovery
drivers/pci/controller/
├── pci-host-common.c         ← generic host bridge
├── pci-host-generic.c        ← simple generic host
├── dwc/pcie-designware.c     ← DesignWare PCIe core
└── pcie-rockchip.c           ← Rockchip PCIe
```

### Commits مهمة في الـ Git History

```bash
# ابحث في الـ git log عن commits متعلقة بـ of_pci
git log --oneline -- drivers/pci/of.c | head -20
git log --oneline -- include/linux/of_pci.h | head -20

# commits مهمة تاريخيًا:
# - إضافة of_pci_find_child_device (2011, IBM Corp)
# - إضافة of_pci_check_probe_only
# - إضافة multifunc-device support في of_pci_find_child_device
```

### مواقع eLinux.org

| الصفحة | الرابط |
|--------|--------|
| Device Tree Reference | [elinux.org/Device_Tree_Reference](https://elinux.org/Device_Tree_Reference) |
| Device Tree Mysteries | [elinux.org/Device_Tree_Mysteries](https://elinux.org/Device_Tree_Mysteries) |
| Linux Drivers Device Tree Guide | [elinux.org/Linux_Drivers_Device_Tree_Guide](https://elinux.org/Linux_Drivers_Device_Tree_Guide) |
| Device Tree Presentations | [elinux.org/Device_Tree_Presentations](https://elinux.org/Device_Tree_Presentations) |
| Device Tree frowand | [elinux.org/Device_Tree_frowand](https://elinux.org/Device_Tree_frowand) |

### kernelnewbies.org

| الصفحة | الرابط |
|--------|--------|
| PCI 'ranges' property discussion | [lists.kernelnewbies.org/pipermail/kernelnewbies/2023-February/022848.html](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2023-February/022848.html) |
| Shared interrupts in devicetree | [lists.kernelnewbies.org/pipermail/kernelnewbies/2015-April/013955.html](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-April/013955.html) |

### الـ Standards والمراجع التقنية

| المرجع | الوصف |
|--------|--------|
| ePAPR (Power.org Embedded Platform Architecture Requirements) | المعيار الرسمي لـ Device Tree على embedded systems — يعرّف PCI `reg` format |
| PCI Local Bus Specification | معيار PCI الأصلي — يشرح interrupt pins INTA-INTD وـ swizzling |
| PCIe Base Specification | معيار PCIe — يشرح legacy interrupt emulation |
| Open Firmware IEEE 1275-1994 | المعيار الأصلي لـ OF/DT |

### الكتب المنصوح بيها

| الكتاب | الأهمية |
|--------|---------|
| **Linux Device Drivers, 3rd Ed (LDD3)** — Corbet, Rubini, Kroah-Hartman | الفصل 12: PCI drivers. متاح مجانًا على lwn.net |
| **Linux Kernel Development, 3rd Ed** — Robert Love | فصل جيد عن device model وـ OF |
| **Embedded Linux Primer** — Christopher Hallinan | يشرح Device Tree بتفصيل للـ embedded systems |
| **Professional Linux Kernel Architecture** — Wolfgang Mauerer | يغطي PCI subsystem بعمق |

### Search Terms للبحث عن معلومات أكثر

```
# للـ Google:
"linux kernel pci of_pci interrupt-map site:lore.kernel.org"
"linux pci device tree interrupt swizzling"
"linux,pci-probe-only device tree"
"of_irq_parse_and_map_pci implementation"
"PCI devicetree bindings interrupt-map format"

# للـ lore.kernel.org:
https://lore.kernel.org/linux-pci/?q=of_pci

# لـ elixir.bootlin.com (cross-reference الكود):
https://elixir.bootlin.com/linux/latest/source/drivers/pci/of.c
https://elixir.bootlin.com/linux/latest/source/include/linux/of_pci.h
```

### الـ Mailing List

الـ patches والنقاشات المتعلقة بهذا الكود بتعدي على:
- **linux-pci@vger.kernel.org**
- **devicetree@vger.kernel.org** (للـ DT bindings)

### أدوات مفيدة

```bash
# dtc - Device Tree Compiler
sudo apt install device-tree-compiler

# تحويل DTB لـ DTS
dtc -I dtb -O dts /boot/dtb/board.dtb -o board.dts

# التحقق من صحة DTS
dtc -I dts -O dtb board.dts -o /dev/null

# fdtdump - عرض DTB بصريًا
fdtdump /boot/dtb/board.dtb | grep -A 30 pcie
```
## Phase 8: Writing simple module

### الـ Module — kprobe على `of_irq_parse_and_map_pci`

الدالة المختارة هي `of_irq_parse_and_map_pci` — هي exported وآمنة للـ kprobe وبترجع قيمة مفيدة (الـ VIRQ number). كل مرة PCI device بياخد IRQ assignment من الـ DT، الـ kprobe هيطبع معلومات الـ device والـ VIRQ.

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * of_pci_irq_probe.c
 * kprobe على of_irq_parse_and_map_pci — يطبع IRQ assignments للـ PCI devices
 * من الـ Device Tree.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>     /* kprobe API */
#include <linux/pci.h>         /* struct pci_dev, PCI_SLOT, PCI_FUNC */
#include <linux/printk.h>      /* pr_info */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Generator");
MODULE_DESCRIPTION("kprobe monitor for of_irq_parse_and_map_pci — shows PCI IRQ DT assignments");

/*
 * الـ pre-handler يشتغل قبل ما الدالة تتنفذ.
 * بنقرأ الـ arguments من الـ registers:
 *   regs->di = arg0 = const struct pci_dev *dev
 *   regs->si = arg1 = u8 slot (unused in callee)
 *   regs->dx = arg2 = u8 pin  (unused in callee)
 *
 * هذا الجزء يعرض معلومات الـ PCI device قبل ما يبدأ الـ IRQ parsing —
 * مفيد لمعرفة أي device طلب IRQ mapping من الـ DT.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* استخرج الـ pci_dev pointer من الـ first argument register */
    struct pci_dev *pdev = (struct pci_dev *)regs->di;

    /*
     * تحقق إن الـ pointer مش NULL قبل ما نقرأ منه —
     * الكيرنل ممكن يستدعي الدالة في أي وقت أثناء الـ PCI scan.
     */
    if (!pdev)
        return 0;

    pr_info("of_pci_irq_probe: PRE → [%04x:%04x] %s "
            "BDF=%04x:%02x:%02x.%x | pin=%u | devfn=0x%02x\n",
            pdev->vendor,
            pdev->device,
            pdev->dev.of_node ? "HAS_DT_NODE" : "NO_DT_NODE",
            pci_domain_nr(pdev->bus),
            pdev->bus->number,
            PCI_SLOT(pdev->devfn),
            PCI_FUNC(pdev->devfn),
            pdev->pin,
            pdev->devfn);

    /*
     * رجوع 0 يعني "استمر في تنفيذ الدالة الأصلية".
     * رجوع 1 كان سيوقف التنفيذ — لا نريد ذلك هنا.
     */
    return 0;
}

/*
 * الـ post-handler يشتغل بعد ما الدالة ترجع قيمتها.
 * بنستخدمه لطباعة الـ VIRQ المُخصص —
 * هذا هو الجزء المفيد لأنه يظهر النتيجة الفعلية.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * الـ return value موجود في regs->ax على x86_64.
     * هو الـ VIRQ number أو 0 لو فشل الـ mapping.
     */
    unsigned long virq = regs->ax;

    if (virq == 0)
        pr_info("of_pci_irq_probe: POST → VIRQ=0 (NO_IRQ) — "
                "no interrupt-map found or device has no IRQ pin\n");
    else
        pr_info("of_pci_irq_probe: POST → VIRQ=%lu mapped successfully "
                "from Device Tree\n", virq);
}

/*
 * تعريف الـ kprobe struct —
 * بنحدد اسم الدالة كـ symbol_name بدل الـ address
 * لأنه أكثر portable بين الـ kernel versions.
 */
static struct kprobe kp = {
    .symbol_name    = "of_irq_parse_and_map_pci",
    .pre_handler    = handler_pre,
    .post_handler   = handler_post,
};

/*
 * module_init — نسجّل الـ kprobe عند تحميل الـ module.
 * لو فشل register_kprobe، يعني إما الدالة مش موجودة
 * (CONFIG_OF_IRQ مش مفعّل) أو الـ kprobes غير مدعومة.
 */
static int __init of_pci_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("of_pci_irq_probe: register_kprobe failed: %d\n", ret);
        pr_err("of_pci_irq_probe: ensure CONFIG_OF_IRQ=y and CONFIG_KPROBES=y\n");
        return ret;
    }

    pr_info("of_pci_irq_probe: kprobe registered on %s at %p\n",
            kp.symbol_name, kp.addr);
    pr_info("of_pci_irq_probe: trigger with: echo 1 > /sys/bus/pci/rescan\n");

    return 0;
}

/*
 * module_exit — نلغي تسجيل الـ kprobe عند إزالة الـ module.
 * ضروري لمنع kernel crash بعد ما الـ module code يتحذف من الذاكرة.
 */
static void __exit of_pci_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("of_pci_irq_probe: kprobe unregistered from %s\n",
            kp.symbol_name);
}

module_init(of_pci_probe_init);
module_exit(of_pci_probe_exit);
```

---

### الـ Makefile لبناء الـ Module

```makefile
# Makefile
obj-m += of_pci_irq_probe.o

# غيّر KDIR لمسار الـ kernel headers على نظامك
KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### كيفية الاستخدام

```bash
# بناء الـ module
make

# تحميل الـ module
sudo insmod of_pci_irq_probe.ko

# trigger الـ PCI rescan لتشغيل الـ kprobe
sudo sh -c 'echo 1 > /sys/bus/pci/rescan'

# أو unload وreload PCI driver معين
sudo modprobe -r iwlwifi && sudo modprobe iwlwifi

# مشاهدة الـ output
dmesg | grep "of_pci_irq_probe"

# مثال output:
# [   10.541] of_pci_irq_probe: kprobe registered on of_irq_parse_and_map_pci at ffffffffc0812340
# [   10.542] of_pci_irq_probe: trigger with: echo 1 > /sys/bus/pci/rescan
# [   12.100] of_pci_irq_probe: PRE → [8086:002a] HAS_DT_NODE BDF=0000:01:00.0 | pin=1 | devfn=0x00
# [   12.101] of_pci_irq_probe: POST → VIRQ=32 mapped successfully from Device Tree
# [   12.200] of_pci_irq_probe: PRE → [10ec:8822] NO_DT_NODE BDF=0000:02:00.0 | pin=1 | devfn=0x00
# [   12.201] of_pci_irq_probe: POST → VIRQ=0 (NO_IRQ) — no interrupt-map found

# إزالة الـ module
sudo rmmod of_pci_irq_probe
```

---

### شرح أجزاء الـ Module

| الجزء | السبب |
|-------|-------|
| `kprobe.symbol_name` | بنستخدم الاسم بدل الـ address علشان يشتغل على أي kernel version بدون ما نحسب الـ address يدويًا |
| `handler_pre` | بيشتغل قبل الدالة — بنقرأ الـ arguments (الـ pci_dev) من الـ registers لأنها لسه موجودة |
| `handler_post` | بيشتغل بعد الـ return — بنقرأ الـ VIRQ result من `regs->ax` |
| `regs->di` | على x86_64 الـ System V ABI، أول argument بيتبعت في RDI register |
| `regs->ax` | الـ return value على x86_64 بيتحط في RAX register |
| `HAS_DT_NODE check` | يوضح لو الـ device عنده DT node مرتبط — مفيد لتشخيص إشكاليات الـ DT mapping |
| `unregister_kprobe` في exit | ضروري لأن الـ kprobe بيعدّل كود الـ kernel مباشرة — لو ما أزلناهش والـ module اتحذف، الـ kernel هيـcrash |

### متطلبات الـ Kernel Config

```
CONFIG_KPROBES=y
CONFIG_KPROBE_EVENTS=y    # اختياري — لـ perf/eBPF integration
CONFIG_OF_IRQ=y           # لازم لوجود of_irq_parse_and_map_pci
CONFIG_PCI=y
CONFIG_OF=y
```
