## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ FDT؟ — تشبيه من الحياة

تخيل إنك اشتريت جهاز إلكتروني جديد، وجه معاه كتيب تعليمات بيقولك: "الجهاز ده فيه شاشة LCD على الـ bus رقم 2، وفيه sensor حرارة على الـ I2C عنوان 0x48، والـ RAM حجمها 512MB تبدأ من العنوان 0x80000000." الـ **Flattened Device Tree (FDT)** هو بالظبط الكتيب ده — بس للـ Linux kernel.

لما الـ bootloader (زي U-Boot) بيشغل الـ kernel، بيبعتله blob صغير في الـ RAM بيوصف كل الـ hardware الموجود على الـ board — من غير ما الـ kernel يحتاج يحفظ كود خاص لكل board. ده الـ **Flattened Device Tree Blob** أو الـ `.dtb`.

### المشكلة التي كانت موجودة

قديمًا، كل معالج ARM كان عنده كود hard-coded جوه الـ kernel يقول "الـ UART على العنوان ده، والـ RAM من هنا لهنا." لما عدد الـ boards وصل للآلاف، الكود بقى كارثة. الـ FDT جه كحل: نفصل وصف الـ hardware عن كود الـ kernel.

### دور الـ `of_fdt.h`

**الـ `of_fdt.h`** هو الـ header الرئيسي اللي بيحدد الـ API الخاص بـ:
1. **قراءة الـ FDT blob** في وقت الـ boot المبكر جدًا (قبل ما الـ memory allocator يشتغل).
2. **unflatten** الـ blob ده وتحويله لشجرة `device_node` تقدر تتعامل معاها بسهولة.
3. **scan** الـ blob مباشرةً عشان تجيب معلومات أساسية زي الـ memory، الـ cmdline، والـ reserved regions.

### القصة كاملة من أول boot

```
[ Power ON ]
      ↓
[ Bootloader (U-Boot / UEFI) ]
   - يحمّل الـ kernel image في RAM
   - يحمّل الـ .dtb blob في RAM
   - يبعت عنوان الـ blob في register (مثلاً r2 على ARM)
      ↓
[ Kernel Entry Point (head.S) ]
   - يحفظ عنوان الـ blob في initial_boot_params
      ↓
[ early_init_devtree() ]   ← معرفة في of_fdt.h
   - يقرأ الـ blob بدون تخصيص ميموري
   - يجيب الـ cmdline والـ memory map
      ↓
[ unflatten_device_tree() ] ← معرفة في of_fdt.h
   - يحول الـ blob لشجرة device_node في الـ RAM
      ↓
[ Drivers يستخدموا of_find_node_by_*() ]
```

### الـ Subsystem

حسب الـ `MAINTAINERS`:
- **Subsystem**: OPEN FIRMWARE AND FLATTENED DEVICE TREE
- **Maintainers**: Rob Herring `<robh@kernel.org>`، Saravana Kannan `<saravanak@google.com>`
- **Mailing List**: `devicetree@vger.kernel.org`
- **Git Tree**: `git://git.kernel.org/pub/scm/linux/kernel/git/robh/linux.git`

### الملفات المرتبطة

| الملف | الدور |
|---|---|
| `include/linux/of_fdt.h` | الـ header موضوع شرحنا — API للـ FDT parsing |
| `include/linux/of.h` | الـ API الرئيسي للتعامل مع `device_node` بعد الـ unflatten |
| `drivers/of/fdt.c` | الـ implementation الفعلي لكل دوال الـ of_fdt.h |
| `drivers/of/base.c` | الـ OF core — التعامل مع شجرة الـ nodes |
| `drivers/of/of_reserved_mem.c` | معالجة الـ reserved-memory nodes |
| `scripts/dtc/` | الـ Device Tree Compiler — يحول `.dts` لـ `.dtb` |
| `arch/arm64/boot/dts/` | ملفات `.dts` الخاصة بكل board |
| `Documentation/devicetree/` | توثيق الـ DT bindings |

### ملفات الـ Core الرئيسية للـ Subsystem

- **Core**: `drivers/of/fdt.c`, `drivers/of/base.c`, `drivers/of/property.c`
- **Headers**: `include/linux/of.h`, `include/linux/of_fdt.h`, `include/linux/fdt.h`
- **HW Drivers**: `drivers/of/platform.c` (ينشئ platform devices من الـ DT)
- **Reserved Memory**: `drivers/of/of_reserved_mem.c`
- **Overlay**: `drivers/of/overlay.c`
- **DT Compiler**: `scripts/dtc/libfdt/` (مكتبة قراءة الـ FDT)
## Phase 2: شرح الـ OF/FDT Framework

### المشكلة

الـ ARM ecosystem فيه مئات الـ SoCs وآلاف الـ boards. قبل الـ Device Tree، كل board كان عنده `arch/arm/mach-*` خاص بيه بيحتوي على:
- عناوين الـ peripherals hard-coded
- حجم الـ RAM وعناوينه
- ترتيب تهيئة الـ devices

ده معناه إن patch واحدة لـ board جديد ممكن تضيف آلاف الأسطر من الكود الـ boilerplate، وكان في وقت فيه أكتر من 1000 ملف `board-*.c` في kernel tree. الـ ARM maintainer Linus Torvalds وصفه بالكارثة في 2011.

### الحل — الـ Device Tree

الـ Device Tree هو standard بيأتي من **Open Firmware** (PowerPC/SPARC). الفكرة: وصف الـ hardware في ملف منفصل (`.dts`) يتحول لـ binary blob (`.dtb`) ويتبعت للـ kernel في الـ boot.

الـ kernel بيقرأ الـ blob ده بأسلوبين:
1. **Early scan**: قراءة مباشرة من الـ blob قبل ما الـ memory management يشتغل.
2. **Unflatten**: تحويل الـ blob لشجرة `device_node` في الـ heap لاستخدام الـ drivers.

### معمارية الـ Framework

```
+------------------+     +------------------+     +------------------+
|   Bootloader     |     |  .dtb blob       |     |    DTS source    |
|  (U-Boot/UEFI)   |---->|  (binary FDT)    |<----|  (human-readable)|
+------------------+     +------------------+     +------------------+
         |                        |
         | passes phys addr       |
         v                        v
+--------------------------------------------------+
|              Linux Kernel Boot                    |
|  +-----------------+   +----------------------+  |
|  | early_init_     |   | early_init_dt_scan_  |  |
|  | devtree()       |   | chosen()             |  |   <-- of_fdt.h API
|  | early_init_dt_  |   | early_init_dt_scan_  |  |
|  | scan()          |   | memory()             |  |
|  +-----------------+   +----------------------+  |
|           |                                       |
|           v                                       |
|  +---------------------------+                    |
|  | unflatten_device_tree()   |   <-- of_fdt.h    |
|  | (allocates device_node    |                    |
|  |  structs from blob)       |                    |
|  +---------------------------+                    |
|           |                                       |
|           v                                       |
|  +---------------------------+                    |
|  |   OF tree (device_node)   |   of.h API        |
|  |   /                       |                    |
|  |   ├── cpus/               |                    |
|  |   ├── memory@80000000     |                    |
|  |   ├── soc/                |                    |
|  |   │   ├── uart@ff000000   |                    |
|  |   │   └── i2c@ff010000    |                    |
|  |   └── chosen              |                    |
|  +---------------------------+                    |
|           |                                       |
|           v                                       |
|  +---------------------------+                    |
|  |  of_platform_populate()   |                    |
|  |  ينشئ platform_device     |                    |
|  |  لكل node في الشجرة       |                    |
|  +---------------------------+                    |
|           |                                       |
|           v                                       |
|  [ Drivers يعملوا probe() ويشتغلوا ]              |
+--------------------------------------------------+
```

### التشبيه المُعمّق

| مفهوم حقيقي | مكافئه في الـ Kernel |
|---|---|
| كتيب تعليمات الجهاز | الـ `.dtb` blob |
| فهرس الكتيب (flat) | الـ FDT structure (مرحلة early) |
| ترتيب المحتوى كشجرة | شجرة `device_node` (بعد unflatten) |
| باحث في الكتيب | `of_scan_flat_dt()` |
| مترجم الكتيب | `unflatten_device_tree()` |
| مستخدم المعلومات | الـ drivers عبر `of.h` API |

### الـ Core Abstraction

**الـ FDT blob** هو binary format بيتكون من:
- **header**: magic number `0xd00dfeed`، version، حجم الـ blob
- **memory reservation block**: مناطق RAM محجوزة
- **structure block**: شجرة الـ nodes والـ properties مُسلسَلة
- **strings block**: أسماء الـ properties

الـ `of_fdt.h` بيوفر:
- **Early API** (قبل الـ allocator): دوال `of_scan_flat_dt_*` و`early_init_dt_*` — تقرأ مباشرةً من الـ blob
- **Post-boot API**: `of_fdt_unflatten_tree()` — تحول الـ blob لشجرة ديناميكية

### ما يملكه الـ Subsystem وما يفوضه

**الـ OF/FDT يملك**:
- تعريف الـ FDT format وقراءته
- تحويل الـ blob لشجرة `device_node`
- الـ early boot memory/cmdline/reserved-mem scanning
- الـ `sysfs` representation لشجرة الـ OF (`/sys/firmware/devicetree/`)

**يفوضه للـ Drivers**:
- تفسير الـ properties الخاصة بكل device (مثلاً `clock-frequency`)
- ربط الـ node بالـ driver عبر `compatible`
- الـ interrupt و clock و regulator parsing (عبر subsystems منفصلة)

### الـ Config Options الجوهرية

| Option | المعنى |
|---|---|
| `CONFIG_OF` | تفعيل الـ OF support الأساسي |
| `CONFIG_OF_FLATTREE` | تفعيل قراءة الـ FDT format |
| `CONFIG_OF_EARLY_FLATTREE` | تفعيل الـ early boot FDT scanning |
| `CONFIG_OF_DYNAMIC` | دعم تعديل الشجرة runtime (overlays) |
| `CONFIG_OF_KOBJ` | تصدير الشجرة عبر sysfs/kobject |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### جدول الـ Macros والـ Flags (Cheatsheet)

| الثابت / الـ Flag | القيمة | المعنى |
|---|---|---|
| `OF_DT_HEADER` | `0xd00dfeed` | الـ magic number اللي بيميز أول كل FDT blob صحيح |
| `FDT_BEGIN_NODE` | `0x00000001` | بداية node في الـ structure block |
| `FDT_END_NODE` | `0x00000002` | نهاية node |
| `FDT_PROP` | `0x00000003` | property داخل node |
| `FDT_NOP` | `0x00000004` | no-operation (filler) |
| `FDT_END` | `0x00000009` | نهاية الـ structure block كله |
| `__initdata` | macro | البيانات اللي بتتحرر بعد boot (مثل `dt_root_addr_cells`) |

### الـ Global Variables المُعرَّفة في `of_fdt.h`

```c
extern int __initdata dt_root_addr_cells;  /* عدد cells للعنوان في root node */
extern int __initdata dt_root_size_cells;  /* عدد cells للحجم في root node */
extern void *initial_boot_params;          /* عنوان الـ FDT blob في virtual memory */
extern phys_addr_t initial_boot_params_pa; /* عنوان الـ FDT blob في physical memory */
extern char __dtb_start[];                 /* بداية DTB مدمج في kernel image */
extern char __dtb_end[];                   /* نهاية DTB مدمج في kernel image */
```

### الـ Structs الأساسية (من `of.h` وفق الـ API)

#### `struct device_node`

```c
struct device_node {
    const char *name;           /* اسم الـ node (آخر component من full_name) */
    phandle phandle;            /* معرف رقمي فريد لكل node */
    const char *full_name;      /* المسار الكامل مثل "/soc/uart@ff000000" */
    struct fwnode_handle fwnode; /* ربط بالـ firmware node abstraction layer */

    struct property *properties; /* linked list لكل الـ properties */
    struct property *deadprops;  /* properties محذوفة (للـ overlay undo) */
    struct device_node *parent;  /* الـ node الأب */
    struct device_node *child;   /* أول node ابن */
    struct device_node *sibling; /* الـ node التالي في نفس المستوى */

    unsigned long _flags;        /* OR_NODE_* flags */
    void *data;                  /* بيانات خاصة بالـ arch */
};
```

#### `struct property`

```c
struct property {
    char  *name;            /* اسم الـ property مثل "compatible" أو "reg" */
    int    length;          /* حجم القيمة بالـ bytes */
    void  *value;           /* مؤشر للقيمة (big-endian) */
    struct property *next;  /* الـ property التالية في نفس الـ node */
};
```

#### FDT Header (libfdt)

```c
/* من scripts/dtc/libfdt/fdt.h — البنية الحقيقية للـ blob */
struct fdt_header {
    fdt32_t magic;              /* 0xd00dfeed */
    fdt32_t totalsize;          /* الحجم الكلي للـ blob */
    fdt32_t off_dt_struct;      /* offset لـ structure block */
    fdt32_t off_dt_strings;     /* offset لـ strings block */
    fdt32_t off_mem_rsvmap;     /* offset لـ memory reservation map */
    fdt32_t version;            /* إصدار الـ format (عادةً 17) */
    fdt32_t last_comp_version;  /* أقل version متوافق معاه */
    fdt32_t boot_cpuid_phys;    /* ID المعالج اللي بيعمل boot */
    fdt32_t size_dt_strings;    /* حجم الـ strings block */
    fdt32_t size_dt_struct;     /* حجم الـ structure block */
};
```

### مخطط العلاقات بين الـ Structs

```
initial_boot_params (void*)
       │
       ▼
+--------------------+
|   fdt_header       |  (flat binary blob in RAM)
|   +-magic          |
|   +-totalsize      |
|   +-off_dt_struct ─┼──► [FDT_BEGIN_NODE]["/"...]
|   +-off_dt_strings─┼──► ["compatible\0reg\0..."]
|   +-off_mem_rsvmap─┼──► [reserved ranges]
+--------------------+
       │
       │ unflatten_device_tree()
       ▼
+--------------------+        +--------------------+
|  device_node (/)   |───────►|  device_node (soc) |
|  +-full_name="/"   |  child |  +-full_name="/soc" |
|  +-properties ─────┼──┐     |  +-properties ──────┼──► property("compatible")
|  +-child ──────────┼──┘     |  +-child ────────────┼──► device_node (uart)
|  +-sibling=NULL    |        |  +-parent ───────────┼──► device_node (/)
+--------------------+        +--------------------+
```

### دورة حياة الـ FDT

```
[1. Boot] ─────────────────────────────────────────────────────────────
  Bootloader يكتب عنوان DTB في register (ARM: r2, ARM64: x0)
  setup_arch() يحفظ العنوان في initial_boot_params

[2. Early Scan] ────────────────────────────────────────────────────────
  early_init_devtree(initial_boot_params)
    ├── early_init_dt_verify()        ← يتحقق من magic number
    ├── early_init_dt_scan_root()     ← يقرأ #address-cells و #size-cells
    ├── early_init_dt_scan_chosen()   ← يجيب kernel cmdline
    └── early_init_dt_scan_memory()   ← يسجل مناطق الـ RAM

[3. Memory Setup] ──────────────────────────────────────────────────────
  الـ memblock allocator يشتغل بناءً على بيانات الـ memory scan

[4. Unflatten] ─────────────────────────────────────────────────────────
  unflatten_device_tree()
    ├── يحسب حجم الشجرة المطلوب
    ├── يخصص ميموري من الـ memblock
    └── يبني شجرة device_node

[5. Driver Probe] ──────────────────────────────────────────────────────
  of_platform_populate() ← ينشئ platform_device من كل node
  Drivers يستخدموا of.h API للوصول للـ properties

[6. Teardown] ──────────────────────────────────────────────────────────
  الـ .init sections تتحرر (بما فيها البيانات __initdata)
  initial_boot_params يمكن تحريره لو مش مدمج في الـ kernel
```

### استراتيجية الـ Locking

في مرحلة الـ early boot: لا يوجد locking — الكود يشتغل على CPU واحد قبل ما يبدأ الـ SMP.

بعد الـ unflatten: الـ OF tree محمية بـ `of_node_lock` (rwlock) معرفة في `drivers/of/base.c`.

```c
/* من drivers/of/base.c */
DEFINE_RAW_SPINLOCK(devtree_lock); /* يحمي التعديل على شجرة الـ device_node */
```

الـ `initial_boot_params` نفسه: read-only بعد الـ unflatten، مفيش حاجة تعدل فيه.

### جدول الـ Config Options وتأثيرها على الـ API

| Config | يُفعّل |
|---|---|
| `CONFIG_OF_FLATTREE` | `of_fdt_unflatten_tree()`, `of_flat_dt_translate_address()`, `of_fdt_limit_memory()` |
| `CONFIG_OF_EARLY_FLATTREE` | كل دوال `early_init_*` و`of_scan_flat_dt_*` و`unflatten_device_tree()` |
| لو الـ configs مش فاعلة | الدوال بتتحول لـ `static inline` فارغة |
## Phase 4: شرح الـ Functions

### جدول الـ API (Cheatsheet)

| الدالة | الـ Config | الغرض |
|---|---|---|
| `of_fdt_unflatten_tree()` | `OF_FLATTREE` | تحويل blob لشجرة device_node |
| `of_flat_dt_translate_address()` | `OF_FLATTREE` | ترجمة عنوان من الـ DT |
| `of_fdt_limit_memory()` | `OF_FLATTREE` | تحديد عدد مناطق الـ memory |
| `of_scan_flat_dt()` | `OF_EARLY_FLATTREE` | scan لكل nodes في الـ blob |
| `of_scan_flat_dt_subnodes()` | `OF_EARLY_FLATTREE` | scan لـ subnodes فقط |
| `of_get_flat_dt_subnode_by_name()` | `OF_EARLY_FLATTREE` | إيجاد subnode باسمه |
| `of_get_flat_dt_prop()` | `OF_EARLY_FLATTREE` | جلب قيمة property من blob |
| `of_flat_dt_get_addr_size_prop()` | `OF_EARLY_FLATTREE` | جلب عناوين وأحجام من property |
| `of_flat_dt_get_addr_size()` | `OF_EARLY_FLATTREE` | جلب أول عنوان وحجم من property |
| `of_flat_dt_read_addr_size()` | `OF_EARLY_FLATTREE` | قراءة entry محدد من property |
| `of_flat_dt_is_compatible()` | `OF_EARLY_FLATTREE` | فحص compatible string لـ node |
| `of_get_flat_dt_root()` | `OF_EARLY_FLATTREE` | جلب الـ root node offset |
| `of_get_flat_dt_phandle()` | `OF_EARLY_FLATTREE` | جلب phandle من node |
| `early_init_dt_scan_chosen()` | `OF_EARLY_FLATTREE` | قراءة cmdline من /chosen |
| `early_init_dt_scan_memory()` | `OF_EARLY_FLATTREE` | تسجيل مناطق الـ RAM |
| `early_init_dt_check_for_usable_mem_range()` | `OF_EARLY_FLATTREE` | فحص linux,usable-memory |
| `early_init_dt_scan_chosen_stdout()` | `OF_EARLY_FLATTREE` | إيجاد الـ early console |
| `early_init_fdt_scan_reserved_mem()` | `OF_EARLY_FLATTREE` | حجز مناطق الـ reserved-memory |
| `early_init_fdt_reserve_self()` | `OF_EARLY_FLATTREE` | حجز الـ DTB blob نفسه في الـ RAM |
| `early_init_dt_add_memory_arch()` | `OF_EARLY_FLATTREE` | تمرير عنوان RAM للـ arch |
| `dt_mem_next_cell()` | `OF_EARLY_FLATTREE` | قراءة cell من property بأمان |
| `early_init_dt_scan_root()` | `OF_EARLY_FLATTREE` | قراءة #address-cells من root |
| `early_init_dt_scan()` | `OF_EARLY_FLATTREE` | scan كامل سريع للـ blob |
| `early_init_dt_verify()` | `OF_EARLY_FLATTREE` | تحقق من magic number وصحة الـ blob |
| `early_init_dt_scan_nodes()` | `OF_EARLY_FLATTREE` | تشغيل كل الـ early scan hooks |
| `of_flat_dt_get_machine_name()` | `OF_EARLY_FLATTREE` | جلب اسم الـ machine |
| `of_flat_dt_match_machine()` | `OF_EARLY_FLATTREE` | مطابقة الـ machine مع compatible |
| `unflatten_device_tree()` | `OF_EARLY_FLATTREE` | تحويل الـ DTB الرسمي لشجرة |
| `unflatten_and_copy_device_tree()` | `OF_EARLY_FLATTREE` | نسخ الـ DTB ثم unflatten |
| `early_init_devtree()` | `OF_EARLY_FLATTREE` | نقطة الدخول الرئيسية للـ early DT |
| `early_get_first_memblock_info()` | `OF_EARLY_FLATTREE` | جلب أول memory region |

---

### تفاصيل الدوال مجمعة بحسب المجموعة الوظيفية

---

#### المجموعة 1: التحقق والـ Early Entry

##### `early_init_dt_verify()`

```c
extern bool early_init_dt_verify(void *dt_virt, phys_addr_t dt_phys);
```

بتتحقق من إن الـ blob الموجود في العنوان `dt_virt` هو فعلاً FDT صحيح عن طريق فحص الـ magic number `0xd00dfeed`. لو الـ verification نجح، بتحفظ العناوين في `initial_boot_params` و`initial_boot_params_pa`. بتُستدعى في أول مراحل boot قبل أي حاجة تانية.

- **Parameters**: `dt_virt` — العنوان الـ virtual للـ blob، `dt_phys` — العنوان الـ physical
- **Return**: `true` لو الـ blob صحيح، `false` لو الـ magic غلط أو الـ pointer NULL
- **Caller**: `setup_arch()` في كل architecture تدعم الـ DT

##### `early_init_dt_scan()`

```c
extern bool early_init_dt_scan(void *dt_virt, phys_addr_t dt_phys);
```

بتجمع `early_init_dt_verify()` و`early_init_dt_scan_nodes()` في خطوة واحدة. لو الـ verification نجح، بتشغل فورًا الـ scan. ده الـ entry point الأشيع اللي بتستخدمه الـ architectures.

- **Return**: `true` لو نجح كل حاجة، `false` لو الـ blob invalid

---

#### المجموعة 2: الـ Flat Blob Scanning

##### `of_scan_flat_dt()`

```c
extern int of_scan_flat_dt(
    int (*it)(unsigned long node, const char *uname, int depth, void *data),
    void *data
);
```

بتمشي على كل nodes في الـ FDT blob واحدة واحدة وبتستدعي الـ callback `it` لكل node. الـ `node` هو offset في الـ blob (مش pointer لـ `device_node`)، والـ `uname` هو اسم الـ node. لو الـ callback رجع قيمة غير صفر، الـ scan بيوقف وبترجع القيمة دي.

- **Parameters**:
  - `it` — callback function بتستقبل الـ node offset والاسم والعمق والـ data
  - `data` — بيانات مخصصة بتتبعتها للـ callback
- **Return**: 0 لو خلص، أو قيمة الـ callback لو وقف مبكرًا
- **Use case**: `early_init_dt_scan_memory()` بتستخدمها عشان تلاقي الـ memory nodes

**Pseudocode**:
```
for each node in FDT structure block:
    name = fdt_get_name(node)
    depth = calculate_depth(node)
    result = it(node, name, depth, data)
    if result != 0: return result
return 0
```

##### `of_scan_flat_dt_subnodes()`

```c
extern int of_scan_flat_dt_subnodes(
    unsigned long node,
    int (*it)(unsigned long node, const char *uname, void *data),
    void *data
);
```

زي `of_scan_flat_dt()` بس بتتحدد في أبناء node معين بس. مفيدة لما تعرف الـ parent node وتعوز تتصفح أبناءه فقط (مثلاً أبناء `/memory`).

##### `of_get_flat_dt_prop()`

```c
extern const void *of_get_flat_dt_prop(
    unsigned long node,
    const char *name,
    int *size
);
```

بتجيب قيمة property معينة من node في الـ flat blob. بترجع pointer مباشر لقيمة الـ property جوه الـ blob (big-endian، محتاج تستخدم `be32_to_cpu()` لقراءة الأرقام). لو الـ property مش موجودة بترجع `NULL`.

- **Return**: pointer للقيمة أو NULL، وبتملا `*size` بحجم القيمة بالـ bytes
- **Locking**: لا يوجد — early boot context

---

#### المجموعة 3: قراءة العناوين

##### `of_flat_dt_get_addr_size_prop()`

```c
extern const __be32 *of_flat_dt_get_addr_size_prop(
    unsigned long node,
    const char *name,
    int *entries
);
```

بتجيب property بتحتوي على قائمة من (address, size) pairs مع حساب عدد الـ cells الصح بناءً على `#address-cells` و`#size-cells`. بتملا `*entries` بعدد الـ (addr,size) pairs.

##### `of_flat_dt_get_addr_size()`

```c
extern bool of_flat_dt_get_addr_size(
    unsigned long node,
    const char *name,
    u64 *addr,
    u64 *size
);
```

بتجيب أول (address, size) pair من property. مفيدة لما تعرف في entry واحدة بس مثل `reg` في غالبية الـ devices.

##### `of_flat_dt_read_addr_size()`

```c
extern void of_flat_dt_read_addr_size(
    const __be32 *prop,
    int entry_index,
    u64 *addr,
    u64 *size
);
```

بتقرأ entry محدد (بالـ index) من property array. بتستخدم `dt_root_addr_cells` و`dt_root_size_cells` لتحديد عدد الـ cells في كل entry.

---

#### المجموعة 4: الـ Early Boot Scan Hooks

##### `early_init_dt_scan_chosen()`

```c
extern int early_init_dt_scan_chosen(char *cmdline);
```

بتدور على الـ `/chosen` node في الـ FDT وتجيب منه `bootargs` (الـ kernel command line) وتحطها في `cmdline`. ده من أهم الـ early scan functions لأن الـ cmdline محتاج يتحدد قبل أي حاجة تانية.

- **Side effects**: بتملا buffer الـ cmdline وبتقدر تُعدّل `initrd_start`/`initrd_end`

##### `early_init_dt_scan_memory()`

```c
extern int early_init_dt_scan_memory(void);
```

بتدور على كل الـ nodes اللي type بتاعها `memory` في الـ FDT وبتسجل كل range في الـ `memblock` allocator. ده اللي بيعرّف للـ kernel فين الـ RAM الفعلي.

- **Caller**: `early_init_devtree()` مبكرًا في الـ boot
- **Side effects**: يضيف memory ranges في `memblock.memory`

##### `early_init_fdt_scan_reserved_mem()`

```c
extern void early_init_fdt_scan_reserved_mem(void);
```

بتقرأ الـ `/reserved-memory` node وبتحجز الـ ranges المذكورة فيه من الـ allocator. ده بيضمن إن الـ kernel محيسبش الـ regions دي كـ available RAM (مثلاً منطقة الـ GPU firmware أو الـ TEE).

##### `early_init_fdt_reserve_self()`

```c
extern void early_init_fdt_reserve_self(void);
```

بتحجز المنطقة اللي فيها الـ DTB blob نفسه في الـ memblock عشان الـ kernel ميكتبش عليه أثناء الـ boot. لازم تتستدعي قبل أي allocation.

---

#### المجموعة 5: الـ Unflatten

##### `unflatten_device_tree()`

```c
extern void unflatten_device_tree(void);
```

الدالة الأساسية اللي بتحول `initial_boot_params` (الـ blob) لشجرة `device_node` كاملة. بتعمل allocation من الـ memblock أو الـ slab حسب المرحلة. النتيجة هي الـ `of_root` الـ global اللي بتستخدمه كل الـ drivers بعد كده.

- **Called by**: `setup_arch()` في كل الـ architectures
- **Side effects**: يملا الـ `of_root` global pointer

##### `unflatten_and_copy_device_tree()`

```c
extern void unflatten_and_copy_device_tree(void);
```

بتعمل نسخة من الـ blob الأصلي في ميموري جديدة ثم بتعمل unflatten. بتُستخدم لما الـ blob الأصلي في منطقة ميموري هتتحرر (مثل الـ initrd).

##### `of_fdt_unflatten_tree()`

```c
extern void *of_fdt_unflatten_tree(
    const unsigned long *blob,
    struct device_node *dad,
    struct device_node **mynodes
);
```

الدالة العامة للـ unflatten — بتسمح بتحويل أي blob (مش بالضرورة الـ blob الرئيسي). مفيدة للـ overlays والـ testing.

- **Parameters**:
  - `blob` — pointer للـ FDT blob
  - `dad` — الـ parent node (NULL للـ root)
  - `mynodes` — الـ output: pointer لأول node في الشجرة الجديدة
- **Return**: pointer لـ buffer الذاكرة المستخدمة

##### `early_init_devtree()`

```c
extern void early_init_devtree(void *);
```

الـ entry point الرئيسي للـ early DT setup — بتجمع verify، scan الـ root، scan الـ chosen، scan الـ memory، وcall الـ arch-specific hooks في ترتيب صحيح.

---

#### المجموعة 6: الـ Machine Matching

##### `of_flat_dt_get_machine_name()`

```c
extern const char *of_flat_dt_get_machine_name(void);
```

بتجيب قيمة أول `compatible` string من الـ root node. بتُستخدم في الـ early boot لطباعة اسم الـ machine في الـ dmesg.

##### `of_flat_dt_match_machine()`

```c
extern const void *of_flat_dt_match_machine(
    const void *default_match,
    const void *(*get_next_compat)(const char * const **)
);
```

بتمشي على قائمة الـ machine descriptors (كل منها بيحتوي على compatible strings) وبتدور على أفضل match مع الـ root node في الـ FDT. بترجع الـ descriptor الأفضل أو `default_match` لو ملقتش. بتُستخدم في الـ ARM machine selection.
## Phase 5: دليل الـ Debugging الشامل

### الـ Kernel Config Options للـ Debugging

| Config | الغرض |
|---|---|
| `CONFIG_OF` | الأساس — لازم يكون enabled |
| `CONFIG_OF_FLATTREE` | دعم قراءة الـ FDT |
| `CONFIG_OF_EARLY_FLATTREE` | الـ early scan functions |
| `CONFIG_OF_DYNAMIC` | دعم تعديل الشجرة runtime (overlays) |
| `CONFIG_OF_UNITTEST` | تشغيل الـ OF unit tests |
| `CONFIG_OF_OVERLAY` | دعم الـ DT overlays |
| `CONFIG_OF_KOBJ` | تصدير الشجرة عبر sysfs |
| `CONFIG_EARLY_PRINTK` | مهم لطباعة رسائل قبل الـ console |
| `CONFIG_DEBUG_OF` | debug checks إضافية في الـ OF layer |

---

### الـ sysfs والـ debugfs

#### تصدير شجرة الـ Device Tree

```bash
# مشاهدة الشجرة كاملة
ls /sys/firmware/devicetree/base/

# قراءة property معينة (مثال: compatible للـ root)
hexdump -C /sys/firmware/devicetree/base/compatible

# عرض كل properties لـ node معين
ls -la /sys/firmware/devicetree/base/soc/

# قراءة reg property
hexdump -C /sys/firmware/devicetree/base/soc/uart@ff000000/reg
```

#### مشاهدة الشجرة كاملة بصيغة مقروءة

```bash
# تحتاج dtc مثبت
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | less

# أو باستخدام fdtdump
fdtdump /sys/firmware/fdt 2>/dev/null | head -100
```

#### ملف `/sys/firmware/fdt`

```bash
# هذا الملف هو الـ DTB blob الأصلي — يمكن حفظه وفحصه
cp /sys/firmware/fdt /tmp/current.dtb
dtc -I dtb -O dts /tmp/current.dtb > /tmp/current.dts
cat /tmp/current.dts | grep -A5 "memory@"
```

---

### الـ ftrace والـ Tracepoints

الـ OF/FDT subsystem لا يملك tracepoints مخصصة له، لكن يمكن الاستفادة من:

```bash
# تتبع الدوال المرتبطة بالـ OF
echo 'of_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغل الأمر المطلوب أو انتظر boot
cat /sys/kernel/debug/tracing/trace | head -50
echo 0 > /sys/kernel/debug/tracing/tracing_on

# تتبع unflatten تحديداً (بعد boot)
echo 'unflatten_device_tree' > /sys/kernel/debug/tracing/set_ftrace_filter
```

---

### الـ Dynamic Debug وـ printk

```bash
# تفعيل debug messages للـ OF subsystem
echo 'module of_fdt +p' > /sys/kernel/debug/dynamic_debug/control

# أو بالـ file
echo 'file drivers/of/fdt.c +p' > /sys/kernel/debug/dynamic_debug/control

# مشاهدة الـ OF messages في الـ dmesg
dmesg | grep -i "device tree\|of_fdt\|DTB\|fdt\|chosen\|memory@"

# الرسائل المبكرة (early boot)
dmesg | head -50 | grep -i "machine\|memory\|dt"
```

---

### طباعة معلومات الـ Boot

```bash
# اسم الـ machine (من root compatible)
dmesg | grep "Machine model"
# مثال output:
# [    0.000000] Machine model: Rockchip RK3562 EVB

# فحص الـ cmdline اللي جا من الـ DT
cat /proc/cmdline

# مناطق الـ Memory المسجلة من الـ DT
dmesg | grep -E "memory:|MEMBLOCK|earlycon"

# الـ reserved memory
dmesg | grep -i "reserved"
cat /proc/iomem | grep "reserved"
```

---

### جدول: أشيع رسائل الـ Error

| رسالة الـ Error | المعنى | الحل |
|---|---|---|
| `FDT: No valid device tree found` | الـ magic number غلط أو العنوان خاطئ | تأكد إن الـ bootloader بيبعت عنوان الـ DTB صح |
| `OF: fdt: not compatible with dt version` | إصدار الـ FDT غير مدعوم | أعد بناء الـ DTB بإصدار حديث من الـ dtc |
| `OF: fdt: fdt_check_header() failed` | الـ blob فيه corruption | افحص نقل الـ DTB في الـ bootloader |
| `Warning: /chosen has no 'bootargs' property` | الـ /chosen node ناقص الـ bootargs | أضف `bootargs` في الـ DTS أو عبر الـ bootloader |
| `No memory defined in device tree` | مفيش `memory` node في الـ DT | أضف `memory@...` node بالـ reg property |
| `earlycon: no matching earlycon found` | الـ stdout-path في /chosen مش صحيح | تأكد من اسم الـ serial node والـ alias |
| `unflatten_device_tree: No device tree found` | `initial_boot_params` = NULL | مشكلة في الـ arch setup أو الـ bootloader |
| `of_fdt: allocated X bytes` | رسالة info عن حجم الشجرة | مش error، بس مفيد لمعرفة الحجم |
| `reserved-memory: failed to reserve memory` | تعارض في الـ memory map | افحص الـ DT reserved-memory مع الـ physical layout |

---

### نقاط الـ Debugging الاستراتيجية

#### في `drivers/of/fdt.c` — أماكن مقترحة لـ WARN_ON

```c
/* الموقع 1: بعد early_init_dt_verify — لو فشل */
if (!early_init_dt_verify(params, phys)) {
    WARN(1, "FDT verification failed at %pa\n", &phys);
}

/* الموقع 2: في of_fdt_unflatten_tree — لو الـ blob NULL */
WARN_ON(!blob);

/* الموقع 3: بعد unflatten — لو الـ of_root NULL */
BUG_ON(!of_root);
```

---

### فحص الـ Hardware — التحقق من الـ DTB

#### عبر الـ U-Boot

```bash
# في U-Boot prompt — فحص الـ DTB
fdt addr $fdt_addr
fdt print /
fdt print /chosen
fdt print /memory

# تحقق من الـ magic
md.b $fdt_addr 4
# لازم يطبع: d0 0d fe ed
```

#### استخراج الـ DTB من كيرنل مضغوط

```bash
# من bzImage / Image
binwalk -e Image --run-as=root
# أو
extract-dtb Image -o dtb_out/

# فحص dtb مستخرج
dtc -I dtb -O dts dtb_out/dtb-1.dtb | head -30
```

#### مقارنة DTB وقت التشغيل بما هو مكتوب في الـ DTS

```bash
dtc -I fs /sys/firmware/devicetree/base -O dts -o /tmp/runtime.dts 2>/dev/null
diff original.dts /tmp/runtime.dts
```

---

### Shell Commands جاهزة للنسخ

```bash
# 1. فحص magic number للـ DTB
hexdump -C /sys/firmware/fdt | head -1
# expected: 00000000  d0 0d fe ed ...

# 2. إيجاد كل compatible strings في الشجرة
find /sys/firmware/devicetree/base -name compatible -exec sh -c \
  'echo "=== {} ===" && strings {}' \;

# 3. فحص الـ memory nodes
find /sys/firmware/devicetree/base -name "memory*" -type d | while read d; do
  echo "Node: $d"
  hexdump -C "$d/reg" 2>/dev/null
done

# 4. قراءة اسم الـ machine
strings /sys/firmware/devicetree/base/compatible | head -3

# 5. فحص الـ reserved-memory
ls /sys/firmware/devicetree/base/reserved-memory/ 2>/dev/null || echo "No reserved-memory node"

# 6. طباعة حجم الـ DTB
wc -c /sys/firmware/fdt
stat -c %s /sys/firmware/fdt

# 7. فحص stdout-path في chosen
cat /sys/firmware/devicetree/base/chosen/stdout-path 2>/dev/null || echo "No stdout-path"

# 8. مشاهدة كل رسائل الـ OF في الـ early boot
dmesg | grep -E "^\[[ ]*0\." | grep -iE "dt|fdt|of:|tree|memory|machine|chosen"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: RK3562 — Industrial Gateway — مشكلة الـ Memory Map

**العنوان**: الـ kernel بيتعطل عند بداية الـ boot على gateway صناعي مبني على RK3562

**السياق**:
فريق engineering بيصمم industrial gateway بـ RK3562 مع 2GB RAM. الـ bootloader U-Boot بيحمل الـ kernel لكن النظام بيـ hang تماماً قبل ما يطبع أي حاجة على الـ serial.

**المشكلة**:
الـ DTB بيحدد منطقة RAM واحدة كبيرة، لكن الـ hardware الفعلي عنده "hole" في الـ memory map بسبب ترتيب الـ SoC interleaving.

**التحليل**:
```
1. early_init_dt_scan() ← يُنفَّذ بنجاح (magic صح)
2. early_init_dt_scan_memory() ← يسجل 0x00000000-0x80000000 (2GB)
3. memblock يضيف المنطقة كاملة
4. الـ kernel يحاول يقرأ من الـ "hole" → page fault مبكر → hang
```

المشكلة في الـ DTS:
```dts
/* خاطئ */
memory@0 {
    device_type = "memory";
    reg = <0x0 0x00000000 0x0 0x80000000>; /* 2GB متواصلة */
};
```

**الحل**:
```dts
/* صحيح — بنفصل المناطق الفعلية */
memory@0 {
    device_type = "memory";
    reg = <0x0 0x00000000 0x0 0x40000000>,  /* 1GB أول */
          <0x0 0x50000000 0x0 0x30000000>;  /* 768MB تانية بعد الـ hole */
};
```

`early_init_dt_scan_memory()` بتقرأ كل entry وبتستدعي `early_init_dt_add_memory_arch()` لكل range على حدة — بكده الـ hole بيتجاهل تلقائياً.

**الدرس المستفاد**: دايمًا اطابق الـ DT memory layout بالـ technical reference manual للـ SoC. استخدم `dmesg | grep MEMBLOCK` للتحقق من المناطق المسجلة.

---

### السيناريو 2: STM32MP1 — Android TV Box — مشكلة الـ Earlycon

**العنوان**: مفيش أي output على الـ serial حتى بعد ما الـ kernel يبدأ

**السياق**:
فريق يعمل على Android TV box بـ STM32MP1. الـ U-Boot شغال وبيطبع، لكن لما الـ kernel يبدأ مفيش output خالص — لا debug ولا حاجة.

**المشكلة**:
الـ `/chosen/stdout-path` في الـ DTS غلط أو مش موجود.

**التحليل**:
```
early_init_dt_scan_chosen()
  ├── يدور على /chosen node
  ├── يقرأ bootargs → تمام
  └── يستدعي early_init_dt_scan_chosen_stdout()
        └── يدور على stdout-path
            └── لو مش موجود → early console مش بيشتغل
```

```dts
/* خاطئ — اسم الـ alias غلط */
chosen {
    stdout-path = "serial1:115200n8";  /* لكن الـ alias معرفه serial0 */
};

aliases {
    serial0 = &usart3;  /* الـ UART الفعلي */
};
```

**الحل**:
```dts
chosen {
    stdout-path = "serial0:115200n8";  /* صحيح */
};
```

وفي الـ cmdline:
```
earlycon=stm32_usart,0x40011000,115200
```

**أمر تشخيص**:
```bash
# في U-Boot
fdt addr $fdt_addr
fdt print /chosen
fdt print /aliases
```

**الدرس المستفاد**: `early_init_dt_scan_chosen_stdout()` بتعتمد على اسم الـ alias المعرف في `/aliases` — لازم يتطابق مع `stdout-path`.

---

### السيناريو 3: i.MX8MQ — IoT Sensor Hub — Reserved Memory Conflict

**العنوان**: الـ GPU firmware بيتخرب بعد بدء الـ kernel على i.MX8MQ

**السياق**:
device IoT بيستخدم i.MX8MQ مع GPU. الـ GPU driver بيشتغل لكن بعد فترة قصيرة الـ firmware بيتلاشى وبيعطي errors.

**المشكلة**:
منطقة GPU firmware في الـ RAM مش محجوزة في الـ DT، فالـ kernel بيكتب عليها.

**التحليل**:
```
early_init_fdt_scan_reserved_mem()
  ├── بتدور على /reserved-memory node
  ├── لو مفيش /reserved-memory → مفيش حجز
  └── الـ memblock allocator بيعتبر كل الـ RAM متاح
      → الـ kernel بيخصص pages فوق منطقة الـ GPU firmware
```

```dts
/* ناقص */
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;
    /* GPU firmware مش محجوز! */
};
```

**الحل**:
```dts
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    gpu_reserved: gpu@7c000000 {
        reg = <0 0x7c000000 0 0x4000000>; /* 64MB للـ GPU */
        no-map;
    };
};
```

الـ `no-map` flag بيخلي `early_init_fdt_scan_reserved_mem()` تحجز المنطقة دي تماماً وتمنع أي استخدام منها.

**تشخيص**:
```bash
dmesg | grep -i "reserved\|cma"
cat /proc/iomem | grep -i "reserved"
```

**الدرس المستفاد**: كل firmware أو carveout منطقة لازم تكون في `/reserved-memory` مع `no-map` عشان `early_init_fdt_scan_reserved_mem()` تحجزها قبل أي allocation.

---

### السيناريو 4: AM62x — Automotive ECU — Board Bring-up / Compatible Mismatch

**العنوان**: الـ kernel بيـ boot لكن الـ drivers مبتشتغلش على ECU مبني على AM62x

**السياق**:
مهندس بيعمل bring-up لـ automotive ECU مبني على Texas Instruments AM62x. الـ kernel بيـ boot لكن معظم الـ SoC peripherals (I2C، SPI، UART) مبتشتغلش.

**المشكلة**:
الـ `compatible` string في الـ root node بيشاور على board قديم مختلف — فالـ machine-specific code مبيتشغلش.

**التحليل**:
```
of_flat_dt_match_machine()
  ├── تمشي على قائمة machine descriptors
  ├── لكل descriptor، بتفحص compatible strings
  ├── compatible = "ti,am625-sk" ← موجود في الـ descriptor list
  └── لو الـ DTS بيقول "ti,am335x-evm" ← مطابقة غلط
      → machine descriptor خاطئ يتحدد
      → board init code خاطئ يشتغل
```

```dts
/* خاطئ — نسخ من board قديم */
/ {
    compatible = "ti,am335x-evm", "ti,am335x"; /* ده board تاني خالص */
    model = "Custom AM62x ECU";
};
```

**الحل**:
```dts
/ {
    compatible = "custom,am62x-ecu", "ti,am625"; /* أول: board-specific، تاني: SoC */
    model = "Custom AM62x Automotive ECU";
};
```

`of_flat_dt_match_machine()` بتختار أفضل match — ولو الـ board-specific compatible مش موجود، بترجع لـ SoC-level match.

**تشخيص**:
```bash
dmesg | grep "Machine model"
cat /sys/firmware/devicetree/base/compatible | strings
```

**الدرس المستفاد**: الـ `compatible` في الـ root node لازم يكون من العام للخاص: أول entry هو الـ board-specific، وآخر entry هو الـ SoC family.

---

### السيناريو 5: Allwinner H616 — Custom Board — DTB Self-Overwrite

**العنوان**: بيانات الـ DTB بتتخرب أثناء الـ boot على board مخصص بـ Allwinner H616

**السياق**:
مهندس بيعمل custom board بـ Allwinner H616 (من اللي بيستخدمها Orange Pi). البورد بيـ boot أحياناً تمام، وأحياناً بيتعطل بعشوائية.

**المشكلة**:
الـ bootloader بيحمل الـ DTB في منطقة RAM، لكن الـ kernel نفسه بيحمل في منطقة أعلى بتـ overlap مع الـ DTB.

**التحليل**:
```
[Bootloader]
  DTB loaded at:    0x4FA00000 (size: 50KB)
  Kernel loaded at: 0x4F800000 (size: 20MB)
  ← Kernel tail يكتب فوق الـ DTB!

early_init_fdt_reserve_self()
  ← المفروض تحجز منطقة الـ DTB
  ← لكن لو الـ overlap حصل قبل ما early_init بيشتغل → خراب
```

الحل الصحيح على مستوى الـ bootloader:
```bash
# في U-Boot script
setenv fdt_addr 0x5FA00000    # حط الـ DTB بعيد عن الـ kernel
setenv kernel_addr 0x40080000
load ${devtype} ${devnum} ${kernel_addr} ${bootfile}
load ${devtype} ${devnum} ${fdt_addr} ${fdtfile}
booti ${kernel_addr} - ${fdt_addr}
```

**تشخيص**:
```bash
# في U-Boot
md.b $fdt_addr 4    # لازم: d0 0d fe ed
# لو القيم مختلفة → الـ DTB اتخرب قبل الـ kernel start

# فحص الـ overlap
echo "Kernel end:" $((0x40080000 + 0x1400000))  # kernel size ~20MB
echo "FDT addr:" $fdt_addr
```

**الدرس المستفاد**: `early_init_fdt_reserve_self()` بتحمي الـ DTB بعد ما الـ kernel يبدأ، لكن الـ overlap في الـ bootloader نفسه يحصل قبل كده. حل الـ memory layout لازم يكون في الـ bootloader configuration.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| العنوان | الرابط |
|---|---|
| ELCE: Grant Likely on device trees (2010) — المحاضرة التاريخية اللي شرحت ليه ARM محتاج DT | [lwn.net/Articles/414016/](https://lwn.net/Articles/414016/) |
| Add basic ARM device tree support — الـ patch الأولى لـ DT support في ARM | [lwn.net/Articles/375296/](https://lwn.net/Articles/375296/) |
| Device tree overlays — شرح آلية الـ overlays الديناميكية | [lwn.net/Articles/616859/](https://lwn.net/Articles/616859/) |
| libfdt: Add support for device tree overlays | [lwn.net/Articles/702638/](https://lwn.net/Articles/702638/) |
| pstore/ram: Add ramoops support for FDT | [lwn.net/Articles/501748/](https://lwn.net/Articles/501748/) |
| Linux and the Devicetree — الـ usage model الرسمي | [static.lwn.net/kerneldoc/devicetree/usage-model.html](https://static.lwn.net/kerneldoc/devicetree/usage-model.html) |
| Infrastructure for PCIe with FDT | [lwn.net/Articles/886943/](https://lwn.net/Articles/886943/) |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة جوه الـ kernel source tree:

```
Documentation/devicetree/
├── usage-model.rst          ← نموذج الاستخدام وشرح المبدأ
├── bindings/                ← توثيق كل binding لكل device
│   ├── memory/              ← memory nodes
│   └── chosen.yaml          ← /chosen node bindings
├── of_unittest.rst          ← كيف تشغل الـ unit tests
└── dynamic-resolution-notes.rst

Documentation/ABI/testing/sysfs-firmware-ofw
← يوثق /sys/firmware/devicetree/
```

**أمر للوصول**:
```bash
# في kernel source tree
less Documentation/devicetree/usage-model.rst
find Documentation/devicetree/bindings -name "*.yaml" | head -10
```

---

### eLinux.org

| الصفحة | الرابط | المحتوى |
|---|---|---|
| Device Tree Reference | [elinux.org/Device_Tree_Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل للـ DT syntax |
| Device Tree Mysteries | [elinux.org/Device_Tree_Mysteries](https://elinux.org/Device_Tree_Mysteries) | حلول لأشيع المشاكل |
| Linux Drivers Device Tree Guide | [elinux.org/Linux_Drivers_Device_Tree_Guide](https://elinux.org/Linux_Drivers_Device_Tree_Guide) | دليل كتابة drivers تستخدم DT |
| Device Tree History | [elinux.org/Device_tree_history](https://elinux.org/Device_tree_history) | تاريخ تطور الـ DT |
| New FDT Format | [elinux.org/New_FDT_format](https://elinux.org/New_FDT_format) | تفاصيل الـ binary format |
| Device Tree frowand | [elinux.org/Device_Tree_frowand](https://elinux.org/Device_Tree_frowand) | مصادر من Frank Rowand |

---

### kernelnewbies.org

| الصفحة | الرابط |
|---|---|
| Linux 3.0 — أول إضافة CONFIG_OF على ARM | [kernelnewbies.org/Linux_3.0_DriverArch](https://kernelnewbies.org/Linux_3.0_DriverArch) |
| Linux 3.17 — توسيع DT support | [kernelnewbies.org/Linux_3.17-DriversArch](https://kernelnewbies.org/Linux_3.17-DriversArch) |
| Linux 6.13 — آخر تطورات DT | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) |

---

### مسارات الكود الأساسية في الـ Kernel

```
# الـ implementation
drivers/of/fdt.c              ← كل دوال of_fdt.h الـ implementation
drivers/of/base.c             ← OF core API
drivers/of/platform.c         ← of_platform_populate() وما يليها
drivers/of/of_reserved_mem.c  ← reserved-memory handling
drivers/of/overlay.c          ← DT overlay support

# الـ libfdt (ديُستخدم من الـ early boot والـ dtc)
scripts/dtc/libfdt/fdt.c
scripts/dtc/libfdt/fdt_ro.c
scripts/dtc/libfdt/fdt_rw.c

# الـ headers
include/linux/of_fdt.h        ← موضوع شرحنا
include/linux/of.h            ← OF API الكامل
include/linux/fdt.h           ← libfdt interface
```

---

### Commits تاريخية مهمة

```bash
# إيجاد commits المهمة في git log
git log --oneline -- drivers/of/fdt.c | head -20
git log --oneline --all --grep="unflatten_device_tree" | head -10
git log --oneline --all --grep="early_init_dt_scan" | head -10

# Commit أول ARM DT support
# 6ab29b8 — ARM: OF: allow device tree firmware to be used on ARM (2011)
```

---

### كتب مقترحة

| الكتاب | المؤلف | الصلة |
|---|---|---|
| **Linux Device Drivers, 3rd Ed. (LDD3)** | Corbet, Rubini, Kroah-Hartman | الفصول الأساسية لفهم الـ device model |
| **Linux Kernel Development, 3rd Ed.** | Robert Love | فصل الـ memory management وبداية الـ boot |
| **Embedded Linux Primer, 2nd Ed.** | Christopher Hallinan | فصول Device Tree بالتفصيل |
| **Mastering Embedded Linux Programming** | Chris Simmonds | فصل كامل للـ DT bring-up |
| **Professional Linux Kernel Architecture** | Wolfgang Mauerer | تفاصيل الـ memory subsystem |

---

### مصطلحات للبحث

```
"flattened device tree" linux kernel
"of_fdt" site:github.com
"unflatten_device_tree" implementation
"early_init_devtree" ARM64
"device tree blob" bootloader passing
dtc "device tree compiler" tutorial
"OF/FDT" subsystem linux kernel development
libfdt API reference
devicetree.org specification
ePAPR specification device tree
```

---

### أدوات مساعدة

```bash
# تثبيت الـ dtc على Linux
apt install device-tree-compiler       # Debian/Ubuntu
dnf install dtc                        # Fedora
# أو بناء من الـ kernel source
make scripts/dtc/dtc

# أدوات مفيدة
dtc --help                             # compiler
fdtdump file.dtb                       # dump blob
fdtget file.dtb /chosen bootargs       # قراءة property محددة
fdtput file.dtb /chosen bootargs "..."  # كتابة property
```

---

### الـ devicetree.org الرسمي

- **Specification**: [devicetree.org](https://www.devicetree.org) — المواصفة الرسمية للـ Device Tree format
- الـ ePAPR specification: تُعرّف الـ binary format المستخدم في الـ FDT
## Phase 8: Writing simple module

### الفكرة

سنستخدم **kprobe** لنعلق على دالة `of_scan_flat_dt()` — وهي دالة exported من الـ OF/FDT subsystem بتُستدعى في الـ boot لكنها ممكن تتستدعى كمان من overlays وغيرها. بنطبع معلومات الـ callback function address وبيانات الـ scan.

سبب اختيار `of_scan_flat_dt()`: هي دالة آمنة للـ kprobe، مش في الـ early boot path، وبتتستدعى أحياناً من كود غير boot (مثل الـ OF unit tests).

**ملاحظة**: الوحدة دي مناسبة للتعلم والتشخيص. في production لا تستخدم kprobes على دوال early-boot لأن الـ init sections بتتحرر.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * of_fdt_probe.c — kprobe على of_scan_flat_dt()
 *
 * يستدعي kprobe كل مرة تتستخدم of_scan_flat_dt()
 * ويطبع معلومات الـ callback وعنوان الـ data pointer
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/of_fdt.h>  /* لتعريفات الـ OF/FDT API */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Linux Kernel Learner");
MODULE_DESCRIPTION("kprobe على of_scan_flat_dt — دراسة الـ FDT scanning");

/*
 * اسم الدالة المستهدفة للـ kprobe.
 * of_scan_flat_dt() بتمشي على كل nodes في الـ FDT blob
 * وبتستدعي الـ callback لكل node — مثالية للمراقبة.
 */
static const char *target_func = "of_scan_flat_dt";

/*
 * الـ pre-handler — بيتنفذ قبل تنفيذ of_scan_flat_dt().
 * pt_regs بيحتوي على قيم الـ registers في لحظة الاستدعاء.
 * على ARM64: regs->regs[0] = أول argument (الـ callback pointer)
 *            regs->regs[1] = تاني argument (الـ data pointer)
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
#ifdef CONFIG_ARM64
    /* على ARM64، الـ arguments بتتبعت في x0 وx1 */
    unsigned long cb_addr = regs->regs[0];  /* الـ callback function */
    unsigned long data    = regs->regs[1];  /* الـ data pointer */
#elif defined(CONFIG_X86_64)
    /* على x86_64، الـ arguments بتتبعت في rdi وrsi */
    unsigned long cb_addr = regs->di;
    unsigned long data    = regs->si;
#else
    unsigned long cb_addr = 0;
    unsigned long data    = 0;
#endif

    /*
     * نطبع معلومات الاستدعاء.
     * %ps بتطبع اسم الدالة من رقمها (kernel symbol lookup).
     */
    pr_info("of_fdt_probe: of_scan_flat_dt() called!\n");
    pr_info("of_fdt_probe:   callback fn  = %pS (addr=0x%lx)\n",
            (void *)cb_addr, cb_addr);
    pr_info("of_fdt_probe:   data pointer = 0x%lx\n", data);

    /*
     * لازم نرجع 0 من الـ pre-handler عشان الـ kprobes framework
     * يكمل تنفيذ الدالة الأصلية. لو رجعنا غير صفر، الـ execution بتوقف.
     */
    return 0;
}

/*
 * الـ post-handler — بيتنفذ بعد تنفيذ of_scan_flat_dt() وبعد كل iter.
 * بنستخدمه للتأكيد إن الدالة رجعت وبنطبع الـ return value.
 * على ARM64 الـ return value في regs->regs[0] بعد الرجوع.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    pr_info("of_fdt_probe: of_scan_flat_dt() returned (post-handler)\n");
}

/*
 * تعريف الـ kprobe struct.
 * بنحدد اسم الدالة المستهدفة فقط — الـ kernel بيبحث عن عنوانها تلقائياً.
 */
static struct kprobe kp = {
    .symbol_name = NULL,  /* سيتحدد في module_init */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/*
 * module_init: بيتنفذ لما تعمل insmod.
 * بيسجل الـ kprobe على الدالة المستهدفة.
 */
static int __init of_fdt_probe_init(void)
{
    int ret;

    /* نحدد اسم الدالة المستهدفة */
    kp.symbol_name = target_func;

    /*
     * register_kprobe() بيدور على عنوان الدالة في الـ kernel symbol table
     * وبيضع breakpoint افتراضي عليها (INT3 على x86، BRK على ARM64).
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("of_fdt_probe: failed to register kprobe on %s: %d\n",
               target_func, ret);
        /*
         * الأخطاء الشائعة:
         * -EINVAL: الدالة مش موجودة أو في blacklist
         * -ENOENT: الاسم مش موجود في الـ symbol table
         * -EBUSY: kprobe تاني موجود بالفعل
         */
        return ret;
    }

    pr_info("of_fdt_probe: kprobe registered on %s at addr %p\n",
            target_func, kp.addr);
    pr_info("of_fdt_probe: module loaded — waiting for of_scan_flat_dt() calls\n");
    pr_info("of_fdt_probe: hint: trigger with 'cat /sys/firmware/devicetree/base/'\n");
    pr_info("of_fdt_probe: or load a DT overlay to trigger a scan\n");

    return 0;
}

/*
 * module_exit: بيتنفذ لما تعمل rmmod.
 * لازم نلغي تسجيل الـ kprobe عشان نرجع الكود لحالته الأصلية.
 */
static void __exit of_fdt_probe_exit(void)
{
    /*
     * unregister_kprobe() بيشيل الـ breakpoint ويرجع الكود الأصلي.
     * لو منعملش ده، الـ kernel هيـ crash عند أول استدعاء للدالة بعد الـ rmmod.
     */
    unregister_kprobe(&kp);
    pr_info("of_fdt_probe: kprobe unregistered — module exiting\n");
}

module_init(of_fdt_probe_init);
module_exit(of_fdt_probe_exit);
```

---

### ملف Makefile للـ Module

```makefile
# Makefile للـ of_fdt_probe module
obj-m += of_fdt_probe.o

# غير هذا المسار لمسار kernel source على جهازك
KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### طريقة التشغيل

```bash
# بناء الـ module
make

# تحميله
sudo insmod of_fdt_probe.ko

# مشاهدة الـ output
sudo dmesg | tail -10

# تشغيل OF scan يدوياً عن طريق OF unit tests (لو مفعلة)
# أو عبر overlay
# أو مجرد ابدأ بناء of_platform من sysfs

# إلغاء التحميل
sudo rmmod of_fdt_probe

# التحقق من النتائج
sudo dmesg | grep "of_fdt_probe"
```

**مثال output متوقع**:
```
[  142.331204] of_fdt_probe: kprobe registered on of_scan_flat_dt at addr ffffffc010a1b234
[  142.331210] of_fdt_probe: module loaded — waiting for of_scan_flat_dt() calls
[  142.331213] of_fdt_probe: hint: trigger with 'cat /sys/firmware/devicetree/base/'
[  155.002445] of_fdt_probe: of_scan_flat_dt() called!
[  155.002448] of_fdt_probe:   callback fn  = early_init_dt_scan_memory+0x0/0x... (addr=0xffffffc010a1c100)
[  155.002450] of_fdt_probe:   data pointer = 0x0
[  155.002451] of_fdt_probe: of_scan_flat_dt() returned (post-handler)
[  178.441102] of_fdt_probe: kprobe unregistered — module exiting
```

---

### شرح كل جزء بالعربي

| الجزء | السبب |
|---|---|
| `#include <linux/kprobes.h>` | بيوفر الـ API لتسجيل وإلغاء الـ kprobes |
| `#include <linux/of_fdt.h>` | بيعرّف API الـ FDT — الهدف من دراستنا |
| `handler_pre()` | بيتنفذ قبل الدالة الأصلية — هنا بنطبع معلومات الـ arguments |
| `handler_post()` | بيتنفذ بعد الدالة — للتأكيد إنها رجعت بنجاح |
| `kp.symbol_name` | الـ kernel بيحول الاسم لعنوان تلقائياً عبر `kallsyms` |
| `register_kprobe()` | بيضع breakpoint افتراضي (software) على الدالة |
| `unregister_kprobe()` | ضروري في الـ exit عشان نتجنب الـ crash بعد تحميل الـ module |
| `pr_info()` | بنستخدم `%pS` لطباعة اسم الدالة من عنوانها (kernel symbol resolution) |
| `module_init` / `module_exit` | نقطتا الدخول والخروج للـ LKM — معيار كل module في الـ kernel |
