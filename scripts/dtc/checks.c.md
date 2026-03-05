## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

الملف `checks.c` جزء من **DTC — Device Tree Compiler**، وده أداة build-time مش kernel module. بيقع تحت subsystem اسمه **"Open Firmware and Flattened Device Tree"** في ملف `MAINTAINERS`، المسؤول عنه Rob Herring وSaravana Kannan، والـ mailing list هو `devicetree@vger.kernel.org`.

---

### الصورة الكبيرة — قبل أي كود

#### تخيل الموقف ده

عندك جهاز embedded — مثلاً Raspberry Pi أو راوتر أو كاميرا — جوّاه:
- CPU
- GPIO pins
- I2C bus فيها sensor
- SPI flash
- Interrupt controller
- Clock generator

الـ Linux kernel لازم يعرف كل ده. المشكلة إن مش كل حاجة بتتعرّف أوتوماتيك زي الـ PCI devices. فمين يقول للـ kernel "الـ I2C sensor ده على address 0x48، وبياخد 2 interrupt cells"؟

الجواب هو الـ **Device Tree** — ملف نصي بـ syntax خاصة اسمها `.dts` (Device Tree Source)، بيوصف hardware الجهاز كشجرة nodes وخصائص (properties). الـ kernel مش بيقرا الـ `.dts` مباشرة، لكن بيقرا النسخة المُجمَّعة منه الاسمها **DTB — Device Tree Blob** (`.dtb`).

اللي بيحوّل `.dts` ← `.dtb` هو الـ **DTC — Device Tree Compiler**، وده tool بيتبنى جنب الـ kernel في `scripts/dtc/`.

---

#### دور `checks.c` تحديداً

التجميع مجرد ترجمة. الأهم هو **التحقق** — إنك قبل ما تعمل compile، تتأكد إن الـ Device Tree صح semantically مش بس syntactically.

`checks.c` هو المكان اللي بيعيش فيه **كل الـ validation rules** اللي الـ DTC بيطبّقها على الـ device tree قبل أو بعد الـ parsing. كل rule هو `struct check` فيه:
- اسم
- function بتنفّذ الـ check
- هل نتيجته warning ولا error
- قائمة prerequisites (checks تانية لازم تنجح الأول)

الـ checks دي بتتقسم لـ 5 أنواع رئيسية:

| النوع | أمثلة |
|-------|--------|
| **Structural** | أسماء nodes مش متكررة، أسماء properties صح، labels مش مكررة |
| **Reference fixups** | حل الـ phandle references وpath references وحذف nodes مش مستخدمة |
| **Semantic** | `#address-cells` يكون cell مش string، `compatible` يكون string list |
| **Bus-specific** | PCI bridge محتاج `#address-cells=3`، I2C عناوين أقل من 7 bits، SPI address cells=1 |
| **Graph/OF-graph** | port/endpoint nodes صح وبيشاوروا على بعض bidirectionally |

---

#### قصة المشكلة

تخيل developer كتب DTS فيه:

```dts
i2c0: i2c@40004000 {
    compatible = "vendor,i2c";
    reg = <0x40004000 0x1000>;
    #address-cells = <1>;
    #size-cells = <0>;

    sensor@200 {    /* عنوان I2C = 0x200 — ده غلط، I2C أقصاه 7 bits (0x7F) */
        compatible = "bosch,bme280";
        reg = <0x200>;
    };
};
```

من غير الـ `checks.c`، الـ DTC هيجمّعها عادي، والـ kernel هيحاول يفتح sensor على عنوان `0x200` ومش هيلاقيه. الـ `check_i2c_bus_reg` في `checks.c` هو اللي هيمسك الغلط ده وقت الـ build ويقولك:

```
Warning (i2c_bus_reg): /i2c@40004000/sensor@200: I2C address must be less than 7-bits, got "0x200"
```

---

### آلية عمل الـ Checks

```
process_checks()
    └── loop على check_table[]
            └── run_check(c, dti)
                    ├── تحقق من prerequisites أولاً (recursive)
                    └── check_nodes_props(c, dti, root_node)
                                └── يطبّق c->fn على كل node في الشجرة (DFS)
```

كل check له status: `UNCHECKED` → `PASSED`/`FAILED`/`PREREQ`.

لو check فشل وهو `error=true`، الـ DTC يوقف بـ exit code 2 ما لم تستخدم `-f` (force).

---

### الـ Prerequisite System

الـ checks مش مستقلة — فيه dependency graph. مثلاً:
- `phandle_references` يعتمد على `duplicate_node_names` و`explicit_phandles`
- `reg_format` يعتمد على `addr_size_cells`
- `i2c_bus_reg` يعتمد على `reg_format` و`i2c_bus_bridge`

الفكرة: ما تتحققش من format الـ `reg` لو الـ `#address-cells` نفسه غلط.

---

### الملفات المرتبطة

#### أساسيات الـ DTC Tool

| الملف | الدور |
|-------|-------|
| `scripts/dtc/dtc.c` | الـ main entry point للأداة |
| `scripts/dtc/dtc.h` | تعريفات `struct node`, `struct property`, `struct dt_info`, `cell_t` وكل الـ data structures |
| `scripts/dtc/checks.c` | **الملف اللي بندرسه** — كل validation rules |
| `scripts/dtc/dtc-lexer.l` | الـ lexer (tokenizer) للـ `.dts` syntax |
| `scripts/dtc/dtc-parser.y` | الـ parser (yacc/bison grammar) اللي يبني الـ AST |
| `scripts/dtc/livetree.c` | تعامل مع الشجرة في الذاكرة (build/merge/delete nodes) |
| `scripts/dtc/flattree.c` | تحويل live tree ← → FDT blob (`.dtb`) |
| `scripts/dtc/fstree.c` | قراءة device tree من `/proc/device-tree` أو `/sys/firmware/fdt` |
| `scripts/dtc/treesource.c` | كتابة الـ tree تاني كـ `.dts` (decompile) |
| `scripts/dtc/srcpos.c` / `srcpos.h` | تتبع source positions للـ error messages الدقيقة |
| `scripts/dtc/data.c` | إدارة raw data blobs للـ properties |
| `scripts/dtc/util.c` / `util.h` | helper functions (memory, strings) |
| `scripts/dtc/yamltree.c` | output الشجرة بـ YAML format |
| `scripts/dtc/libfdt/` | مكتبة للتعامل مع الـ FDT binary format |

#### ملفات kernel ذات صلة

| الملف | الدور |
|-------|-------|
| `drivers/of/` | الـ OF (Open Firmware) layer اللي بيقرا الـ DTB في runtime |
| `include/linux/of*.h` | headers للـ OF API اللي بيستخدمها الـ drivers |
| `arch/*/boot/dts/` | ملفات `.dts` الفعلية لكل board |
| `include/dt-bindings/` | constants بتُستخدم في ملفات الـ DTS |
| `Documentation/devicetree/` | bindings documentation (YAML schema) |
| `tools/testing/selftests/dt/` | اختبارات runtime للـ device tree |
| `scripts/Makefile.dtb*` | build rules لتجميع ملفات الـ `.dts` |

---

### ملخص الهدف

**الـ `checks.c` هو "محامي صحة" الـ Device Tree** — بيتأكد قبل ما الـ kernel يشوف الـ DTB إن كل حاجة فيه منطقية: العناوين صح، الـ phandles بتوصّل لـ nodes موجودة، الـ bus protocols محترمة، والـ graph connections bidirectional. ده بيوفّر ساعات debug لو الغلط اتمسك وقت الـ build بدل ما يظهر كـ kernel panic أو device مش شغّال.
## Phase 2: شرح الـ DTC Semantic Checks Framework

### المشكلة اللي بيحلها الـ subsystem ده

الـ **Device Tree Compiler (DTC)** بيأخد ملفات الـ `.dts` (Device Tree Source) ويحولها لـ `.dtb` (Device Tree Blob) — binary format بيستخدمه الـ bootloader عشان يوصف الـ hardware للـ kernel.

المشكلة: ملف الـ `.dts` ممكن يكون syntactically صح (valid C-like syntax) لكن semantically غلط — يعني:
- node عنده `reg` property بس من غير unit address في الاسم
- phandle بيشاور على node مش موجودة
- I2C device عنده address أكبر من 7-bit من غير ما يعلنها 10-bit
- `compatible` property مش string list

الـ kernel مش هيقولك إيه الغلط — هيتعطل بصمت أو يتجاهل الـ device. الـ `checks.c` موجود عشان يمسك الأخطاء دي **قبل** ما الـ blob يوصل للـ board.

---

### الحل: نظام Checks قابل للتوسعة بـ Prerequisites

الـ DTC عنده **check engine** مبني على:
1. **`struct check`** — الوحدة الأساسية لكل فحص
2. **dependency graph** — كل check ممكن يعتمد على checks تانية (prereqs)
3. **recursive tree walk** — كل check بيتطبق على كل node في الـ tree
4. **severity levels** — WARNING أو ERROR، والـ framework يفرق بينهم

---

### الـ Big Picture: فين DTC Checks في الـ Toolchain؟

```
   .dts file
      │
      ▼
 ┌──────────┐
 │  Lexer   │  (dtc-lexer.l)   ← tokenize
 └────┬─────┘
      │
      ▼
 ┌──────────┐
 │  Parser  │  (dtc-parser.y)  ← build live tree (struct node/property)
 └────┬─────┘
      │
      ▼
 ┌─────────────────────────────────────────────┐
 │           checks.c  ← نحن هنا               │
 │                                             │
 │   process_checks(force, dti)                │
 │         │                                   │
 │   for each check in check_table[]           │
 │         │                                   │
 │   run_check(c, dti)                         │
 │         │                                   │
 │   check_nodes_props(c, dti, root)           │
 │    └── recursive: visits every node        │
 └────────────────┬────────────────────────────┘
                  │
                  ▼
         ┌────────────────┐
         │  dt_to_blob()  │  → .dtb output
         └────────────────┘
```

الـ checks بتشتغل على الـ **live tree** (in-memory representation) مش على الـ binary — وده مهم لأن الـ source position info لسا موجودة هنا لإعطاء error messages واضحة.

---

### الـ Core Abstraction: الـ `struct check`

```c
struct check {
    const char *name;        // اسم الفحص، يتعرض في رسائل الأخطاء
    check_fn fn;             // الدالة اللي بتنفذ الفحص فعلاً
    const void *data;        // بيانات extra بتحملها الدالة (مثلاً اسم الـ property)
    bool warn, error;        // severity: هل ده warning أو error؟
    enum checkstatus status; // UNCHECKED / PREREQ / PASSED / FAILED
    bool inprogress;         // cycle detection في الـ dependency graph
    int num_prereqs;         // عدد الـ prerequisites
    struct check **prereq;   // array of pointers لـ checks تانية
};
```

```
struct check
 ├── name: "reg_format"
 ├── fn: check_reg_format()
 ├── data: NULL
 ├── warn: true / error: false
 ├── status: UNCHECKED → PASSED/FAILED
 └── prereq[0] → &addr_size_cells  ← dependency
```

الـ `check_fn` signature ثابت:

```c
typedef void (*check_fn)(struct check *c, struct dt_info *dti, struct node *node);
```

كل check function بتاخد:
- **`c`**: الـ check نفسه (عشان تقدر تقرأ `c->data` وتكتب `c->status`)
- **`dti`**: الـ tree كاملة (الـ root + flags)
- **`node`**: الـ node الحالية اللي بيتفحص عليها

---

### الـ Macro System: تعريف الـ Checks بطريقة declarative

```c
#define CHECK_ENTRY(nm_, fn_, d_, w_, e_, ...)       \
    static struct check *nm_##_prereqs[] = { __VA_ARGS__ }; \
    static struct check nm_ = {                      \
        .name = #nm_,                                \
        .fn = (fn_),                                 \
        .data = (d_),                                \
        .warn = (w_),                                \
        .error = (e_),                               \
        .status = UNCHECKED,                         \
        .num_prereqs = ARRAY_SIZE(nm_##_prereqs),    \
        .prereq = nm_##_prereqs,                     \
    };

#define WARNING(nm_, fn_, d_, ...)  CHECK_ENTRY(nm_, fn_, d_, true,  false, __VA_ARGS__)
#define ERROR(nm_, fn_, d_, ...)    CHECK_ENTRY(nm_, fn_, d_, false, true,  __VA_ARGS__)
#define CHECK(nm_, fn_, d_, ...)    CHECK_ENTRY(nm_, fn_, d_, false, false, __VA_ARGS__)
```

مثال:
```c
// هذا السطر الواحد ينشئ struct كامل + prereq array
ERROR(node_name_format, check_node_name_format, NULL, &node_name_chars);
```

ده بيعمل:
```c
static struct check *node_name_format_prereqs[] = { &node_name_chars };
static struct check node_name_format = {
    .name = "node_name_format",
    .fn   = check_node_name_format,
    .data = NULL,
    .warn = false, .error = true,
    .num_prereqs = 1,
    .prereq = node_name_format_prereqs,
};
```

---

### الـ Dependency Graph: run_check وكيف بيتعامل مع الـ prerequisites

```c
static bool run_check(struct check *c, struct dt_info *dti)
{
    if (c->status != UNCHECKED)
        goto out;               // مش هيشتغل أكتر من مرة

    c->inprogress = true;

    for (i = 0; i < c->num_prereqs; i++) {
        error |= run_check(prq, dti);   // recursive: شغّل الـ prereq الأول
        if (prq->status != PASSED) {
            c->status = PREREQ;          // فشل prereq = skip الـ check ده
        }
    }

    check_nodes_props(c, dti, dt);      // امشي على كل الـ nodes

    if (c->status == UNCHECKED)
        c->status = PASSED;
}
```

الـ `inprogress` flag ده cycle detection — لو check A يعتمد على B ويعتمد على A، الـ assert هيوقف البرنامج.

مثال dependency chain:
```
reg_format
    └── prereq: addr_size_cells
                    └── prereq: address_cells_is_cell
                    └── prereq: size_cells_is_cell
```

لو `address_cells_is_cell` فشل، `addr_size_cells` بيتعلم الـ status `PREREQ`، وبالتالي `reg_format` نفسه بيتعلم `PREREQ` وما بيشتغلش.

---

### تصنيفات الـ Checks: إيه بيفحصوا؟

#### 1. Structural Checks

| Check | المشكلة اللي بيكشفها |
|---|---|
| `duplicate_node_names` | نفس اسم الـ node مرتين تحت نفس الـ parent |
| `duplicate_property_names` | نفس الـ property مرتين في نفس الـ node |
| `duplicate_label` | نفس الـ label على أكتر من node أو property |
| `explicit_phandles` | phandle مكرر أو قيمة 0 / 0xFFFFFFFF |
| `node_name_chars` | حروف مش مسموح بيها في اسم الـ node |

#### 2. Reference Fixup "Checks" (بيعدلوا الـ tree مش بس بيفحصوا)

```c
static void fixup_phandle_references(struct check *c, struct dt_info *dti,
                                     struct node *node)
{
    for_each_property(node, prop) {
        for_each_marker_of_type(m, REF_PHANDLE) {
            refnode = get_node_by_ref(dt, m->ref);
            phandle = get_node_phandle(dt, refnode);
            // Write the phandle value into the property binary data
            *((fdt32_t *)(prop->val.val + m->offset)) = cpu_to_fdt32(phandle);
        }
    }
}
ERROR(phandle_references, fixup_phandle_references, NULL,
      &duplicate_node_names, &explicit_phandles);
```

ده check بيحل الـ symbolic references — لما تكتب `&uart0` في الـ dts، الـ parser بيحطها كـ `REF_PHANDLE` marker، والـ fixup ده بيحولها لـ رقم فعلي.

#### 3. Semantic / Format Checks

```c
WARNING_IF_NOT_CELL(address_cells_is_cell, "#address-cells");
WARNING_IF_NOT_STRING(compatible_is_string_list, "compatible");
```

دي macros بتنشئ checks جاهزة بـ generic functions زي `check_is_cell` و`check_is_string`.

#### 4. Bus-Specific Checks

الـ framework عنده concept الـ **`struct bus_type`**:

```c
struct bus_type {
    const char *name;
};

static const struct bus_type pci_bus    = { .name = "PCI" };
static const struct bus_type i2c_bus    = { .name = "i2c-bus" };
static const struct bus_type spi_bus    = { .name = "spi-bus" };
static const struct bus_type simple_bus = { .name = "simple-bus" };
```

الـ `struct node` عنده field:
```c
const struct bus_type *bus;
```

الـ framework بيمشي على الـ tree، لما يلاقي node بـ `device_type = "pci"` يعيّن `node->bus = &pci_bus`، وبعدين كل child بيتفحص من خلال bus-specific checks.

```
Root
  └── pci@1,0  ← check_pci_bridge يعيّن node->bus = &pci_bus
        └── device@0   ← check_pci_device_reg يتحقق إن bus num صح
        └── device@1,1 ← check_pci_device_bus_num يتحقق من range
```

##### مثال: I2C address validation

```c
static void check_i2c_bus_reg(struct check *c, struct dt_info *dti, struct node *node)
{
    if (!node->parent || (node->parent->bus != &i2c_bus))
        return;  // مش تحت I2C bus → skip

    reg = fdt32_to_cpu(*cells);
    reg &= ~I2C_OWN_SLAVE_ADDRESS;  // ignore bit 30

    if (reg & I2C_TEN_BIT_ADDRESS) {
        if ((reg & ~I2C_TEN_BIT_ADDRESS) > 0x3ff)
            FAIL_PROP(..., "I2C address must be less than 10-bits");
    } else if (reg > 0x7f) {
        FAIL_PROP(..., "I2C address must be less than 7-bits");
    }
}
```

#### 5. Phandle Provider/Consumer Checks

أكتر جزء متطور في الـ file:

```c
struct provider {
    const char *prop_name;   // "clocks", "dmas", "interrupts-extended", إلخ
    const char *cell_name;   // "#clock-cells", "#dma-cells", إلخ
    bool optional;
};
```

الـ macro:
```c
#define WARNING_PROPERTY_PHANDLE_CELLS(nm, propname, cells_name, ...) \
    static struct provider nm##_provider = { (propname), (cells_name), __VA_ARGS__ }; \
    WARNING_IF_NOT_CELL(nm##_is_cell, cells_name); \
    WARNING(nm##_property, check_provider_cells_property, &nm##_provider, ...)
```

استخدام:
```c
WARNING_PROPERTY_PHANDLE_CELLS(clocks,   "clocks",   "#clock-cells");
WARNING_PROPERTY_PHANDLE_CELLS(dmas,     "dmas",     "#dma-cells");
WARNING_PROPERTY_PHANDLE_CELLS(iommus,   "iommus",   "#iommu-cells");
WARNING_PROPERTY_PHANDLE_CELLS(pwms,     "pwms",     "#pwm-cells");
// ... إلخ
```

الـ logic في `check_property_phandle_args`:
```
property "clocks = <&clk_div 2>"
                    │       │
                    phandle  args
                    │
                    └── get_node_by_phandle(root, phandle)
                              │
                              └── get_property(node, "#clock-cells")
                                        │
                                        └── cellsize = propval_cell(prop)
                                                  │
                                                  └── verify args count == cellsize
```

لو `cellsize = 1`، الـ consumer لازم يحط phandle + 1 cell بعده. لو حط 0 أو 2 → warning.

---

### الـ struct الكاملة: علاقة كل struct بالتاني

```
struct dt_info
 ├── dtsflags  (DTSF_V1, DTSF_PLUGIN)
 ├── reservelist → struct reserve_info
 ├── boot_cpuid_phys
 └── dt → struct node  ← root of the tree
              │
              ├── name, fullpath, basenamelen
              ├── phandle  (cell_t = uint32_t)
              ├── addr_cells, size_cells  (set by fixup_addr_size_cells)
              ├── bus → struct bus_type  (set by check_pci_bridge, etc.)
              ├── labels → struct label → label → next
              ├── proplist → struct property
              │                ├── name
              │                ├── val → struct data
              │                │         ├── len
              │                │         ├── val (raw bytes)
              │                │         └── markers → struct marker
              │                │                        ├── type (REF_PHANDLE, REF_PATH, LABEL, ...)
              │                │                        ├── offset (في الـ val buffer)
              │                │                        └── ref (اسم الـ symbol)
              │                └── next
              ├── children → struct node (first child)
              └── next_sibling → struct node
```

---

### الـ check_table: الـ Registry المركزي

```c
static struct check *check_table[] = {
    &duplicate_node_names, &duplicate_property_names,
    &node_name_chars, &node_name_format, ...
    &explicit_phandles,
    &phandle_references, &path_references,
    &omit_unused_nodes,
    &addr_size_cells, &reg_format, &ranges_format,
    &pci_bridge, &pci_device_reg, ...
    &i2c_bus_bridge, &i2c_bus_reg,
    &spi_bus_bridge, &spi_bus_reg,
    &clocks_property, &dmas_property, ...
    &gpios_property,
    &interrupts_property, &interrupt_provider, &interrupt_map,
    &graph_nodes, &graph_port, &graph_endpoint,
    &always_fail,
};
```

الـ `process_checks()` بتمشي على الـ array وبتشغل كل check عنده `warn || error`:

```c
void process_checks(bool force, struct dt_info *dti)
{
    for (i = 0; i < ARRAY_SIZE(check_table); i++) {
        struct check *c = check_table[i];
        if (c->warn || c->error)
            error = error || run_check(c, dti);
    }

    if (error && !force) {
        fprintf(stderr, "ERROR: Input tree has errors, aborting\n");
        exit(2);
    }
}
```

الـ `force` flag ده `dtc -f` — يخلي الـ compiler يكمل حتى لو في errors.

---

### Enable/Disable من command line

```c
void parse_checks_option(bool warn, bool error, const char *arg)
{
    // "no-reg_format" → name = "reg_format", enable = false
    // "-W reg_format" → warn=true, enable=true

    for each check in check_table:
        if streq(c->name, name):
            enable_warning_error(c, warn, error)
}
```

الـ `enable_warning_error` بيعمل propagation للأعلى — لو شغّلت check معين، الـ prereqs بتاعته بتتشغل كمان أوتوماتيك.
الـ `disable_warning_error` بيعمل propagation للأسفل — لو عطّلت check، كل اللي بيعتمد عليه بيتعطل.

---

### تشبيه واقعي: Lint Tool لكود C

الـ DTC checks يشبه تماماً `clang-tidy` أو `cppcheck` على كود C:

| DTC checks | clang-tidy |
|---|---|
| `struct check` | تعريف كل rule |
| `prereq[]` | rule A يعتمد على rule B |
| `check_table[]` | القائمة الكاملة للـ rules المفعّلة |
| `check_fn` | visitor بيمشي على الـ AST |
| WARNING / ERROR severity | `-Wwarning` / `-Werror` |
| `parse_checks_option("no-reg_format")` | `-Wno-unused-variable` |
| `force` flag | `--keep-going` |
| `struct node` = AST node | `clang::Stmt` أو `clang::Decl` |

لكن الفرق الجوهري: بعض الـ DTC checks مش بس بتفحص — بتـ **modify** الـ tree (زي `fixup_phandle_references` و`fixup_path_references` و`fixup_omit_unused_nodes`). يعني الـ check engine هو كمان جزء من الـ compilation pipeline نفسه.

---

### الـ Framework يملك / يفوّض لمين؟

| الـ Framework (checks.c) يملك | الـ Driver / Check Function تفعل |
|---|---|
| تعريف `struct check` وإدارة lifecycle | منطق الفحص الفعلي بداخل `check_fn` |
| tree traversal (يمشي على كل node) | تقرر إيه اللي تفحصه في كل node |
| dependency resolution بين الـ checks | تحدد الـ prerequisites لكل check |
| severity system (warn/error) | تستخدم `FAIL` أو `FAIL_PROP` |
| propagation لـ enable/disable | — |
| الـ check_table registry | تضيف نفسها للـ table |
| cycle detection بالـ `inprogress` flag | — |
| source position في رسائل الأخطاء | تمرر الـ node/prop للـ check_msg |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Flags — Cheatsheet

#### `enum checkstatus` — حالة الـ check

| القيمة | المعنى |
|--------|---------|
| `UNCHECKED = 0` | لسه ما اتشغلش |
| `PREREQ` | فشل لأن prereq فشل |
| `PASSED` | اتشغل وعدى |
| `FAILED` | اتشغل وطلع error/warning |

#### `enum markertype` — نوع الـ marker في الـ data blob

| القيمة | المعنى |
|--------|---------|
| `TYPE_NONE` | مفيش نوع |
| `REF_PHANDLE` | reference لـ phandle |
| `REF_PATH` | reference لـ path string |
| `LABEL` | label في الـ DTS source |
| `TYPE_UINT8` | بيانات uint8 |
| `TYPE_UINT16` | بيانات uint16 |
| `TYPE_UINT32` | بيانات uint32 |
| `TYPE_UINT64` | بيانات uint64 |
| `TYPE_STRING` | بيانات string |

#### الـ `dtsflags` Bits في `dt_info`

| الـ Flag | القيمة | المعنى |
|---------|---------|---------|
| `DTSF_V1` | `0x0001` | الملف بيستخدم `/dts-v1/` |
| `DTSF_PLUGIN` | `0x0002` | overlay plugin — يسمح unresolved phandles |

#### الـ `phandle_format` Options

| الـ Flag | القيمة | المعنى |
|---------|---------|---------|
| `PHANDLE_LEGACY` | `0x1` | يستخدم `linux,phandle` |
| `PHANDLE_EPAPR` | `0x2` | يستخدم `phandle` |
| `PHANDLE_BOTH` | `0x3` | يكتب الاتنين |

#### Macros تعريف الـ Checks

| الـ Macro | `warn` | `error` | الاستخدام |
|-----------|--------|---------|-----------|
| `WARNING(nm, fn, d, ...)` | `true` | `false` | تحذير بس |
| `ERROR(nm, fn, d, ...)` | `false` | `true` | error فعلي |
| `CHECK(nm, fn, d, ...)` | `false` | `false` | تشغيل بدون إبلاغ |
| `CHECK_ENTRY(nm, fn, d, w, e, ...)` | custom | custom | تحكم كامل |

#### Character Sets للأسماء

| الـ Macro | المحتوى | الاستخدام |
|-----------|---------|-----------|
| `NODECHARS` | lowercase + uppercase + digits + `,._+-@` | node names |
| `PROPCHARS` | lowercase + uppercase + digits + `,._+*#?-` | property names |
| `PROPNODECHARSSTRICT` | lowercase + uppercase + digits + `,-` | strict mode |

---

### كل الـ Structs المهمة

#### 1. `struct check` — الـ unit الأساسية للـ validation

**الغرض:** بيمثل check واحد (rule) بالاسم والـ function والـ dependencies.

```c
struct check {
    const char *name;        /* اسم الـ check — يظهر في رسائل الـ error */
    check_fn fn;             /* الـ function اللي بتنفذ الـ check فعلاً */
    const void *data;        /* بيانات extra — مثلاً اسم الـ property */
    bool warn, error;        /* مستوى الخطورة */
    enum checkstatus status; /* الحالة الحالية */
    bool inprogress;         /* لكشف الـ circular dependency */
    int num_prereqs;         /* عدد الـ prerequisites */
    struct check **prereq;   /* مصفوفة pointers للـ checks اللي لازم تعدي الأول */
};
```

**العلاقات:**
- بيشير على نفسه (self-referential) عبر `prereq[]`
- الـ `check_fn` بتاخد pointer على `struct check` نفسه + `dt_info` + `node`

---

#### 2. `struct dt_info` — الـ Device Tree بالكامل

**الغرض:** الـ container الرئيسي للـ device tree أثناء الـ compilation.

```c
struct dt_info {
    unsigned int dtsflags;         /* DTSF_V1 | DTSF_PLUGIN */
    struct reserve_info *reservelist; /* قائمة /memreserve/ entries */
    uint32_t boot_cpuid_phys;      /* الـ boot CPU ID */
    struct node *dt;               /* root node للشجرة */
    const char *outname;           /* اسم ملف الـ output، أو "-" للـ stdout */
};
```

**العلاقات:**
- `dt` → `struct node` (الـ root، وكل الشجرة بتبدأ منه)
- `reservelist` → `struct reserve_info` (linked list)

---

#### 3. `struct node` — عقدة في الـ Device Tree

**الغرض:** بيمثل node واحدة في شجرة الـ DT.

```c
struct node {
    bool deleted;                  /* اتمسحت أثناء المعالجة؟ */
    char *name;                    /* اسم الـ node */
    struct property *proplist;     /* linked list للـ properties */
    struct node *children;         /* أول child */
    struct node *parent;           /* الـ parent node */
    struct node *next_sibling;     /* الـ sibling اللي بعده */
    char *fullpath;                /* المسار الكامل مثل /cpus/cpu@0 */
    int basenamelen;               /* طول الاسم بدون الـ unit address */
    cell_t phandle;                /* الـ phandle المخصص للـ node */
    int addr_cells, size_cells;    /* من #address-cells و #size-cells — -1 = مش موجود */
    struct label *labels;          /* الـ labels المرتبطة بالـ node */
    const struct bus_type *bus;    /* نوع الـ bus لو اتعرف (PCI/I2C/SPI/...) */
    struct srcpos *srcpos;         /* موقع الـ node في ملف الـ source */
    bool omit_if_unused;           /* احذفه لو مفيش reference ليه */
    bool is_referenced;            /* فيه reference ليه من حاجة تانية؟ */
};
```

**العلاقات:**
- `parent` ↔ `children` + `next_sibling` = بناء الشجرة الكاملة
- `proplist` → `struct property`
- `labels` → `struct label`
- `bus` → `struct bus_type` (static const — PCI/I2C/SPI/graph)
- `srcpos` → `struct srcpos`

---

#### 4. `struct property` — property داخل node

**الغرض:** بيمثل property واحدة (اسم + قيمة).

```c
struct property {
    bool deleted;          /* اتمسحت؟ */
    char *name;            /* اسم الـ property */
    struct data val;       /* البيانات (raw bytes + markers) */
    struct property *next; /* الـ property اللي بعدها في القائمة */
    struct label *labels;  /* labels مرتبطة بالـ property */
    struct srcpos *srcpos; /* موقعها في الـ source */
};
```

**العلاقات:**
- `val` → `struct data` (اللي فيها الـ markers كمان)
- `labels` → `struct label`

---

#### 5. `struct data` — raw data blob

**الغرض:** بيخزن البيانات الخام لـ property مع metadata للـ type والـ references.

```c
struct data {
    unsigned int len;     /* الحجم بالبايت */
    char *val;            /* البيانات الخام */
    struct marker *markers; /* linked list للـ markers (phandle refs, labels, types) */
};
```

---

#### 6. `struct marker` — annotation على الـ data

**الغرض:** بيحدد نقطة معينة في الـ data blob ويقول إيه نوعها (phandle ref / path ref / label / type).

```c
struct marker {
    enum markertype type; /* نوع الـ marker */
    unsigned int offset;  /* الـ offset من بداية الـ data */
    char *ref;            /* اسم الـ label أو الـ path */
    struct marker *next;
};
```

---

#### 7. `struct label` — label مرتبطة بـ node أو property

```c
struct label {
    bool deleted;
    char *label;         /* نص الـ label */
    struct label *next;
};
```

---

#### 8. `struct bus_type` — نوع الـ bus

**الغرض:** marker بسيط لتحديد نوع الـ bus — بيتستخدم كـ pointer comparison مش string comparison.

```c
struct bus_type {
    const char *name;  /* "PCI", "simple-bus", "i2c-bus", "spi-bus", "graph-port", ... */
};
```

الـ instances الموجودة في `checks.c`:
- `pci_bus`
- `simple_bus`
- `i2c_bus`
- `spi_bus`
- `graph_port_bus`
- `graph_ports_bus`

---

#### 9. `struct provider` — معلومات الـ phandle provider

**الغرض:** بيوصف الـ property اللي بتشير لـ provider وإيه اسم الـ cell-count property.

```c
struct provider {
    const char *prop_name;  /* اسم الـ property اللي بتحتوي phandles، مثلاً "clocks" */
    const char *cell_name;  /* اسم الـ cells property، مثلاً "#clock-cells" */
    bool optional;          /* الـ cell_name اختياري؟ */
};
```

---

#### 10. `struct reserve_info` — /memreserve/ entry

```c
struct reserve_info {
    uint64_t address, size;  /* العنوان والحجم */
    struct reserve_info *next;
    struct label *labels;
};
```

---

#### 11. `struct srcpos` — موقع في الـ source file

```c
struct srcpos {
    int first_line, first_column;
    int last_line, last_column;
    struct srcfile_state *file; /* الملف اللي جاي منه */
    struct srcpos *next;        /* لو اتعرف في أماكن متعددة */
};
```

---

### رسم العلاقات بين الـ Structs

```
dt_info
  │
  ├── reservelist ──► reserve_info ──► reserve_info ──► NULL
  │                      └── labels ──► label ──► ...
  │
  └── dt (root node)
        │
        └── struct node (root "/")
              ├── proplist ──► property ──► property ──► NULL
              │                  ├── val (struct data)
              │                  │     └── markers ──► marker ──► marker ──► NULL
              │                  ├── labels ──► label ──► ...
              │                  └── srcpos
              │
              ├── children ──► node ("cpus") ──► node ("memory") ──► NULL
              │                   │                   │
              │                   │ next_sibling ──────┘
              │                   └── children ──► node ("cpu@0") ──► NULL
              │
              ├── parent ──► NULL  (لأنه root)
              ├── bus ──► NULL (أو &pci_bus / &simple_bus / ...)
              └── srcpos ──► srcpos
                               └── file ──► srcfile_state
```

```
struct check
  │
  ├── fn ──► check_fn(c, dti, node)   [function pointer]
  ├── data ──► (const void*) — قد يكون string أو struct provider*
  ├── status (enum checkstatus)
  └── prereq[] ──► &check_A
                ──► &check_B
                ──► ...
```

```
struct provider
  ├── prop_name  ──► "clocks"
  ├── cell_name  ──► "#clock-cells"
  └── optional   ──► false/true

       يتستخدم في:
       check_provider_cells_property()
       check_property_phandle_args()
            └──► get_node_by_phandle(root, phandle)
                  └──► struct node (provider node)
                         └──► get_property(node, cell_name)
                                └──► propval_cell()
```

---

### Lifecycle Diagrams

#### دورة حياة الـ `struct check`

```
[تعريف static]
      │
      ▼
CHECK_ENTRY(nm, fn, data, warn, error, prereqs...)
      │
      ▼   يُنشئ:
  static struct check *nm_prereqs[] = { ... }
  static struct check nm = { .name, .fn, .data, .warn, .error,
                              .status=UNCHECKED, .num_prereqs, .prereq }
      │
      ▼
check_table[] يحتوي pointer على nm
      │
      ▼
parse_checks_option() [اختياري — يغير warn/error flags]
      │
      ▼
process_checks(force, dti)
      │
      ▼  لكل check في check_table:
run_check(c, dti)
      │
      ├── c->status != UNCHECKED? ──► goto out (skip)
      │
      ├── c->inprogress = true
      │
      ├── لكل prereq:
      │     run_check(prereq, dti)  [recursive]
      │     prereq->status != PASSED? ──► c->status = PREREQ
      │
      ├── c->status == UNCHECKED?
      │     └──► check_nodes_props(c, dti, root_node)
      │               └── c->fn(c, dti, node) لكل node بالـ DFS
      │               └── FAIL() أو FAIL_PROP() يغير c->status = FAILED
      │
      ├── c->status == UNCHECKED? ──► c->status = PASSED
      │
      └── c->inprogress = false
            c->status != PASSED && c->error ──► error = true
```

#### دورة حياة الـ `struct node`

```
[Parser يقرأ الـ DTS]
      │
      ▼
build_node(proplist, children, srcpos)
      │
      ▼
name_node(node, name)  ──► يضبط name, fullpath, basenamelen
      │
      ▼
add_child(parent, child)  ──► يضبط parent pointer + next_sibling
      │
      ▼
[checks تشتغل عليه]
      │
      ├── fixup_addr_size_cells() ──► يضبط addr_cells, size_cells
      ├── check_pci_bridge()      ──► يضبط node->bus = &pci_bus
      ├── fixup_phandle_references() ──► يضبط phandle قيمة فعلية
      ├── fixup_omit_unused_nodes()  ──► delete_node() لو omit_if_unused
      │
      ▼
[Output: dt_to_blob / dt_to_source]
      │
      ▼
[free() — الذاكرة بتتحرر يدوياً]
```

---

### Call Flow Diagrams

#### `process_checks()` — الـ flow الرئيسي

```
process_checks(force, dti)
  │
  ├── for each check_table[i]:
  │     if (c->warn || c->error):
  │         run_check(c, dti)
  │               │
  │               ├── c->inprogress = true
  │               ├── for each prereq:
  │               │       run_check(prereq, dti)   ◄── recursive
  │               │       prereq->status != PASSED
  │               │           └──► c->status = PREREQ
  │               │               check_msg(c, dti, NULL, NULL, "Failed prereq")
  │               │
  │               └── check_nodes_props(c, dti, dti->dt)
  │                         │
  │                         ├── c->fn(c, dti, node)   ◄── الـ check function
  │                         │       └──► FAIL(c, dti, node, ...)
  │                         │               └──► c->status = FAILED
  │                         │                   check_msg(c, dti, node, NULL, ...)
  │                         │
  │                         └── for_each_child(node, child):
  │                                 check_nodes_props(c, dti, child)  ◄── DFS
  │
  └── error && !force ──► exit(2)
      error && force  ──► print warning only
```

#### `check_msg()` — طباعة رسائل الـ error/warning

```
check_msg(c, dti, node, prop, fmt, ...)
  │
  ├── !(c->warn && quiet<1) && !(c->error && quiet<2) ──► return  [suppressed]
  │
  ├── اختيار الـ source position:
  │     prop->srcpos ──► يستخدمه
  │     node->srcpos ──► يستخدمه
  │     dti->outname == "-" ──► "<stdout>"
  │     else ──► dti->outname
  │
  ├── يبني الرسالة:
  │     "file:line: ERROR/Warning (check_name): "
  │     + "node_path:prop_name: "
  │     + رسالة الـ format
  │
  ├── لو node بس (مش prop):
  │     يطبع كل srcpos->next ("also defined at ...")
  │
  └── fputs(str, stderr)
```

#### `fixup_phandle_references()` — resolve الـ phandle refs

```
fixup_phandle_references(c, dti, node)
  │
  └── for_each_property(node, prop):
        markers = prop->val.markers
        for_each_marker_of_type(m, REF_PHANDLE):
            │
            ├── get_node_by_ref(dt, m->ref) ──► refnode
            │     refnode == NULL && !PLUGIN ──► FAIL("non-existent node")
            │     refnode == NULL &&  PLUGIN ──► write 0xffffffff
            │
            ├── get_node_phandle(dt, refnode) ──► phandle (يخصص لو مش موجود)
            │
            ├── *((fdt32_t*)(prop->val.val + m->offset)) = cpu_to_fdt32(phandle)
            │
            └── reference_node(refnode) ──► refnode->is_referenced = true
```

#### `check_property_phandle_args()` — validate phandle + cells

```
check_property_phandle_args(c, dti, node, prop, provider)
  │
  ├── prop->val.len % sizeof(cell_t) != 0 ──► FAIL_PROP
  │
  └── for cell=0; cell < len/4; cell += cellsize+1:
        phandle = propval_cell_n(prop, cell)
        │
        ├── !phandle_is_valid ──► cellsize=0, continue
        │
        ├── verify marker at offset is REF_PHANDLE
        │
        ├── provider_node = get_node_by_phandle(root, phandle)
        │     NULL ──► FAIL_PROP
        │
        ├── cellprop = get_property(provider_node, provider->cell_name)
        │     found  ──► cellsize = propval_cell(cellprop)
        │     !found && optional ──► cellsize = 0
        │     !found && !optional ──► FAIL
        │
        └── expected = (cell + cellsize + 1) * 4
              prop->val.len < expected ──► FAIL_PROP
```

#### `run_check()` — cycle detection

```
run_check(c, dti)
  │
  ├── assert(!c->inprogress)   ◄── يكشف الـ circular dependency فوراً بـ assert
  │
  ├── c->status != UNCHECKED ──► goto out  (already done)
  │
  └── c->inprogress = true
        │
        ├── [prereqs loop — recursive calls]
        │
        └── c->inprogress = false
```

---

### الـ Locking Strategy

**الـ `checks.c` مفيهوش أي locking.** ده بـ design:

- الـ DTC (Device Tree Compiler) بيشتغل **single-threaded** بالكامل
- مفيش shared state بين threads
- كل الـ `struct check` instances هي `static` globals — بس بيتوصلولها من thread واحد
- الـ `struct node` / `struct property` trees بيتبنوا وبيتشيكوا في نفس الـ thread

**الـ reentrancy:**
- `run_check()` بيستخدم `c->inprogress = true` كـ guard ضد الـ cycles — مش mutex
- لو حصل circular prereq، `assert(!c->inprogress)` بتـ crash فوراً في debug builds

**الـ global state اللي محتاج attention:**
```c
extern int quiet;           /* read-only أثناء الـ checks */
extern int generate_symbols; /* read-only أثناء الـ checks */
static struct check *check_table[]; /* static array — read-only بعد التعريف */
```

كل الـ mutation بتحصل في parsing phase قبل ما `process_checks()` تتشغل.

---

### ملخص العلاقات الكاملة

```
check_table[]
  └──► struct check (nm)
            ├── fn ──────────────────────────────────► check_fn
            │                                              │
            │                                      يشتغل على:
            │                                         struct dt_info
            │                                              │
            │                                         ├── dt ──► struct node (root)
            │                                         │              ├── proplist ──► struct property
            │                                         │              │                   ├── val (struct data)
            │                                         │              │                   │     └── markers ──► struct marker
            │                                         │              │                   └── labels ──► struct label
            │                                         │              ├── children ──► struct node (child)
            │                                         │              ├── bus ──► struct bus_type (static)
            │                                         │              └── srcpos ──► struct srcpos
            │                                         └── reservelist ──► struct reserve_info
            │
            ├── data ──► string أو struct provider
            │                   └── (prop_name, cell_name, optional)
            │
            └── prereq[] ──► &check_A ──► &check_B ──► ...
                                │               └── check له prereqs تانية
                                └── نفس البنية recursive
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs

#### جدول الـ Core Engine Functions

| Function | النوع | الغرض |
|---|---|---|
| `check_msg` | static inline | طباعة رسالة warning/error مع source location |
| `check_nodes_props` | static | تطبيق check واحد على كل nodes في الشجرة recursively |
| `is_multiple_of` | static | helper: هل `multiple` هو مضاعف صحيح لـ `divisor` |
| `run_check` | static | تشغيل check واحد مع prerequisites |
| `enable_warning_error` | static | رفع severity مستوى check وكل prerequisites |
| `disable_warning_error` | static | خفض severity مع propagation لـ dependents |
| `parse_checks_option` | public | تفسير `-W`/`-E` CLI options |
| `process_checks` | public | نقطة الدخول الرئيسية لتشغيل كل الـ checks |

#### جدول الـ Structural Check Functions

| Check Function | المستوى | ما يتحقق منه |
|---|---|---|
| `check_duplicate_node_names` | ERROR | أسماء nodes مكررة تحت نفس الـ parent |
| `check_duplicate_property_names` | ERROR | خصائص مكررة في نفس الـ node |
| `check_node_name_chars` | ERROR | أحرف غير مسموحة في اسم الـ node |
| `check_node_name_chars_strict` | CHECK | أحرف غير موصى بها (strict mode) |
| `check_node_name_format` | ERROR | وجود `@` متعدد في اسم الـ node |
| `check_node_name_not_empty` | ERROR | اسم الـ node فارغ |
| `check_node_name_vs_property_name` | WARNING | تعارض اسم الـ node مع property في الـ parent |
| `check_property_name_chars` | ERROR | أحرف غير مسموحة في اسم الـ property |
| `check_property_name_chars_strict` | CHECK | أحرف غير موصى بها في أسماء الـ properties |
| `check_duplicate_label` | ERROR (helper) | label مكرر في مكانين مختلفين |
| `check_duplicate_label_node` | ERROR | يمشي على كل labels وmarkers في node |

#### جدول الـ Phandle & Reference Functions

| Function | النوع | الغرض |
|---|---|---|
| `check_phandle_prop` | static helper | قراءة والتحقق من قيمة phandle property |
| `check_explicit_phandles` | ERROR | phandle مكرر أو يشير لـ node آخر |
| `fixup_phandle_references` | ERROR | كتابة قيم phandle الفعلية في الـ properties |
| `fixup_path_references` | ERROR | كتابة الـ full path في REF_PATH markers |
| `fixup_omit_unused_nodes` | ERROR | حذف nodes الغير مستخدمة |

#### جدول الـ Bus-Specific Check Functions

| Function | Bus | ما يتحقق منه |
|---|---|---|
| `check_pci_bridge` | PCI | #address-cells=3, #size-cells=2, ranges, bus-range |
| `check_pci_device_bus_num` | PCI | bus number ضمن نطاق bus-range الـ parent |
| `check_pci_device_reg` | PCI | صيغة reg في PCI config space |
| `check_simple_bus_bridge` | simple-bus | الكشف عن simple-bus compatible |
| `check_simple_bus_reg` | simple-bus | unit-address == reg address |
| `check_i2c_bus_bridge` | I2C | #address-cells=1, #size-cells=0 |
| `check_i2c_bus_reg` | I2C | عنوان صحيح ≤7bit أو 10bit |
| `check_spi_bus_bridge` | SPI | #address-cells=1, #size-cells=0 |
| `check_spi_bus_reg` | SPI | unit-address == chip select |

#### جدول الـ Semantic & Style Check Functions

| Function | المستوى | الغرض |
|---|---|---|
| `check_reg_format` | WARNING | طول reg property صحيح بناءً على #address/#size-cells |
| `check_ranges_format` | WARNING | صحة ranges/dma-ranges |
| `check_unit_address_vs_reg` | WARNING | وجود unit-address بدون reg والعكس |
| `check_unit_address_format` | WARNING | لا leading 0x أو leading zeros |
| `check_avoid_default_addr_size` | WARNING | الاعتماد على قيم default لـ #address/#size-cells |
| `check_avoid_unnecessary_addr_size` | WARNING | #address/#size-cells غير ضروريين |
| `check_unique_unit_address` | WARNING | unit-address مكرر بين siblings |
| `check_unique_unit_address_if_enabled` | CHECK | نفسه لكن يتجاهل disabled nodes |
| `check_names_is_string_list` | WARNING | كل `*-names` properties هي string lists |
| `check_alias_paths` | WARNING | aliases تشير لـ paths حقيقية وأسماء lowercase |

#### جدول الـ Interrupt & Provider Check Functions

| Function | الغرض |
|---|---|
| `check_property_phandle_args` | helper: التحقق من phandle+args في أي provider property |
| `check_provider_cells_property` | wrapper يشغّل check على property محددة |
| `check_gpios_property` | التحقق من #gpio-cells لكل *-gpios properties |
| `check_deprecated_gpio_property` | تحذير من استخدام `[*-]gpio` بدل `[*-]gpios` |
| `node_is_interrupt_provider` | helper: هل الـ node يحمل interrupt-controller أو interrupt-map |
| `check_interrupt_provider` | WARNING: #interrupt-cells لازم على interrupt providers |
| `check_interrupt_map` | WARNING: التحقق من صحة interrupt-map بالكامل |
| `check_interrupts_property` | WARNING: حجم interrupts property يتطابق مع #interrupt-cells |

#### جدول الـ Graph Check Functions

| Function | الغرض |
|---|---|
| `check_graph_nodes` | الكشف عن graph port/endpoint nodes وتعيين bus type |
| `check_graph_reg` | reg وunit-address صحيحين في graph nodes |
| `check_graph_port` | اسم الـ port صحيح وتحقق من reg |
| `get_remote_endpoint` | helper: جلب الـ node الـ remote عبر phandle |
| `check_graph_endpoint` | endpoint name صحيح + bidirectional connection |

---

### Group 1: Core Engine — البنية التحتية لنظام الـ Checks

هذه المجموعة هي القلب الذي يُشغّل كل الـ checks. بدونها ما في شيء يعمل. تتعامل مع الـ state machine للـ check، وتطبّق prerequisite ordering، وتطبع الرسائل بالصيغة الصحيحة.

---

#### `check_msg`

```c
static inline void PRINTF(5, 6) check_msg(struct check *c,
                                           struct dt_info *dti,
                                           struct node *node,
                                           struct property *prop,
                                           const char *fmt, ...)
```

**ما تعمله:** بتبني string كاملة للـ warning/error message وتطبعها على `stderr`. بتحدد أول حاجة مصدر الرسالة (source position من الـ `.dts` file أو stdout)، وتضيف اسم الـ check والـ node/property path، وبعدين تضيف الـ message الفعلية. لو الـ node عنده multiple source positions (من merge)، بتطبع كل الـ "also defined at" locations.

**Parameters:**
- `c` — الـ check اللي الرسالة جاية منه؛ بيستخدم `c->error` و`c->name`
- `dti` — الـ device tree info؛ بيستخدم `dti->outname` fallback لو مفيش source position
- `node` — الـ node اللي فيه المشكلة؛ ممكن يكون NULL
- `prop` — الـ property اللي فيها المشكلة؛ ممكن يكون NULL
- `fmt, ...` — format string للـ message الفعلية

**Return value:** void

**Key details:**
- ما بتطبعش حاجة لو `quiet >= 1` ومفيش warning، أو `quiet >= 2` ومفيش error — يعني الـ quiet level بيتحكم في الـ verbosity
- بتستخدم `xasprintf`/`xavsprintf_append` اللي بتـ`die()` عند فشل الـ allocation
- الـ `PRINTF(5, 6)` attribute بيخلي GCC يـcheck الـ format string statically
- **بتُستدعى من:** `FAIL` macro، `FAIL_PROP` macro، `run_check` لما prerequisites تفشل

---

#### `check_nodes_props`

```c
static void check_nodes_props(struct check *c,
                               struct dt_info *dti,
                               struct node *node)
```

**ما تعمله:** بتمشي على شجرة الـ DT كلها recursively وبتطبق الـ check function الموجودة في `c->fn` على كل node. الـ traversal بالـ DFS pre-order — الـ parent قبل الـ children.

**Parameters:**
- `c` — الـ check اللي هيتطبق؛ `c->fn` هو الـ callback
- `dti` — الـ device tree info المرتبط بالـ tree
- `node` — الـ node الحالية في الـ traversal

**Return value:** void

**Key details:**
- لو `c->fn == NULL`، الـ check بيمشي على كل الـ tree من غير أي عمل (useful للـ checks اللي بس بتعتمد على prerequisites لـ side effects زي `fixup_addr_size_cells`)
- بتستخدم `for_each_child` macro اللي بيتخطى الـ deleted nodes
- لا locking — الـ DTC tool يشتغل single-threaded

---

#### `is_multiple_of`

```c
static bool is_multiple_of(int multiple, int divisor)
```

**ما تعمله:** بترجع `true` لو `multiple` هو مضاعف صحيح لـ `divisor`. Edge case: لو `divisor == 0` بترجع `true` فقط لو `multiple == 0`.

**Parameters:**
- `multiple` — القيمة اللي هنتحقق منها
- `divisor` — القسمة المطلوبة

**Return value:** `bool` — النتيجة

**Key details:** مستخدمة كتير في `check_reg_format` و`check_ranges_format` و`check_interrupt_map` للتحقق من أن الـ property length مضاعف لـ entry size.

---

#### `run_check`

```c
static bool run_check(struct check *c, struct dt_info *dti)
```

**ما تعمله:** ينفذ check واحد على الـ device tree الكامل مع prerequisite resolution. ده الـ scheduler المحوري للنظام. بيمنع circular dependencies بفضل `inprogress` flag، وبيكشف prerequisites لم تمر، وبيشغّل الـ check function على كل nodes لو الـ prerequisites نجحت.

**Parameters:**
- `c` — الـ check المطلوب تشغيله
- `dti` — الـ device tree info

**Return value:** `bool error` — يرجع `true` لو الـ check ده (أو أي prerequisite) كان error-level وفشل

**Key details:**
- لو `c->status != UNCHECKED` بيتخطى الـ execution مباشرة (idempotent)
- `assert(!c->inprogress)` بيكشف الـ circular prerequisites في development
- لو أي prerequisite فشل، الـ check يتسجل كـ `PREREQ` (مش `FAILED`) ويطبع رسالة "Failed prerequisite"
- بعد `check_nodes_props`، لو الـ status لسه `UNCHECKED` (ما انقلبش لـ`FAILED` من داخل الـ fn)، يترقّى لـ `PASSED`
- الـ error return فقط بيكون `true` لو `c->error == true` و`c->status != PASSED`

**Pseudocode flow:**
```
run_check(c, dti):
  assert !c->inprogress
  if c->status != UNCHECKED: goto out

  c->inprogress = true
  for each prereq prq:
    error |= run_check(prq, dti)          // recursive
    if prq->status != PASSED:
      c->status = PREREQ
      print "Failed prerequisite prq->name"

  if c->status == UNCHECKED:
    check_nodes_props(c, dti, dt->root)   // run the actual check
    if c->status == UNCHECKED:
      c->status = PASSED

out:
  c->inprogress = false
  if c->status != PASSED && c->error: error = true
  return error
```

---

#### `enable_warning_error`

```c
static void enable_warning_error(struct check *c, bool warn, bool error)
```

**ما تعمله:** برفع severity level لـ check معين. لو بترفع، بتعمل propagation للـ prerequisites لأنهم لازم يكونوا enabled كمان. مثلاً لو enable error على `reg_format`، لازم `addr_size_cells` يكون enabled.

**Parameters:**
- `c` — الـ check المطلوب رفعه
- `warn` — هل هنرفع لـ warning level
- `error` — هل هنرفع لـ error level

**Return value:** void

**Key details:** الـ propagation upstream (للـ prerequisites) مش downstream. يُستدعى من `parse_checks_option`.

---

#### `disable_warning_error`

```c
static void disable_warning_error(struct check *c, bool warn, bool error)
```

**ما تعمله:** بيخفض severity level لـ check. الـ propagation هنا downstream — بيمشي على كل الـ `check_table` ولأي check بيعتمد على `c` كـ prerequisite بيعمل recursive disable. كده لو حطيت `no-reg_format`، الـ checks الـ downstream اللي بتعتمد عليه هتتأثر.

**Parameters:**
- `c` — الـ check المطلوب خفضه
- `warn` — هل هنخفض الـ warn flag
- `error` — هل هنخفض الـ error flag

**Return value:** void

**Key details:** بيمشي على كل `ARRAY_SIZE(check_table)` في كل call — O(n²) في worst case لكن الـ table صغير (~50 checks) فمعلش.

---

#### `parse_checks_option`

```c
void parse_checks_option(bool warn, bool error, const char *arg)
```

**ما تعمله:** بتفسّر `-W check_name` أو `-E check_name` من الـ CLI. لو الاسم يبدأ بـ `no-` أو `no_` بتشيل الـ prefix وبتـ disable الـ check. غير كده بتـ enable. بتموت على اسم مجهول.

**Parameters:**
- `warn` — الـ check هيُفعَّل كـ warning
- `error` — الـ check هيُفعَّل كـ error
- `arg` — اسم الـ check (مثلاً `"reg_format"` أو `"no-reg_format"`)

**Return value:** void

**Key details:**
- `die()` عند اسم غير موجود في `check_table`
- يُستدعى من الـ CLI parser في `dtc.c`
- يقبل كلا الصيغتين `no-` و `no_` (flexibility للـ users)

---

#### `process_checks`

```c
void process_checks(bool force, struct dt_info *dti)
```

**ما تعمله:** نقطة الدخول الرئيسية لكل الـ checks. بتمشي على كل الـ `check_table` وبتشغّل الـ checks اللي عندها `warn` أو `error` enabled. لو في أي error وـ`force == false` يموت بـ `exit(2)`. لو `force == true` يطبع warning ويكمّل.

**Parameters:**
- `force` — الـ `-f` flag اللي بيخلي الـ DTC يكمّل رغم الأخطاء
- `dti` — الـ complete device tree info

**Return value:** void — ممكن تـ`exit(2)`

**Key details:**
- بيستدعي `run_check` على كل check في الـ table؛ الـ idempotency في `run_check` بتضمن ما يتشغلش check مرتين
- `quiet < 3` شرط طباعة الـ "output forced" warning
- يُستدعى من `main()` بعد parsing وقبل output generation

---

### Group 2: Utility Checks — التحقق من نوع الـ Properties

هذه المجموعة توفر building blocks للـ checks الأعلى مستوى. بتتحقق من نوع property معين في node معين.

---

#### `check_is_string`

```c
static void check_is_string(struct check *c, struct dt_info *dti,
                             struct node *node)
```

**ما تعمله:** بتجيب الـ property اللي اسمها `c->data` من الـ node وبتتحقق إنها string واحدة null-terminated. لو الـ property مش موجودة بتعتبرها OK.

**Parameters:**
- `c->data` — اسم الـ property (`const char *`)
- `node` — الـ node المطلوب فحصه

**Return value:** void — بتستخدم `FAIL_PROP` لو في مشكلة

**Key details:** مستخدمة كـ base لـ `WARNING_IF_NOT_STRING` و`ERROR_IF_NOT_STRING` macros. بتستدعي `data_is_one_string()` من `dtc.h`.

---

#### `check_is_string_list`

```c
static void check_is_string_list(struct check *c, struct dt_info *dti,
                                  struct node *node)
```

**ما تعمله:** بتتحقق إن الـ property هي قائمة من strings كل واحدة null-terminated. بتمشي على الـ data byte by byte وبتتحقق إن كل string لها `\0` terminator قبل نهاية الـ data.

**Key details:** الـ edge case: لو `strnlen(str, rem) == rem` معناه مفيش `\0` قبل نهاية الـ buffer — ده الـ failure condition. مستخدمة في `compatible`, `*-names` properties.

---

#### `check_is_cell`

```c
static void check_is_cell(struct check *c, struct dt_info *dti,
                           struct node *node)
```

**ما تعمله:** بتتحقق إن الـ property حجمها بالضبط `sizeof(cell_t)` = 4 bytes. مستخدمة لـ properties زي `#address-cells`, `#size-cells`, `#interrupt-cells`.

---

### Group 3: Structural Checks — التحقق من البنية

---

#### `check_duplicate_node_names`

```c
static void check_duplicate_node_names(struct check *c, struct dt_info *dti,
                                        struct node *node)
```

**ما تعمله:** بتمشي على كل الـ children pairs لـ node معين وبتتحقق ما فيش اثنين بنفس الاسم. الـ comparison بالـ `name` مش `fullpath`.

**Key details:** O(n²) في عدد الـ children لكن الـ DT nodes قليلة فعملياً مفيش مشكلة. بتستخدم `for_each_child` اللي بيتجاهل الـ deleted nodes.

---

#### `check_duplicate_label`

```c
static void check_duplicate_label(struct check *c, struct dt_info *dti,
                                   const char *label, struct node *node,
                                   struct property *prop, struct marker *mark)
```

**ما تعمله:** helper function بتتحقق إن label معين مش معرّف في أكثر من مكان واحد في الـ tree كلها. بتدور بـ `get_node_by_label`، `get_property_by_label`، `get_marker_label` على الـ tree كلها.

**Parameters:**
- `label` — الـ label string المطلوب البحث عنه
- `node`, `prop`, `mark` — الـ location الحالي اللي وجدنا فيه الـ label

**Key details:** لو اتلاقى الـ label في نفس الـ (node, prop, mark) combination بيعتبره OK (مش error). الـ error بس لو في مكانين مختلفين.

---

### Group 4: Phandle & Reference Fixups

هذه المجموعة هي الأهم بعد compilation — بتحول الـ symbolic references (`&node_label`) لـ numeric phandle values فعلية في الـ binary output.

---

#### `check_phandle_prop`

```c
static cell_t check_phandle_prop(struct check *c, struct dt_info *dti,
                                  struct node *node, const char *propname)
```

**ما تعمله:** بتقرأ قيمة phandle property (`"phandle"` أو `"linux,phandle"`) وبتتحقق من صحتها. بتتحقق من الحجم، وإن القيمة ليست self-reference، وإنها valid (مش 0 أو ~0).

**Return value:** `cell_t` — قيمة الـ phandle لو valid، أو 0 لو مفيش أو invalid

**Key details:**
- لو الـ property فيها `REF_PHANDLE` marker يشير لـ node آخر = error
- لو `REF_PHANDLE` يشير للـ node نفسه = valid (طلب allocate phandle) فيرجع 0 (سيُحدَّد لاحقاً)

---

#### `check_explicit_phandles`

```c
static void check_explicit_phandles(struct check *c, struct dt_info *dti,
                                     struct node *node)
```

**ما تعمله:** بتتحقق إن الـ phandle values الـ explicit (اللي الـ user كتبها في الـ DTS) صحيحة ومش مكررة. بتفحص كلاً من `phandle` و`linux,phandle` properties وبتتأكد إنهم متطابقين لو الاثنين موجودين. بتخزن الـ phandle في `node->phandle`.

**Key details:**
- `assert(!node->phandle)` يضمن ما في phase سابقة حطت phandle
- يُستدعى كـ ERROR check — أي failure يوقف البناء بدون `-f`

---

#### `fixup_phandle_references`

```c
static void fixup_phandle_references(struct check *c, struct dt_info *dti,
                                      struct node *node)
```

**ما تعمله:** بتمشي على كل الـ properties وبتدور على الـ `REF_PHANDLE` markers. لكل marker بتجيب الـ node المقصود بالـ reference، بتجيب أو تُنشئ phandle له، وبتكتب الـ phandle value في الـ property data بـ big-endian. كمان بتعمل `reference_node` لتعليم الـ node إنه مستخدم.

**Key details:**
- الـ overlay support: لو الـ referenced node مش موجود في overlay (`DTSF_PLUGIN`)، بتكتب `0xffffffff` placeholder بدل الـ die
- بيعتمد كـ prerequisite على `duplicate_node_names` و`explicit_phandles`
- بعد ما يكمّل، الـ binary data في الـ properties بتكون جاهزة للكتابة كـ FDT

---

#### `fixup_path_references`

```c
static void fixup_path_references(struct check *c, struct dt_info *dti,
                                   struct node *node)
```

**ما تعمله:** مشابهة لـ `fixup_phandle_references` لكن للـ `REF_PATH` markers. بدل ما تكتب phandle number بتكتب الـ full path string للـ node المقصود (مثلاً `/soc/uart@1000000`). بتستخدم `data_insert_at_marker` لإدراج الـ string في مكانها الصحيح.

---

#### `fixup_omit_unused_nodes`

```c
static void fixup_omit_unused_nodes(struct check *c, struct dt_info *dti,
                                     struct node *node)
```

**ما تعمله:** بتشيل الـ nodes اللي عليها `omit-if-unused` ومش أي حد بيشير ليها. Exception: لو `generate_symbols` مفعّل والـ node عنده labels، بتبقيه.

---

### Group 5: Address & Size Cells Checks

---

#### `fixup_addr_size_cells`

```c
static void fixup_addr_size_cells(struct check *c, struct dt_info *dti,
                                   struct node *node)
```

**ما تعمله:** بتقرأ `#address-cells` و`#size-cells` من الـ node وبتخزنهم في `node->addr_cells` و`node->size_cells`. لو مش موجودين بتخلّيهم -1 (للـ default detection لاحقاً).

**Key details:** ده check بـ side-effects مش validation. بيُشغَّل كـ prerequisite لكل الـ checks اللي محتاجة addr/size cells values. الـ macros `node_addr_cells(n)` و`node_size_cells(n)` بتعمل fallback: -1 → 2 و-1 → 1 على التوالي (DP defaults).

---

#### `check_reg_format`

```c
static void check_reg_format(struct check *c, struct dt_info *dti,
                              struct node *node)
```

**ما تعمله:** بتتحقق إن طول الـ `reg` property هو مضاعف صحيح لـ `(addr_cells + size_cells) * 4`. يعني لو الـ parent عنده `#address-cells = 2` و`#size-cells = 1`، فكل entry لازم تكون 12 bytes.

**Key details:**
- بتتجاهل الـ root node (مفيهاش reg)
- بتفشل لو `prop->val.len == 0`
- بيعتمد على `addr_size_cells` prerequisite

---

#### `check_ranges_format`

```c
static void check_ranges_format(struct check *c, struct dt_info *dti,
                                 struct node *node)
```

**ما تعمله:** بتتحقق من صحة الـ `ranges` أو `dma-ranges` property. كل entry لازم تكون `(parent_addr + child_addr + child_size) * 4` bytes. الـ edge case: empty ranges property مقبولة بس لازم الـ addr/size cells تتطابق مع الـ parent.

**Parameters:**
- `c->data` — اسم الـ property (`"ranges"` أو `"dma-ranges"`)

---

### Group 6: Bus-Specific Checks

كل bus type عنده من function لـ bridge detection ومن function لـ reg validation.

---

#### `check_pci_bridge`

```c
static void check_pci_bridge(struct check *c, struct dt_info *dti,
                              struct node *node)
```

**ما تعمله:** بتتعرف على الـ PCI bridge nodes عبر `device_type = "pci"` وبتتحقق من:
- اسم الـ node يبدأ بـ `pci` أو `pcie`
- وجود `ranges` property
- `#address-cells == 3` (PCI address space encoding)
- `#size-cells == 2`
- `bus-range` property صحيحة (2 cells، first ≤ second، max ≤ 0xff)

**Key details:** بتعين `node->bus = &pci_bus` كـ side effect — الـ child checks بتستخدمه للـ detection.

---

#### `check_pci_device_reg`

```c
static void check_pci_device_reg(struct check *c, struct dt_info *dti,
                                  struct node *node)
```

**ما تعمله:** للـ nodes تحت PCI bus، بتتحقق من صيغة الـ `reg` property وفق PCI address encoding. Cells[1] وCells[2] لازم تكون 0 (config space). بتحسب `dev` و`func` من cells[0] وبتتحقق إن الـ unit-address يطابق `"dev"` أو `"dev,func"`.

**Key details:** PCI config space encoding: bits[25:24] = space type (00 = config)، bits[18:16] = function، bits[22:19] = device.

---

#### `node_is_compatible`

```c
static bool node_is_compatible(struct node *node, const char *compat)
```

**ما تعمله:** بتفحص `compatible` property وبتدور على كل string في الـ list وبترجع `true` لو `compat` موجود.

---

#### `check_i2c_bus_bridge`

```c
static void check_i2c_bus_bridge(struct check *c, struct dt_info *dti,
                                  struct node *node)
```

**ما تعمله:** بتكشف I2C bus controllers. Node اسمه `i2c-bus` أو `i2c-arb` → يُعيَّن فوراً. Node اسمه `i2c` → يُفحص لو مفيش child اسمه `i2c-bus`. بتتحقق من `#address-cells == 1` و`#size-cells == 0`.

**Key details:** الـ I2C slave check بيفحص لو العنوان > 0x7f بدون `I2C_TEN_BIT_ADDRESS` flag أو > 0x3ff معاه.

---

#### `check_spi_bus_bridge`

```c
static void check_spi_bus_bridge(struct check *c, struct dt_info *dti,
                                  struct node *node)
```

**ما تعمله:** بتكشف SPI bus controllers. لو اسم الـ node مش `spi` بتدور على الـ children وبتشوف لو في properties تبدأ بـ `spi-` (heuristic). الـ `spi-slave` property بتخلّي `spi_addr_cells = 0`.

---

### Group 7: Semantic Property Checks

---

#### `check_names_is_string_list`

```c
static void check_names_is_string_list(struct check *c, struct dt_info *dti,
                                        struct node *node)
```

**ما تعمله:** بتمشي على كل الـ properties وبتلاقي اللي اسمها بيخلص بـ `-names`. بتتحقق إن كل واحدة منهم string list. بيغطي `clock-names`, `dma-names`, `power-supply-names`, إلخ.

---

#### `check_alias_paths`

```c
static void check_alias_paths(struct check *c, struct dt_info *dti,
                               struct node *node)
```

**ما تعمله:** للـ `/aliases` node بس، بتتحقق إن:
- كل property value تشير لـ path حقيقي في الـ tree
- أسماء الـ properties lowercase letters وأرقام وـ `-` فقط

**Key details:** بتتخطى الـ `phandle` و`linux,phandle` properties في الـ aliases node. في overlays (`DTSF_PLUGIN`) بتتخطى الـ path check.

---

#### `check_avoid_default_addr_size`

```c
static void check_avoid_default_addr_size(struct check *c, struct dt_info *dti,
                                           struct node *node)
```

**ما تعمله:** لو الـ node عنده `reg` أو `ranges` والـ parent لم يُحدد `#address-cells` أو `#size-cells` (قيمتهم -1)، بتطبع warning. الاعتماد على الـ defaults (2 و1) مش best practice لأنه ضمني.

---

#### `check_unique_unit_address_common`

```c
static void check_unique_unit_address_common(struct check *c,
                                              struct dt_info *dti,
                                              struct node *node,
                                              bool disable_check)
```

**ما تعمله:** بتتحقق إن كل children في node لهم unit-addresses مختلفة. لو `disable_check == true` بتتجاهل الـ disabled nodes (status = "disabled"). ده مهم لأنه ممكن يكون في منفذين لنفس العنوان لكن واحد بس enabled.

**Key details:** O(n²) في عدد الـ children. بتتجاهل الـ nodes اللي مفيهاش unit-address (`strlen(addr_a) == 0`).

---

### Group 8: Provider & Phandle-Args Checks

هذه المجموعة بتتحقق من الـ properties اللي بتشير لـ hardware providers (clocks, interrupts, GPIOs, etc) وبتتأكد إن عدد الـ args الصح موجود لكل provider.

---

#### `check_property_phandle_args`

```c
static void check_property_phandle_args(struct check *c,
                                         struct dt_info *dti,
                                         struct node *node,
                                         struct property *prop,
                                         const struct provider *provider)
```

**ما تعمله:** ده الـ core helper للـ provider checks. بيمشي على الـ property cells: بيقرأ phandle، بيجيب الـ provider node، بيقرأ الـ cell count property (مثلاً `#clock-cells`)، وبيتقدم `cellsize + 1` cells. بيتحقق من الـ property size في كل step.

**Parameters:**
- `prop` — الـ property اللي فيها الـ phandle+args pairs
- `provider` — struct فيه `prop_name`، `cell_name` (مثلاً `"#clock-cells"`)، `optional`

**Key details:**
- Cells بقيمة 0 أو ~0 (invalid phandle) → تُتخطى بدون error (بعض الـ bindings بتستخدمها كـ placeholder)
- لو في markers، بيتحقق إن الـ cell الحالي فعلاً `REF_PHANDLE` marker
- `provider->optional == true` معناه لو مفيش `cell_name` property على الـ provider، نفترض 0 cells

---

#### `check_provider_cells_property`

```c
static void check_provider_cells_property(struct check *c,
                                           struct dt_info *dti,
                                           struct node *node)
```

**ما تعمله:** wrapper بسيط — بيجيب الـ property المحددة في `provider->prop_name` ويشغّل `check_property_phandle_args` عليها. ده الـ `check_fn` المستخدم في `WARNING_PROPERTY_PHANDLE_CELLS` macro.

---

#### `check_gpios_property`

```c
static void check_gpios_property(struct check *c, struct dt_info *dti,
                                  struct node *node)
```

**ما تعمله:** بتمشي على كل الـ properties وبتدور على الـ GPIO properties (اللي أسمائها `gpios`، `*-gpios`، `gpio`، `*-gpio` وليس `,nr-gpios`). لكل property بتشغّل `check_property_phandle_args` مع `#gpio-cells`.

**Key details:** بتتخطى الـ gpio-hog nodes لأنهم بيستخدموا `gpios` property بمعنى مختلف.

---

#### `prop_is_gpio`

```c
static bool prop_is_gpio(struct property *prop)
```

**ما تعمله:** helper بيحدد إن property هي GPIO property أم لا. `strends(prop->name, ",nr-gpios")` هو الـ false positive الوحيد المعروف اللي لازم يُستثنى.

---

#### `check_interrupt_provider`

```c
static void check_interrupt_provider(struct check *c, struct dt_info *dti,
                                      struct node *node)
```

**ما تعمله:** بتتحقق من consistency: لو الـ node provider للـ interrupts (عنده `interrupt-controller` أو `interrupt-map`)، لازم عنده `#interrupt-cells`. والعكس: لو عنده `#interrupt-cells` بدون كونه provider → error.

---

#### `check_interrupt_map`

```c
static void check_interrupt_map(struct check *c, struct dt_info *dti,
                                 struct node *node)
```

**ما تعمله:** من أعقد الـ checks — بتتحقق من صحة `interrupt-map` property بالكامل. كل entry في الـ map: `[child-addr-cells][#interrupt-cells][parent-phandle][parent-addr-cells][parent-#interrupt-cells]`.

**Pseudocode flow:**
```
for each entry in interrupt-map:
  skip (addr_cells + irq_cells) child specifier cells
  read phandle
  lookup provider_node
  read #interrupt-cells from provider
  read #address-cells from provider (default 0)
  advance cell by (1 + parent_addr_cells + parent_irq_cells)
```

**Key details:**
- بتتحقق أولاً إن `#address-cells` موجود على الـ node نفسه (مش الـ parent)
- `interrupt-map-mask` لازم حجمها = `(addr_cells + #interrupt-cells) * 4`

---

#### `check_interrupts_property`

```c
static void check_interrupts_property(struct check *c, struct dt_info *dti,
                                       struct node *node)
```

**ما تعمله:** بتتحقق من الـ `interrupts` property. بتصعد شجرة الـ parent لتلاقي الـ interrupt provider (عبر `interrupt-parent` phandle أو بالـ topology). بتتحقق إن طول الـ `interrupts` property مضاعف لـ `#interrupt-cells * 4`.

---

### Group 9: Graph Checks (V4L2/media)

الـ graph bindings بتستخدم `port` و`endpoint` nodes لتمثيل الـ media links. هذه الـ checks بتتحقق من صحة هذه البنية.

---

#### `check_graph_nodes`

```c
static void check_graph_nodes(struct check *c, struct dt_info *dti,
                               struct node *node)
```

**ما تعمله:** بتكشف graph port nodes — لو الـ node عنده child اسمه `endpoint` أو عنده `remote-endpoint` property. بتعيّن `node->bus = &graph_port_bus`. بتكشف كمان graph ports containers (`ports` node أو node عنده reg).

---

#### `check_graph_reg`

```c
static void check_graph_reg(struct check *c, struct dt_info *dti,
                             struct node *node)
```

**ما تعمله:** للـ graph nodes (ports وendpoints)، بتتحقق إن الـ `reg` property هي cell واحدة وإن الـ unit-address يطابق قيمتها. بتتحقق كمان إن الـ parent عنده `#address-cells == 1` و`#size-cells == 0`.

---

#### `get_remote_endpoint`

```c
static struct node *get_remote_endpoint(struct check *c, struct dt_info *dti,
                                         struct node *endpoint)
```

**ما تعمله:** helper بيقرأ `remote-endpoint` phandle ويجيب الـ node المقصود. بيرجع `NULL` لو الـ property مش موجودة أو الـ phandle invalid (overlay case) أو الـ node مش موجود.

---

#### `check_graph_endpoint`

```c
static void check_graph_endpoint(struct check *c, struct dt_info *dti,
                                  struct node *node)
```

**ما تعمله:** للـ endpoint nodes (children لـ graph port nodes)، بتتحقق من:
- اسم الـ node يبدأ بـ `endpoint`
- الـ `remote-endpoint` يشير لـ node حقيقي
- الـ connection **bidirectional**: الـ remote node لازم يشير للـ node ده بالـ `remote-endpoint`

**Key details:** بتشيل الـ graph checks في overlays (`DTSF_PLUGIN`) لأن الـ remote endpoint ممكن يكون external.

---

### ملاحظات هامة على الـ Macros

#### `CHECK_ENTRY` Family

```c
#define CHECK_ENTRY(nm_, fn_, d_, w_, e_, ...) \
    static struct check *nm_##_prereqs[] = { __VA_ARGS__ }; \
    static struct check nm_ = { \
        .name = #nm_, \
        .fn = (fn_), \
        .data = (d_), \
        .warn = (w_), .error = (e_), \
        .status = UNCHECKED, \
        .num_prereqs = ARRAY_SIZE(nm_##_prereqs), \
        .prereq = nm_##_prereqs, \
    };
```

- `WARNING(nm, fn, d, ...)` — `warn=true, error=false`
- `ERROR(nm, fn, d, ...)` — `warn=false, error=true`
- `CHECK(nm, fn, d, ...)` — `warn=false, error=false` (silent by default, قابل للتفعيل بـ CLI)

الـ `__VA_ARGS__` بيتحول لـ array من `struct check *` prerequisites. الـ `ARRAY_SIZE` بيحسب عددهم automatically.

#### `FAIL` و`FAIL_PROP`

```c
#define FAIL(c, dti, node, ...) \
    do { \
        TRACE((c), "\t\tFAILED at %s:%d", __FILE__, __LINE__); \
        (c)->status = FAILED; \
        check_msg((c), dti, node, NULL, __VA_ARGS__); \
    } while (0)
```

الفرق الوحيد: `FAIL_PROP` بيمرر `prop` لـ `check_msg` بدل `NULL` — ده بيعمل الـ source location أدق (level الـ property مش الـ node).

#### `WARNING_PROPERTY_PHANDLE_CELLS`

```c
#define WARNING_PROPERTY_PHANDLE_CELLS(nm, propname, cells_name, ...) \
    static struct provider nm##_provider = { (propname), (cells_name), __VA_ARGS__ }; \
    WARNING_IF_NOT_CELL(nm##_is_cell, cells_name); \
    WARNING(nm##_property, check_provider_cells_property, \
            &nm##_provider, &nm##_is_cell, &phandle_references);
```

ده macro بيولّد automatically 2 checks لكل provider type: check إن الـ `#xxx-cells` property هي cell واحدة، وcheck إن الـ consumer property صحيحة. بيُطبَّق على: clocks، dmas، phys، resets، pwms، iommus، mboxes، وغيرهم.
## Phase 5: دليل الـ Debugging الشامل

> **السياق:** الملف `scripts/dtc/checks.c` هو محرك الـ **semantic validation** في الـ Device Tree Compiler (dtc). بيشغّل سلسلة من الـ checks على الـ DT tree بعد الـ parsing — كل check عبارة عن `struct check` بيتنفذ recursively على كل node. الـ errors والـ warnings بتتطبع على `stderr` مع اسم الـ check ومكانه في الـ source file.

---

### Software Level

#### 1. debugfs entries

الـ dtc مش kernel module — هو userspace tool، فمفيش debugfs entries مباشرةً ليه. لكن لو شغّلت الـ DTB على kernel، فيه entries مفيدة:

```bash
# قراءة الـ DT المحمّل حالياً من الـ kernel
ls /sys/firmware/devicetree/base/

# شوف كل property في node معين
cat /sys/firmware/devicetree/base/chosen/bootargs

# الـ overlay application errors
cat /sys/kernel/debug/of/resolver
```

| debugfs path | المحتوى |
|---|---|
| `/sys/kernel/debug/of/` | live device tree state |
| `/sys/kernel/debug/of/resolver` | phandle resolution errors |
| `/sys/firmware/devicetree/base/` | الـ DTB المحمّل كـ sysfs tree |

#### 2. sysfs entries

```bash
# تحقق من compatibility الـ node
cat /sys/bus/platform/devices/<device>/of_node/compatible

# شوف الـ reg property
xxd /sys/firmware/devicetree/base/<path>/reg

# شوف status property
cat /sys/firmware/devicetree/base/<path>/status

# list كل nodes في الـ DT
find /sys/firmware/devicetree/base -name "compatible" | xargs grep -l "simple-bus"
```

#### 3. ftrace — tracepoints للـ DT subsystem

```bash
# enable الـ OF (Open Firmware) tracepoints
echo 1 > /sys/kernel/debug/tracing/events/of/enable

# trace الـ phandle resolution
echo 1 > /sys/kernel/debug/tracing/events/of/of_find_node_by_phandle/enable

# trace الـ device probing من الـ DT
echo 1 > /sys/kernel/debug/tracing/events/device/enable

# قراءة الـ trace
cat /sys/kernel/debug/tracing/trace
```

```bash
# trace dtc كـ process (userspace)
strace -e openat,read,write dtc -I dts -O dtb input.dts -o output.dtb 2>&1 | grep -E "ERROR|Warning"
```

#### 4. printk / dynamic debug

الـ dtc بيطبع على `stderr` مباشرةً — مش بيستخدم kernel printk. لكن لو الـ DT بيتحمّل في kernel:

```bash
# enable dynamic debug للـ OF subsystem
echo "module of_fdt +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/of/*.c +p" > /sys/kernel/debug/dynamic_debug/control

# شوف نتيجة الـ DT validation عند boot
dmesg | grep -E "OF:|DT:|devicetree|phandle|#address-cells"
```

لتفعيل `TRACE_CHECKS` في الـ dtc نفسه (compile-time):
```bash
# أضف للـ Makefile أو CFLAGS
make CFLAGS="-DTRACE_CHECKS" scripts/dtc/
```

ده هيخلي كل check يطبع على stderr:
```
=== duplicate_node_names: /soc/uart@1234
=== phandle_references: /cpus/cpu@0
```

#### 5. Kernel config options للـ debugging

| Config | الغرض |
|---|---|
| `CONFIG_OF` | تفعيل دعم الـ Open Firmware / Device Tree |
| `CONFIG_OF_FLATTREE` | دعم الـ FDT (Flattened Device Tree) |
| `CONFIG_OF_OVERLAY` | دعم الـ DT overlays |
| `CONFIG_OF_UNITTEST` | unit tests للـ OF subsystem |
| `CONFIG_OF_KOBJ` | expose الـ DT كـ kobjects في sysfs |
| `CONFIG_OF_DYNAMIC` | تغيير الـ DT ديناميكياً وقت الـ runtime |
| `CONFIG_DEBUG_OF` | تفعيل extra logging في OF subsystem |
| `CONFIG_PROC_DEVICETREE` | expose الـ DT عبر `/proc/device-tree` (legacy) |
| `CONFIG_DTC` | بناء الـ dtc كجزء من الـ kernel build |

```bash
# تحقق من الـ config الحالي
grep -E "CONFIG_OF|CONFIG_DTC|CONFIG_DEBUG_OF" /boot/config-$(uname -r)
```

#### 6. أدوات خاصة بالـ subsystem

```bash
# dtc نفسه: تشغيل كل الـ checks مع verbose output
dtc -I dts -O dtb -W all -E all input.dts -o output.dtb

# تفعيل check معين كـ warning
dtc -W phandle_references input.dts -o /dev/null

# تعطيل check معين
dtc -W no-unit_address_vs_reg input.dts -o /dev/null

# تحويل DTB → DTS للفحص اليدوي
dtc -I dtb -O dts /boot/dtb-$(uname -r).dtb -o /tmp/current.dts

# فحص الـ DTB المحمّل حالياً
dtc -I fs -O dts /sys/firmware/devicetree/base

# fdtdump: dump الـ DTB
fdtdump /boot/dtb-$(uname -r).dtb | less

# fdtget/fdtput: قراءة/تعديل properties
fdtget /boot/dtb.dtb /cpus/cpu@0 compatible
fdtput -t s test.dtb /node compatible "vendor,chip"

# python-devicetree (لو متاحة)
pip install dtschema
dt-validate -s /path/to/bindings/ input.dts
```

#### 7. جدول رسائل الـ errors الشائعة

| رسالة الـ Error | الـ Check المسبّب | السبب | الحل |
|---|---|---|---|
| `Duplicate node name` | `duplicate_node_names` | نودين بنفس الاسم تحت نفس الـ parent | غيّر اسم واحد منهم أو أضف unit address |
| `Duplicate property name` | `duplicate_property_names` | property مكررة في نفس الـ node | احذف التكرار |
| `property is not a string` | `check_is_string` | property كـ `compatible` مش null-terminated | استخدم `"value"` بدل `<0x...>` |
| `property is not a single cell` | `check_is_cell` | `#address-cells` مش 4 bytes | استخدم `<1>` أو `<2>` |
| `Bad character '%c' in node name` | `node_name_chars` | حرف غير مسموح في اسم الـ node | استخدم فقط `[a-zA-Z0-9,._+-@]` |
| `node has a reg or ranges property, but no unit name` | `unit_address_vs_reg` | الـ node عنده `reg` بس مفيش `@addr` في اسمه | أضف `@address` لاسم الـ node |
| `node has a unit name, but no reg or ranges property` | `unit_address_vs_reg` | العكس — اسم فيه `@` بس مفيش `reg` | أضف `reg` أو احذف الـ unit address |
| `Duplicate label '%s'` | `duplicate_label` | label اتعرّف في أكتر من مكان | اجعل كل label فريد في الـ DTS |
| `duplicated phandle 0x%x` | `explicit_phandles` | قيمة phandle اتعرّفت مرتين | لا تعيّن phandle يدوياً إلا لو ضروري |
| `Reference to non-existent node or label` | `phandle_references` | `&label` بيشير لـ label مش موجودة | تأكد إن الـ label معرّفة في نفس الـ DTS أو الـ base DTB |
| `missing ranges for PCI bridge` | `pci_bridge` | node بيه `device_type = "pci"` بدون `ranges` | أضف `ranges` property للـ PCI bridge |
| `incorrect #address-cells for I2C bus` | `i2c_bus_bridge` | I2C bus محتاج `#address-cells = <1>` | صحّح القيمة |
| `I2C address must be less than 7-bits` | `i2c_bus_reg` | عنوان I2C > 0x7F بدون I2C_TEN_BIT_ADDRESS flag | استخدم `(I2C_TEN_BIT_ADDRESS \| addr)` أو صحّح العنوان |
| `Missing '#interrupt-cells' in interrupt provider` | `interrupt_provider` | node عنده `interrupt-controller` بدون `#interrupt-cells` | أضف `#interrupt-cells = <N>` |
| `graph connection to node '%s' is not bidirectional` | `graph_endpoint` | `remote-endpoint` مش بيشير على بعض | تأكد إن كل endpoint بيشير على الـ endpoint التاني |
| `"name" property is incorrect` | `name_properties` | الـ `name` property مش مطابقة لاسم الـ node | احذف الـ `name` property — الـ dtc بيحسبها أوتوماتيك |
| `aliases property is not a valid node` | `alias_paths` | الـ alias بيشير لـ path مش موجود | تأكد إن الـ path موجود في الـ DTS |
| `Relying on default #address-cells value` | `avoid_default_addr_size` | parent مفيش `#address-cells` | أضف `#address-cells` و`#size-cells` صراحةً |
| `Failed prerequisite '%s'` | `run_check` | check prerequisite فشل | صلّح الـ prerequisite check الأول |

#### 8. أماكن استراتيجية لـ dump_stack() / WARN_ON()

في الـ dtc (userspace) مفيش `dump_stack()` بالمعنى الكرنل. لكن في الـ kernel OF code:

```c
/* في drivers/of/base.c — لو phandle resolution فشل */
node = of_find_node_by_phandle(phandle);
WARN_ON(!node);  /* هنا لو الـ DTB corrupt */

/* في drivers/of/fdt.c — لو الـ FDT magic number غلط */
if (fdt_magic(blob) != FDT_MAGIC) {
    WARN(1, "Invalid FDT magic\n");
    /* dump_stack() هنا مفيد لمعرفة من استدعى الـ DT loader */
    dump_stack();
}

/* في checks.c نفسه — لو أضفت debugging */
assert(!c->inprogress);  /* ده بيوقف لو في circular dependency بين الـ checks */
```

نقاط حرجة للـ `WARN_ON` في kernel:
- بعد `get_node_by_phandle()` لو المفروض يرجع node معروف
- في `fixup_phandle_references()` لو الـ offset تجاوز الـ property length
- في `check_reg_format()` لو `addr_cells + size_cells == 0`

---

### Hardware Level

#### 1. التحقق إن الـ hardware state مطابق للـ kernel state

```bash
# قارن الـ reg property في DT مع الـ physical address من datasheet
fdtget /boot/dtb.dtb /soc/uart@10000000 reg
# يرجع: 0x10000000 0x1000 (base address + size)

# تحقق إن الـ kernel map الـ address فعلاً
cat /proc/iomem | grep -i uart

# تأكد من الـ interrupt number
fdtget /boot/dtb.dtb /soc/uart@10000000 interrupts
# قارن مع
cat /proc/interrupts | grep uart
```

#### 2. Register Dump Techniques

```bash
# باستخدام devmem2 — قراءة register من physical address
devmem2 0x10000000 w        # قراءة 32-bit من base address الـ UART
devmem2 0x10000004 w        # قراءة الـ register التاني

# /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((0x10000000/4)) 2>/dev/null | xxd

# io utility (من package ioport)
io -4 -r 0x10000000

# قراءة block من registers
devmem2 0x10000000 w; devmem2 0x10000004 w; devmem2 0x10000008 w
```

**مثال على الـ output وتفسيره:**
```bash
$ devmem2 0x44e09000 w
/dev/mem opened.
Memory mapped at address 0xb6f8e000.
Value at address 0x44E09000 (0xb6f8e000): 0x00000021
# 0x21 = UART_LSR: TX empty (bit 5) + Transmitter Shift Register empty (bit 6)
# لو رجع 0xFFFFFFFF: الـ address مش mapped أو الـ peripheral مش enabled
```

#### 3. Logic Analyzer / Oscilloscope Tips

لـ debugging الـ bus protocols المذكورة في checks.c (I2C, SPI, PCI):

**I2C (check_i2c_bus_reg):**
- اتحقق إن الـ SCL/SDA lines بيشتغلوا بالـ frequency الصح
- لو الـ dtc اشتكى من عنوان > 0x7F: verify إن الـ device فعلاً بيدعم 10-bit addressing
- ابحث عن ACK/NACK — NACK يعني العنوان في الـ DT غلط

**SPI (check_spi_bus_reg):**
- الـ chip select number في الـ DT reg property لازم يطابق الـ CS line الفعلي
- تحقق من CPOL/CPHA settings

**PCI (check_pci_bridge, check_pci_device_reg):**
- الـ configuration space address في reg[0] لازم يكون `0x00SSDDFF00` (S=bus, D=dev, F=func)
- لو الـ check اشتكى من bus number: تحقق من الـ physical bus topology مع `lspci -tv`

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في kernel log | الـ check المرتبط |
|---|---|---|
| الـ peripheral مش موجود بالـ address ده | `OF: ERROR: Bad cell count for /soc/uart@1234` | `reg_format` |
| الـ I2C device مش بيرد | `i2c i2c-0: Device not found at 0x50` | `i2c_bus_reg` |
| الـ interrupt line مش متوصل | `irq: no irq domain found for /soc/gpio` | `interrupts_property` |
| الـ clock source مش available | `clk: Failed to get 'uart' clock` | `clocks_property` |
| الـ phandle بيشير لـ node غلط | `OF: Bad phandle 0x5 in /cpus/cpu@0:clocks[0]` | `phandle_references` |

```bash
# فلترة الـ DT-related kernel messages
dmesg | grep -E "OF:|DT:|of_|fdt_|devicetree|phandle" | head -50

# شوف الـ deferred probing (غالباً بسبب DT dependency مش resolved)
cat /sys/kernel/debug/devices_deferred
```

#### 5. Device Tree Debugging — التحقق إن الـ DT مطابق للـ Hardware

```bash
# قارن الـ DTS source مع الـ DTB المحمّل
dtc -I dtb -O dts /boot/my-board.dtb -o /tmp/from-dtb.dts
diff original.dts /tmp/from-dtb.dts

# تحقق من الـ #address-cells و #size-cells في hierarchy
fdtget /boot/dtb.dtb /soc "#address-cells"
fdtget /boot/dtb.dtb /soc "#size-cells"
# لو الـ values غلط: check_reg_format و check_ranges_format هيشتكوا

# تحقق من الـ compatible strings مقابل الـ drivers المتاحة
fdtget /boot/dtb.dtb /soc/uart@10000000 compatible
# ثم
ls /sys/bus/platform/drivers/ | grep uart

# تحقق من الـ phandle references
fdtget /boot/dtb.dtb /cpus/cpu@0 clocks  # يرجع phandle value
# ثم اعرف الـ node المشار إليه
fdtget /boot/dtb.dtb / -p | xargs -I{} sh -c 'h=$(fdtget /boot/dtb.dtb /{} phandle 2>/dev/null); [ "$h" = "0x<N>" ] && echo {}'

# التحقق من overlay application
cat /sys/kernel/config/device-tree/overlays/*/status
```

---

### Practical Commands

#### مجموعة كاملة من الأوامر الجاهزة للنسخ

**1. تشغيل الـ dtc مع كل الـ checks:**
```bash
# تحويل DTS → DTB مع كل الـ warnings كـ errors
dtc -I dts -O dtb -W all -E all -f myboard.dts -o myboard.dtb 2>&1 | tee dtc-errors.log

# تشغيل check محدد فقط
dtc -I dts -O dtb -W phandle_references myboard.dts -o /dev/null

# تحقق من DTS بدون output
dtc -I dts -O null -W all myboard.dts
```

**تفسير الـ output:**
```
arch/arm64/boot/dts/myboard.dts:42.1-10: Warning (unit_address_vs_reg): \
  /soc/i2c@ff160000/sensor@48: node has a unit name, but no reg or ranges property
#  ^-- file:line          ^-- check name              ^-- node path
```

**2. dump الـ DTB المحمّل وفحصه:**
```bash
# تحويل الـ live DT إلى DTS
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null > /tmp/live-dt.dts

# مقارنة سريعة
diff <(dtc -I dtb -O dts original.dtb 2>/dev/null) \
     <(dtc -I fs  -O dts /sys/firmware/devicetree/base 2>/dev/null)
```

**3. فحص phandle references:**
```bash
# اعرف كل الـ phandles الموجودة
fdtdump myboard.dtb 2>/dev/null | grep "phandle ="

# تحقق من reference محددة
fdtget myboard.dtb /soc/clk@10000 "#clock-cells"

# فحص interrupt hierarchy
fdtget myboard.dtb /soc/uart@10000000 interrupts
fdtget myboard.dtb /soc/uart@10000000 interrupt-parent
```

**4. تفعيل الـ TRACE_CHECKS وقت الـ build:**
```bash
cd linux/scripts/dtc
make clean
make CFLAGS="-DTRACE_CHECKS -g" dtc

# الـ output هيكون:
# === duplicate_node_names: /soc/i2c@ff160000
# === phandle_references: /soc/i2c@ff160000
#     FAILED at checks.c:628
```

**5. فحص الـ I2C bus addresses:**
```bash
# تحقق من كل I2C addresses في الـ DT
fdtget -l /boot/dtb.dtb /soc/i2c@ff160000 | while read node; do
  addr=$(fdtget /boot/dtb.dtb /soc/i2c@ff160000/$node reg 2>/dev/null)
  echo "$node: 0x$addr"
done

# قارن مع الـ i2cdetect
i2cdetect -y 0  # scan الـ I2C bus فعلياً
# لو الـ address في DT موجود هنا = hardware ومطابق
# لو مش موجود = مشكلة hardware أو address غلط في DT
```

**6. تفعيل الـ kernel OF debugging:**
```bash
# kernel cmdline
# أضف: of_debug=1

# أو بعد الـ boot
echo 8 > /proc/sys/kernel/printk  # max verbosity
echo "file drivers/of/*.c +pflm" > /sys/kernel/debug/dynamic_debug/control

# شوف النتيجة
dmesg | grep "of_"
```

**7. فحص graph nodes (الـ media subsystem):**
```bash
# تحقق من bidirectional remote-endpoint connections
fdtget myboard.dtb /csi/port@0/endpoint@0 remote-endpoint
fdtget myboard.dtb /isp/port@0/endpoint@0 remote-endpoint
# كل واحد لازم يشير على التاني

# شوف media topology
media-ctl -d /dev/media0 --print-topology
```

**8. فحص complete check status بعد الـ dtc run:**
```bash
# لو عندك dtc مبني بـ TRACE_CHECKS
dtc -I dts -O dtb input.dts -o output.dtb 2>&1 | grep -E "FAILED|PASSED|PREREQ"

# عدد الـ errors والـ warnings
dtc -I dts -O null -W all -E all input.dts 2>&1 | \
  awk '/ERROR/{e++} /Warning/{w++} END{print "Errors:", e, "Warnings:", w}'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: RK3562 — Industrial Gateway — فشل compile الـ DTS بسبب `phandle` مكرر

#### العنوان
**Duplicate phandle** في DTS خاص بـ industrial gateway على RK3562

#### السياق
شركة تبني industrial IoT gateway على RK3562. الـ BSP engineer بيعدّل DTS الـ main board عشان يضيف custom I2C sensor hub. اتنين engineers شتغلوا على نفس الملف في نفس الوقت وكل واحد فيهم زود node بـ `phandle = <1>;` يدوي.

#### المشكلة
لما بيشغّل:
```bash
make dtbs
```
بيطلع error:
```
arch/arm64/boot/dts/rockchip/rk3562-gateway.dts: ERROR (explicit_phandles): \
/i2c2/sensor@48: duplicated phandle 0x1 (seen before at /i2c1/eeprom@50)
```
الكيرنل مش بيتبني خالص.

#### التحليل
الـ `check_explicit_phandles` في `checks.c` بتتنفّذ من `run_check` لكل node:

```c
/* checks.c: check_explicit_phandles */
static void check_explicit_phandles(struct check *c, struct dt_info *dti,
                                    struct node *node)
{
    struct node *root = dti->dt;
    struct node *other;
    cell_t phandle, linux_phandle;

    /* Nothing should have assigned phandles yet */
    assert(!node->phandle);

    phandle = check_phandle_prop(c, dti, node, "phandle");
    linux_phandle = check_phandle_prop(c, dti, node, "linux,phandle");

    /* ... */

    other = get_node_by_phandle(root, phandle);
    if (other && (other != node)) {
        FAIL(c, dti, node, "duplicated phandle 0x%x (seen before at %s)",
             phandle, other->fullpath);  /* <-- هنا بييجي الـ error */
        return;
    }

    node->phandle = phandle;
}
ERROR(explicit_phandles, check_explicit_phandles, NULL);
```

الـ check دي مسجّلة كـ `ERROR` مش `WARNING`، فبالتالي `run_check` بيرجع `true` وفي `process_checks`:

```c
void process_checks(bool force, struct dt_info *dti)
{
    /* ... */
    if (error) {
        if (!force) {
            fprintf(stderr, "ERROR: Input tree has errors, aborting "
                "(use -f to force output)\n");
            exit(2);  /* <-- عدم إكمال البناء */
        }
    }
}
```

#### الحل
الحل الصحيح: إزالة الـ `phandle` اليدوي وترك الـ DTC يعملها تلقائي:

```dts
/* قبل: غلط */
eeprom: eeprom@50 {
    compatible = "atmel,24c02";
    reg = <0x50>;
    phandle = <1>;  /* احذف السطر ده */
};

sensor_hub: sensor@48 {
    compatible = "vendor,sensor-hub";
    reg = <0x48>;
    phandle = <1>;  /* احذف السطر ده كمان */
};

/* بعد: صح */
eeprom: eeprom@50 {
    compatible = "atmel,24c02";
    reg = <0x50>;
    /* DTC هيحدد الـ phandle تلقائي من الـ label */
};

sensor_hub: sensor@48 {
    compatible = "vendor,sensor-hub";
    reg = <0x48>;
};
```

لو في مكان بيعمل reference:
```dts
nvmem-provider = <&eeprom>;
sensor = <&sensor_hub>;
```

#### الدرس المستفاد
الـ `phandle` اليدوي خطر في أي مشروع بيشتغل عليه أكتر من واحد. ابعد عنه خالص واستخدم الـ labels مع الـ `&reference`. الـ DTC بيخصص الـ phandles تلقائي من غير تعارض.

---

### السيناريو الثاني: STM32MP1 — IoT Sensor Board — Warning في `#address-cells` مش محدد

#### العنوان
**Relying on default `#address-cells`** في DTS خاص بـ IoT sensor board على STM32MP1

#### السياق
فريق يبني IoT sensor board على STM32MP157. الـ firmware engineer بيضيف SPI flash memory للـ datalogger. بيضيف الـ node بسرعة من غير ما يحدد `#address-cells` و`#size-cells` في الـ SPI controller node.

#### المشكلة
البناء بيعدّي بـ warnings:
```
stm32mp157-iot-board.dts: Warning (avoid_default_addr_size): \
/soc/spi@44004000/flash@0: Relying on default #address-cells value
stm32mp157-iot-board.dts: Warning (avoid_default_addr_size): \
/soc/spi@44004000/flash@0: Relying on default #size-cells value
```
الـ DTB اتبنى بس الـ kernel بيطلع warning في boot وفي بعض الكيرنل versions الـ SPI driver مش بيشوف الـ flash صح.

#### التحليل
الـ check `avoid_default_addr_size` في `checks.c`:

```c
static void check_avoid_default_addr_size(struct check *c, struct dt_info *dti,
                                          struct node *node)
{
    struct property *reg, *ranges;

    if (!node->parent)
        return; /* Ignore root node */

    reg = get_property(node, "reg");
    ranges = get_property(node, "ranges");

    if (!reg && !ranges)
        return;

    if (node->parent->addr_cells == -1)
        FAIL(c, dti, node, "Relying on default #address-cells value");
    /* -1 معناه مش محدد، شوف fixup_addr_size_cells */

    if (node->parent->size_cells == -1)
        FAIL(c, dti, node, "Relying on default #size-cells value");
}
WARNING(avoid_default_addr_size, check_avoid_default_addr_size, NULL,
        &addr_size_cells);
```

الـ `fixup_addr_size_cells` بتحط `-1` لو الـ property مش موجودة:

```c
static void fixup_addr_size_cells(struct check *c, struct dt_info *dti,
                                  struct node *node)
{
    node->addr_cells = -1;  /* default = غير محدد */
    node->size_cells = -1;

    prop = get_property(node, "#address-cells");
    if (prop)
        node->addr_cells = propval_cell(prop);
    /* ... */
}
```

الـ macro `node_addr_cells` بيستخدم قيمة default:
```c
#define node_addr_cells(n) \
    (((n)->addr_cells == -1) ? 2 : (n)->addr_cells)
#define node_size_cells(n) \
    (((n)->size_cells == -1) ? 1 : (n)->size_cells)
```

لكن الـ check الـ explicit بتحذّر إنك بتعتمد على الـ default بدل ما تحددها صراحة.

#### الحل
```dts
/* قبل: ناقص */
&spi4 {
    status = "okay";
    cs-gpios = <&gpiof 6 GPIO_ACTIVE_LOW>;

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <50000000>;
    };
};

/* بعد: صح */
&spi4 {
    status = "okay";
    cs-gpios = <&gpiof 6 GPIO_ACTIVE_LOW>;
    #address-cells = <1>;   /* أضف السطرين دول */
    #size-cells = <0>;      /* SPI devices: size-cells = 0 دايما */

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <50000000>;
    };
};
```

تحقق من النتيجة:
```bash
dtc -I dts -O dtb -W no-avoid_default_addr_size \
    stm32mp157-iot-board.dts -o /dev/null 2>&1
# يجب ما يطلعش أي warnings
```

#### الدرس المستفاد
الـ SPI bus دايمًا `#address-cells = <1>` و`#size-cells = <0>`. الـ I2C bus كمان نفس الشيء. الـ DTC بيحذّر لو اعتمدت على الـ default بدل ما تكتبها صراحة عشان الـ DTS يكون self-documenting ومش عارضة لتغيير الـ defaults.

---

### السيناريو الثالث: i.MX8MM — Android TV Box — خطأ في `reg` property مش متوافق مع `#address-cells`

#### العنوان
**`reg` property invalid length** في HDMI bridge node على i.MX8MM

#### السياق
شركة بتبني Android TV box على i.MX8MM. الـ display engineer بيضيف HDMI bridge chip (IT66121) متوصل بـ I2C. بيكتب الـ reg بطريقة غلط من BSP قديم كان بيستخدم `#address-cells = <2>`.

#### المشكلة
```
imx8mm-tv-box.dts: Warning (reg_format): \
/i2c2/hdmi-bridge@4c: property has invalid length (4 bytes) \
(#address-cells == 2, #size-cells == 0)
```
الـ HDMI bridge مش بيشتغل. الـ kernel مش لاقي الـ device.

#### التحليل
الـ `check_reg_format` في `checks.c`:

```c
static void check_reg_format(struct check *c, struct dt_info *dti,
                             struct node *node)
{
    struct property *prop;
    int addr_cells, size_cells, entrylen;

    prop = get_property(node, "reg");
    if (!prop)
        return;

    if (!node->parent) {
        FAIL(c, dti, node, "Root node has a \"reg\" property");
        return;
    }

    if (prop->val.len == 0)
        FAIL_PROP(c, dti, node, prop, "property is empty");

    addr_cells = node_addr_cells(node->parent);  /* قيمة من الـ parent */
    size_cells = node_size_cells(node->parent);
    entrylen = (addr_cells + size_cells) * sizeof(cell_t);  /* bytes مطلوبة */

    if (!is_multiple_of(prop->val.len, entrylen))
        FAIL_PROP(c, dti, node, prop,
            "property has invalid length (%d bytes) "
            "(#address-cells == %d, #size-cells == %d)",
            prop->val.len, addr_cells, size_cells);
        /* 4 bytes != (2+0)*4 = 8 bytes */
}
WARNING(reg_format, check_reg_format, NULL, &addr_size_cells);
```

الـ parent node (i2c2) عنده `#address-cells = <2>` من BSP قديم بيدعم 10-bit I2C. بس الـ reg للـ bridge بس `<0x4c>` = 4 bytes. المطلوب 8 bytes = `<0x00 0x4c>`.

#### الحل
```bash
# أولاً: اعرف القيم الحالية
dtc -I dtb -O dts /boot/imx8mm-tv-box.dtb | grep -A5 "i2c2"
```

```dts
/* قبل: غلط (legacy BSP) */
&i2c2 {
    clock-frequency = <400000>;
    #address-cells = <2>;   /* قديم وغلط لـ I2C عادي */
    #size-cells = <0>;

    hdmi_bridge: hdmi-bridge@4c {
        compatible = "ite,it66121";
        reg = <0x4c>;   /* 4 bytes بس، مش متوافق مع #address-cells=2 */
    };
};

/* بعد: صح */
&i2c2 {
    clock-frequency = <400000>;
    #address-cells = <1>;   /* I2C standard: 1 cell */
    #size-cells = <0>;

    hdmi_bridge: hdmi-bridge@4c {
        compatible = "ite,it66121";
        reg = <0x4c>;   /* الآن صح: 4 bytes = 1 cell */
        interrupts-extended = <&gpio1 9 IRQ_TYPE_LEVEL_LOW>;
    };
};
```

التحقق:
```bash
make dtbs 2>&1 | grep -i "reg_format\|address-cells"
# يجب ما يطلعش أي errors
```

#### الدرس المستفاد
**الـ I2C bus دايمًا `#address-cells = <1>` و`#size-cells = <0>`**. الـ DTC بيتحقق إن حجم الـ `reg` property متوافق مع عدد الـ cells المحددة في الـ parent. لو ورثت BSP قديم، اتحقق من `#address-cells` في كل bus node.

---

### السيناريو الرابع: AM62x — Automotive ECU — `compatible` مش string list

#### العنوان
**`compatible` property غلط format** في custom sensor node على AM62x

#### السياق
فريق automotive بيبني ECU على TI AM62x. الـ junior engineer بيضيف node لـ CAN transceiver متوصل بـ SPI. بيكتب الـ `compatible` property بصيغة binary cells بدل strings بسبب سوء فهم.

#### المشكلة
```
am62x-ecu.dts: Warning (compatible_is_string_list): \
/soc/spi@2000000/can-transceiver@0: property 'compatible' \
is not a string list
```
بالإضافة لـ:
```
am62x-ecu.dts: Warning (spi_bus_reg): /soc/spi@2000000/can-transceiver@0: \
SPI bus unit address format error, expected "0"
```

الـ kernel مش عارف يـmatch الـ driver للـ device.

#### التحليل
الـ `compatible_is_string_list` في `checks.c`:

```c
/* السطر البسيط في checks.c */
WARNING_IF_NOT_STRING_LIST(compatible_is_string_list, "compatible");
```

اللي بيوسّع لـ:

```c
static void check_is_string_list(struct check *c, struct dt_info *dti,
                                 struct node *node)
{
    int rem, l;
    struct property *prop;
    const char *propname = c->data;  /* = "compatible" */
    char *str;

    prop = get_property(node, propname);
    if (!prop)
        return;

    str = prop->val.val;
    rem = prop->val.len;
    while (rem > 0) {
        l = strnlen(str, rem);
        if (l == rem) {
            /* وصلنا لنهاية البيانات من غير null terminator */
            FAIL_PROP(c, dti, node, prop,
                "property is not a string list");
            break;
        }
        rem -= l + 1;
        str += l + 1;
    }
}
```

الكود المغلوط كان:
```dts
can-transceiver@0 {
    compatible = <0x1234 0x5678>;  /* cells مش strings! */
    reg = <0x0>;
};
```

ده بيخلي `prop->val.val` مش فيها null terminators، فـ `strnlen` بيرجع `rem` بالضبط = no null found.

#### الحل
```dts
/* قبل: غلط */
can-transceiver@0 {
    compatible = <0x1234 0x5678>;
    reg = <0x0>;
    spi-max-frequency = <10000000>;
};

/* بعد: صح */
can-transceiver@0 {
    compatible = "microchip,mcp2518fd", "microchip,mcp25xxfd";
    /* أو compatible = "ti,tcan4x5x"; حسب الـ chip */
    reg = <0>;   /* SPI CS index */
    spi-max-frequency = <10000000>;
    interrupts-extended = <&gpio0 15 IRQ_TYPE_EDGE_FALLING>;
    clocks = <&can_osc>;
};
```

للتحقق من كل الـ warnings دفعة واحدة:
```bash
dtc -I dts -O dtb -W all am62x-ecu.dts -o /dev/null 2>&1 | \
    grep -E "Warning|ERROR"
```

#### الدرس المستفاد
الـ `compatible` دايمًا **string list** مش cells. كل string مفصولة بـ null byte. الـ DTC بيتحقق من ده بـ `check_is_string_list`. الـ driver matching في الـ kernel بيعتمد على مطابقة الـ strings دي بالـ `of_device_id` table في الـ driver.

---

### السيناريو الخامس: Allwinner H616 — Custom Board Bring-up — `graph endpoint` غير bidirectional لـ HDMI

#### العنوان
**graph connection غير bidirectional** في DTS الخاص بـ HDMI pipeline على Allwinner H616

#### السياق
مهندس بيعمل board bring-up لـ custom board بيستخدم Allwinner H616 (مشهور في الـ Android TV boxes الصينية). بيحاول يوصّل الـ HDMI encoder بالـ TCON0 (timing controller) باستخدام الـ OF graph. بيكتب الـ `remote-endpoint` في طرف واحد بس وبينسى الطرف الثاني.

#### المشكلة
```
sun50i-h616-custom.dts: Warning (graph_endpoint): \
/display-engine/tcon@5461000/ports/port@0/endpoint@0: \
graph connection to node \
'/display-engine/hdmi/ports/port@0/endpoint@0' is not bidirectional
```
الـ display system مش شغّال. الـ HDMI مش بيطلع إشارة.

#### التحليل
الـ `check_graph_endpoint` في `checks.c`:

```c
static void check_graph_endpoint(struct check *c, struct dt_info *dti,
                                 struct node *node)
{
    struct node *remote_node;

    if (!node->parent || node->parent->bus != &graph_port_bus)
        return;

    check_graph_reg(c, dti, node);

    if (dti->dtsflags & DTSF_PLUGIN)
        return;

    if (!strprefixeq(node->name, node->basenamelen, "endpoint"))
        FAIL(c, dti, node,
            "graph endpoint node name should be 'endpoint'");

    remote_node = get_remote_endpoint(c, dti, node);
    if (!remote_node)
        return;

    /* التحقق من الاتجاهين */
    if (get_remote_endpoint(c, dti, remote_node) != node)
        FAIL(c, dti, node,
            "graph connection to node '%s' is not bidirectional",
            remote_node->fullpath);
}
```

الـ `get_remote_endpoint` بيتبع الـ `remote-endpoint` phandle:

```c
static struct node *get_remote_endpoint(struct check *c, struct dt_info *dti,
                                        struct node *endpoint)
{
    cell_t phandle;
    struct node *node;
    struct property *prop;

    prop = get_property(endpoint, "remote-endpoint");
    if (!prop)
        return NULL;

    phandle = propval_cell(prop);
    if (!phandle_is_valid(phandle))
        return NULL;

    node = get_node_by_phandle(dti->dt, phandle);
    if (!node)
        FAIL_PROP(c, dti, endpoint, prop, "graph phandle is not valid");

    return node;
}
```

المشكلة: الـ TCON endpoint بيشاور على الـ HDMI endpoint بـ `remote-endpoint`, بس الـ HDMI endpoint مش بيشاور على الـ TCON endpoint. الـ check بيتحقق إن `remote_of_remote == current_node`.

#### الحل

```dts
/* قبل: غلط — طرف واحد بس */
tcon0: tcon@5461000 {
    /* ... */
    ports {
        #address-cells = <1>;
        #size-cells = <0>;
        tcon0_out: port@1 {
            #address-cells = <1>;
            #size-cells = <0>;
            tcon0_out_hdmi: endpoint@1 {
                reg = <1>;
                remote-endpoint = <&hdmi_in_tcon0>;  /* شايل على hdmi */
            };
        };
    };
};

hdmi: hdmi@1ee0000 {
    /* ... */
    ports {
        #address-cells = <1>;
        #size-cells = <0>;
        hdmi_in: port@0 {
            #address-cells = <1>;
            #size-cells = <0>;
            hdmi_in_tcon0: endpoint@0 {
                reg = <0>;
                /* remote-endpoint مش موجود! */
            };
        };
    };
};

/* بعد: صح — bidirectional */
tcon0: tcon@5461000 {
    /* ... */
    ports {
        #address-cells = <1>;
        #size-cells = <0>;
        tcon0_out: port@1 {
            #address-cells = <1>;
            #size-cells = <0>;
            tcon0_out_hdmi: endpoint@1 {
                reg = <1>;
                remote-endpoint = <&hdmi_in_tcon0>;
            };
        };
    };
};

hdmi: hdmi@1ee0000 {
    /* ... */
    ports {
        #address-cells = <1>;
        #size-cells = <0>;
        hdmi_in: port@0 {
            #address-cells = <1>;
            #size-cells = <0>;
            hdmi_in_tcon0: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&tcon0_out_hdmi>;  /* أضف ده */
            };
        };
    };
};
```

للتحقق:
```bash
# تحقق من الـ graph بعد الإصلاح
dtc -I dts -O dts sun50i-h616-custom.dts 2>&1 | grep -i "graph\|endpoint"

# runtime: تحقق من الـ pipeline
cat /sys/kernel/debug/dri/0/state 2>/dev/null
# أو
modetest -M sun4i-drm -c 2>/dev/null
```

#### الدرس المستفاد
الـ OF graph لازم يكون **bidirectional** دايمًا. كل endpoint لازم يشاور على الـ remote endpoint بتاعه. الـ DTC بيتحقق من ده بـ `check_graph_endpoint`. الـ display pipeline في الـ kernel (خصوصًا DRM/KMS) بيعتمد على الـ graph علشان يبني الـ pipeline تلقائي — لو الـ graph مكسور، الـ display مش هيشتغل خالص حتى لو الـ hardware متوصل صح.
## Phase 7: مصادر ومراجع

### مصادر رسمية — Kernel Documentation

| المصدر | الرابط |
|--------|--------|
| Linux and the Devicetree (usage model) | [docs.kernel.org/devicetree/usage-model.html](https://docs.kernel.org/devicetree/usage-model.html) |
| Open Firmware and Devicetree — index | [docs.kernel.org/devicetree/index.html](https://docs.kernel.org/devicetree/index.html) |
| Writing Devicetree Bindings in json-schema | [docs.kernel.org/devicetree/bindings/writing-schema.html](https://docs.kernel.org/devicetree/bindings/writing-schema.html) |
| DOs and DON'Ts for Devicetree bindings | [docs.kernel.org/devicetree/bindings/writing-bindings.html](https://docs.kernel.org/devicetree/bindings/writing-bindings.html) |
| Devicetree Sources (DTS) Coding Style | [docs.kernel.org/devicetree/bindings/dts-coding-style.html](https://docs.kernel.org/devicetree/bindings/dts-coding-style.html) |
| Submitting Devicetree binding patches | [docs.kernel.org/devicetree/bindings/submitting-patches.html](https://docs.kernel.org/devicetree/bindings/submitting-patches.html) |
| DeviceTree Kernel API | [docs.kernel.org/devicetree/kernel-api.html](https://docs.kernel.org/devicetree/kernel-api.html) |

**الـ** `Documentation/` paths المقابلة في الـ kernel source tree:

```
Documentation/devicetree/usage-model.rst
Documentation/devicetree/bindings/writing-schema.rst
Documentation/devicetree/bindings/writing-bindings.rst
Documentation/devicetree/bindings/submitting-patches.rst
Documentation/devicetree/kernel-api.rst
```

---

### مقالات LWN.net

دي أهم مقالات LWN.net اللي بتشرح تطور الـ DTC وآليات الـ checking:

| العنوان | الرابط | الأهمية |
|---------|--------|---------|
| Device trees I: Are we having fun yet? | [lwn.net/Articles/572692](https://lwn.net/Articles/572692/) | مقدمة شاملة للـ DT في Linux |
| Device trees II: The harder parts | [lwn.net/Articles/574852](https://lwn.net/Articles/574852/) | تعمق في مشاكل الـ DT العملية |
| Device Tree schemas and validation | [lwn.net/Articles/568217](https://lwn.net/Articles/568217/) | schema-based validation مبكر |
| Device-tree schemas (dt_binding_check) | [lwn.net/Articles/771621](https://lwn.net/Articles/771621/) | الـ YAML schema validation الحديث في kernel 4.20+ |
| Device tree troubles | [lwn.net/Articles/560523](https://lwn.net/Articles/560523/) | مشاكل وتحديات الـ DT |
| Device trees as ABI | [lwn.net/Articles/561462](https://lwn.net/Articles/561462/) | الـ DT كـ ABI وتداعياته |
| Device tree overlays | [lwn.net/Articles/616859](https://lwn.net/Articles/616859/) | الـ overlays وتأثيرها على الـ checks |
| Device Tree Overlays - 8th time's the charm | [lwn.net/Articles/618334](https://lwn.net/Articles/618334/) | تطور دعم الـ overlays في dtc |
| An alternative device-tree source language | [lwn.net/Articles/730217](https://lwn.net/Articles/730217/) | بدائل لغة الـ DTS |
| Platform devices and device trees | [lwn.net/Articles/448502](https://lwn.net/Articles/448502/) | ربط platform devices بالـ DT |

---

### مصادر eLinux.org

**الـ** eLinux.org بتحتوي على أهم مراجع عملية للـ Device Tree:

| الصفحة | الرابط |
|--------|--------|
| Device Tree Reference (المرجع الأساسي) | [elinux.org/Device_Tree_Reference](https://elinux.org/Device_Tree_Reference) |
| Device Tree Mysteries (شرح الـ properties الغامضة) | [elinux.org/Device_Tree_Mysteries](https://elinux.org/Device_Tree_Mysteries) |
| Device Tree unittest | [elinux.org/Device_Tree_unittest](https://elinux.org/Device_Tree_unittest) |
| Device tree history | [elinux.org/Device_tree_history](https://elinux.org/Device_tree_history) |
| Device Tree frowand (tools & debugging) | [elinux.org/Device_Tree_frowand](https://elinux.org/Device_Tree_frowand) |
| Device tree kernel summit 2017 | [elinux.org/Device_tree_kernel_summit_2017_etherpad](https://elinux.org/Device_tree_kernel_summit_2017_etherpad) |
| Device tree plumbers 2016 | [elinux.org/Device_tree_plumbers_2016_etherpad](https://elinux.org/Device_tree_plumbers_2016_etherpad) |

---

### مصادر kernelnewbies.org

**الـ** kernelnewbies بيوثق تغييرات الـ DT في كل kernel release:

| الصفحة | الرابط |
|--------|--------|
| Linux 3.17 — DriversArch (DT changes) | [kernelnewbies.org/Linux_3.17-DriversArch](https://kernelnewbies.org/Linux_3.17-DriversArch) |
| Linux 3.18 — DriversArch | [kernelnewbies.org/Linux_3.18-DriversArch](https://kernelnewbies.org/Linux_3.18-DriversArch) |
| Linux 4.5 — DriversArch | [kernelnewbies.org/Linux_4.5-DriversArch](https://kernelnewbies.org/Linux_4.5-DriversArch) |
| Mailing List guidelines | [kernelnewbies.org/mailinglistguidelines](https://kernelnewbies.org/mailinglistguidelines) |

---

### مستودعات الكود

**الـ** upstream DTC project مستقل عن الـ kernel ويتضمن `checks.c`:

| المستودع | الرابط |
|---------|--------|
| DTC upstream (David Gibson) | [github.com/dgibson/dtc](https://github.com/dgibson/dtc) |
| DTC on kernel.googlesource.com | [kernel.googlesource.com/pub/scm/utils/dtc/dtc](https://kernel.googlesource.com/pub/scm/utils/dtc/dtc/) |
| DTC manual (documentation) | [github.com/vagrantc/device-tree-compiler — manual.txt](https://github.com/vagrantc/device-tree-compiler/blob/master/Documentation/manual.txt) |

**الـ** `scripts/dtc/` في kernel هو mirror من الـ upstream repository — أي تغيير في `checks.c` بيتعمل upstream أولاً.

---

### Commits مهمة في الـ Kernel

للبحث عن commits مرتبطة بـ `scripts/dtc/checks.c` في kernel git:

```bash
# كل commits لملف checks.c
git log --oneline -- scripts/dtc/checks.c

# تاريخ إضافة check معين، مثلاً node_name_chars
git log -S "node_name_chars" -- scripts/dtc/checks.c

# الـ sync commits من upstream DTC
git log --oneline --grep="dtc: sync" -- scripts/dtc/
```

**الـ** commit الخاص بإضافة `dt_binding_check` و `dtbs_check` targets:

```bash
# في kernel tree
git log --oneline --grep="dt_binding_check"
git log --oneline --grep="dtbs_check"
```

---

### مصادر إضافية لـ Validation الحديث

**الـ** dtschema project اللي بيوفر الـ YAML-based validation:

| المصدر | الرابط |
|--------|--------|
| dtschema على PyPI | [pypi.org/project/dtschema](https://pypi.org/project/dtschema) |
| Linaro: Tips for Validating DT with Schema | [linaro.org/blog/tips-and-tricks-for-validating-devicetree-sources-with-the-devicetree-schema](https://www.linaro.org/blog/tips-and-tricks-for-validating-devicetree-sources-with-the-devicetree-schema/) |

---

### كتب موصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 14** — The Linux Device Model: بيشرح كيف بتُعبّر عن الـ hardware في الـ kernel
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- الـ DTC وُجد بعد LDD3، لكن الـ device model fundamentals لازلت مهمة

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17** — Devices and Modules: device registration و platform devices
- بيشرح الـ infrastructure اللي بيتعامل معها الـ DT nodes بعد parsing

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 16** — Kernel Initialization: بيشرح كيف الـ kernel بيستخدم الـ DTB اللي الـ DTC ولّده
- **الفصل 15** — Debugging Embedded Linux: بيشرح تشخيص مشاكل الـ DT

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 6** — Device Drivers: platform devices وربطها بالـ OF (Open Firmware / DT)

---

### مصطلحات للبحث

للعثور على معلومات إضافية، استخدم المصطلحات دي:

```
dtc checks.c semantic check
device tree compiler warning error
DT binding validation kernel
dtc YAML schema dt_binding_check
dtbs_check kernel makefile
device tree phandle check
device tree node name check
libfdt validation
OF open firmware linux kernel
scripts/dtc linux kernel
```

**للبحث في mailing list archives:**

```bash
# على lore.kernel.org
https://lore.kernel.org/devicetree/?q=checks.c

# أو
https://lore.kernel.org/devicetree/?q=dtc+check
```

---

### ملخص سريع — أهم 5 روابط للبدء

| الأولوية | المصدر | السبب |
|---------|--------|-------|
| 1 | [lwn.net/Articles/572692](https://lwn.net/Articles/572692/) | أفضل مقدمة للـ DT في Linux |
| 2 | [elinux.org/Device_Tree_Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل للـ properties والـ syntax |
| 3 | [lwn.net/Articles/771621](https://lwn.net/Articles/771621/) | فهم schema validation الحديث |
| 4 | [docs.kernel.org/devicetree/bindings/writing-schema.html](https://docs.kernel.org/devicetree/bindings/writing-schema.html) | كتابة bindings صحيحة تعدي الـ checks |
| 5 | [github.com/dgibson/dtc](https://github.com/dgibson/dtc) | الـ upstream source لـ `checks.c` نفسه |
## Phase 8: Writing simple module

### الفكرة

**`checks.c`** هو جزء من **DTC (Device Tree Compiler)** — أداة userspace بتـvalidate وتـcheck الـ device tree قبل ما تتحول لـ DTB. المقابل الحقيقي في الـ kernel هو الكود اللي بيـparse وبيـunflatten الـ DTB في runtime — وتحديداً الدالة **`unflatten_device_tree()`** اللي بتتحول فيها الـ flat blob لـ live tree من `struct device_node`.

الـ module هيستخدم **kprobe** على `unflatten_device_tree` عشان يـlog كل مرة الـ kernel بيعمل فيها parse للـ device tree — ده مفيد جداً في الـ embedded systems debugging.

---

### الـ Kernel Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * dtc_checks_probe.c
 *
 * Hooks into unflatten_device_tree() via kprobe to log
 * every time the kernel parses the device tree blob.
 *
 * Mirrors the validation role of scripts/dtc/checks.c but
 * from inside the kernel at parse time.
 */

#include <linux/module.h>      /* MODULE_* macros, module_init/exit  */
#include <linux/kernel.h>      /* pr_info, KERN_INFO                 */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe ... */
#include <linux/of_fdt.h>      /* initial_boot_params (DTB pointer)  */
#include <linux/libfdt.h>      /* fdt_totalsize(), fdt_version()     */

/* ------------------------------------------------------------------ */
/* 1. Pre-handler: fires just BEFORE unflatten_device_tree() executes  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * initial_boot_params هو pointer للـ DTB الـ raw في الـ memory.
     * بنقرأ منه معلومات بسيطة زي الـ total size والـ FDT version.
     */
    if (initial_boot_params) {
        pr_info("dtc_checks_probe: unflatten_device_tree() called\n");
        pr_info("dtc_checks_probe:   DTB @ %px | total_size=%u bytes | fdt_version=%d\n",
                initial_boot_params,
                fdt_totalsize(initial_boot_params),
                fdt_version(initial_boot_params));
    } else {
        pr_info("dtc_checks_probe: unflatten_device_tree() called (no DTB pointer yet)\n");
    }

    return 0; /* 0 = let the original function continue normally */
}

/* ------------------------------------------------------------------ */
/* 2. Post-handler: fires just AFTER unflatten_device_tree() returns   */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * بعد ما الـ unflatten خلص، الـ device tree بقى live tree.
     * بنـlog رسالة بسيطة تأكيداً إن العملية اتمت.
     */
    pr_info("dtc_checks_probe: unflatten_device_tree() completed — live OF tree is ready\n");
}

/* ------------------------------------------------------------------ */
/* 3. تعريف الـ kprobe struct                                           */
/* ------------------------------------------------------------------ */
static struct kprobe dtc_kp = {
    .symbol_name = "unflatten_device_tree", /* الدالة اللي هنـhook عليها */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* 4. module_init: بنـregister الـ kprobe                              */
/* ------------------------------------------------------------------ */
static int __init dtc_checks_probe_init(void)
{
    int ret;

    /*
     * register_kprobe() بتزرع breakpoint في الـ kernel text
     * عند بداية الدالة المحددة في .symbol_name.
     */
    ret = register_kprobe(&dtc_kp);
    if (ret < 0) {
        pr_err("dtc_checks_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("dtc_checks_probe: kprobe planted on %s @ %px\n",
            dtc_kp.symbol_name, dtc_kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* 5. module_exit: لازم نـunregister عشان نرجع الـ original bytes      */
/* ------------------------------------------------------------------ */
static void __exit dtc_checks_probe_exit(void)
{
    /*
     * unregister_kprobe() بترجع الـ original instruction اللي
     * استبدلناها بالـ breakpoint — لو ما عملناش ده هيحصل kernel panic.
     */
    unregister_kprobe(&dtc_kp);
    pr_info("dtc_checks_probe: kprobe removed from %s\n", dtc_kp.symbol_name);
}

module_init(dtc_checks_probe_init);
module_exit(dtc_checks_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe on unflatten_device_tree() — mirrors dtc/checks.c validation role");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل الـ API بتاعها |
| `linux/of_fdt.h` | بيعطينا `initial_boot_params` — الـ pointer للـ DTB الخام |
| `linux/libfdt.h` | بنستخدم `fdt_totalsize()` و`fdt_version()` عشان نقرأ header الـ DTB |

الـ `of_fdt.h` و`libfdt.h` مهمين هنا لأن `checks.c` نفسه شغلته كلها حول validate الـ FDT — فأحنا بنطبق نفس الفكرة من جوه الـ kernel.

---

#### الـ `handler_pre`

**الـ pre-handler** بيتنفذ فور ما الـ CPU يوصل لأول instruction في `unflatten_device_tree()` قبل ما تنفذ. بنستخدم `initial_boot_params` عشان نجيب:
- الـ **physical address** للـ DTB في الـ memory
- **`fdt_totalsize()`**: حجم الـ blob كاملاً بالـ bytes — نفس المعلومة اللي `checks.c` بتـprocess فيها الـ tree
- **`fdt_version()`**: إصدار الـ FDT format (عادةً 17)

لو `initial_boot_params` كان NULL (نادر جداً) بنـlog رسالة بديلة بدل crash.

---

#### الـ `handler_post`

**الـ post-handler** بيتنفذ بعد رجوع الدالة. بنـlog رسالة تأكيد إن الـ live OF tree اتبنى — ده مفيد في الـ debugging لأنك بتعرف بالظبط إيمتى الـ device tree بقى available للـ drivers.

---

#### الـ `struct kprobe`

```c
static struct kprobe dtc_kp = {
    .symbol_name = "unflatten_device_tree",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};
```

الـ **`symbol_name`** بيخلي الـ kernel يـresolve العنوان تلقائياً من الـ kallsyms — مش محتاج تحط عنوان hardcoded. الـ kprobe بيحل محل أول instruction بـ breakpoint (int3 على x86).

---

#### الـ `module_init` و `module_exit`

**`register_kprobe()`** بتزرع الـ breakpoint وبتربط الـ handlers. لو فشلت (مثلاً الدالة مش exported أو الـ CONFIG_KPROBES مش مفعل) بترجع error code سالب.

**`unregister_kprobe()`** في الـ exit **ضرورية جداً** — بتعمل restore للـ original bytes. لو ما عملتيهاش والـ module اتـunload، الـ handler هيتنفذ من memory اتـfree وهيحصل **use-after-free** وبعدين **kernel panic**.

---

### بناء الـ Module

```bash
# Makefile
obj-m += dtc_checks_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# تحقق إن الـ kprobes مفعلة
grep CONFIG_KPROBES /boot/config-$(uname -r)
# CONFIG_KPROBES=y

# بناء وتحميل
make
sudo insmod dtc_checks_probe.ko

# شوف الـ output (الدالة بتتنفذ مرة عند boot عادةً، قد تكون سبق)
sudo dmesg | grep dtc_checks_probe

# إزالة الـ module
sudo rmmod dtc_checks_probe
```

---

### الـ Output المتوقع في `dmesg`

```
[    0.842315] dtc_checks_probe: kprobe planted on unflatten_device_tree @ ffffffff81a3c4e0
[    0.842380] dtc_checks_probe: unflatten_device_tree() called
[    0.842381] dtc_checks_probe:   DTB @ 00000000abcd1234 | total_size=65536 bytes | fdt_version=17
[    0.843210] dtc_checks_probe: unflatten_device_tree() completed — live OF tree is ready
```

> **ملاحظة**: لو الـ module اتحمل بعد الـ boot، مش هتشوف الـ pre/post output لأن `unflatten_device_tree()` بتتنفذ مرة واحدة بس في early boot. في الـ QEMU أو عند تجربة الـ module في VM ممكن تـtrigger الـ probe بـreboot.

---

### العلاقة بـ `checks.c`

| `dtc/checks.c` (userspace) | الـ kernel module (runtime) |
|---|---|
| يتنفذ على الـ `.dts` / `.dtb` قبل الـ boot | يتنفذ داخل الـ kernel أثناء الـ boot |
| `run_check()` تمشي على كل الـ checks | `handler_pre()` بيلتقط لحظة بداية الـ parse |
| يطبع warnings وأخطاء في stderr | يطبع معلومات الـ DTB في `dmesg` |
| `process_checks()` هي نقطة الدخول | `unflatten_device_tree()` هي نقطة الدخول في الـ kernel |
