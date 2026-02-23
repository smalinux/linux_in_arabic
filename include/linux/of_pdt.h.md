## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

**الـ** `include/linux/of_pdt.h` هو header صغير جداً — 39 سطراً فقط — لكنه يربط عالمَين مختلفَين تماماً: **Open Firmware PROM** (firmware قديم موجود في العتاد نفسه) و**device tree** الذي يستخدمه Linux kernel الحديث لاكتشاف الأجهزة.

---

### القصة: الكمبيوتر الذي يحمل دليله بنفسه

تخيّل أنك اشتريت جهاز SPARC أو PowerMac قديماً. هذا الجهاز لا يحتوي على BIOS عادي — بل يحتوي على شيء اسمه **Open Firmware (OF)**، وهو firmware مدمج في الـ ROM الخاص بالجهاز. هذا الـ firmware يعرف كل شيء عن الجهاز: الـ CPU، الذاكرة، الباصات، الأجهزة الطرفية — ويُخزّن هذه المعلومات في شجرة بيانات داخلية اسمها **PROM device tree**.

عندما يبدأ Linux بالتشغيل على هذه الأجهزة، يجد أمامه مشكلة:

- Linux الحديث يعمل مع **device tree** بصيغة موحّدة (`struct device_node`, `struct property`)
- لكن هذه المعلومات موجودة فعلاً داخل الـ PROM firmware بصيغة خاصة بكل معمارية
- الوصول إليها يختلف بين SPARC وPowerPC وغيرها

**الحل:** `of_pdt` — جسر يستدعي الـ PROM مباشرة أثناء الـ boot المبكر، يسحب منه كل معلومات الأجهزة، ويبني منها شجرة `device_node` يفهمها بقية الـ kernel.

---

### تشبيه بسيط

الـ PROM كالموظف القديم الذي يعرف كل شيء عن المبنى (أين الغرف، ما فيها، كيف تتصل ببعض) لكنه يتحدث لهجة خاصة.

**الـ** `of_pdt` هو المترجم الذي يسأل هذا الموظف بلغته، ثم يكتب تقريراً موحداً (device tree) يفهمه كل Linux.

---

### ما الذي يعرّفه هذا الـ Header؟

#### `struct of_pdt_ops` — جدول العمليات القابل للتخصيص

```c
struct of_pdt_ops {
    /* اقرأ اسم الخاصية التالية من node معيّن */
    int (*nextprop)(phandle node, char *prev, char *buf);

    /* احصل على طول خاصية معيّنة */
    int (*getproplen)(phandle node, const char *prop);

    /* احصل على قيمة الخاصية */
    int (*getproperty)(phandle node, const char *prop, char *buf, int bufsize);

    /* انتقل للابن الأول أو للأخ التالي في الشجرة */
    phandle (*getchild)(phandle parent);
    phandle (*getsibling)(phandle node);

    /* حوّل phandle إلى مسار نصي كامل مثل /cpus/cpu@0 */
    int (*pkg2path)(phandle node, char *buf, const int buflen, int *len);
};
```

هذا الـ struct هو **واجهة برمجية قابلة للتخصيص (abstraction layer)**. كل معمارية (SPARC، PowerPC) تُعبئ هذه الـ function pointers بدوالها الخاصة التي تتحدث مع الـ PROM الخاص بها.

#### الدالتان المُعلَنتان

```c
/* تخصيص ذاكرة مبكراً قبل أن يعمل allocator الاعتيادي */
extern void *prom_early_alloc(unsigned long size);

/* الدالة الرئيسية: ابنِ شجرة device_node من الـ PROM */
extern void of_pdt_build_devicetree(phandle root_node, struct of_pdt_ops *ops);
```

---

### كيف تعمل العملية كاملاً؟

```
 ┌─────────────────────────────────────────────────────┐
 │               Hardware PROM (ROM)                   │
 │   يحتوي شجرة nodes: CPU، RAM، PCI، Serial، ...      │
 └──────────────────────┬──────────────────────────────┘
                        │  استدعاءات PROM (nextprop، getchild...)
                        ▼
 ┌─────────────────────────────────────────────────────┐
 │         of_pdt_build_devicetree()  [pdt.c]          │
 │  يمشي الشجرة بشكل recursive                        │
 │  يبني struct device_node لكل node                   │
 │  يبني struct property لكل خاصية                     │
 └──────────────────────┬──────────────────────────────┘
                        │
                        ▼
 ┌─────────────────────────────────────────────────────┐
 │         Linux Device Tree (of_root)                 │
 │   شجرة موحّدة يفهمها كل الـ kernel                  │
 │   drivers، subsystems، interrupts، ...              │
 └─────────────────────────────────────────────────────┘
```

---

### أي Subsystem ينتمي إليه؟

**الـ** `of_pdt.h` ينتمي إلى subsystem **OF (Open Firmware) / Device Tree** داخل Linux kernel. هذا الـ subsystem مسؤول عن:
- قراءة معلومات العتاد من مصادر مختلفة (FDT blob، PROM، ACPI)
- توحيدها في بنية `device_node` واحدة
- توفيرها لبقية الـ kernel ولـ drivers

---

### متى يُستخدم هذا المسار تحديداً؟

فقط على المعماريات التي تملك **Open Firmware PROM حي وقابل للاستدعاء** أثناء boot:
- **SPARC / SPARC64** (Sun workstations، UltraSPARC)
- **PowerPC** (Apple PowerMac القديمة)

أما الأنظمة الحديثة (ARM، x86، RISC-V) فتستخدم **FDT (Flattened Device Tree)** — ملف ثنائي جاهز يُمرَّر للـ kernel عند الإقلاع، ويُعالَج عبر `drivers/of/fdt.c` لا `pdt.c`.

---

### الملفات المرتبطة التي يجب معرفتها

| الملف | الدور |
|---|---|
| `drivers/of/pdt.c` | التنفيذ الكامل لـ `of_pdt_build_devicetree()` وكل دوال بناء الشجرة |
| `include/linux/of.h` | تعريفات `struct device_node`، `struct property`، `phandle` |
| `arch/sparc/include/asm/prom.h` | يستخدم `of_pdt_ops` ويعرّف `prom_build_devicetree()` الخاصة بـ SPARC |
| `drivers/of/fdt.c` | المسار البديل للأنظمة الحديثة (FDT blob بدلاً من PROM حي) |
| `drivers/of/base.c` | العمليات الأساسية على شجرة الـ device tree بعد بنائها |
| `drivers/of/of_private.h` | دوال داخلية مشتركة بين ملفات `drivers/of/` |

---

### ملفات الـ Subsystem الأساسية

**Core:**
- `drivers/of/pdt.c` — بناء الشجرة من PROM
- `drivers/of/fdt.c` — بناء الشجرة من FDT blob
- `drivers/of/base.c` — العمليات الأساسية (البحث، القراءة)
- `drivers/of/dynamic.c` — التعديل الديناميكي للشجرة
- `drivers/of/platform.c` — ربط الـ device_node بـ platform_device

**Headers:**
- `include/linux/of.h` — التعريفات الأساسية
- `include/linux/of_pdt.h` — واجهة PROM builder (هذا الملف)

**معمارية SPARC:**
- `arch/sparc/include/asm/prom.h` — تعريفات SPARC الخاصة
- `arch/sparc/kernel/prom_*.c` — تنفيذ `of_pdt_ops` لـ SPARC
## Phase 2: شرح الـ OF-PDT (Open Firmware PROM Device Tree) Framework

---

### المشكلة التي يحلها هذا الـ Subsystem

في الأنظمة الكلاسيكية مثل **SPARC** و **PowerPC** وبعض أجهزة x86 الخاصة كـ **OLPC**، يوجد firmware حي يُعرف بـ **Open Firmware (OFW)**. هذا الـ firmware لا يقدم ملف DTB ثابتاً في الذاكرة — بل يحتفظ بشجرة الأجهزة داخله ويُجيب على استعلامات runtime عبر **PROM (Programmable Read-Only Memory) call interface**.

المشكلة المحددة:

- الـ Linux kernel يحتاج `struct device_node` شجرة كاملة في الذاكرة **قبل** أي driver يعمل.
- الـ OFW يخزن شجرة الأجهزة كـ **PROM nodes** بعناوين مرجعية تُسمى **phandles**.
- كل معمارية لها طريقتها الخاصة للتحدث مع الـ PROM (SPARC يستخدم `prom_*` calls، OLPC يستخدم `olpc_ofw("peer", ...)` إلخ).

**بدون** هذا الـ subsystem: كل معمارية تحتاج تكتب كودها الخاص لبناء الـ device tree من الصفر — تكرار ضخم وأخطاء متعددة.

---

### الحل — ما الذي يفعله الـ Kernel؟

الكيرنل يقدم **طبقة تجريد** عبر `of_pdt`:

1. تُعرِّف كل معمارية مجموعة من **callback functions** تُغلَّف داخل `struct of_pdt_ops`.
2. تستدعي المعمارية `of_pdt_build_devicetree(root_phandle, &ops)` مرة واحدة أثناء الـ boot.
3. الـ framework يسير عبر شجرة الـ PROM بشكل تعاودي، يستدعي الـ callbacks لاستخراج كل node وكل property، ويبني `struct device_node` مقابل كل عقدة PROM.
4. النتيجة هي `of_root` — نقطة الدخول الموحدة لشجرة الأجهزة في الكيرنل.

---

### التشابه مع الواقع — المترجم الفوري

تخيل شخصاً يتحدث لغة قديمة نادرة (الـ PROM firmware). لا يستطيع كل موظف في الوزارة تعلم هذه اللغة.

الحل: نوظف **مترجماً فورياً** (الـ `of_pdt_ops`) يعرف هذه اللغة. الوزارة (الـ kernel) تعطي المترجم قائمة أسئلة محددة:
- "اسأله عن أطفاله" → `getchild()`
- "اسأله عن إخوته" → `getsibling()`
- "اسأله عن خصائصه" → `nextprop()` + `getproperty()`

المترجم يتحدث مع الشخص بلغته، ويُعيد الإجابات بالعربية (أي بـ `struct device_node` التي يفهمها الكيرنل). الوزارة لا تحتاج أن تعرف شيئاً عن اللغة القديمة بعد ذلك.

**الربط الكامل للتشابه بالكود:**

| عنصر التشابه | المقابل الحقيقي |
|---|---|
| اللغة القديمة النادرة | PROM interface الخاص بكل معمارية |
| المترجم الفوري | `struct of_pdt_ops` + تنفيذها في كل معمارية |
| الأسئلة المحددة | function pointers: `getchild`, `nextprop`, `getproperty` |
| ترجمة الإجابة | بناء `struct device_node` و `struct property` |
| الأرشيف الرسمي للوزارة | `of_root` شجرة الـ device nodes في الكيرنل |
| موظف الوزارة يسأل الأرشيف | أي driver يستخدم `of_find_node_by_name()` إلخ |

---

### المكان في معمارية الكيرنل — الصورة الكبيرة

```
+------------------------------------------------------------------+
|                        HARDWARE / PROM                           |
|                                                                  |
|   OFW PROM Firmware (SPARC oplib / OLPC OFW / PowerPC PROM)     |
|   يحتفظ بشجرة الأجهزة كـ phandle nodes                          |
+---------------------------+--------------------------------------+
                            |  PROM calls (arch-specific)
                            v
+------------------------------------------------------------------+
|               ARCH-SPECIFIC PDT OPS LAYER                        |
|                                                                  |
|  arch/sparc/kernel/prom_common.c   → struct of_pdt_ops sparc_ops |
|  arch/x86/platform/olpc/olpc_dt.c → struct of_pdt_ops olpc_ops  |
|                                                                  |
|  كل ops تُنفذ: nextprop / getproplen / getproperty               |
|               getchild / getsibling / pkg2path                   |
+---------------------------+--------------------------------------+
                            |  of_pdt_build_devicetree(root, ops)
                            v
+------------------------------------------------------------------+
|                 drivers/of/pdt.c  —  OF-PDT CORE                 |
|                                                                  |
|  of_pdt_build_devicetree()                                       |
|    └─ of_pdt_create_node()       → struct device_node *          |
|         └─ of_pdt_build_prop_list()  → struct property *         |
|              └─ of_pdt_build_one_prop()                          |
|    └─ of_pdt_build_tree()        (recursive DFS traversal)       |
|    └─ of_alias_scan()            (parse /aliases node)           |
+---------------------------+--------------------------------------+
                            |  of_root (global device tree)
                            v
+------------------------------------------------------------------+
|                    OF CORE  (drivers/of/*.c)                     |
|                                                                  |
|  of_find_node_by_name(), of_get_property(), of_match_device()   |
|  ... كل الـ OF API العادي يعمل من هنا                             |
+---------------------------+--------------------------------------+
                            |
                            v
+------------------------------------------------------------------+
|                        DEVICE DRIVERS                            |
|                                                                  |
|  platform_driver, i2c_driver, spi_driver ...                     |
|  يستخدمون of_node للـ probing والـ configuration                 |
+------------------------------------------------------------------+
```

---

### التجريد المحوري — الـ Core Abstraction

**الـ `phandle`** هو القلب. إنه عدد صحيح من 32 بت (`typedef u32 phandle`) يُعبر عن **مرجع** لعقدة في شجرة الـ PROM. فكر فيه كـ "pointer" لكن بدلاً من عنوان ذاكرة في كيرنل-سبيس، هو **معرف داخلي** في الـ firmware.

الـ `struct of_pdt_ops` هو الـ vtable — جدول الدوال الافتراضية. يُعرِّف "كيف تتحدث مع الـ PROM" بشكل مجرد:

```c
struct of_pdt_ops {
    /* iterate properties: prev=NULL يُعطي أول property */
    int (*nextprop)(phandle node, char *prev, char *buf);

    /* طول الـ property بالبايت، أو -1 عند الخطأ */
    int (*getproplen)(phandle node, const char *prop);

    /* اقرأ قيمة الـ property في buf */
    int (*getproperty)(phandle node, const char *prop,
                       char *buf, int bufsize);

    /* navigation: 0 يعني لا يوجد child/sibling */
    phandle (*getchild)(phandle parent);
    phandle (*getsibling)(phandle node);

    /* حوِّل phandle إلى مسار نصي مثل "/cpus/cpu@0" */
    int (*pkg2path)(phandle node, char *buf,
                    const int buflen, int *len);
};
```

---

### كيف تتصل الـ Structs ببعضها

```
                    phandle (u32)
                        │
                        │  ops->getchild() / ops->getsibling()
                        ▼
              ┌─────────────────────────┐
              │    struct device_node   │
              │─────────────────────────│
              │ name         (char*)    │◄── ops->getproperty("name")
              │ phandle      (u32)      │◄── phandle رقم الـ PROM node
              │ full_name    (char*)    │◄── ops->pkg2path()
              │ parent       (dn*)      │◄── يُعيَّن أثناء الـ traversal
              │ child        (dn*)      │◄── ops->getchild()
              │ sibling      (dn*)      │◄── ops->getsibling()
              │ properties   (prop*)   ─┼──┐
              └─────────────────────────┘  │
                                           │
                                           ▼
              ┌─────────────────────────┐   ┌─────────────────────────┐
              │    struct property      │──►│    struct property       │
              │─────────────────────────│   │─────────────────────────│
              │ name   = "compatible"   │   │ name   = "reg"          │
              │ length = 12             │   │ length = 4              │
              │ value  = "ti,omap3430"  │   │ value  = 0x00000000     │
              │ next   ─────────────────┘   │ next   = NULL           │
              └─────────────────────────┘   └─────────────────────────┘
```

**ملاحظة مهمة:** أول property تُبنى دائماً هي `.node` وهي خاصة — تحتوي على قيمة الـ `phandle` الخام. هذا يسمح للكود لاحقاً بإيجاد الـ phandle من أي `device_node`.

---

### تسلسل البناء أثناء الـ Boot — خطوة بخطوة

```
of_pdt_build_devicetree(root_phandle, &ops)
│
├─ [1] حفظ ops في of_pdt_prom_ops (global __initdata)
│
├─ [2] of_pdt_create_node(root_phandle, NULL)
│       ├─ prom_early_alloc(sizeof(device_node))  // تخصيص من memblock
│       ├─ dp->phandle = root_phandle
│       ├─ dp->name = ops->getproperty(node, "name")
│       ├─ dp->properties = of_pdt_build_prop_list(node)
│       │       ├─ أول prop: ".node" = phandle value نفسها
│       │       └─ حلقة: nextprop() → getproplen() → getproperty()
│       └─ dp->full_name = ops->pkg2path(node)  // مثل "/cpus/cpu@0"
│
├─ [3] of_root = dp  // تعيين الجذر العالمي
│       of_root->full_name = "/"  // override للجذر
│
├─ [4] of_pdt_build_tree(of_root, ops->getchild(root))
│       └─ حلقة while(node != 0):
│               ├─ dp = of_pdt_create_node(node, parent)
│               ├─ dp->child = of_pdt_build_tree(dp, getchild(node))  // تعاود
│               └─ node = ops->getsibling(node)
│
└─ [5] of_alias_scan()  // parse /aliases و /chosen
```

---

### مثال حقيقي — OLPC XO Laptop

جهاز **OLPC XO-1** هو كمبيوتر x86 يحمل firmware من نوع Open Firmware بدلاً من BIOS/UEFI التقليدي. في `/workspace/external/linux/arch/x86/platform/olpc/olpc_dt.c`:

```c
/* كل دالة تستدعي olpc_ofw() — وهي syscall للـ OFW firmware */

static phandle __init olpc_dt_getchild(phandle node)
{
    const void *args[] = { (void *)node };
    void *res[] = { &node };

    if ((s32)node == -1)
        return 0;

    /* "child" هي OFW standard method لجلب أول ابن */
    if (olpc_ofw("child", args, res) || (s32)node == -1)
        return 0;

    return node;
}

/* الـ ops table المكتملة */
static struct of_pdt_ops prom_olpc_ops __initdata = {
    .nextprop    = olpc_dt_nextprop,
    .getproplen  = olpc_dt_getproplen,
    .getproperty = olpc_dt_getproperty,
    .getchild    = olpc_dt_getchild,
    .getsibling  = olpc_dt_getsibling,
    .pkg2path    = olpc_dt_pkg2path,
};
```

في وقت الـ boot، يستدعي كود OLPC:
```c
of_pdt_build_devicetree(root_node, &prom_olpc_ops);
```

ومن هنا يعمل باقي الكيرنل كأنه على SPARC أو PowerPC — لأن الـ `of_root` واحد على جميع المعماريات.

---

### الـ `prom_early_alloc` — تخصيص الذاكرة قبل الـ Allocator

**الـ `prom_early_alloc`** هو allocator بسيط يعمل قبل أن يكون الـ `kmalloc` متاحاً. الكيرنل في مرحلة الـ early boot لا يملك slab allocator بعد، لذا يُخصص من الـ **memblock** (الـ subsystem المسؤول عن إدارة الذاكرة الأولية أثناء الـ boot). كل الـ `device_node` و `property` structs المبنية من الـ PROM تُخصَّص هنا وتبقى طول عمر الكيرنل.

```c
/* من olpc_dt.c — مثال على تنفيذ prom_early_alloc */
void * __init prom_early_alloc(unsigned long size)
{
    static u8 *mem;
    static size_t free_mem;
    void *res;

    if (free_mem < size) {
        /* اطلب PAGE_SIZE على الأقل من memblock */
        res = memblock_alloc_or_panic(chunk_size, SMP_CACHE_BYTES);
        free_mem = chunk_size;
        mem = res;
    }

    /* أعط المستدعي جزءاً من الـ chunk المحجوز */
    free_mem -= size;
    res = mem;
    mem += size;
    return res;
}
```

**الـ memblock subsystem**: طبقة إدارة ذاكرة بسيطة جداً تعمل قبل الـ buddy allocator. تُسجِّل مناطق الذاكرة المتاحة والمحجوزة وتُعطي chunks للمستخدمين خلال الـ early boot.

---

### ما يملكه الـ OF-PDT مقابل ما يفوّضه

| المسؤولية | OF-PDT Core (`drivers/of/pdt.c`) | الـ Architecture-Specific Ops |
|---|---|---|
| traversal منطق الشجرة (DFS) | نعم | لا |
| بناء `struct device_node` | نعم | لا |
| بناء `struct property` linked list | نعم | لا |
| التحدث مع الـ PROM firmware | لا | نعم |
| تفسير صيغة الـ phandle | لا | نعم |
| `prom_early_alloc` تنفيذه | لا (تعريف فقط) | نعم (كل arch تُعرّفه) |
| تعيين `of_root` العالمي | نعم | لا |
| استدعاء `of_alias_scan()` | نعم | لا |

**ملاحظة الـ CONFIG_OF_PROMTREE**: هذا الـ subsystem كله محمي بـ `CONFIG_OF_PROMTREE`. على المعماريات التي تستخدم FDT (Flattened Device Tree مثل ARM) يُستخدم بدلاً من ذلك `of_fdt` الذي يقرأ ملف DTB ثابت من الذاكرة بدلاً من الاستعلام من الـ firmware.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### Config Options المتعلقة بـ `of_pdt`

| الخيار | النوع | الوصف |
|--------|-------|-------|
| `CONFIG_OF_PROMTREE` | `bool` | يُفعِّل تجميع `pdt.o` — يُختار تلقائياً على SPARC وOLPC/x86 |
| `CONFIG_SPARC` | arch | يُفعِّل مسار `unique_id` والـ `irq_trans` داخل `device_node` |
| `CONFIG_OF_DYNAMIC` | `bool` | يضيف حقل `_flags` إلى `struct property` |
| `CONFIG_OF_PROMTREE` | `bool` | يضيف حقل `unique_id` إلى `struct property` |
| `CONFIG_OF_KOBJ` | `def_bool SYSFS` | يضيف `struct bin_attribute attr` إلى `property` وkobj إلى `device_node` |

#### Flags الـ `device_node`

| الـ Flag | القيمة | المعنى |
|----------|--------|--------|
| `OF_DYNAMIC` | 1 | العقدة أو الـ property مُخصَّصة بـ `kmalloc` |
| `OF_DETACHED` | 2 | منفصلة عن شجرة الأجهزة |
| `OF_POPULATED` | 3 | الجهاز قد تم إنشاؤه مسبقاً |
| `OF_POPULATED_BUS` | 4 | تم إنشاء platform bus للأبناء |
| `OF_OVERLAY` | 5 | مُخصَّصة لـ overlay |
| `OF_OVERLAY_FREE_CSET` | 6 | ضمن overlay cset قيد الحذف |

> ملاحظة: في سياق `of_pdt`، كل الذاكرة تُخصَّص عبر `prom_early_alloc` (ذاكرة early-boot ثابتة) وليس `kmalloc`، لذا لا يُرفع `OF_DYNAMIC` أبداً على هذه العقد.

---

### الـ Structs الأساسية

---

#### 1. `struct of_pdt_ops`

**الغرض:** واجهة عمليات abstraction تُغلِّف استدعاءات الـ **Open Firmware PROM** — كل معمارية تُقدِّم تنفيذها الخاص لهذه المؤشرات الوظيفية.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `nextprop` | `int (*)(phandle, char*, char*)` | يجلب اسم الـ property التالية بعد `prev`؛ المخزن `buf` يجب أن يكون 32 byte؛ `prev=NULL` يعني الأولى |
| `getproplen` | `int (*)(phandle, const char*)` | يُعيد طول الـ property بالبايت، أو `-1` عند الخطأ |
| `getproperty` | `int (*)(phandle, const char*, char*, int)` | يقرأ قيمة الـ property إلى `buf` بحجم `bufsize`؛ يُعيد طول المكتوب أو `-1` |
| `getchild` | `phandle (*)(phandle)` | يُعيد الـ phandle للابن الأول؛ `0` إن لم يوجد |
| `getsibling` | `phandle (*)(phandle)` | يُعيد الـ phandle للأخ التالي؛ `0` عند النهاية |
| `pkg2path` | `int (*)(phandle, char*, int, int*)` | يحوِّل الـ phandle إلى مسار نصي كامل؛ يملأ `len` بالطول |

**الارتباطات:**
- يُخزَّن في المتغير الثابت `of_pdt_prom_ops` (مُعرَّف بـ `__initdata`) طوال عملية البناء.
- يُمرَّر من المعمارية إلى `of_pdt_build_devicetree()` ثم يُستخدم في جميع دوال البناء الداخلية.

---

#### 2. `struct property`

**الغرض:** يُمثِّل **خاصية واحدة** (property) لعقدة في شجرة الأجهزة — اسم + طول + قيمة.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `name` | `char *` | اسم الـ property (مثل `"compatible"`, `"reg"`) |
| `length` | `int` | حجم `value` بالبايت |
| `value` | `void *` | بيانات الـ property الخام |
| `next` | `struct property *` | مؤشر للـ property التالية في القائمة المرتبطة |
| `_flags` | `unsigned long` | (اختياري: `CONFIG_OF_DYNAMIC` أو `CONFIG_SPARC`) |
| `unique_id` | `unsigned int` | (اختياري: `CONFIG_OF_PROMTREE`) معرِّف فريد على SPARC |
| `attr` | `struct bin_attribute` | (اختياري: `CONFIG_OF_KOBJ`) لعرضها في sysfs |

**في سياق `of_pdt`:**
- تُخصَّص بـ `prom_early_alloc(sizeof(struct property) + 32)` — الـ 32 بايت الإضافية تخزِّن الاسم مباشرة بعد الـ struct.
- `p->name = (char *)(p + 1)` — الاسم يسكن في نفس كتلة الذاكرة مباشرة.
- أول property خاصة اسمها `".node"` وقيمتها الـ phandle نفسه.

---

#### 3. `struct device_node`

**الغرض:** يُمثِّل **عقدة واحدة** في شجرة الأجهزة — تُكافئ عقدة PROM بعد تحويلها إلى تمثيل kernel داخلي.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `name` | `const char *` | اسم العقدة (من property `"name"`) |
| `phandle` | `phandle` | المعرِّف العددي للعقدة في PROM |
| `full_name` | `const char *` | المسار الكامل مثل `"/cpus/cpu@0"` |
| `fwnode` | `struct fwnode_handle` | واجهة firmware node العامة |
| `properties` | `struct property *` | رأس قائمة الـ properties المرتبطة |
| `deadprops` | `struct property *` | properties محذوفة (غير مستخدمة في of_pdt) |
| `parent` | `struct device_node *` | مؤشر للعقدة الأب |
| `child` | `struct device_node *` | مؤشر للابن الأول |
| `sibling` | `struct device_node *` | مؤشر للأخ التالي |
| `_flags` | `unsigned long` | أعلام حالة العقدة (`OF_DYNAMIC` إلخ) |
| `data` | `void *` | بيانات خاصة بالمعمارية |
| `unique_id` | `unsigned int` | (SPARC فقط) معرِّف فريد |
| `irq_trans` | `struct of_irq_controller *` | (SPARC فقط) مترجم الـ IRQ |

---

### مخطط علاقات الـ Structs (ASCII)

```
  ┌─────────────────────────────────────────────────────────┐
  │                   struct of_pdt_ops                     │
  │  nextprop()  getproplen()  getproperty()                │
  │  getchild()  getsibling()  pkg2path()                   │
  └──────────────────────────┬──────────────────────────────┘
                             │  يُستخدَم عبر
                             │  of_pdt_prom_ops (global ptr)
                             ▼
  ┌──────────────────────────────────────────────────────────┐
  │                  struct device_node                      │
  │                                                          │
  │  name ──────────────────► "cpus"                        │
  │  full_name ─────────────► "/cpus"                       │
  │  phandle ───────────────► 0x03 (PROM handle)            │
  │  fwnode ────────────────► struct fwnode_handle           │
  │                                                          │
  │  properties ────────────────────────────────────────┐   │
  │  parent ────────────────► device_node (root)        │   │
  │  child  ────────────────► device_node (cpu@0)       │   │
  │  sibling ───────────────► device_node (memory)      │   │
  └─────────────────────────────────────────────────────│───┘
                                                        │
                             ┌──────────────────────────┘
                             ▼
  ┌────────────────────────────────────────┐
  │           struct property              │
  │                                        │
  │  name   ──► ".node"                   │
  │  length ──► 4                         │
  │  value  ──► &phandle                  │
  │  next ──────────────────────────────┐ │
  └─────────────────────────────────────│─┘
                                        │
                                        ▼
  ┌────────────────────────────────────────┐
  │           struct property              │
  │  name   ──► "compatible"              │
  │  length ──► 12                        │
  │  value  ──► "sun4u\0..."              │
  │  next ──────────────────────────────┐ │
  └─────────────────────────────────────│─┘
                                        │
                                        ▼
                                      NULL
```

---

### مخطط شجرة الـ device_node في الذاكرة

```
of_root
  │
  ├─► [/] device_node
  │      parent = NULL
  │      child ──────────────────────────────────┐
  │      sibling = NULL                          │
  │                                              ▼
  │                                     [/cpus] device_node
  │                                        parent = of_root
  │                                        child ──────────┐
  │                                        sibling ──────► [/memory] device_node
  │                                                │            sibling ──► [/chosen] ...
  │                                                ▼
  │                                      [/cpus/cpu@0] device_node
  │                                         parent = /cpus
  │                                         child = NULL
  │                                         sibling ──► [/cpus/cpu@1] ...
```

---

### مخطط دورة الحياة: من PROM إلى شجرة Kernel

```
  [Boot]
    │
    ▼
  معمارية SPARC/x86 تُنشئ struct of_pdt_ops
  وتملأ مؤشرات الدوال من PROM firmware
    │
    ▼
  of_pdt_build_devicetree(root_phandle, ops)
    │  ① يُسجِّل ops في of_pdt_prom_ops
    │  ② يُنشئ العقدة الجذر
    │  ③ يضع full_name = "/"
    │  ④ يستدعي of_pdt_build_tree() للأبناء
    │  ⑤ يستدعي of_alias_scan() لتسجيل /aliases و/chosen
    │
    ▼
  of_pdt_create_node(phandle, parent)
    │  ① prom_early_alloc(sizeof(device_node))
    │  ② of_node_init() — يضبط refcount=1، يُسجِّل fwnode_ops
    │  ③ of_pdt_incr_unique_id() (SPARC فقط)
    │  ④ يجلب property "name" من PROM
    │  ⑤ يبني قائمة properties
    │  ⑥ يبني full_name عبر pkg2path أو fallback
    │  ⑦ irq_trans_init() (SPARC فقط)
    │
    ▼
  of_pdt_build_prop_list(phandle)
    │  ① يُنشئ property ".node" كأول عنصر (phandle كقيمة)
    │  ② يُكرِّر nextprop() لجلب كل الأسماء
    │  ③ لكل اسم: getproplen() ثم getproperty()
    │  ④ يبني قائمة مرتبطة من struct property
    │
    ▼
  of_pdt_build_tree(parent, child_phandle)   [recursive]
    │  ① يُنشئ عقدة لكل phandle
    │  ② يربط siblings في قائمة أفقية
    │  ③ يتكرر بشكل عميق على getchild()
    │  ④ يتقدم أفقياً عبر getsibling()
    │
    ▼
  [شجرة of_root جاهزة في الذاكرة — kernel يبدأ استخدامها]
```

---

### مخطط تدفق الاستدعاء التفصيلي

```
arch_init() / prom_build_devicetree()
  └─► of_pdt_build_devicetree(root_node, ops)
        ├─ BUG_ON(!ops)
        ├─ of_pdt_prom_ops = ops          [global init]
        ├─ of_root = of_pdt_create_node(root_node, NULL)
        │     ├─ prom_early_alloc(sizeof(*dp))
        │     ├─ of_node_init(dp)
        │     │     └─ fwnode_init(&dp->fwnode, &of_fwnode_ops)
        │     ├─ dp->name = of_pdt_get_one_property(node, "name")
        │     │     ├─ ops->getproplen(node, "name")
        │     │     └─ ops->getproperty(node, "name", buf, len)
        │     ├─ dp->phandle = node
        │     ├─ dp->properties = of_pdt_build_prop_list(node)
        │     │     ├─ of_pdt_build_one_prop(node, NULL, ".node", &node, 4)
        │     │     │     └─ prom_early_alloc(sizeof(property) + 32)
        │     │     └─ loop: of_pdt_build_one_prop(node, prev, NULL,...)
        │     │           ├─ ops->nextprop(node, prev, p->name)
        │     │           ├─ ops->getproplen(node, p->name)
        │     │           └─ ops->getproperty(node, p->name, buf, len)
        │     └─ dp->full_name = of_pdt_build_full_name(dp)
        │           ├─ [non-SPARC]: ops->pkg2path(dp->phandle, path, ...)
        │           └─ [fallback]: "name@unknownN"
        ├─ of_root->full_name = "/"
        ├─ of_root->child = of_pdt_build_tree(of_root, ops->getchild(root))
        │     └─ loop per sibling:
        │           ├─ of_pdt_create_node(node, parent)   [كما أعلاه]
        │           ├─ dp->child = of_pdt_build_tree(dp, ops->getchild(node))
        │           │                                      [تكرار عميق]
        │           └─ node = ops->getsibling(node)
        └─ of_alias_scan(kernel_tree_alloc)
              └─ kernel_tree_alloc() → prom_early_alloc()
```

---

### استراتيجية الـ Locking

#### ملخص

| العنصر | نوع الحماية | السبب |
|--------|------------|-------|
| `of_pdt_prom_ops` | بلا قفل | `__initdata` — يُكتَب مرة واحدة قبل تشغيل أي core آخر |
| `of_pdt_unique_id` | بلا قفل | `__initdata` — يُعدَّل فقط في early-boot على SPARC، وحيد المسار |
| `prom_early_alloc` | بلا قفل | يعمل في single-thread boot قبل SMP |
| `of_root` وشجرة `device_node` | بعد البناء: `of_node_lock` | بعد `of_alias_scan`، يبدأ kernel استخدام قفل الـ OF tree |
| `struct property` المبنية | بلا قفل أثناء البناء | البناء يحدث قبل أي وصول متزامن |

#### التحليل:
- **الـ `of_pdt` يعمل كلياً في early-boot** — مرحلة single-CPU، بدون scheduler، بدون SMP. لذا لا يحتاج إلى قفل داخلي.
- بعد انتهاء `of_pdt_build_devicetree()`، تصبح شجرة `of_root` ملكاً للـ OF core العام الذي يحميها بـ `of_node_lock` (spinlock) وبـ `of_node_get/put` للـ reference counting.
- المتغير `tmp` الثابت في `of_pdt_build_one_prop()` آمن لأن الدالة لا تُستدعى بشكل متزامن.

#### ترتيب الأقفال بعد البناء (OF core):
```
of_node_lock (spinlock)
  └─► يحمي: قوائم properties، deadprops، sibling/child pointers
        يجب الحصول عليه قبل تعديل أي حقل في device_node أو property
```
## Phase 4: شرح الـ Functions

---

### ملخص عام — جدول الـ Functions والـ APIs

| Function / Op | النوع | الملف | الغرض الرئيسي |
|---|---|---|---|
| `of_pdt_build_devicetree` | Public API | `pdt.c` | نقطة الدخول الوحيدة — يبني شجرة الـ device tree كاملة من الـ PROM |
| `prom_early_alloc` | Public API | `of_pdt.h` | allocator مبكر يُستخدم قبل تفعيل الـ slab allocator |
| `of_pdt_create_node` | Internal `__init` | `pdt.c` | ينشئ `struct device_node` واحداً لـ phandle معين |
| `of_pdt_build_tree` | Internal `__init` | `pdt.c` | يمشي الـ PROM tree بالـ DFS ويبني siblings/children |
| `of_pdt_build_prop_list` | Internal `__init` | `pdt.c` | يبني linked list من كل properties لـ node معين |
| `of_pdt_build_one_prop` | Internal `__init` | `pdt.c` | يبني `struct property` واحدة (اسم + قيمة) |
| `of_pdt_get_one_property` | Internal `__init` | `pdt.c` | يقرأ قيمة property واحدة من الـ PROM |
| `of_pdt_build_full_name` | Internal `__init` | `pdt.c` | يبني الـ full path string للـ node |
| `kernel_tree_alloc` | Internal `__init` | `pdt.c` | wrapper حول `prom_early_alloc` يُمرَّر لـ `of_alias_scan` |
| `of_pdt_incr_unique_id` | Macro/Inline | `pdt.c` | يُعيِّن `unique_id` تصاعدياً (خاص بـ SPARC) |
| **`struct of_pdt_ops`** | Interface struct | `of_pdt.h` | مجموعة function pointers تمثّل واجهة الـ PROM |

---

### ops callbacks — جدول سريع لـ `struct of_pdt_ops`

| الـ Op | Signature | القيمة المُعادة | الغرض |
|---|---|---|---|
| `nextprop` | `int (*)(phandle, char *prev, char *buf)` | 0 نجاح، غير 0 خطأ | يجلب اسم الـ property التالية |
| `getproplen` | `int (*)(phandle, const char *prop)` | طول الـ property أو -1 | يعيد حجم قيمة الـ property |
| `getproperty` | `int (*)(phandle, const char *prop, char *buf, int bufsize)` | طول البيانات أو -1 | يقرأ قيمة الـ property |
| `getchild` | `phandle (*)(phandle parent)` | phandle أو 0 | يُعيد أول child لـ node معين |
| `getsibling` | `phandle (*)(phandle node)` | phandle أو 0 | يُعيد الـ sibling التالي |
| `pkg2path` | `int (*)(phandle, char *buf, int buflen, int *len)` | 0 نجاح، غير 0 خطأ | يحوّل الـ phandle إلى full path string |

---

### المجموعة الأولى: الـ Public API

#### `of_pdt_build_devicetree`

```c
void __init of_pdt_build_devicetree(phandle root_node, struct of_pdt_ops *ops);
```

**الـ entry point الوحيد** لبناء الـ device tree من الـ Open Firmware PROM. تُستدعى مرة واحدة فقط أثناء الـ boot، وتبني `of_root` وكل النسب من تحتها، ثم تُشغّل `of_alias_scan` لتعبئة `of_aliases` و `of_chosen`.

**المعاملات:**
- `root_node`: الـ `phandle` للـ root node كما يُعيده الـ PROM (عادةً ما يكون phandle 1 أو ما يُعادله في الـ firmware).
- `ops`: مؤشر لـ `struct of_pdt_ops` مليء بـ callbacks تُناسب الـ firmware الفعلي (SPARC OBP، OLPC PROM، إلخ).

**القيمة المُعادة:** لا شيء (`void`).

**التفاصيل الجوهرية:**
- تستدعي `BUG_ON(!ops)` مباشرةً — لا تسامح مع `NULL` ops.
- تُسند `ops` إلى المتغير الـ static `of_pdt_prom_ops` الذي يُستخدم داخلياً في كل functions الـ `__init`.
- تُعيد كتابة `of_root->full_name` إلى `"/"` يدوياً بعد إنشاء الـ root node، لأن `of_pdt_build_full_name` قد تُعيد اسماً غير صحيح للـ root.
- تنتهي بـ `of_alias_scan(kernel_tree_alloc)` لبناء قائمة الـ aliases.

**السياق:** تُستدعى من `setup_arch()` في معمارية SPARC أو من `x86/olpc_dt.c`، دائماً في سياق الـ boot الأحادي (single CPU، قبل SMP).

**Pseudocode:**

```
of_pdt_build_devicetree(root_node, ops):
    BUG_ON(!ops)
    of_pdt_prom_ops = ops                       // store globally
    of_root = of_pdt_create_node(root_node, NULL)
    of_root->full_name = "/"                    // override root name
    of_root->child = of_pdt_build_tree(of_root,
                         ops->getchild(of_root->phandle))
    of_alias_scan(kernel_tree_alloc)            // populate aliases/chosen
```

---

#### `prom_early_alloc`

```c
extern void *prom_early_alloc(unsigned long size);
```

**الـ allocator المبكر** الذي يُستخدم قبل أن يصبح الـ kernel heap جاهزاً. مُعرَّف خارج هذه الوحدة (في كود الـ architecture — مثلاً `arch/sparc/` أو `arch/x86/`). يُخصص ذاكرة من منطقة fixed أو من memblock، ولا يُرجَع هذه الذاكرة أبداً (لا يوجد `prom_early_free`).

**المعاملات:**
- `size`: عدد البايتات المطلوبة.

**القيمة المُعادة:** مؤشر للذاكرة المُخصصة (مضمونة صفرية أو غير صفرية حسب التنفيذ). في حال الفشل سيـ `panic` الـ kernel (الـ boot-time allocators لا تُعيد NULL عادةً).

**ملاحظة:** الـ `__initdata` marker على `of_pdt_prom_ops` يعني أن كل هذه البنية تُحرَّر بعد انتهاء الـ init.

---

### المجموعة الثانية: بناء الـ Node — Internal Functions

#### `of_pdt_create_node`

```c
static struct device_node * __init of_pdt_create_node(phandle node,
                                                       struct device_node *parent);
```

يُنشئ `struct device_node` كاملاً لـ phandle واحد بالاستعلام عن الـ PROM. يربط الـ node بـ parent ويملؤه بالاسم والـ phandle والـ properties والـ full_name.

**المعاملات:**
- `node`: الـ `phandle` من الـ PROM للـ node المطلوب بناؤه.
- `parent`: مؤشر للـ parent node المُنشأ سابقاً (أو `NULL` للـ root).

**القيمة المُعادة:** مؤشر لـ `struct device_node` مُخصص من `prom_early_alloc`، أو `NULL` إذا كان `node == 0` (أي لا يوجد node).

**التفاصيل:**
- يستدعي `of_node_init(dp)` الذي يُهيئ الـ `fwnode_handle` ويضبط الـ refcount.
- يملأ `dp->name` من property `"name"` في الـ PROM عبر `of_pdt_get_one_property`.
- يستدعي `of_pdt_build_prop_list` لبناء linked list من كل الـ properties.
- يستدعي `of_pdt_build_full_name` لتحديد الـ `full_name` (سيُعاد تعيينه يدوياً للـ root لاحقاً).
- في SPARC: يستدعي `irq_trans_init(dp)` لتهيئة مترجم الـ interrupts.

**Pseudocode:**

```
of_pdt_create_node(node, parent):
    if node == 0: return NULL
    dp = prom_early_alloc(sizeof(*dp))
    of_node_init(dp)                    // init fwnode + refcount
    dp->parent   = parent
    dp->name     = get_prop(node, "name")
    dp->phandle  = node
    dp->properties = build_prop_list(node)
    dp->full_name  = build_full_name(dp)
    irq_trans_init(dp)                  // SPARC only
    return dp
```

---

#### `of_pdt_build_tree`

```c
static struct device_node * __init of_pdt_build_tree(struct device_node *parent,
                                                      phandle node);
```

يمشي الـ PROM tree بـ DFS iterative للـ siblings و recursive للـ children. يبني linked list من الـ siblings ويُعيد رأس القائمة.

**المعاملات:**
- `parent`: الـ parent node المُنشأ مسبقاً لربط الـ children به.
- `node`: الـ `phandle` لأول sibling في المستوى الحالي.

**القيمة المُعادة:** مؤشر لأول `device_node` في مستوى الـ siblings، أو `NULL` إذا كان المستوى فارغاً.

**التفاصيل:**
- الحلقة تُكمل عبر `getsibling()` في كل دورة حتى تُعيد 0.
- كل node تُستدعى عليها `of_pdt_build_tree` recursively لبناء الـ children: `dp->child = of_pdt_build_tree(dp, getchild(node))`.
- يبني الـ sibling chain بتعيين `prev_sibling->sibling = dp` في كل دورة.

**ASCII Diagram:**

```
PROM Tree:                   Kernel device_node Tree:

 root                         of_root
  ├─ cpu                       ├─ child → cpu ─── sibling → memory
  ├─ memory                                └─ child → core0 ─ sibling → core1
  │    └─ none
  └─ aliases
```

**Pseudocode:**

```
of_pdt_build_tree(parent, node):
    ret = NULL, prev = NULL
    while True:
        dp = of_pdt_create_node(node, parent)
        if !dp: break
        if prev: prev->sibling = dp
        if !ret: ret = dp
        prev = dp
        dp->child = of_pdt_build_tree(dp, ops->getchild(node))
        node = ops->getsibling(node)
    return ret
```

---

### المجموعة الثالثة: بناء الـ Properties — Internal Functions

#### `of_pdt_build_prop_list`

```c
static struct property * __init of_pdt_build_prop_list(phandle node);
```

يبني linked list كاملة من `struct property` لكل properties الـ node في الـ PROM. يبدأ دائماً بـ property خاصة اسمها `.node` تحتوي على قيمة الـ `phandle` نفسه.

**المعاملات:**
- `node`: الـ `phandle` للـ node المُستعلَم عنه.

**القيمة المُعادة:** مؤشر لأول `struct property` في القائمة (`.node` دائماً).

**التفاصيل:**
- الـ property الأولى `.node` هي pseudo-property تحمل الـ phandle الرقمي لتسهيل الـ lookup.
- تستدعي `of_pdt_build_one_prop(node, NULL, NULL, NULL, 0)` لأول property حقيقية (تبدأ iteration من أول property في الـ PROM).
- تستمر في الاستدعاء بتمرير `tail->name` كـ `prev` حتى تُعيد `NULL` (أي لا مزيد من الـ properties).

**بنية القائمة الناتجة:**

```
head → [.node | phandle_val] → [prop1 | val1] → [prop2 | val2] → ... → NULL
```

---

#### `of_pdt_build_one_prop`

```c
static struct property * __init of_pdt_build_one_prop(phandle node, char *prev,
                                                       char *special_name,
                                                       void *special_val,
                                                       int special_len);
```

يبني `struct property` واحدة إما من بيانات يُمرَّر مباشرةً (`special_*`) أو بالاستعلام عن الـ PROM. هو الـ function الأكثر تعقيداً في هذه الوحدة وله منطق reuse للذاكرة.

**المعاملات:**
- `node`: الـ `phandle` للـ node المالك للـ property.
- `prev`: اسم الـ property السابقة (لكي يُعيد الـ PROM التالية)؛ `NULL` للأولى.
- `special_name`: إذا غير `NULL` يُنشئ property بهذا الاسم مباشرةً (لا يستعلم الـ PROM).
- `special_val`: قيمة الـ special property.
- `special_len`: طول `special_val` بالبايتات.

**القيمة المُعادة:** مؤشر لـ `struct property` مُنشأة، أو `NULL` إذا لم تعد هناك properties.

**تفاصيل جوهرية:**
- يستخدم متغير `static struct property *tmp` لإعادة استخدام آخر كتلة ذاكرة في حال أن الـ property الأخيرة كانت empty (لا اسم لها) — هذا optimization لتجنب낭낭낭 낭 낭 낭 낭 낭 낭 낭 الـ waste للذاكرة الـ boot-time.
- عندما يُعيد `nextprop()` خطأً (لا مزيد)، تُحفظ الـ property غير المُستخدمة في `tmp` لاستخدامها في الدورة التالية.
- اسم الـ property يُخزَّن مباشرةً بعد `struct property` في نفس كتلة الـ 32 بايت: `p->name = (char *)(p + 1)`.
- يضيف null terminator يدوياً بعد القيمة: `((unsigned char *)p->value)[p->length] = '\0'`.
- إذا كان `getproplen` يُعيد `<= 0` تُعيَّن `p->length = 0` ولا تُخصص ذاكرة للقيمة.

**Pseudocode:**

```
of_pdt_build_one_prop(node, prev, special_name, special_val, special_len):
    // reuse saved allocation if available
    if tmp != NULL:
        p = tmp; memset(p, 0, sizeof(*p) + 32); tmp = NULL
    else:
        p = prom_early_alloc(sizeof(struct property) + 32)

    p->name = (char*)(p + 1)   // name buffer follows struct in memory

    if special_name:
        strcpy(p->name, special_name)
        p->length = special_len
        p->value  = prom_early_alloc(special_len)
        memcpy(p->value, special_val, special_len)
    else:
        err = ops->nextprop(node, prev, p->name)
        if err:
            tmp = p            // save for reuse
            return NULL        // no more properties
        p->length = ops->getproplen(node, p->name)
        if p->length > 0:
            p->value = prom_early_alloc(p->length + 1)
            len = ops->getproperty(node, p->name, p->value, p->length)
            if len <= 0: p->length = 0
            p->value[p->length] = '\0'   // null terminate

    return p
```

---

#### `of_pdt_get_one_property`

```c
static char * __init of_pdt_get_one_property(phandle node, const char *name);
```

يقرأ قيمة property واحدة من الـ PROM ويُعيدها كـ `char *` مُخصصة من `prom_early_alloc`. يُستخدم حصراً لجلب `"name"` property في `of_pdt_create_node`.

**المعاملات:**
- `node`: الـ `phandle` للـ node.
- `name`: اسم الـ property المطلوبة (مثل `"name"`).

**القيمة المُعادة:** مؤشر لـ buffer تحتوي قيمة الـ property، أو الـ literal string `"<NULL>"` إذا كانت الـ property غير موجودة أو طولها صفر. لا يُعيد `NULL`.

**تفاصيل:**
- استخدام `"<NULL>"` بدلاً من `NULL` يحمي بقية الكود من الـ null dereference عند طباعة `dp->name`.

---

### المجموعة الرابعة: بناء الـ Full Name — معالجة الـ Path

#### `of_pdt_build_full_name` (non-SPARC)

```c
static char * __init of_pdt_build_full_name(struct device_node *dp);
```

يُنشئ اسم كامل مناسب للـ node (الجزء الأخير من الـ path) باستخدام `pkg2path` ops callback. في حال الفشل يُنشئ اسماً بديلاً من الـ `"name"` property وعداد `failsafe_id`.

**المعاملات:**
- `dp`: الـ `device_node` المُنشأ حديثاً (لابد أن يكون `dp->phandle` مُعيَّناً مسبقاً).

**القيمة المُعادة:** مؤشر لـ string مُخصص من `prom_early_alloc` يحتوي اسم الـ node (الجزء الأخير من الـ path، مثل `"cpu@0"` أو `"memory@80000000"`).

**التفاصيل:**
- يستدعي `ops->pkg2path()` للحصول على الـ full path، ثم `kbasename()` لاستخراج الجزء الأخير.
- الـ fallback: `sprintf(buf, "%s@unknown%i", name, failsafe_id++)` مع `pr_err`.
- في SPARC: الـ implementation مختلفة — تستدعي `build_path_component(dp)` المُعرَّفة في `arch/sparc`.

---

#### `of_pdt_build_full_name` (SPARC)

```c
static char * __init of_pdt_build_full_name(struct device_node *dp);
```

**الـ SPARC variant** يفوّض العمل بالكامل إلى `build_path_component(dp)` المُعرَّفة في `arch/sparc/kernel/prom_common.c`، وتتعامل مع الـ OBP (Open Boot PROM) naming conventions الخاصة بـ SPARC.

---

### المجموعة الخامسة: الـ Helpers والـ Macros

#### `kernel_tree_alloc`

```c
static void * __init kernel_tree_alloc(u64 size, u64 align);
```

Wrapper بسيط حول `prom_early_alloc` بتوقيع متوافق مع ما تتوقعه `of_alias_scan`. يتجاهل معامل `align` لأن `prom_early_alloc` لا تدعمه.

**المعاملات:**
- `size`: حجم الـ allocation المطلوب.
- `align`: محاذاة مطلوبة (مُتجاهَلة في الـ implementation).

**القيمة المُعادة:** نفس ما تُعيده `prom_early_alloc(size)`.

---

#### `of_pdt_incr_unique_id` (SPARC فقط)

```c
/* Macro: */
#define of_pdt_incr_unique_id(p) do { (p)->unique_id = of_pdt_unique_id++; } while (0)

/* non-SPARC: */
static inline void of_pdt_incr_unique_id(void *p) { }
```

يُعيِّن `unique_id` تصاعدياً لكل `struct device_node` و`struct property` في SPARC. هذا الـ ID يُستخدم كـ stable identifier في الـ SPARC PROM subsystem. في الـ architectures الأخرى هو no-op.

**المعاملات:**
- `p`: مؤشر لأي struct يحتوي حقل `unique_id` (إما `device_node` أو `property`).

---

### خريطة تدفق البناء الكاملة (ASCII)

```
of_pdt_build_devicetree(root_phandle, ops)
│
├── BUG_ON(!ops)
├── of_pdt_prom_ops = ops
│
├── of_pdt_create_node(root_phandle, NULL)   ← ينشئ of_root
│   ├── prom_early_alloc(sizeof(device_node))
│   ├── of_node_init()                       ← fwnode + refcount
│   ├── of_pdt_get_one_property("name")      ← يقرأ من PROM
│   ├── of_pdt_build_prop_list()
│   │   ├── of_pdt_build_one_prop(.node, phandle_val)  ← pseudo-prop
│   │   └── of_pdt_build_one_prop(node, NULL, ...)     ← loop via nextprop
│   └── of_pdt_build_full_name()             ← pkg2path → kbasename
│
├── of_root->full_name = "/"                 ← override
│
├── of_pdt_build_tree(of_root, getchild(root))
│   └── [loop over siblings]
│       ├── of_pdt_create_node(sibling, parent)
│       └── dp->child = of_pdt_build_tree(dp, getchild(sibling))  ← recursive
│
└── of_alias_scan(kernel_tree_alloc)
    └── يملأ of_aliases و of_chosen
```

---

### علاقة `struct of_pdt_ops` بالـ Architecture

**الـ `struct of_pdt_ops`** هي الـ abstraction layer الرئيسية التي تفصل كود بناء الـ device tree عن تفاصيل الـ firmware لكل architecture. كل architecture تُهيئ هذه الـ struct بطريقتها:

| Architecture | الملف | الملاحظات |
|---|---|---|
| SPARC | `arch/sparc/kernel/prom_common.c` | يستخدم OBP (Open Boot PROM) calls مباشرةً |
| x86 OLPC | `arch/x86/platform/olpc/olpc_dt.c` | يستخدم OLPC firmware interface |

```c
/* مثال تهيئة من OLPC (x86): */
static struct of_pdt_ops olpc_pdt_ops __initdata = {
    .nextprop   = olpc_pdt_nextprop,
    .getproplen = olpc_pdt_getproplen,
    .getproperty= olpc_pdt_getproperty,
    .getchild   = olpc_pdt_getchild,
    .getsibling = olpc_pdt_getsibling,
    .pkg2path   = olpc_pdt_pkg2path,
};

/* ثم في boot: */
of_pdt_build_devicetree(root, &olpc_pdt_ops);
```

---

### نقاط مهمة للـ Embedded Developer

1. **الـ `__initdata` و `__init`**: كل هذه الكود يُحذف من الذاكرة بعد `free_initmem()`. لا يجوز الإبقاء على مؤشرات لها.
2. **الـ `prom_early_alloc` غير قابلة للتحرير**: الذاكرة المُخصصة تبقى طوال عمر الـ kernel. لهذا الـ device tree يُبنى مرة واحدة فقط.
3. **الـ `of_alias_scan`** في نهاية `of_pdt_build_devicetree` ضرورية: بدونها لن تعمل `of_find_node_by_alias()` وما يعتمد عليها من drivers.
4. **الـ `tmp` static في `of_pdt_build_one_prop`**: يمثّل optimization دقيقاً — يمنع낭낭낭 낭 낭 낭 낭 낭 낭 낭 낭낭낭낭낭낭낭 낭 낭 낭 낭 낭 낭 낭 낭낭낭낭낭낭낭 waste ذاكرة الـ boot allocator عند اكتشاف نهاية الـ properties.
5. **الـ `.node` pseudo-property**: موجودة في كل node وتحمل قيمة الـ `phandle` الرقمية، مما يُتيح الـ reverse lookup من الـ `device_node` إلى الـ PROM handle.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem المعني هو **OF PDT** (Open Firmware PROM Device Tree)، المسؤول عن بناء الـ device tree في الـ kernel من خلال استدعاء الـ PROM مباشرةً أثناء الـ boot على معماريات مثل SPARC. الكود الرئيسي في `drivers/of/pdt.c` والـ interface في `include/linux/of_pdt.h`.

---

### Software Level

#### 1. مدخلات الـ debugfs ذات الصلة

الـ OF PDT يعمل فقط خلال مرحلة الـ **early boot** (`__init`)، لذا لا توجد مدخلات debugfs مباشرة له. لكن نتيجة عمله — شجرة الـ device nodes — تظهر كاملةً عبر:

| المسار | الوصف |
|---|---|
| `/sys/firmware/devicetree/base/` | الشجرة الكاملة التي بناها OF PDT أو FDT |
| `/sys/kernel/debug/of/` | يوجد في بعض kernels لعرض OF nodes |

```bash
# قراءة خاصية من node معين
cat /sys/firmware/devicetree/base/chosen/bootargs

# سرد كل nodes في الجذر
ls /sys/firmware/devicetree/base/

# قراءة phandle لـ node
xxd /sys/firmware/devicetree/base/cpus/phandle
```

للتحقق أن `of_root` تم بناؤه بشكل صحيح:
```bash
# التحقق من وجود /chosen و /aliases
ls /sys/firmware/devicetree/base/chosen 2>/dev/null && echo "chosen OK" || echo "chosen MISSING"
ls /sys/firmware/devicetree/base/aliases 2>/dev/null && echo "aliases OK" || echo "aliases MISSING"
```

---

#### 2. مدخلات الـ sysfs ذات الصلة

| المسار | الوصف |
|---|---|
| `/sys/firmware/devicetree/base/` | تمثيل sysfs لشجرة OF بأكملها |
| `/sys/firmware/devicetree/base/#address-cells` | عدد خلايا العنوان في الجذر |
| `/sys/firmware/devicetree/base/#size-cells` | عدد خلايا الحجم في الجذر |
| `/sys/bus/platform/devices/` | الأجهزة المستخرجة من OF |

```bash
# قراءة compatible string لجهاز معين
cat /sys/firmware/devicetree/base/compatible | tr '\0' '\n'

# سرد خصائص node معين بصيغة hex
for f in /sys/firmware/devicetree/base/memory@*; do
    echo "=== $f ==="; xxd "$f/reg"
done
```

---

#### 3. الـ ftrace — الـ tracepoints والـ events

لا يوجد tracepoint مخصص لـ OF PDT لأنه يعمل في `__init`، لكن يمكن تتبع الدوال المرتبطة:

```bash
# تفعيل function tracer على دوال OF
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'of_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# مراقبة استدعاءات of_alias_scan وما يتبعها
echo of_alias_scan >> /sys/kernel/debug/tracing/set_ftrace_filter
echo of_node_init >> /sys/kernel/debug/tracing/set_ftrace_filter

# قراءة النتائج
cat /sys/kernel/debug/tracing/trace | head -50
```

لتتبع OF events الموجودة:
```bash
# سرد كل OF events المتاحة
ls /sys/kernel/debug/tracing/events/of/

# تفعيل كل OF events
echo 1 > /sys/kernel/debug/tracing/events/of/enable
cat /sys/kernel/debug/tracing/trace_pipe &
```

---

#### 4. الـ printk والـ dynamic debug

الـ OF PDT يستخدم `pr_err` عند فشل `pkg2path`. لتفعيل dynamic debug لكل OF subsystem:

```bash
# تفعيل كل debug messages في drivers/of/
echo 'file drivers/of/* +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug messages في pdt.c تحديداً
echo 'file drivers/of/pdt.c +p' > /sys/kernel/debug/dynamic_debug/control

# التحقق مما تم تفعيله
grep 'of/pdt' /sys/kernel/debug/dynamic_debug/control

# تفعيل مع line numbers وfunction names
echo 'file drivers/of/pdt.c +pflm' > /sys/kernel/debug/dynamic_debug/control
```

لرفع مستوى الـ printk مؤقتاً:
```bash
# طباعة كل رسائل KERN_DEBUG
echo 8 > /proc/sys/kernel/printk

# أو عبر kernel cmdline أثناء الـ boot (مهم لـ early boot messages)
# أضف للـ bootloader: loglevel=8 earlycon
```

---

#### 5. خيارات الـ kernel config للـ debugging

| CONFIG | الوصف |
|---|---|
| `CONFIG_OF` | تفعيل OF/DT الأساسي — ضروري |
| `CONFIG_OF_FLATTREE` | دعم FDT (Flattened DT) |
| `CONFIG_OF_EARLY_FLATTREE` | scanning مبكر لـ FDT |
| `CONFIG_OF_UNITTEST` | unit tests لـ OF subsystem |
| `CONFIG_OF_OVERLAY` | دعم DT overlays |
| `CONFIG_DEBUG_DRIVER` | debug عام لـ driver core |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug |
| `CONFIG_KALLSYMS` | أسماء الرموز في stack traces |
| `CONFIG_DEBUG_INFO` | معلومات DWARF للـ debugging |
| `CONFIG_SPARC` (على SPARC) | يفعّل `of_pdt_unique_id` ومسارات SPARC-specific |
| `CONFIG_PROVE_LOCKING` | للكشف عن deadlocks في OF lock |

```bash
# التحقق من تفعيل الخيارات
zcat /proc/config.gz | grep -E 'CONFIG_OF|CONFIG_DEBUG_DRIVER|CONFIG_DYNAMIC_DEBUG'
```

---

#### 6. أدوات خاصة بالـ subsystem

**أداة `dtc` (Device Tree Compiler):**
```bash
# تحويل الـ live device tree إلى DTB
dtc -I fs -O dts /sys/firmware/devicetree/base > /tmp/live_tree.dts
cat /tmp/live_tree.dts

# مقارنة الشجرة الحالية بالـ DTS المتوقع
diff expected.dts /tmp/live_tree.dts
```

**أداة `fdtdump`:**
```bash
# فحص DTB مباشرة
fdtdump /boot/dtb-$(uname -r) | less

# استخراج property معينة
fdtget /boot/dtb-$(uname -r) /chosen bootargs
```

**قراءة PROM properties مباشرة (على SPARC):**
```bash
# على SPARC — قراءة PROM tree
cat /proc/openprom/
ls /proc/openprom/

# قراءة property من PROM node
cat /proc/openprom/SUNW,UltraAX-i2/@0/name
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ kernel log | السبب | الإصلاح |
|---|---|---|
| `of_pdt_build_full_name: pkg2path failed; assigning <name>@unknownN` | فشل `ops->pkg2path` في تحويل phandle إلى مسار | تحقق من صحة تنفيذ `pkg2path` في الـ platform ops، تأكد أن الـ PROM يستجيب |
| `OF: fdt: No valid device tree found` | لم يُمرَّر FDT صحيح أو PDT لم يُستدعَ | تحقق من bootloader config، تأكد أن `of_pdt_build_devicetree()` تُستدعى |
| `OF: resolver: phandle collision ...` | تعارض في الـ phandle values | مشكلة في `of_pdt_unique_id` أو phandles متكررة في PROM |
| `BUG: of_pdt_build_devicetree called with NULL ops` | تمرير `NULL` كـ `ops` | تأكد أن `struct of_pdt_ops` ممتلئة قبل الاستدعاء |
| `WARNING: CPU: 0 PID: 0 at drivers/of/pdt.c:61` | فشل `pkg2path` مع `pr_err` | راجع تنفيذ PROM calls في الـ platform code |
| `of: property ... has invalid length` | `getproplen` تُعيد قيمة سالبة غير متوقعة | تحقق من الـ PROM ومن صحة الـ node phandle |
| `prom_early_alloc: out of memory` | نفد الـ early boot memory | زد `CONFIG_PROM_ALLOC_SIZE` أو راجع حجم الـ DT |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في of_pdt_build_devicetree — التحقق من ops */
void __init of_pdt_build_devicetree(phandle root_node, struct of_pdt_ops *ops)
{
    WARN_ON(!ops->nextprop);          /* يجب أن تكون جميع العمليات غير NULL */
    WARN_ON(!ops->getproplen);
    WARN_ON(!ops->getproperty);
    WARN_ON(!ops->getchild);
    WARN_ON(!ops->getsibling);
    BUG_ON(!ops);
    /* ... */
}

/* في of_pdt_create_node — عند فشل تخصيص الـ node */
static struct device_node * __init of_pdt_create_node(phandle node,
                                                       struct device_node *parent)
{
    struct device_node *dp;
    if (!node)
        return NULL;

    dp = prom_early_alloc(sizeof(*dp));
    if (WARN_ON(!dp)) {              /* نقطة استراتيجية: نفاد الـ early memory */
        dump_stack();
        return NULL;
    }
    /* ... */
}

/* في of_pdt_build_one_prop — عند فشل getproperty */
if (len <= 0) {
    WARN_ONCE(1, "getproperty failed for node %x prop %s\n",
              node, p->name);        /* تحذير لمرة واحدة لتفادي الـ log flood */
}
```

---

### Hardware Level

#### 1. التحقق أن حالة الـ hardware تطابق حالة الـ kernel

الـ OF PDT يعكس ما يقوله الـ PROM. التحقق يتم عبر:

```bash
# مقارنة ما قرأه الـ kernel من PROM مع ما يجب أن يكون
# على SPARC:
cat /proc/openprom/options/boot-device
# يجب أن يطابق ما في /sys/firmware/devicetree/base/chosen/

# التحقق من عدد الـ CPUs
cat /sys/firmware/devicetree/base/cpus/\#address-cells | xxd
ls /sys/firmware/devicetree/base/cpus/ | grep -c cpu

# مقارنة مع الـ CPUs الفعلية
nproc --all
```

#### 2. تقنيات الـ register dump

الـ PROM على SPARC يُعرَّض عادةً عبر `/dev/mem` أو عبر PROM calls. على SPARC UltraSPARC:

```bash
# قراءة عنوان ذاكرة معين (يتطلب CONFIG_DEVMEM=y)
# devmem2 <address> [type] [value]
devmem2 0xffd00000 w       # قراءة PROM base address كـ word

# أو مباشرة عبر /dev/mem
dd if=/dev/mem bs=4 count=1 skip=$((0xffd00000 / 4)) 2>/dev/null | xxd

# io utility للـ I/O ports (على x86 فقط)
# io -r -1 0x3f8    # قراءة byte من port
```

لقراءة الـ device tree من الـ PROM مباشرةً (قبل الـ kernel):
```
# في OpenBoot PROM على SPARC:
ok> show-devs          # يسرد كل devices في الـ PROM tree
ok> .properties        # يطبع properties للـ node الحالي
ok> " /memory" find-device .properties
```

#### 3. نصائح الـ logic analyzer والـ oscilloscope

الـ OF PDT يتعامل مع الـ PROM عبر software calls، لكن إذا كان الـ PROM يتواصل عبر bus:

```
SPARC PROM (OpenBoot) Interface:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  CPU  ──[trap]──▶  PROM ROM
                      │
                   [SBUS/PCIe]
                      │
                   Devices
```

- على الـ logic analyzer: راقب خط الـ **JTAG/SBUS** أثناء الـ boot لرصد PROM calls
- راقب خطوط **SDA/SCL** إذا كان هناك EEPROM/NVRAM يحمل الـ device tree
- الـ oscilloscope: تحقق من **power rails** للـ PROM chip أثناء الـ early boot
- ابحث عن **address strobe signals** على الـ memory bus أثناء `prom_early_alloc`

#### 4. مشاكل الـ hardware الشائعة وأنماط الـ kernel log

| المشكلة | نمط الـ log | الحل |
|---|---|---|
| PROM لا يستجيب | `OF: fdt: No valid device tree found` أو `BUG at pdt.c` | تحقق من الـ PROM chip وتوصيلاته |
| NVRAM تالفة (SPARC) | `prom: idprom checksum error` | استبدال NVRAM chip |
| فشل قراءة property | `pkg2path failed; assigning @unknownN` متكرر | تحقق من سلامة الـ PROM firmware |
| تعارض phandles | `phandle collision` أو nodes مكررة | تحديث PROM firmware أو إعادة flash |
| نفاد early memory | `prom_early_alloc: out of memory` | تقليل عدد DT nodes أو زيادة early alloc pool |

#### 5. الـ Device Tree debugging (التحقق من توافق DT مع الـ hardware)

```bash
# 1. استخراج الـ live DT من الـ kernel
dtc -I fs -O dts -o /tmp/live.dts /sys/firmware/devicetree/base 2>/dev/null

# 2. مقارنة مع الـ DTS المصدري
diff /path/to/board.dts /tmp/live.dts | head -40

# 3. التحقق من compatible strings
grep -r compatible /sys/firmware/devicetree/base/ | head -20

# 4. التحقق من صحة الـ phandles
dtc -I fs -O dts /sys/firmware/devicetree/base 2>&1 | grep -i 'error\|warning'

# 5. التحقق من أن الـ memory regions في DT تطابق الفعلية
cat /proc/iomem
# قارن مع:
fdtget /boot/dtb /memory reg | xargs printf "0x%08x\n"

# 6. على SPARC — التحقق من PROM version
cat /proc/openprom/openprom/version

# 7. فحص الـ aliases
cat /sys/firmware/devicetree/base/aliases/* 2>/dev/null
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**فحص شامل لـ OF PDT build:**
```bash
#!/bin/bash
# of_pdt_debug.sh — فحص شامل لنتائج OF PDT build

echo "=== OF Root Node ==="
ls /sys/firmware/devicetree/base/

echo ""
echo "=== Compatible String ==="
cat /sys/firmware/devicetree/base/compatible 2>/dev/null | tr '\0' '\n'

echo ""
echo "=== Chosen Node ==="
ls /sys/firmware/devicetree/base/chosen/ 2>/dev/null
cat /sys/firmware/devicetree/base/chosen/bootargs 2>/dev/null
echo ""

echo "=== CPU Count in DT ==="
ls /sys/firmware/devicetree/base/cpus/ 2>/dev/null | grep -c cpu@

echo ""
echo "=== Memory Nodes ==="
for mem in /sys/firmware/devicetree/base/memory*; do
    echo "Node: $mem"
    xxd "$mem/reg" 2>/dev/null
done

echo ""
echo "=== OF Kernel Messages ==="
dmesg | grep -iE 'OF:|prom:|devicetree|pdt|phandle' | head -30
```

**تفعيل dynamic debug لـ OF أثناء التشغيل:**
```bash
# تفعيل كل OF debug messages
echo 'file drivers/of/*.c +p' > /sys/kernel/debug/dynamic_debug/control

# مراقبة الـ kernel log في الوقت الفعلي
dmesg -w | grep -i 'of\|prom\|pdt'
```

**فحص PROM على SPARC:**
```bash
# سرد الـ PROM tree
ls -la /proc/openprom/

# قراءة PROM version
cat /proc/openprom/openprom/version 2>/dev/null

# قراءة board serial number من NVRAM
cat /proc/openprom/options/\#name 2>/dev/null
```

**استخراج وتحليل DT كامل:**
```bash
# تحويل DT الحي إلى DTS مقروء
dtc -I fs -O dts /sys/firmware/devicetree/base > /tmp/kernel_dt.dts 2>/dev/null
echo "DT size: $(wc -l < /tmp/kernel_dt.dts) lines"

# البحث عن node بـ compatible معينة
grep -A5 'compatible = ".*uart\|serial"' /tmp/kernel_dt.dts

# فحص كل phandles للتحقق من عدم التكرار
grep 'phandle' /tmp/kernel_dt.dts | sort | uniq -d
```

**فحص early boot messages المتعلقة بـ PDT:**
```bash
# البحث في dmesg عن رسائل early boot الخاصة بـ OF PDT
dmesg | grep -E 'pkg2path|prom_early|of_pdt|of_alias_scan|of_root'

# التحقق من وجود خطأ في بناء الـ full_name
dmesg | grep 'unknown'

# فحص عدد nodes التي تم بناؤها
find /sys/firmware/devicetree/base -type d | wc -l
```

#### مثال على الـ output وكيفية تفسيره

```
$ dmesg | grep -E 'OF:|prom:|pdt'

[    0.000000] OF: fdt: Ignoring memory range 0x0 - 0x1000
[    0.000000] OF: fdt: fdt_reserved_mem_save_node: found 2 reserved mem nodes
[    0.000000] OF: early_parse_phandle_with_args: found phandle 0x1
[    0.000000] OF: aliases: /aliases/serial0 -> /soc/serial@101f1000
```

**التفسير:**
- السطر الأول: `OF: fdt` يعني الـ FDT path (ليس PDT) — يُشير إلى أن boot جاء عبر FDT لا PROM مباشرة
- `found 2 reserved mem nodes`: الـ DT يحتوي على منطقتَي ذاكرة محجوزة — طبيعي
- `found phandle 0x1`: أول phandle — يجب أن يبدأ من 1 وليس 0
- `aliases/serial0`: `of_alias_scan()` نجحت — آخر خطوة في `of_pdt_build_devicetree`

```
$ find /sys/firmware/devicetree/base -type d | wc -l
142
# 142 device node — رقم معقول لـ embedded system
# إذا كان 0 أو 1 → فشل البناء
# إذا كان أكبر من 1000 → احتمال loop في getchild/getsibling
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Sparc-based Industrial Gateway — kernel panic أثناء بناء الـ device tree

#### العنوان
**الـ `pkg2path` callback يُعيد خطأ على بوابة صناعية قديمة تعمل بـ SPARC**

#### السياق
شركة تصنيع تستخدم بوابة صناعية (industrial gateway) مبنية على معالج SPARC قديم لجمع بيانات من أجهزة استشعار عبر I2C وإرسالها بـ Ethernet. البوابة تعمل منذ سنوات، وبعد تحديث الـ kernel من 5.15 إلى 6.6، أصبح الجهاز يتوقف مبكراً أثناء الـ boot.

#### المشكلة
الـ kernel يطبع:

```
of_pdt_build_full_name: pkg2path failed; assigning uart@unknown0
of_pdt_build_full_name: pkg2path failed; assigning i2c@unknown1
...
Kernel panic - not syncing: VFS: Unable to mount root fs
```

العقد في الـ device tree تأخذ أسماء مثل `uart@unknown0` بدل `uart@ff010000`، مما يُخفق في تطابق drivers مع العقد.

#### التحليل
في `pdt.c`، الدالة `of_pdt_build_full_name()`:

```c
static char * __init of_pdt_build_full_name(struct device_node *dp)
{
    ...
    if (!of_pdt_prom_ops->pkg2path(dp->phandle, path, sizeof(path), &len)) {
        name = kbasename(path);
        buf = prom_early_alloc(strlen(name) + 1);
        strcpy(buf, name);
        return buf;
    }

    /* fallback: pkg2path failed */
    name = of_get_property(dp, "name", &len);
    buf = prom_early_alloc(len + 16);
    sprintf(buf, "%s@unknown%i", name, failsafe_id++);
    pr_err("%s: pkg2path failed; ...\n", __func__, buf);
    return buf;
}
```

الـ `pkg2path` في `struct of_pdt_ops` (معرّفة في `of_pdt.h`) هي callback يجب أن يوفّرها الـ PROM driver:

```c
/* return 0 on success; fill in 'len' with number of bytes in path */
int (*pkg2path)(phandle node, char *buf, const int buflen, int *len);
```

في kernel 6.6 تغيّر حجم `buflen` المتوقع — الـ PROM القديم يعيد path أطول من `sizeof(path) = 256`، فالـ callback يُفشل العملية ويُعيد كود خطأ غير صفري.

#### الحل

**1. تشخيص المشكلة:**

```bash
# طباعة رسائل الـ early boot كاملة
dmesg | grep "pkg2path"
dmesg | grep "unknown"
```

**2. تعديل الـ PROM driver** الذي يُنفّذ `pkg2path` لاستيعاب paths أطول:

```c
/* في ملف arch/sparc/prom/tree.c أو ما يعادله */
static int sparc_pkg2path(phandle node, char *buf, const int buflen, int *len)
{
    /* increase internal buffer to 512 to handle legacy PROM long paths */
    char tmp[512];
    int ret = prom_pathtranslate(node, tmp, sizeof(tmp));
    if (ret)
        return ret;
    *len = strlen(tmp);
    if (*len >= buflen)
        return -ENOMEM;   /* signal failure cleanly */
    strcpy(buf, tmp);
    return 0;
}
```

**3. في `of_pdt_build_full_name`** زيادة حجم الـ buffer المحلي:

```c
char path[512];  /* was 256 — handle older PROM with long paths */
```

#### الدرس المستفاد
الـ `of_pdt_ops` هي **واجهة تجريدية** بين الـ kernel وبين PROM محدد. أي تغيير في افتراضات الأحجام (buffer sizes) في أحد الطرفين يكسر الطرف الآخر. عند ترقية الـ kernel على منصات SPARC قديمة، يجب مراجعة كل callback في `struct of_pdt_ops` وتحقق من التوافق مع الـ PROM الفعلي.

---

### السيناريو الثاني: Android TV Box بمعالج Allwinner H616 — خصائص مفقودة في الـ device tree

#### العنوان
**`getproplen` يُعيد `-1` لخاصية `reg` في عقدة HDMI — الشاشة لا تعمل**

#### السياق
جهاز Android TV box يستخدم Allwinner H616. الـ HDMI يعمل بشكل طبيعي في الإصدار السابق من الـ firmware، لكن بعد تحديث الـ bootloader الذي يبني الـ device tree ديناميكياً عبر PROM interface، أصبح HDMI غير معترف به.

#### المشكلة
```
sunxi-hdmi 6000000.hdmi: reg property not found in DT node
sunxi-hdmi 6000000.hdmi: probe failed: -EINVAL
```

الـ driver لا يجد خاصية `reg` في العقدة رغم أن الـ bootloader يحتوي عليها.

#### التحليل
في `pdt.c`، الدالة `of_pdt_build_one_prop()` تستدعي:

```c
p->length = of_pdt_prom_ops->getproplen(node, p->name);
if (p->length <= 0) {
    p->length = 0;   /* property ignored silently! */
} else {
    ...
    len = of_pdt_prom_ops->getproperty(node, p->name,
            p->value, p->length);
    if (len <= 0)
        p->length = 0;
}
```

الـ `getproplen` callback (من `of_pdt.h`) يجب أن يُعيد طول الخاصية أو `-1` عند الخطأ:

```c
/* for both functions, return proplen on success; -1 on error */
int (*getproplen)(phandle node, const char *prop);
```

في الـ bootloader الجديد، تم تغيير الـ encoding لخاصية `reg` — أصبحت تُخزَّن بـ big-endian بينما كانت little-endian سابقاً. `getproplen` يعمل بشكل صحيح ويُعيد `8`، لكن `getproperty` يُعيد القيمة بـ encoding خاطئ فيُفشل تحقق الـ driver.

أما الخطأ الصامت (`p->length = 0`) يعني أن الخاصية تُبنى في الـ tree لكن بطول صفر — الـ driver يجدها لكن بيانات فارغة.

#### الحل

**1. تتبع المشكلة عبر debugfs:**

```bash
# بعد الـ boot
cat /sys/firmware/devicetree/base/hdmi@6000000/reg | xxd
# إذا كان الإخراج فارغاً أو خاطئاً، المشكلة في getproperty
```

**2. إضافة debug مؤقت في `of_pdt_build_one_prop`:**

```c
len = of_pdt_prom_ops->getproperty(node, p->name,
        p->value, p->length);
if (len <= 0) {
    pr_warn("of_pdt: getproperty('%s') returned %d\n",
            p->name, len);
    p->length = 0;
}
```

**3. تصحيح الـ bootloader** ليُعيد `reg` بـ big-endian صحيح وفق مواصفات DT.

#### الدرس المستفاد
الخطأ الصامت `p->length = 0` في `of_pdt_build_one_prop` يجعل الـ debugging صعباً لأن العقدة تظهر في الـ tree لكن بخصائص فارغة. عند تطوير الـ bootloader لأجهزة Allwinner، يجب اختبار كل callback في `of_pdt_ops` بشكل مستقل قبل تكاملها مع الـ kernel.

---

### السيناريو الثالث: STM32MP1 IoT Sensor Board — تعليق الـ kernel في حلقة `of_pdt_build_tree`

#### العنوان
**الـ `getsibling` يُعيد نفس الـ phandle — حلقة لا نهائية أثناء بناء الـ device tree**

#### السياق
لوحة IoT مخصصة تستخدم STM32MP1 لقراءة بيانات من حساسات عبر SPI وإرسالها بـ LoRa. الفريق يطور PROM abstraction layer خاصاً لاستبدال DTB الثابت ببناء ديناميكي من الـ flash. أثناء أول boot بعد تفعيل `CONFIG_OF_PROMTREE`، الـ kernel يتعلق إلى الأبد.

#### المشكلة
```
[    0.000000] OF: fdt: Machine model: Custom STM32MP1 IoT Board
[    0.000000] <hang — no further output>
```

لا kernel panic، لا watchdog trigger (الـ watchdog لم يُفعّل بعد في هذه المرحلة).

#### التحليل
الدالة `of_pdt_build_tree()` في `pdt.c`:

```c
static struct device_node * __init of_pdt_build_tree(struct device_node *parent,
                                                   phandle node)
{
    struct device_node *ret = NULL, *prev_sibling = NULL;
    struct device_node *dp;

    while (1) {
        dp = of_pdt_create_node(node, parent);
        if (!dp)
            break;              /* exits only if node == 0 */
        ...
        node = of_pdt_prom_ops->getsibling(node);  /* must reach 0 */
    }
    return ret;
}
```

الـ `getsibling` callback (من `of_pdt.h`) يجب أن تُعيد `0` عندما لا يوجد sibling:

```c
/* phandles are 0 if no child or sibling exists */
phandle (*getsibling)(phandle node);
```

في الـ implementation الخاطئة:

```c
/* BUG: returns node itself if no sibling found */
static phandle my_getsibling(phandle node)
{
    struct prom_node *n = find_node(node);
    if (!n || !n->sibling)
        return node;   /* should return 0! */
    return n->sibling->phandle;
}
```

هذا يسبب حلقة لا نهائية لأن `of_pdt_create_node()` لا تُعيد `NULL` إلا إذا كان `node == 0`.

#### الحل

**1. إصلاح الـ `getsibling` callback فوراً:**

```c
static phandle my_getsibling(phandle node)
{
    struct prom_node *n = find_node(node);
    if (!n || !n->sibling)
        return 0;   /* correct: 0 means no sibling */
    return n->sibling->phandle;
}
```

**2. إضافة guard مؤقت للـ debugging:**

```c
/* in of_pdt_build_tree — add cycle detection */
int max_siblings = 1000;
while (1) {
    if (--max_siblings <= 0) {
        pr_err("of_pdt: sibling loop detected at phandle %u\n", node);
        break;
    }
    ...
}
```

**3. كذلك يجب تصحيح `getchild`** بنفس المنطق.

#### الدرس المستفاد
قاعدة أساسية في `of_pdt.h` مكتوبة صراحة: **`phandles are 0 if no child or sibling exists`**. أي implementation لـ `getchild` أو `getsibling` يجب أن تُعيد `0` وليس قيمة أخرى عند نهاية القائمة. عند كتابة PROM abstraction layer جديد لـ STM32MP1 أو غيره، هذا الشرط غير قابل للتفاوض.

---

### السيناريو الرابع: i.MX8 Automotive ECU — خاصية `.node` مكررة تُسبب crash في `of_alias_scan`

#### العنوان
**`of_pdt_build_prop_list` تبني خاصية `.node` مكررة على ECU بمعالج i.MX8**

#### السياق
وحدة تحكم إلكترونية (ECU) لسيارة كهربائية تستخدم NXP i.MX8. الـ ECU يتواصل مع شبكة CAN وعدة buses عبر I2C وSPI. الفريق ينقل الـ BSP من نظام PROM قديم إلى نظام DTB هجين — الـ kernel يستخدم `of_pdt_build_devicetree` لبناء جزء من الـ tree ثم يدمجه مع DTB ثابت. بعد الدمج، الـ kernel يُعطي `BUG_ON` في `of_alias_scan`.

#### المشكلة
```
kernel BUG at drivers/of/base.c:2145!
Call trace:
  of_alias_scan+0x...
  of_pdt_build_devicetree+0x...
```

#### التحليل
في `pdt.c`، الدالة `of_pdt_build_prop_list()`:

```c
static struct property * __init of_pdt_build_prop_list(phandle node)
{
    struct property *head, *tail;

    /* first property is always ".node" (synthetic) */
    head = tail = of_pdt_build_one_prop(node, NULL,
                         ".node", &node, sizeof(node));

    /* then iterate all real properties */
    tail->next = of_pdt_build_one_prop(node, NULL, NULL, NULL, 0);
    ...
}
```

والدالة `of_pdt_build_devicetree()`:

```c
void __init of_pdt_build_devicetree(phandle root_node, struct of_pdt_ops *ops)
{
    BUG_ON(!ops);
    of_pdt_prom_ops = ops;
    of_root = of_pdt_create_node(root_node, NULL);
    of_root->full_name = "/";
    of_root->child = of_pdt_build_tree(of_root,
                of_pdt_prom_ops->getchild(of_root->phandle));
    of_alias_scan(kernel_tree_alloc);  /* <-- crash here */
}
```

المشكلة: الـ PROM الخاص بالـ i.MX8 يُعيد `.node` كأول خاصية حقيقية عبر `nextprop`، فتُبنى مرتين — مرة كـ synthetic property ومرة من الـ PROM. `of_alias_scan` يتوقع بنية نظيفة بدون تكرار.

#### الحل

**1. تعديل `nextprop` callback لتخطي `.node`:**

```c
static int imx8_nextprop(phandle node, char *prev, char *buf)
{
    int ret;
    do {
        ret = prom_next_prop(node, prev, buf);
        if (ret) return ret;
        prev = buf;
    } while (strcmp(buf, ".node") == 0);  /* skip synthetic name */
    return 0;
}
```

**2. أو تعديل `of_pdt_build_prop_list` للتحقق من التكرار:**

```c
tail->next = of_pdt_build_one_prop(node, NULL, NULL, NULL, 0);
tail = tail->next;
while (tail) {
    if (strcmp(tail->name, ".node") == 0) {
        /* skip duplicate .node property */
        tail->next = of_pdt_build_one_prop(node, tail->name,
                                            NULL, NULL, 0);
    } else {
        tail->next = of_pdt_build_one_prop(node, tail->name,
                                            NULL, NULL, 0);
    }
    tail = tail->next;
}
```

#### الدرس المستفاد
الخاصية `.node` هي **synthetic property** تُضيفها `of_pdt_build_prop_list` دائماً كأول عنصر لتخزين الـ phandle. الـ `nextprop` callback في `of_pdt_ops` يجب ألا تُعيد `.node` أبداً لأنها محجوزة للاستخدام الداخلي. عند كتابة PROM driver لـ i.MX8 أو أي منصة أخرى، يجب استبعاد هذا الاسم من نتائج `nextprop`.

---

### السيناريو الخامس: AM62x Custom Board Bring-up — الـ `prom_early_alloc` ينفد أثناء بناء شجرة معقدة

#### العنوان
**استنزاف الـ early memory allocator أثناء بناء الـ device tree على لوحة AM62x بها عقد USB و HDMI وI2C معقدة**

#### السياق
لوحة تطوير مخصصة تستخدم Texas Instruments AM62x لتطبيق HMI صناعي. اللوحة تحتوي على USB 3.0، HDMI، ثلاثة buses I2C، أربعة buses UART، وعدة GPIOs. الفريق يستخدم PROM-based device tree building (عبر `of_pdt_build_devicetree`) لاختبار مرونة الـ BSP. الـ kernel يتوقف بعد رسالة:

```
prom_early_alloc: out of memory, request 32 bytes
Kernel panic - not syncing: Attempted to kill the idle task!
```

#### المشكلة
الـ early memory pool الذي يخدم `prom_early_alloc` ينفد قبل اكتمال بناء الـ device tree. اللوحة بها عشرات العقد وكل عقدة تستهلك allocations متعددة.

#### التحليل
تتبع تسلسل الـ allocations في `pdt.c`:

```
of_pdt_build_devicetree()
  └─ of_pdt_create_node()          → prom_early_alloc(sizeof(device_node))   ~120 bytes
       └─ of_pdt_get_one_property() → prom_early_alloc(len)                   per property
       └─ of_pdt_build_prop_list()
            └─ of_pdt_build_one_prop()
                 ├─ prom_early_alloc(sizeof(property) + 32)                   ~80 bytes
                 └─ prom_early_alloc(p->length + 1)                           per value
```

لكل عقدة في الـ tree:
- allocation واحدة لـ `device_node` نفسه
- allocation لـ `full_name` عبر `of_pdt_build_full_name`
- allocation لكل خاصية: `sizeof(property) + 32` للاسم + allocation للقيمة

لوحة AM62x المعقدة بها ~150 عقدة وآلاف الخصائص. الحساب:

```
150 عقدة × 120 bytes = 18KB
150 × 10 خصائص × 80 bytes = 120KB
150 × 10 × متوسط 20 bytes قيمة = 30KB
المجموع: ~168KB
```

الـ `prom_early_alloc` pool الافتراضي أصغر من ذلك للوحات المدمجة.

#### الحل

**1. تشخيص حجم الـ tree:**

```bash
# من الـ bootloader، حساب عدد العقد والخصائص
grep -c "^" /proc/device-tree/.../  # أو عبر fdtdump
fdtdump /boot/board.dtb | wc -l
```

**2. زيادة حجم الـ early allocator pool:**

```c
/* في arch/arm64/mm/init.c أو ما يعادله للـ AM62x */
#define PROM_EARLY_ALLOC_SIZE   (512 * 1024)  /* increase from 256KB to 512KB */
```

**3. تحسين الاستخدام بتجنب allocations غير ضرورية — الـ `tmp` pointer في `of_pdt_build_one_prop`:**

```c
static struct property * __init of_pdt_build_one_prop(...)
{
    static struct property *tmp = NULL;
    struct property *p;

    if (tmp) {
        p = tmp;
        memset(p, 0, sizeof(*p) + 32);
        tmp = NULL;   /* reuse allocation when nextprop fails */
    } else {
        p = prom_early_alloc(sizeof(struct property) + 32);
    }
    ...
    if (err) {
        tmp = p;   /* save for reuse next call */
        return NULL;
    }
```

هذا الـ reuse mechanism موجود أصلاً في الكود — لكنه يعمل فقط لخاصية واحدة في كل مرة. إذا كانت `getproplen` تُعيد `0` أو أقل لخصائص كثيرة، تُهدر allocations بدون إعادة استخدام.

**4. مراجعة الـ `nextprop` callback** لتخطي الخصائص ذات الطول الصفري مبكراً قبل الوصول لـ `of_pdt_build_one_prop`:

```c
static int am62x_nextprop(phandle node, char *prev, char *buf)
{
    while (1) {
        int ret = prom_nextprop(node, prev, buf);
        if (ret) return ret;
        /* skip zero-length properties early to save allocations */
        if (prom_getproplen(node, buf) > 0)
            return 0;
        prev = buf;
    }
}
```

#### الدرس المستفاد
الـ `prom_early_alloc` مخصص لمرحلة ما قبل الـ allocator العام — لا يوجد `free` ولا يمكن استرداد الذاكرة. على لوحات AM62x ذات الـ device tree المعقدة، يجب حساب حجم الـ pool المطلوب مسبقاً بدقة. كذلك يجب الاستفادة من آلية `tmp` الموجودة في `of_pdt_build_one_prop` وتأكد من أن الـ `nextprop` callback لا تُعيد خصائص فارغة غير ضرورية.
## Phase 7: مصادر ومراجع

---

### مقدمة

**الـ** `of_pdt.h` header يُعرِّف واجهة بناء شجرة الأجهزة عبر استدعاء PROM الخاص بـ Open Firmware. المصادر التالية تغطي هذا الموضوع من جميع زوايا: توثيق النواة الرسمي، مقالات LWN، نقاشات mailing list، والكتب المرجعية.

---

### مقالات LWN.net

| العنوان | الوصف |
|---------|-------|
| [Open Firmware device tree virtual filesystem](https://lwn.net/Articles/215853/) | مقالة مبكرة تناقش تطبيق واجهة Open Firmware Device Tree كـ virtual filesystem داخل النواة |
| [x86: OLPC: add OLPC device-tree support (v3)](https://lwn.net/Articles/411365/) | patch series يستخدم مباشرةً `<linux/of_pdt.h>` لبناء device tree على x86 عبر OLPC Open Firmware — أبرز مثال حقيقي لـ `of_pdt_build_devicetree()` خارج SPARC |
| [ELCE: Grant Likely on device trees](https://lwn.net/Articles/414016/) | تغطية لمحاضرة Grant Likely حول توحيد كود device tree عبر المعماريات المختلفة |
| [[PATCH] Device Tree on ARM platform](https://lwn.net/Articles/334826/) | نقاش تقنيق نقل البنية التحتية لـ DT إلى ARM وتعميمها |
| [Open Firmware and Devicetree — kernel docs mirror](https://static.lwn.net/kerneldoc/devicetree/index.html) | نسخة LWN من التوثيق الرسمي لكامل subsystem |

---

### توثيق النواة الرسمي — `Documentation/`

```
Documentation/devicetree/usage-model.rst       # نموذج الاستخدام الكامل لـ DT
Documentation/devicetree/booting-without-of.txt # الإقلاع بدون Open Firmware
Documentation/devicetree/index.rst             # فهرس Open Firmware & Devicetree
```

روابط HTML المنشورة:

- [Linux and the Devicetree — usage model](https://www.kernel.org/doc/html/latest/devicetree/usage-model.html)
- [Open Firmware and Devicetree — index](https://docs.kernel.org/devicetree/index.html)
- [Booting without Open Firmware](https://www.kernel.org/doc/html/v5.9/devicetree/booting-without-of.html)

---

### الملفات المصدرية ذات الصلة في النواة

| الملف | الدور |
|-------|-------|
| `include/linux/of_pdt.h` | تعريف `struct of_pdt_ops` والـ API العام |
| `drivers/of/pdt.c` | التنفيذ الفعلي لـ `of_pdt_build_devicetree()` |
| `arch/sparc/kernel/prom_common.c` | أول مستخدم تاريخي — SPARC PROM callbacks |
| `arch/microblaze/kernel/prom.c` | تطبيق Microblaze لنفس الـ ops |
| `arch/x86/kernel/devicetree.c` | تطبيق x86/OLPC — مثال على نقل الكود عبر المعماريات |

رابط مرجعي لملف SPARC:
- [arch/sparc/kernel/prom_common.c على GitHub](https://github.com/torvalds/linux/blob/master/arch/sparc/kernel/prom_common.c)
- [arch/microblaze/kernel/prom.c على GitHub](https://github.com/torvalds/linux/blob/master/arch/microblaze/kernel/prom.c)
- [arch/x86/kernel/devicetree.c على GitHub](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/devicetree.c)

---

### Commits ومناقشات Mailing List ذات الصلة

الـ patch series الذي أنشأ `drivers/of/pdt.c` كملف مستقل عام 2010 (Andres Salomon):

- [[PATCH 2/4] sparc: break out some prom device-tree building code into drivers/of](https://lists.ozlabs.org/pipermail/devicetree-discuss/2010-July/002527.html)
  - فصل كود PROM من `arch/sparc` إلى `drivers/of/pdt.c` لإعادة استخدامه

- [Linux-Kernel Archive: [PATCH 4/9] sparc: make drivers/of/pdt.c no longer sparc-only](https://lkml.iu.edu/hypermail/linux/kernel/1008.3/02762.html)
  - الـ patch الذي حوّل الكود من sparc-only إلى عام متاح لجميع المعماريات

- [Re: [PATCH 4/4] x86: OLPC: add OLPC device-tree support — LKML](https://lkml.iu.edu/hypermail/linux/kernel/1006.3/02190.html)
  - نقاش تقني حول استخدام `of_pdt_build_devicetree()` على x86

- [tip:x86/platform — x86, olpc: Use device tree for platform identification — Patchwork](https://lore.kernel.org/patchwork/patch/241200/)
  - patch لاحق يستخدم DT لتحديد المنصة على OLPC

---

### صفحات eLinux.org

| الصفحة | المحتوى |
|--------|---------|
| [Device Tree Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل لبنية DT وصياغتها |
| [Device tree history](https://elinux.org/Device_tree_history) | تاريخ تطور Device Tree من Open Firmware حتى FDT |
| [Device Tree Information](https://www.elinux.org/Device_Tree_Information) | نقطة بداية لكل ما يخص DT في Linux embedded |
| [Device Tree unittest](https://elinux.org/Device_Tree_unittest) | كيفية اختبار كود DT في النواة |
| [Device Tree Mysteries](https://elinux.org/Device_Tree_Mysteries) | تفسير نقاط الغموض الشائعة في DT |
| [Linux Drivers Device Tree Guide](https://elinux.org/Linux_Drivers_Device_Tree_Guide) | دليل كتابة drivers تعتمد على DT |

---

### صفحات KernelNewbies.org

| الصفحة | السبب |
|--------|-------|
| [Linux_5.2](https://kernelnewbies.org/Linux_5.2) | تتبع تغييرات DT subsystem في الإصدارات الحديثة |
| [Linux_2_6_25](https://kernelnewbies.org/Linux_2_6_25) | أولى الإضافات الكبيرة لـ OF/DT في kernelnewbies |
| [Documents](https://kernelnewbies.org/Documents) | فهرس وثائق KernelNewbies كنقطة بداية |

---

### مصادر إضافية مفيدة

- [Device Tree Hacking — OLPC Wiki](https://wiki.laptop.org/go/Device_Tree_Hacking)
  — توثيق OLPC لكيفية التفاعل مع Open Firmware PROM على x86

- [The Device Tree for embedded Linux and Xilinx FPGAs](https://billauer.co.il/blog/2011/08/dts-of-open-firmware-microblaze/)
  — مقالة تشرح DT على Microblaze مع Open Firmware

- [CONFIG_OF — kernelconfig.io](https://www.kernelconfig.io/config_of)
  — معلومات Kconfig لتفعيل دعم Open Firmware / Device Tree

- [Devicetree — Wikipedia](https://en.wikipedia.org/wiki/Devicetree)
  — خلفية تاريخية شاملة من IEEE 1275 حتى FDT

---

### الكتب المرجعية

#### Linux Device Drivers (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول ذات الصلة**:
  - Chapter 14: *The Linux Device Model* — يشرح كيف يُبنى نموذج الأجهزة الذي يعتمد عليه DT
- **ملاحظة**: LDD3 قديم نسبياً (2005) ولا يتناول `of_pdt` مباشرةً لكنه أساس لفهم `struct bus_type` وكيف تُسجَّل الأجهزة

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول ذات الصلة**:
  - Chapter 17: *Devices and Modules* — يغطي driver model وسجلات الأجهزة التي تُعبّأ من DT
- **القيمة**: يشرح دورة حياة `platform_device` التي تُنشئها نواة DT لاحقاً

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصول ذات الصلة**:
  - Chapter 7: *Bootloaders* — يشرح كيف يمرر U-Boot أو Open Firmware شجرة الأجهزة للنواة
  - Chapter 8: *Device Tree* — يتناول بنية DTS/DTB وعلاقتها بـ PROM
- **القيمة**: أوضح كتاب يربط بين Open Firmware legacy وعالم FDT الحديث على المنظومات المدمجة

---

### مصطلحات البحث

للعثور على مزيد من المعلومات استخدم هذه المصطلحات:

```
of_pdt_build_devicetree
of_pdt_ops linux kernel
prom_early_alloc sparc kernel
open firmware PROM linux phandle
drivers/of/pdt.c kernel
linux open firmware device tree OLPC x86
IEEE 1275 open firmware linux
flattened device tree FDT vs open firmware live tree
sparc prom device tree linux kernel history
microblaze open firmware device tree linux
```
## Phase 8: Writing simple module

### الهدف

**`of_pdt_build_devicetree`** هي الدالة الوحيدة المُصدَّرة من `of_pdt.h`، وهي تُبنى أثناء `__init` فقط — لا يمكن وضع kprobe عليها مباشرةً لأنها تُحرَّر من الذاكرة بعد الإقلاع. بدلاً من ذلك، نضع **kprobe** على `of_alias_scan`، وهي دالة مُصدَّرة (`EXPORT_SYMBOL`) تُستدعى في نهاية `of_pdt_build_devicetree` وتبقى في الذاكرة طوال دورة حياة الـ kernel — ما يجعلها هدفاً آمناً وممتعاً للمراقبة.

الـ kprobe يطبع معلومات عن جذر شجرة الـ device tree (`of_root`) لحظة استكمال بنائها من PROM.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_of_alias_scan.c
 *
 * Hooks of_alias_scan() which is called at the end of
 * of_pdt_build_devicetree() to inspect the device tree root
 * node right after it is built from PROM data.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/of.h>        /* struct device_node, of_root */
#include <linux/of_pdt.h>    /* of_pdt_build_devicetree declaration */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs <example@kernel.org>");
MODULE_DESCRIPTION("kprobe on of_alias_scan to inspect OF PDT device tree root");

/* ------------------------------------------------------------------ */
/* pre-handler: يُستدعى قُبيل تنفيذ of_alias_scan()                  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct device_node *root = of_root; /* global set by of_pdt_build_devicetree */

    if (root) {
        /* Print the root node name and phandle assigned by PROM */
        pr_info("kprobe[of_alias_scan]: of_root name=\"%s\" phandle=0x%x\n",
                root->full_name ? root->full_name : "<none>",
                root->phandle);

        /* Count direct children of root */
        {
            struct device_node *child;
            int count = 0;
            for (child = root->child; child; child = child->sibling)
                count++;
            pr_info("kprobe[of_alias_scan]: root has %d direct child node(s)\n",
                    count);
        }
    } else {
        pr_info("kprobe[of_alias_scan]: of_root is NULL (PROM tree not built?)\n");
    }

    return 0; /* 0 = continue normal execution */
}

/* ------------------------------------------------------------------ */
/* تعريف الـ kprobe نفسه                                              */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "of_alias_scan", /* الدالة المستهدفة               */
    .pre_handler = handler_pre,     /* callback قبل التنفيذ           */
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
static int __init kprobe_of_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_of: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_of: planted kprobe at %s (%p)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit kprobe_of_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe_of: removed kprobe from %s\n", kp.symbol_name);
}

module_init(kprobe_of_init);
module_exit(kprobe_of_exit);
```

---

### Makefile

```makefile
obj-m += kprobe_of_alias_scan.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل قسم

#### الـ includes

```c
#include <linux/kprobes.h>   // struct kprobe + register/unregister_kprobe
#include <linux/of.h>        // struct device_node + of_root
#include <linux/of_pdt.h>    // واجهة PROM device tree
```

**الـ** `kprobes.h` يُعرِّف آلية الـ hook الديناميكي في الـ kernel؛ **الـ** `of.h` ضروري للوصول إلى `of_root` وهو المتغير العالمي الذي يُخزِّن جذر شجرة الـ device tree التي بناها `of_pdt_build_devicetree`.

---

#### اختيار `of_alias_scan` هدفاً

`of_pdt_build_devicetree` نفسها مُعلَّمة بـ `__init` أي تُحذف من الذاكرة بعد الإقلاع، فلا يمكن وضع kprobe عليها. **الـ** `of_alias_scan` تُستدعى في آخر سطر من `of_pdt_build_devicetree`، تبقى في الذاكرة، ومُصدَّرة بـ `EXPORT_SYMBOL` — ما يعني أن رمزها متاح للـ kprobe subsystem.

---

#### `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

يُستدعى هذا الـ callback لحظة وصول تنفيذ الـ CPU إلى أول تعليمة في `of_alias_scan`، قبل أن تبدأ الدالة عملها. نقرأ `of_root` في هذه اللحظة لأن `of_pdt_build_devicetree` أكملت بناء الشجرة بالكامل — بما في ذلك ضبط `of_root->full_name = "/"` — قبل استدعاء `of_alias_scan`.

القيمة المُعادة `0` تعني "أكمل تنفيذ الدالة الأصلية بشكل طبيعي".

---

#### `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "of_alias_scan",
    .pre_handler = handler_pre,
};
```

**الـ** `.symbol_name` يُخبر الـ kernel بالبحث عن عنوان `of_alias_scan` في جدول الرموز وكتابة نقطة توقف (breakpoint instruction) عنده تلقائياً عند تسجيل الـ kprobe. لا حاجة لتحديد العنوان يدوياً.

---

#### `module_init` و `module_exit`

`register_kprobe` تكتب تعليمة `int3` (على x86) أو ما يعادلها على بنى أخرى في بداية `of_alias_scan`، وتربطها بـ `handler_pre`. **من الضروري** استدعاء `unregister_kprobe` في `module_exit` لاستعادة التعليمة الأصلية وتحرير موارد الـ kprobe — وإلا يبقى الـ breakpoint في الذاكرة بعد تفريغ الـ module مما يُسبب kernel panic عند استدعاء الدالة لاحقاً.

---

### مثال على الناتج (dmesg)

```
kprobe_of: planted kprobe at of_alias_scan (ffffffffc0a12340)
kprobe[of_alias_scan]: of_root name="/" phandle=0x1
kprobe[of_alias_scan]: root has 7 direct child node(s)
kprobe_of: removed kprobe from of_alias_scan
```

---

### ملاحظات عملية

| النقطة | التفاصيل |
|--------|----------|
| البنى المدعومة | x86, ARM64, RISC-V — أي بنية تدعم `CONFIG_KPROBES` |
| متى يُطلق الـ kprobe؟ | فقط عند بناء الشجرة من PROM (OLPC, SPARC) — على x86 العادي `of_alias_scan` تُستدعى من مسارات أخرى أيضاً |
| الحماية من NULL | نتحقق من `of_root != NULL` قبل الوصول إلى حقوله |
| `CONFIG_KPROBES` | يجب تفعيله في تكوين الـ kernel (`make menuconfig → Kernel hacking → Kprobes`) |
