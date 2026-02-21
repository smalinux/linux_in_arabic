# `syscon.h`

> **PATH**: `include/linux/mfd/syscon.h`
> **Subsystem**: MFD (Multi-Function Device) / System Configuration (SYSCON)
> **الوظيفة الأساسية**: واجهة برمجية تتيح لأي driver في الكيرنل الوصول إلى registers المشتركة بين أجهزة متعددة عبر regmap موحّد

---

## Phase 1: الصورة الكبيرة

### المشكلة اللي بيحلها syscon

تخيل عندك لوحة إلكترونية (SoC — System on Chip) فيها عشرات الأجهزة: USB، GPU، Clock، Reset، Ethernet. كل هذه الأجهزة تحتاج أحياناً تقرأ أو تكتب في **نفس منطقة الذاكرة** (نفس register block). مثلاً:
- الـ USB controller يحتاج يعدّل bit في register مشترك عشان يفعّل الـ PHY.
- الـ Ethernet controller يحتاج يقرأ نفس الـ register block عشان يعرف حالة الـ power.
- الـ Clock driver يكتب في نفس المنطقة عشان يضبط سرعة النواة.

لو كل driver عمل `ioremap()` منفصل لنفس العنوان — يصير فوضى وتعارض. **syscon** هو الحل: driver واحد يملك الـ regmap لهذه المنطقة المشتركة، والباقون يطلبون منه الوصول بشكل آمن.

---

### ما هو syscon بالضبط؟

**syscon** = **Sys**tem **Con**figuration — هو framework صغير جداً مكوّن من:
1. **driver** (`drivers/mfd/syscon.c`) يسجّل نفسه مع Device Tree كـ`"syscon"` node.
2. **header** (`include/linux/mfd/syscon.h`) يعلن الـ API للـ drivers الأخرى.

الـ driver يعمل `ioremap()` مرة واحدة على منطقة الـ registers، يلفّها بـ **regmap** (وهو layer موحّد للقراءة والكتابة في الـ registers)، ثم يحفظها في قائمة عامة (`syscon_list`). أي driver آخر يحتاج الوصول لنفس المنطقة يطلبها بالاسم أو بالـ phandle من الـ Device Tree، ويحصل على نفس الـ regmap مباشرة.

---

### القصة كاملة — رحلة driver يبحث عن register مشترك

```
┌─────────────────────────────────────────────────────────────┐
│                     Device Tree (DTS)                        │
│                                                              │
│  syscon_node: syscon@40000000 {                              │
│      compatible = "simple-bus", "syscon";                    │
│      reg = <0x40000000 0x1000>;   ← منطقة الـ registers      │
│  };                                                          │
│                                                              │
│  usb_phy: usb-phy@40100000 {                                 │
│      syscon = <&syscon_node>;  ← phandle يشير للـ syscon      │
│  };                                                          │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│               syscon driver (drivers/mfd/syscon.c)           │
│                                                              │
│  1. يقرأ الـ reg من Device Tree                              │
│  2. يعمل ioremap() على العنوان 0x40000000                    │
│  3. ينشئ regmap (32-bit registers)                           │
│  4. يضيفها لـ syscon_list                                    │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│           USB PHY Driver يريد يكتب في الـ register           │
│                                                              │
│  struct regmap *map =                                        │
│      syscon_regmap_lookup_by_phandle(np, "syscon");          │
│                                                              │
│  regmap_update_bits(map, PHY_CTRL_REG, BIT(3), BIT(3));     │
└─────────────────────────────────────────────────────────────┘
```

بدون syscon: كل driver يعمل `ioremap()` منفصل → تعارض وأخطاء.
مع syscon: ورقة عنوان واحدة، والكل يستعير منها.

---

### ماذا يوفر هذا الـ header تحديداً؟

الملف `syscon.h` بسيط جداً — فقط **إعلانات الدوال** (function declarations). يوفر 6 دوال:

| الدالة | ماذا تفعل؟ |
|--------|-----------|
| `device_node_to_regmap(np)` | يحوّل أي device node لـ regmap (بدون تحقق من resources) |
| `syscon_node_to_regmap(np)` | نفس السابق لكن يتحقق أن الـ node هو فعلاً `"syscon"` |
| `syscon_regmap_lookup_by_compatible(s)` | يبحث عن syscon بالـ compatible string |
| `syscon_regmap_lookup_by_phandle(np, property)` | يبحث عبر phandle في الـ Device Tree |
| `syscon_regmap_lookup_by_phandle_args(np, property, arg_count, out_args)` | نفس السابق + يجلب arguments إضافية |
| `syscon_regmap_lookup_by_phandle_optional(np, property)` | نفس phandle لكن يرجع NULL بدل error لو ما وُجد |
| `of_syscon_register_regmap(np, regmap)` | يسمح لـ driver خارجي يسجّل regmap خاص به في syscon |

---

### لماذا `CONFIG_MFD_SYSCON`؟

الـ header يستخدم `#ifdef CONFIG_MFD_SYSCON`:
- لو syscon **مفعّل**: الدوال حقيقية تُنفَّذ في `drivers/mfd/syscon.c`.
- لو syscon **مُعطَّل**: كل الدوال تصبح `static inline` ترجع `ERR_PTR(-ENOTSUPP)` — بحيث الـ drivers الأخرى تبقى تُكمبَل بدون أخطاء، وتتعامل مع الـ error بشكل عادي.

هذا نمط دفاعي شائع في الكيرنل: لا تكسر الـ build لو feature اختيارية غير موجودة.

---

### أين يقع هذا الـ header في منظومة الكيرنل؟

```
include/linux/mfd/
└── syscon.h          ← هذا الملف (الـ API للـ drivers)

drivers/mfd/
└── syscon.c          ← التطبيق الفعلي (implementation)

Documentation/devicetree/bindings/
├── arm/stm32/st,stm32-syscon.yaml
├── soc/cirrus/cirrus,ep9301-syscon.yaml
├── soc/starfive/starfive,jh7110-syscon.yaml
└── ...               ← توثيق الـ Device Tree لكل platform
```

---

### من يستخدم syscon.h؟

المستخدمون كُثُر جداً — أمثلة:
- **`drivers/fpga/zynq-fpga.c`** — الـ FPGA driver على Xilinx Zynq يحتاج registers مشتركة مع الـ PS (Processing System).
- **`drivers/hwspinlock/qcom_hwspinlock.c`** — Qualcomm hardware spinlock يصل للـ registers عبر syscon.
- **`drivers/net/can/flexcan/flexcan-core.c`** — الـ CAN bus driver على NXP يستخدمه لضبط الـ PHY.
- **`drivers/crypto/aspeed/aspeed-acry.c`** — crypto accelerator على ASPEED SoCs.

---

### الملفات الأساسية التي يجب معرفتها

| الملف | الدور |
|-------|-------|
| `include/linux/mfd/syscon.h` | **هذا الملف** — الـ public API |
| `drivers/mfd/syscon.c` | التطبيق الكامل للـ framework |
| `include/linux/regmap.h` | تعريف `struct regmap` والـ API الخاص به |
| `include/linux/of.h` | دوال قراءة الـ Device Tree (`of_parse_phandle`، إلخ) |
| `Documentation/devicetree/bindings/arm/stm32/st,stm32-syscon.yaml` | مثال توثيق DT binding لـ syscon |
| `Documentation/devicetree/bindings/soc/starfive/starfive,jh7110-syscon.yaml` | مثال آخر على platform مختلف |

---

### خلاصة

**الـ** `syscon.h` هو بوابة صغيرة وقوية: يتيح لأي driver في الكيرنل الوصول الآمن إلى **registers مشتركة** بين أجهزة متعددة على نفس الـ SoC، دون الحاجة لعمل `ioremap()` متعدد على نفس العنوان. الـ framework كله لا يتجاوز ملفين (header + implementation)، لكنه يُستخدَم من مئات الـ drivers عبر جميع architectures المدعومة في الكيرنل.
## Phase 2: شرح الـ syscon Framework

### المشكلة — ليش موجود أصلاً؟

في عالم الـ SoC (System-on-Chip) — يعني الشريحة اللي فيها كل شيء مدمج: CPU، GPU، USB، Ethernet، وغيرها — الوضع دايماً كالتالي:

عندك كتلة واحدة من الـ registers في الذاكرة (منطقة MMIO محددة بعنوان واحد وحجم واحد)، لكن هذه الكتلة **لا تخص قطعة hardware واحدة**. هي تحكم أشياء كثيرة مختلفة في آن واحد: clocks، power domains، GPIO muxing، USB PHY settings، reset lines... إلخ.

**المشكلة الحقيقية:** كيف يصل كل driver إلى هذه الـ registers بشكل آمن ومنظم؟

الحلول السيئة اللي كانت موجودة قبل syscon:

1. **كل driver يعمل iomap خاص به** — يعني نفس العنوان يُؤخذ مرتين أو أكثر. هذا خطأ لأن الـ kernel يرفض map نفس المنطقة مرتين في حالات كثيرة.

2. **driver يستدعي functions من driver آخر مباشرة** — coupling قوي جداً، يصعّب الـ modularization.

3. **platform_data تحتوي pointers لـ registers** — هذا نهج قديم، لا يدعم Device Tree بشكل نظيف.

4. **shared global variables** — كارثة كاملة من ناحية الـ synchronization والـ maintainability.

**الـ syscon جاء ليحل هذه المعضلة تحديداً.**

---

### الحل — ماذا يفعل syscon؟

الـ **syscon** (System Controller) هو framework بسيط جداً لكنه ذكي: يوفر **نقطة وصول مركزية واحدة** لأي منطقة registers مشتركة بين عدة drivers.

الفكرة الجوهرية:
- أي block من الـ registers يُسجَّل **مرة واحدة فقط** تحت syscon.
- كل driver يريد الوصول يطلب **regmap** لهذا الـ block عبر syscon API.
- الـ **regmap** هو abstraction layer موحّد للقراءة والكتابة على الـ registers — يدعم locking، caching، endianness، hardware spinlocks.
- الـ syscon يضمن أن نفس الـ regmap يُعاد لكل من يطلبه (لا يُنشأ مرتين).

بكلمة واحدة: **syscon = registry عالمي للـ shared register banks، يُقدّمها كـ regmap.**

---

### التشبيه الحقيقي — المكتبة العامة

تخيل مدينة فيها مكتبة عامة واحدة (= منطقة الـ registers المشتركة). كل شخص في المدينة يحتاج كتباً من هذه المكتبة:

- لو كل شخص فتح المكتبة بمفتاح خاص به من الصفر — فوضى وتعارض.
- لو أحد سرق كتاباً وحمله لبيته بشكل دائم — الكتاب لا يعود متاحاً للآخرين.

الـ syscon هو **أمين المكتبة**: تسجّل عنده المكتبة مرة واحدة، وأي شخص يريد كتاباً يتوجه لأمين المكتبة، ويعطيه نسخة آمنة (regmap) مع ضمان عدم التعارض مع الآخرين.

---

### البنية المعمارية الكاملة

```
┌─────────────────────────────────────────────────────────────────┐
│                        Device Tree (DTS)                        │
│                                                                 │
│  syscon_node: syscon@1c00000 {                                  │
│      compatible = "syscon";                                     │
│      reg = <0x1c00000 0x1000>;   ← عنوان وحجم الـ register block│
│  };                                                             │
│                                                                 │
│  usb_phy: phy@... {                                             │
│      syscon = <&syscon_node>;   ← phandle يشير للـ syscon       │
│  };                                                             │
│                                                                 │
│  clock_ctrl: clock@... {                                        │
│      syscon = <&syscon_node>;   ← نفس الـ phandle               │
│  };                                                             │
└─────────────────────┬───────────────────────────────────────────┘
                      │  of_parse_phandle / of_find_compatible_node
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                     syscon Framework                            │
│                                                                 │
│  syscon_list (linked list محمي بـ mutex):                       │
│  ┌──────────────────────────────────┐                           │
│  │ struct syscon {                  │                           │
│  │   struct device_node *np;  ◄─── مفتاح البحث (Device Tree node)│
│  │   struct regmap *regmap;   ◄─── نقطة الوصول الموحدة         │
│  │   struct reset_control *reset;  │                           │
│  │   struct list_head list;        │                           │
│  │ }                               │                           │
│  └──────────────────────────────────┘                           │
│         │                                                       │
│         │ regmap_init_mmio()                                    │
│         ▼                                                       │
│  ┌──────────────────────────────────┐                           │
│  │         regmap layer             │                           │
│  │  - reg_bits = 32                 │                           │
│  │  - val_bits = 32 (أو 8/16)      │                           │
│  │  - reg_stride = 4                │                           │
│  │  - endianness config             │                           │
│  │  - hwspinlock (اختياري)          │                           │
│  │  - locking داخلي                │                           │
│  └──────────────────────────────────┘                           │
│         │                                                       │
│         │ of_iomap()                                            │
│         ▼                                                       │
│  ┌──────────────────────────────────┐                           │
│  │    Physical MMIO Region          │                           │
│  │    (void __iomem *base)          │                           │
│  │  0x1c00000 ─ 0x1c00FFF          │                           │
│  └──────────────────────────────────┘                           │
└─────────────────────────────────────────────────────────────────┘
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
┌─────────────────┐    ┌─────────────────────┐
│   USB PHY Driver│    │  Clock Control Driver│
│                 │    │                      │
│ regmap = syscon_│    │ regmap = syscon_     │
│  regmap_lookup_ │    │  regmap_lookup_      │
│  by_phandle()   │    │  by_phandle()        │
│                 │    │                      │
│ regmap_update_  │    │ regmap_read(regmap,  │
│  bits(regmap,   │    │   CLK_REG, &val);    │
│   USB_REG,...); │    │                      │
└─────────────────┘    └─────────────────────┘
```

---

### البنى الداخلية — كيف ترتبط ببعض

#### 1. الـ `struct syscon` — الوحدة الأساسية

```c
/* تعريفها في drivers/mfd/syscon.c (static، غير مُصدَّرة) */
struct syscon {
    struct device_node *np;       /* مفتاح التعرف: أي node في الـ DT؟ */
    struct regmap *regmap;        /* الـ handle الموحد للقراءة/الكتابة */
    struct reset_control *reset;  /* reset line اختياري */
    struct list_head list;        /* ربطها في القائمة العامة */
};
```

هذه الـ struct **خاصة تماماً** بالـ syscon framework. الـ drivers الخارجية لا ترى هذه الـ struct أبداً — هم يرون فقط الـ `struct regmap *`.

#### 2. القائمة العامة

```c
static DEFINE_MUTEX(syscon_list_lock);  /* حماية thread-safe */
static LIST_HEAD(syscon_list);          /* قائمة كل الـ syscons المسجّلة */
```

هذا هو "دفتر العناوين" المركزي. كل مرة يطلب driver regmap لـ node معين، الـ framework يبحث في هذه القائمة أولاً. إذا وجد — يُعيد نفس الـ regmap. إذا لم يجد — ينشئ واحداً جديداً.

#### 3. الـ `struct regmap_config` الافتراضية

```c
static const struct regmap_config syscon_regmap_config = {
    .reg_bits  = 32,  /* عناوين الـ registers بـ 32 bit */
    .val_bits  = 32,  /* قيم الـ registers بـ 32 bit */
    .reg_stride = 4,  /* كل register يبعد 4 bytes عن التالي */
};
```

هذه القيم الافتراضية قابلة للتخصيص عبر خصائص الـ DT:
- `reg-io-width` تغيّر عرض الـ register (مثلاً `1` لـ 8-bit registers).
- `big-endian` / `little-endian` / `native-endian` تحدد ترتيب البايتات.

---

### تدفق التسجيل — خطوة بخطوة

```
Driver يستدعي: syscon_regmap_lookup_by_phandle(np, "syscon")
                      │
                      ▼
          of_parse_phandle() ← يحول الـ phandle في الـ DT
                      │           إلى struct device_node *
                      ▼
          syscon_node_to_regmap(syscon_np)
                      │
                      ▼
          device_node_get_regmap(np, create=true, check_res=true)
                      │
                      ├─► mutex_lock(syscon_list_lock)
                      │
                      ├─► البحث في syscon_list هل هناك إدخال بنفس np؟
                      │       │
                      │   موجود؟ ──► أعد regmap الموجود مباشرةً ✓
                      │       │
                      │   غير موجود؟
                      │       │
                      ▼       ▼
              of_syscon_register(np, check_res=true)
                      │
                      ├─► of_address_to_resource() ← احصل على العنوان الفيزيائي
                      ├─► of_iomap()               ← map الـ physical memory
                      ├─► قراءة خصائص الـ DT (endianness, reg-io-width)
                      ├─► of_hwspin_lock_get_id()  ← هل يوجد hardware spinlock؟
                      ├─► regmap_init_mmio()        ← أنشئ الـ regmap
                      ├─► of_clk_get() + regmap_mmio_attach_clk() ← اختياري
                      ├─► of_reset_control_get() + reset_control_deassert()
                      ├─► list_add_tail() ← أضف للقائمة العامة
                      │
                      └─► أعد struct syscon * ← mutex_unlock
                                    │
                                    ▼
                          return syscon->regmap  ✓
```

---

### الـ API المُصدَّرة وأين تستخدمها

| Function | متى تستخدمها؟ |
|---|---|
| `syscon_node_to_regmap(np)` | عندك الـ device_node مباشرة، وتريد regmap للـ syscon المرتبط به. الأكثر أماناً. |
| `device_node_to_regmap(np)` | مثلها لكن بدون check للـ clocks/resets. للحالات البسيطة جداً حيث الـ node ليس بالضرورة "syscon" رسمياً. |
| `syscon_regmap_lookup_by_compatible(s)` | تعرف الـ compatible string (مثل `"ti,sysc"`) وتريد الـ regmap مباشرة. |
| `syscon_regmap_lookup_by_phandle(np, property)` | الأكثر شيوعاً: driver عنده property في الـ DT تشير بـ phandle للـ syscon. |
| `syscon_regmap_lookup_by_phandle_args(np, prop, arg_count, out_args)` | مثل السابق لكن مع arguments إضافية بجانب الـ phandle (مثل offset أو بيانات تكوين). |
| `syscon_regmap_lookup_by_phandle_optional(np, property)` | مثل phandle لكن إذا لم يوجد الـ property يُعيد NULL بدل error. للـ features الاختيارية. |
| `of_syscon_register_regmap(np, regmap)` | إذا أنشأت الـ regmap بنفسك (driver خاص) وتريد تسجيله في syscon ليستفيد منه الآخرين. |

---

### مفهوم الـ regmap — ليش هو مهم هنا؟

الـ **regmap** ليس مجرد pointer للذاكرة. هو طبقة كاملة تحتوي:

```
┌──────────────────────────────────────────────────┐
│                  struct regmap                   │
│                                                  │
│  ┌─────────────┐   ┌──────────────────────────┐  │
│  │  Locking    │   │       Cache Layer         │  │
│  │  (mutex or  │   │  (REGCACHE_NONE في syscon)│  │
│  │  hwspinlock)│   └──────────────────────────┘  │
│  └─────────────┘                                  │
│                                                  │
│  ┌──────────────────────────────────────────────┐│
│  │          Bus Operations                       ││
│  │  .read()  → readl(base + reg)                 ││
│  │  .write() → writel(val, base + reg)           ││
│  └──────────────────────────────────────────────┘│
│                                                  │
│  ┌──────────────────────────────────────────────┐│
│  │       Endianness Handling                     ││
│  │  Big/Little/Native Endian conversion          ││
│  └──────────────────────────────────────────────┘│
└──────────────────────────────────────────────────┘
```

عندما يستدعي driver مثلاً:
```c
regmap_update_bits(regmap, USB_PHY_CTRL, BIT(3), BIT(3));
```

الـ regmap يقوم داخلياً بـ:
1. أخذ الـ lock (mutex أو hwspinlock).
2. قراءة القيمة الحالية من الـ register.
3. تعديل البتات المطلوبة فقط (read-modify-write آمن).
4. كتابة القيمة الجديدة.
5. تحرير الـ lock.

هذا يعني أن driver الـ USB وdriver الـ Clock يمكنهما الكتابة في نفس الـ register block في نفس الوقت **بأمان تام** بدون race conditions.

---

### الـ Hardware Spinlock — التزامن بين processors

في أنظمة الـ SoC الحديثة، ليس فقط kernel threads متعددة تتنافس — أحياناً يكون فيه **processor ثانٍ كلياً** (مثل Cortex-M بجانب Cortex-A) يكتب في نفس الـ registers!

الـ mutex العادي لا يحمي في هذه الحالة لأنه لا يعمل إلا داخل نفس الـ CPU/OS.

لذلك الـ syscon يدعم الـ **hwspinlock** (Hardware Spinlock):

```c
ret = of_hwspin_lock_get_id(np, 0);
if (ret > 0 || (IS_ENABLED(CONFIG_HWSPINLOCK) && ret == 0)) {
    syscon_config.use_hwlock = true;
    syscon_config.hwlock_id = ret;
    syscon_config.hwlock_mode = HWLOCK_IRQSTATE; /* آمن حتى داخل interrupt */
}
```

الـ hwspinlock هو دائرة في الـ hardware نفسه تحكم الوصول المشترك بين processors مختلفة. وضع `HWLOCK_IRQSTATE` يعني أن الـ lock يحفظ حالة الـ interrupts أيضاً — لا يُعطّل الـ IRQs نهائياً، بل يحفظها ويستعيدها بعد الانتهاء.

---

### مثال واقعي: USB PHY على Allwinner SoC

في Allwinner H3/H5 مثلاً، رقاقة الـ USB PHY تحتاج تعديل bits في منطقة registers تُسمى "System Control" وتخص أيضاً الـ clock controller وغيره.

في الـ DTS:
```
/* ملاحظة: هذا مثال توضيحي */
syscon: syscon@1c00000 {
    compatible = "allwinner,sun8i-h3-system-control", "syscon";
    reg = <0x01c00000 0x1000>;
};

usbphy: phy@1c19400 {
    compatible = "allwinner,sun8i-h3-usb-phy";
    syscon = <&syscon>;   /* ← phandle يشير للـ syscon */
};
```

في driver الـ USB PHY:
```c
/* الحصول على regmap من الـ syscon */
regmap = syscon_regmap_lookup_by_phandle(dev->of_node, "syscon");
if (IS_ERR(regmap))
    return PTR_ERR(regmap);

/* تعديل USB PHY enable bit بأمان */
regmap_update_bits(regmap,
    SUN8I_H3_PHY_CTRL,      /* offset داخل الـ register block */
    SUN8I_H3_PHY_ENABLE,    /* bitmask */
    SUN8I_H3_PHY_ENABLE);   /* القيمة المطلوبة */
```

لا iomap، لا raw pointer، لا تعارض مع الـ clock driver الذي يستخدم نفس الـ regmap.

---

### مكانة syscon في الـ kernel

```
┌──────────────────────────────────────────────────────┐
│                  Kernel Subsystems                   │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ USB PHY  │  │  Clock   │  │  GPIO / Pinctrl  │   │
│  │ Driver   │  │ Driver   │  │  Driver          │   │
│  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │
│       │              │                 │             │
│       └──────────────┴─────────────────┘             │
│                       │                              │
│                       ▼                              │
│            ┌──────────────────────┐                  │
│            │   syscon Framework   │  ← MFD subsystem │
│            │  (drivers/mfd/syscon)│                  │
│            └──────────┬───────────┘                  │
│                       │                              │
│                       ▼                              │
│            ┌──────────────────────┐                  │
│            │    regmap Layer      │  ← lib/          │
│            └──────────┬───────────┘                  │
│                       │                              │
│                       ▼                              │
│            ┌──────────────────────┐                  │
│            │   MMIO (ioremap)     │                  │
│            └──────────┬───────────┘                  │
└───────────────────────┼──────────────────────────────┘
                        │
                        ▼
             ┌──────────────────────┐
             │  Physical SoC        │
             │  System Control Regs │
             │  (Shared Register    │
             │   Block)             │
             └──────────────────────┘
```

الـ syscon يقع ضمن مجلد `drivers/mfd/` (Multi-Function Device) لأن المنطقة التي يديرها تخدم وظائف متعددة — لكنه أبسط بكثير من MFD كامل لأنه لا ينشئ child devices، فقط يوفر regmap مشتركاً.

---

### ملخص المفاهيم الجوهرية

| المفهوم | التفسير |
|---|---|
| **Shared Register Block** | منطقة MMIO واحدة تخدم أجزاء hardware متعددة |
| **syscon** | framework يُسجّل هذه المنطقة مرة واحدة ويوزع regmap لمن يطلب |
| **regmap** | abstraction موحّد للقراءة/الكتابة مع locking وendianness |
| **phandle** | مؤشر في Device Tree يربط driver بالـ syscon node |
| **hwspinlock** | حماية hardware-level للتزامن بين processors مختلفة |
| **syscon_list** | قائمة عامة تضمن عدم إنشاء نفس الـ regmap مرتين |
| **of_syscon_register** | التسجيل التلقائي عند أول طلب (lazy initialization) |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums وخيارات الـ Config — Cheatsheet

#### الـ `CONFIG_MFD_SYSCON` — مفتاح التشغيل الرئيسي

| الحالة | النتيجة |
|--------|---------|
| `CONFIG_MFD_SYSCON=y` | كل الـ API يشتغل فعلاً |
| `CONFIG_MFD_SYSCON` غير موجود | كل الدوال ترجع `ERR_PTR(-ENOTSUPP)` أو `NULL` أو `-EOPNOTSUPP` |

> فكرة بسيطة: لو الـ syscon مش مفعّل في الـ kernel config، الكود بيتحول لـ stubs فارغة تماماً — زي مفتاح كهرباء مقفول.

---

#### الـ `enum regmap_endian` — ترتيب البايتات

| القيمة | المعنى |
|--------|--------|
| `REGMAP_ENDIAN_DEFAULT` (= 0) | اتبع الـ bus default |
| `REGMAP_ENDIAN_BIG` | big-endian (MSB أول) |
| `REGMAP_ENDIAN_LITTLE` | little-endian (LSB أول) |
| `REGMAP_ENDIAN_NATIVE` | ترتيب المعالج الحالي |

**في syscon:** يتم تحديده من الـ Device Tree عبر properties:
- `big-endian` → `REGMAP_ENDIAN_BIG`
- `little-endian` → `REGMAP_ENDIAN_LITTLE`
- `native-endian` → `REGMAP_ENDIAN_NATIVE`

---

#### الـ `enum regcache_type` — نوع الـ Cache

| القيمة | المعنى |
|--------|--------|
| `REGCACHE_NONE` | بلا cache، كل قراءة تروح للـ hardware |
| `REGCACHE_RBTREE` | cache بشجرة ثنائية (legacy) |
| `REGCACHE_FLAT` | مصفوفة مسطحة، تبدأ بـ zero |
| `REGCACHE_MAPLE` | maple tree (الأحدث والأفضل) |
| `REGCACHE_FLAT_S` | flat sparse (لا تعبئة بـ zero) |

**في syscon:** الـ syscon_regmap_config لا يحدد cache_type يعني `REGCACHE_NONE` بشكل ضمني — منطقي لأن الـ syscon registers حساسة وتتغير من خارج الـ kernel.

---

#### الـ DT Properties المهمة في syscon

| الـ Property | النوع | المعنى |
|---|---|---|
| `big-endian` | bool | يجبر big-endian على القراءة/الكتابة |
| `little-endian` | bool | يجبر little-endian |
| `native-endian` | bool | يستخدم endian المعالج |
| `reg-io-width` | u32 | عرض الـ register بالبايتات (1, 2, 4) |
| `hwspinlocks` | phandle | hardware spinlock اختياري للحماية |

---

#### الـ hwlock_mode الممكنة

| القيمة | المعنى |
|--------|--------|
| `HWLOCK_IRQSTATE` | يحفظ ويعطّل الـ IRQs أثناء القفل |
| `HWLOCK_IRQ` | يعطّل الـ IRQs فقط |
| `0` | بلا حماية IRQ |

**في syscon:** يستخدم `HWLOCK_IRQSTATE` دائماً — لأن الـ syscon registers قد تُلمس من داخل interrupt handler.

---

### 1. الـ Structs الأساسية

---

#### `struct syscon` — البطل الخفي

```c
/* من: drivers/mfd/syscon.c — غير مُصدَّر خارج الملف */
struct syscon {
    struct device_node *np;      /* مؤشر لعقدة الـ Device Tree */
    struct regmap      *regmap;  /* الـ regmap الذي يمثل السجلات */
    struct reset_control *reset; /* reset control اختياري */
    struct list_head   list;     /* ربطه بالقائمة العالمية syscon_list */
};
```

**الغرض:** هو "البطاقة الشخصية" لكل جهاز syscon. يربط بين عقدة الـ DT والـ regmap المقابل له.

| الحقل | النوع | الدور |
|-------|-------|-------|
| `np` | `struct device_node *` | مفتاح البحث — بيه نعرف هذا الـ syscon لأي node |
| `regmap` | `struct regmap *` | الـ handle للوصول للـ registers |
| `reset` | `struct reset_control *` | للتحكم في reset الجهاز (اختياري) |
| `list` | `struct list_head` | حلقة الربط بالقائمة العالمية |

---

#### `struct regmap_config` — وصفة بناء الـ regmap

```c
/* من: include/linux/regmap.h */
struct regmap_config {
    const char *name;           /* اسم الـ regmap (للـ debugfs) */
    int reg_bits;               /* عدد بتات العنوان */
    int reg_stride;             /* المسافة بين registers متجاورة */
    int val_bits;               /* عدد بتات القيمة */
    unsigned int max_register;  /* أعلى عنوان مسموح */
    bool max_register_is_0;     /* workaround لو max=0 حقيقي */

    /* Endianness */
    enum regmap_endian val_format_endian;
    enum regmap_endian reg_format_endian;

    /* Hardware spinlock */
    bool use_hwlock;
    unsigned int hwlock_id;
    unsigned int hwlock_mode;

    /* Locking بديل */
    bool disable_locking;
    regmap_lock lock;
    regmap_unlock unlock;
    void *lock_arg;

    /* Cache */
    enum regcache_type cache_type;
    /* ... و callbacks للتحقق من صلاحية الـ registers */
};
```

**في syscon، الـ config الافتراضي هو:**
```c
static const struct regmap_config syscon_regmap_config = {
    .reg_bits = 32,   /* عناوين 32-bit */
    .val_bits = 32,   /* قيم 32-bit */
    .reg_stride = 4,  /* كل register بُعده 4 bytes عن التالي */
};
```
ثم تُعدَّل هذه القيم بناءً على `reg-io-width` من الـ DT.

---

#### `struct regmap` — الصندوق الأسود (opaque)

لا يظهر تعريفه في الـ header العام — فقط forward declaration:
```c
struct regmap;
```
الـ syscon يحتفظ بـ pointer له ويمرره للـ clients. الـ clients يستخدمونه مع `regmap_read()` / `regmap_write()`.

---

#### `struct device_node` — عقدة الـ Device Tree

```c
struct device_node; /* forward declaration فقط في syscon.h */
```
هي تمثيل الـ kernel لعقدة من ملف الـ DTS. مثلاً:
```
syscon@40000000 {
    compatible = "syscon";
    reg = <0x40000000 0x1000>;
    reg-io-width = <4>;
};
```
الـ `np` في `struct syscon` يشير لهذه العقدة.

---

#### `struct regmap_range` — نطاق عناوين

```c
struct regmap_range {
    unsigned int range_min;  /* أول عنوان في النطاق */
    unsigned int range_max;  /* آخر عنوان في النطاق */
};
```

---

#### `struct reg_default` — قيمة افتراضية

```c
struct reg_default {
    unsigned int reg;  /* عنوان الـ register */
    unsigned int def;  /* القيمة الافتراضية بعد reset */
};
```

---

### 2. مخطط العلاقات بين الـ Structs

```
القائمة العالمية (global)
═══════════════════════════

syscon_list (LIST_HEAD)
    │
    ├──► struct syscon ──────────────────────────────────────┐
    │        ├── np      ──────────────► struct device_node  │
    │        │                               (DT node)       │
    │        ├── regmap  ──────────────► struct regmap       │
    │        │                          (opaque, internal)   │
    │        ├── reset   ──────────────► struct reset_control│
    │        │                          (اختياري)             │
    │        └── list    ─────────────────────────────────────┘
    │                    (حلقة مزدوجة تربط كل الـ syscons)
    │
    ├──► struct syscon (جهاز ثانٍ)
    │        └── ...
    │
    └──► struct syscon (جهاز ثالث)
             └── ...


struct regmap_config (تُبنى محلياً في of_syscon_register)
    │
    ▼
regmap_init_mmio()
    │
    ▼
struct regmap (يعيش طول فترة حياة الـ syscon)
    │
    ▼
يُخزَّن في syscon->regmap
    │
    ▼
يُعاد للـ client drivers عبر API


Client Driver
    │
    ├── يطلب syscon_regmap_lookup_by_phandle()
    │        │
    │        ▼
    │   struct regmap * ◄─── syscon->regmap
    │
    └── يستخدم regmap_read() / regmap_write()
              │
              ▼
         Hardware Registers (MMIO)
```

---

### 3. مخطط دورة الحياة — Lifecycle

```
══════════════════════════════════════════════════════════════
                    LIFECYCLE OF A SYSCON
══════════════════════════════════════════════════════════════

[Boot / Driver Probe]
        │
        ▼
client driver يطلب syscon_node_to_regmap(np)
        │
        ▼
device_node_get_regmap(np, create=true, check_res=true)
        │
        ├─── هل الـ np موجود مسبقاً في syscon_list؟
        │         YES ──────────────────────────────────────────► رجع regmap موجود
        │         NO
        │         │
        ▼         ▼
  [CREATION — of_syscon_register(np, check_res)]
        │
        ├── 1. kzalloc(struct syscon)
        │
        ├── 2. of_address_to_resource(np) → physical address
        │
        ├── 3. of_iomap(np) → virtual address (base)
        │
        ├── 4. قراءة DT properties:
        │       - endianness (big/little/native)
        │       - reg-io-width
        │
        ├── 5. of_hwspin_lock_get_id() → hwlock (اختياري)
        │
        ├── 6. حساب regmap_config:
        │       - reg_stride = reg_io_width
        │       - val_bits = reg_io_width * 8
        │       - max_register = res_size - reg_io_width
        │
        ├── 7. regmap_init_mmio(NULL, base, &config) → struct regmap
        │
        ├── 8. [إذا check_res=true]:
        │       - of_clk_get() → clk (اختياري)
        │       - regmap_mmio_attach_clk()
        │       - of_reset_control_get_optional_exclusive()
        │       - reset_control_deassert()
        │
        ├── 9. syscon->regmap = regmap
        │       syscon->np = np
        │
        ├── 10. list_add_tail(&syscon->list, &syscon_list)
        │         [محمي بـ syscon_list_lock]
        │
        └── 11. إرجاع syscon->regmap للـ client

[USAGE — طول فترة تشغيل النظام]
        │
        ├── regmap_read(regmap, reg, &val)
        ├── regmap_write(regmap, reg, val)
        └── regmap_update_bits(regmap, reg, mask, val)

[TEARDOWN]
        │
        ▼
   (لا يوجد teardown صريح في syscon العادي!)
   الـ syscon يعيش طول عمر النظام — لا يُحذف من القائمة
   إلا في حالات خاصة مع platform driver كامل
```

---

### 4. مخططات تدفق الاستدعاء — Call Flow

#### 4.1 تدفق `syscon_regmap_lookup_by_compatible()`

```
client driver
    │
    └── syscon_regmap_lookup_by_compatible("vendor,my-syscon")
              │
              ├── of_find_compatible_node(NULL, NULL, s)
              │       البحث في شجرة الـ DT عن compatible string
              │       ترجع: struct device_node *syscon_np
              │
              ├── syscon_node_to_regmap(syscon_np)
              │       │
              │       └── device_node_get_regmap(np,
              │               create = of_device_is_compatible(np, "syscon"),
              │               check_res = true)
              │                   │
              │                   ├── mutex_lock(&syscon_list_lock)
              │                   │
              │                   ├── البحث في syscon_list
              │                   │       وجدناه؟ → استخدمه مباشرة
              │                   │       لم نجده؟ → of_syscon_register(np, true)
              │                   │
              │                   ├── mutex_unlock(&syscon_list_lock)
              │                   │
              │                   └── return syscon->regmap
              │
              ├── of_node_put(syscon_np)  ← تحرير مرجع الـ DT node
              │
              └── return struct regmap *
```

---

#### 4.2 تدفق `syscon_regmap_lookup_by_phandle()`

```
client driver DTS:
    my-device {
        syscon-handle = <&syscon0>;
    };

client driver code:
    syscon_regmap_lookup_by_phandle(dev->of_node, "syscon-handle")
              │
              ├── of_parse_phandle(np, "syscon-handle", 0)
              │       تحليل الـ phandle من الـ DTS
              │       ترجع: struct device_node *syscon_np (= &syscon0)
              │
              ├── syscon_node_to_regmap(syscon_np)
              │       (نفس التدفق في 4.1)
              │
              ├── of_node_put(syscon_np)
              │
              └── return struct regmap *
```

---

#### 4.3 تدفق `syscon_regmap_lookup_by_phandle_args()`

```
client driver DTS:
    my-device {
        power-domains = <&syscon0 3 7>;  /* phandle + 2 args */
    };

client driver code:
    unsigned int args[2];
    syscon_regmap_lookup_by_phandle_args(np, "power-domains", 2, args)
              │
              ├── of_parse_phandle_with_fixed_args(np, property, 2, 0, &args)
              │       يفك: np=syscon0, args[0]=3, args[1]=7
              │
              ├── syscon_node_to_regmap(args.np)
              │
              ├── out_args[0] = args.args[0]  (= 3)
              │   out_args[1] = args.args[1]  (= 7)
              │
              ├── of_node_put(syscon_np)
              │
              └── return struct regmap *
              /* الـ client يستخدم القيم 3 و 7 لتحديد domain أو offset */
```

---

#### 4.4 تدفق التسجيل الخارجي `of_syscon_register_regmap()`

```
driver يملك regmap خاص به (مش syscon عادي)
    │
    └── of_syscon_register_regmap(np, my_regmap)
              │
              ├── التحقق: np != NULL && regmap != NULL
              │
              ├── kzalloc(struct syscon)
              │
              ├── mutex_lock(&syscon_list_lock)
              │
              ├── البحث في syscon_list:
              │       هل np موجود مسبقاً؟ → return -EEXIST
              │
              ├── syscon->regmap = regmap  ← الـ regmap الخارجي
              │   syscon->np = np
              │
              ├── list_add_tail(&syscon->list, &syscon_list)
              │
              ├── mutex_unlock(&syscon_list_lock)
              │
              └── return 0
              /* الآن أي driver آخر يقدر يجيب هذا الـ regmap عبر API العادي */
```

---

#### 4.5 تدفق `device_node_to_regmap()` مقابل `syscon_node_to_regmap()`

```
device_node_to_regmap(np)
    └── device_node_get_regmap(np, create=TRUE, check_res=FALSE)
              │
              └── يُنشئ regmap لأي node حتى لو مش "syscon"
                  بدون clock/reset management
                  مناسب للـ nodes البسيطة


syscon_node_to_regmap(np)
    └── device_node_get_regmap(np,
            create = of_device_is_compatible(np, "syscon"),
            check_res = TRUE)
              │
              ├── إذا الـ node compatible مع "syscon":
              │       ينشئ regmap مع clock و reset management
              │
              └── إذا مش "syscon":
                      create=false → ينتظر (EPROBE_DEFER)
                      يعني: لازم حد آخر يسجّله أولاً
```

---

### 5. استراتيجية الـ Locking

الـ syscon يستخدم **طبقتين من القفل**:

---

#### 5.1 الـ `syscon_list_lock` — حارس القائمة العالمية

```c
static DEFINE_MUTEX(syscon_list_lock);
static LIST_HEAD(syscon_list);
```

| السؤال | الجواب |
|--------|--------|
| نوع القفل | `mutex` (يقبل النوم) |
| ما يحميه | القائمة `syscon_list` كاملة |
| متى يُؤخذ | عند البحث أو الإضافة في القائمة |
| متى يُطلق | فور انتهاء عملية القائمة |

```
device_node_get_regmap():
    mutex_lock(&syscon_list_lock)     ← اقفل
    list_for_each_entry(...)          ← بحث
    if (!found) → of_syscon_register()  ← داخل القفل! (WARN_ON تتحقق)
    mutex_unlock(&syscon_list_lock)   ← افتح

of_syscon_register_regmap():
    mutex_lock(&syscon_list_lock)     ← اقفل
    list_for_each_entry(...)          ← تحقق من عدم التكرار
    list_add_tail(...)                ← أضف
    mutex_unlock(&syscon_list_lock)   ← افتح
```

**ملاحظة مهمة:** `of_syscon_register()` تحتوي على:
```c
WARN_ON(!mutex_is_locked(&syscon_list_lock));
```
هذا يعني أنها **مصممة لتُستدعى دائماً داخل القفل** — حماية من الاستدعاء الخاطئ.

---

#### 5.2 الـ regmap internal lock — حارس السجلات

الـ regmap بداخله قفله الخاص (mutex أو spinlock حسب الـ config). في syscon:

```
الحالة الافتراضية:
    regmap يستخدم mutex داخلي تلقائياً

إذا وُجد hwspinlock في الـ DT:
    syscon_config.use_hwlock = true
    syscon_config.hwlock_id = ret
    syscon_config.hwlock_mode = HWLOCK_IRQSTATE
    → regmap يستخدم hardware spinlock بدلاً من المتعارف

إذا أردت fast_io:
    .fast_io = true → regmap يستخدم spinlock بدلاً من mutex
    (syscon لا يفعّل هذا افتراضياً)
```

---

#### 5.3 ترتيب القفل — Lock Ordering

```
القاعدة الصارمة:
═══════════════

syscon_list_lock  (المستوى الخارجي)
        │
        └──► regmap internal lock  (المستوى الداخلي)

يعني: لو أخذت syscon_list_lock، يمكنك بعدها تأخذ regmap lock.
العكس محظور! لا تأخذ syscon_list_lock وأنت حامل regmap lock.

في الواقع العملي:
- البحث في القائمة يحتاج syscon_list_lock
- العمليات على الـ registers (read/write) تحتاج regmap lock فقط
- لا يوجد سيناريو يحتاج الاثنين معاً في نفس الوقت
  ← لهذا لا يوجد deadlock risk
```

---

#### 5.4 ملخص الـ Locking بصرياً

```
┌─────────────────────────────────────────────────────┐
│                  syscon_list_lock (mutex)            │
│                                                      │
│  يحمي: ┌──────────────────────────────┐             │
│         │  syscon_list (linked list)   │             │
│         │  ┌─────────┐  ┌─────────┐   │             │
│         │  │ syscon  │  │ syscon  │   │             │
│         │  │  .np    │  │  .np    │   │             │
│         │  │  .regmap│  │  .regmap│   │             │
│         │  └─────────┘  └─────────┘   │             │
│         └──────────────────────────────┘             │
└─────────────────────────────────────────────────────┘

بعد إطلاق syscon_list_lock:

┌─────────────────────────────────────────────────────┐
│           regmap internal lock                       │
│  (mutex عادي أو hwspinlock حسب الـ DT)              │
│                                                      │
│  يحمي: عمليات القراءة والكتابة على الـ MMIO         │
│                                                      │
│  regmap_read()  → lock → MMIO read  → unlock        │
│  regmap_write() → lock → MMIO write → unlock        │
└─────────────────────────────────────────────────────┘
```
## Phase 4: شرح الـ Functions

### جدول الـ Functions

| Function | Type | Purpose |
|----------|------|---------|
| `of_syscon_register()` | static | تسجيل node جديد كـ syscon وإنشاء الـ regmap له |
| `device_node_get_regmap()` | static | البحث في القائمة عن syscon أو إنشاء واحد جديد |
| `of_syscon_register_regmap()` | EXPORT_SYMBOL_GPL | تسجيل regmap خارجي جاهز مع الـ syscon |
| `device_node_to_regmap()` | EXPORT_SYMBOL_GPL | الحصول على regmap لأي device node بدون قيود |
| `syscon_node_to_regmap()` | EXPORT_SYMBOL_GPL | الحصول على regmap لـ node من نوع syscon حقيقي |
| `syscon_regmap_lookup_by_compatible()` | EXPORT_SYMBOL_GPL | البحث عن syscon بالـ compatible string |
| `syscon_regmap_lookup_by_phandle()` | EXPORT_SYMBOL_GPL | البحث عن syscon عبر phandle في الـ DT |
| `syscon_regmap_lookup_by_phandle_args()` | EXPORT_SYMBOL_GPL | البحث عن syscon عبر phandle مع استخراج arguments |
| `syscon_regmap_lookup_by_phandle_optional()` | EXPORT_SYMBOL_GPL | نفس phandle lookup لكن يرجع NULL بدل error لو مش موجود |

---

### الفئة الأولى: الـ Internal Helpers (داخلي فقط)

هذه الـ functions هي العمود الفقري للـ driver — لا يراها أحد من خارج الملف، لكنها اللي بتعمل الشغل الحقيقي.

---

#### `of_syscon_register()`

```c
static struct syscon *of_syscon_register(struct device_node *np, bool check_res)
```

**ما تعمله:**
هذه الـ function هي "المصنع" اللي بيبني الـ syscon من الصفر. تاخد device_node وتحوله لـ struct syscon كامل مع regmap يشتغل على MMIO. لو `check_res` كانت true، تجيب الـ clock والـ reset وتتأكد إن الـ hardware جاهز.

تخيل إنك بتفتح دكانة جديدة — لازم تاخد المفتاح (iomap)، تزين المحل (regmap config)، وتشغل الكهرباء (clock) وتتأكد إن الماء شغال (reset deassert).

**الـ Parameters:**
- `np` — الـ `struct device_node` اللي يمثل الـ hardware block في الـ Device Tree
- `check_res` — لو `true`، يجيب الـ clock والـ reset ويعمل deassert للـ reset (ده بيتطلب إن الـ block فعلاً موجود كـ hardware حقيقي)

**الـ Return Value:**
- pointer لـ `struct syscon` جديد في حالة النجاح
- `ERR_PTR(-ENOMEM)` لو فشل الـ allocation أو الـ iomap
- `ERR_PTR(-EFAULT)` لو حجم الـ resource أصغر من الـ io width
- `ERR_PTR(ret)` في أي خطأ تاني

**Key Details:**
- **يشترط** إن `syscon_list_lock` يكون مقفول قبل الاستدعاء (`WARN_ON(!mutex_is_locked(...))`)
- بيستخدم `__free(kfree)` cleanup mechanism — لو حصل error، الـ memory بتتحرر أوتوماتيك
- بيقرأ properties من الـ DT: `big-endian`، `little-endian`، `native-endian`، `reg-io-width`
- بيدعم الـ **hardware spinlock** عبر `of_hwspin_lock_get_id()` للحماية من التعارض بين processors مختلفة
- الـ `regmap` بيتعمل بـ `regmap_init_mmio()` — يعني كل read/write هيروح مباشرة للـ registers

**Pseudocode Flow:**
```
of_syscon_register(np, check_res):
    تأكد إن syscon_list_lock مقفول
    alloc syscon struct
    احسب resource من np
    عمل iomap للـ base address

    قرأ endianness من DT
    قرأ reg-io-width (default=4)
    شيك على hwspinlock

    تحقق إن res_size >= reg_io_width
    ابني اسم الـ config
    أنشئ regmap_init_mmio

    لو check_res:
        جيب الـ clock (اختياري)
        attach clock للـ regmap
        جيب الـ reset control
        عمل reset_control_deassert

    حط syscon في list_tail
    ارجع syscon

error paths:
    حرر reset → حرر clock → exit regmap → iounmap → ارجع ERR_PTR
```

**من يستدعيها:**
تُستدعى فقط من `device_node_get_regmap()` وهي مقفولة بالـ mutex دايماً.

---

#### `device_node_get_regmap()`

```c
static struct regmap *device_node_get_regmap(struct device_node *np,
                                              bool create_regmap,
                                              bool check_res)
```

**ما تعمله:**
دي الـ "بوابة" اللي كل الـ public functions بتمر منها. بتدور في الـ `syscon_list` على الـ node المطلوب — لو لقيته، بترجع الـ regmap بتاعه. لو ملقيتوش ولو `create_regmap` كانت true، بتعمل واحد جديد. لو `create_regmap` كانت false وملقيتوش، بترجع `-EPROBE_DEFER` علامة إن الـ device لسه ما اشتغلش.

تخيلها زي مكتب استعلامات في مستشفى — لو المريض موجود، بيديك ملفه. لو مش موجود وعندك صلاحية، بيفتح له ملف جديد. لو مش موجود وما عندكش صلاحية، بيقولك "ارجع بكره".

**الـ Parameters:**
- `np` — الـ device node المطلوب
- `create_regmap` — لو true، ينشئ syscon جديد لو مش موجود
- `check_res` — بيتمرر لـ `of_syscon_register()` لتحديد هل يجيب resources أم لا

**الـ Return Value:**
- `struct regmap *` صالح لو نجح
- `ERR_PTR(-EPROBE_DEFER)` لو مش موجود ومش مسموح بالإنشاء
- أي error من `of_syscon_register()` لو فشل الإنشاء

**Key Details:**
- يقفل `syscon_list_lock` بالـ mutex طول فترة البحث والإنشاء — يعني thread-safe
- الـ list search خطي O(n) — مناسب لأن عدد الـ syscon nodes في أي نظام قليل جداً

**من يستدعيها:**
تُستدعى من `device_node_to_regmap()` و `syscon_node_to_regmap()` فقط.

---

### الفئة الثانية: الـ Registration API

هذه الـ functions للـ drivers اللي عندهم regmap خاص بهم ويريدون تسجيله مع الـ syscon framework.

---

#### `of_syscon_register_regmap()`

```c
int of_syscon_register_regmap(struct device_node *np, struct regmap *regmap)
```

**ما تعمله:**
بدل ما الـ syscon driver يبني الـ regmap بنفسه، هذه الـ function بتخليك "تجيب" regmap جاهز وتسجله مع الـ syscon. مفيد لما الـ driver بيدير الـ hardware بطريقته الخاصة لكن يريد السماح لـ drivers تانية بالوصول عبر الـ syscon API.

تخيلها زي إنك عندك مفتاح بيتك وتعطي نسخة للجار عشان يدخل عند الطوارئ — البيت موجود بالفعل، بس بتسجله في نظام مشترك.

**الـ Parameters:**
- `np` — الـ device node اللي يربط الـ regmap بالـ hardware
- `regmap` — الـ regmap الجاهز اللي هيتسجل

**الـ Return Value:**
- `0` لو تم التسجيل بنجاح
- `-EINVAL` لو `np` أو `regmap` كانوا NULL
- `-ENOMEM` لو فشل الـ kzalloc
- `-EEXIST` لو الـ node ده مسجل بالفعل

**Key Details:**
- يقفل `syscon_list_lock` طول فترة الفحص والإضافة — atomic operation
- لو الـ node موجود بالفعل، بيرجع `-EEXIST` وبيحرر الـ memory المحجوزة
- الـ regmap المُسجَّل **لا يتحرر** من قِبَل الـ syscon — المسؤولية على الـ caller

**من يستدعيها:**
الـ drivers اللي بتنشئ regmap بنفسها (مثلاً عبر platform device) وتريد تشاركه مع الـ syscon framework.

---

### الفئة الثالثة: الـ Lookup API (العامة للـ Clients)

هذه الـ functions هي اللي الـ client drivers بتستخدمها يومياً للحصول على الـ regmap الخاص بالـ syscon.

---

#### `device_node_to_regmap()`

```c
struct regmap *device_node_to_regmap(struct device_node *np)
```

**ما تعمله:**
أبسط طريقة للحصول على regmap من device node. بتديها الـ node وترجعلك الـ regmap — لو مش موجود، بتنشئه. الفرق عن `syscon_node_to_regmap()` إنها **مش بتجيب resources** (clock/reset) — يعني مناسبة للـ nodes اللي بتوصّلها مباشرة بدون ما تكون syscon device حقيقية.

**الـ Parameters:**
- `np` — الـ device node المطلوب

**الـ Return Value:**
- `struct regmap *` صالح لو نجح
- `ERR_PTR` بكود الخطأ لو فشل

**Key Details:**
- تستدعي `device_node_get_regmap(np, true, false)` — create=true، check_res=false
- لأن check_res=false، لا تجيب clock ولا reset — مناسب للـ nodes البسيطة
- **يُحذَّر** من استخدامها لو الـ node عنده resources تحتاج إدارة

**من يستدعيها:**
الـ drivers اللي تعرف إن الـ node اللي بتوصله ليس بالضرورة compatible مع "syscon" لكن لازم تقرأ منه registers.

---

#### `syscon_node_to_regmap()`

```c
struct regmap *syscon_node_to_regmap(struct device_node *np)
```

**ما تعمله:**
النسخة "الذكية" من `device_node_to_regmap()`. بتفحص الـ node — لو كان compatible مع `"syscon"`، بتنشئ regmap جديد مع resources (clock/reset). لو مش compatible، تدور عليه في القائمة بس مش بتنشئه من الصفر (يعني لو مش موجود مسبقاً هترجع error).

هذي الـ function الأنسب للاستخدام العام لأنها بتتعامل صح مع الـ resources.

**الـ Parameters:**
- `np` — الـ device node المطلوب

**الـ Return Value:**
- `struct regmap *` صالح لو نجح
- `ERR_PTR(-EPROBE_DEFER)` لو الـ node مش compatible وما كانش متسجل
- أي error تاني من الإنشاء لو فشل

**Key Details:**
- تستدعي `device_node_get_regmap(np, of_device_is_compatible(np, "syscon"), true)`
- الـ `create_regmap` parameter بيتحدد ديناميكياً بناءً على الـ compatible
- الـ check_res=true هنا — يعني لو اتنشأ، هيجيب clock وreset ويعمل deassert

**من يستدعيها:**
تُستدعى من `syscon_regmap_lookup_by_compatible()` و `syscon_regmap_lookup_by_phandle()` و `syscon_regmap_lookup_by_phandle_args()`.

---

#### `syscon_regmap_lookup_by_compatible()`

```c
struct regmap *syscon_regmap_lookup_by_compatible(const char *s)
```

**ما تعمله:**
بتدور في كل الـ Device Tree عن أول node compatible مع الـ string اللي بتديها، وبترجع الـ regmap بتاعه. مفيدة لما بتعرف الـ compatible string بس مش عندك الـ node pointer.

تخيلها زي البحث في دليل التليفون باسم الشركة — بدل ما تحتاج العنوان الكامل.

**الـ Parameters:**
- `s` — الـ compatible string المطلوب (مثلاً `"vendor,chipname-syscon"`)

**الـ Return Value:**
- `struct regmap *` صالح لو لقى الـ node ونجح
- `ERR_PTR(-ENODEV)` لو ما لقاش node compatible
- أي error من `syscon_node_to_regmap()` لو فشل

**Key Details:**
- تستخدم `of_find_compatible_node()` للبحث في الـ DT
- **مهم جداً:** تستدعي `of_node_put()` بعد الانتهاء لتحرير الـ reference — بدون ده هيحصل memory leak
- تأخذ أول node فقط — لو عندك أكثر من node بنفس الـ compatible، هتاخد الأول

**من يستدعيها:**
الـ drivers اللي تعرف الـ compatible string للـ syscon لكن ما عندهاش reference مباشر للـ node.

---

#### `syscon_regmap_lookup_by_phandle()`

```c
struct regmap *syscon_regmap_lookup_by_phandle(struct device_node *np,
                                                const char *property)
```

**ما تعمله:**
بتقرأ الـ phandle من property معينة في الـ DT، وبتتبع الـ phandle ده للـ syscon node، وبترجع الـ regmap بتاعه. ده الأسلوب الأشيع في الـ DT — driver بيقول "أنا محتاج الـ syscon اللي في property كذا".

تخيلها زي رقم المرجع في الأوراق الرسمية — "ارجع للمستند رقم X" وتلاقي فيه كل المعلومات.

**الـ Parameters:**
- `np` — الـ device node اللي عنده الـ property
- `property` — اسم الـ property اللي فيها الـ phandle (مثلاً `"syscon"` أو `"fsl,syscon"`). لو NULL، بيستخدم `np` مباشرة كـ syscon node

**الـ Return Value:**
- `struct regmap *` صالح لو نجح
- `ERR_PTR(-ENODEV)` لو ما لقاش الـ phandle
- أي error من `syscon_node_to_regmap()`

**Key Details:**
- لو `property` كانت NULL، بيتعامل مع `np` نفسه كـ syscon node — شورتكت مريح
- يستدعي `of_node_put()` بس لو `property` مش NULL (لأن الـ node اتجاب من الـ parse)
- **الأكثر استخداماً** في الـ kernel لاستدعاء الـ syscon

**مثال DT:**
```dts
/* في node الـ driver */
my-driver {
    syscon = <&gpr>;  /* phandle للـ syscon */
};
```
```c
/* في الـ driver code */
regmap = syscon_regmap_lookup_by_phandle(np, "syscon");
```

**من يستدعيها:**
الـ drivers اللي بتحتاج تقرأ أو تكتب في registers موجودة في syscon مشترك.

---

#### `syscon_regmap_lookup_by_phandle_args()`

```c
struct regmap *syscon_regmap_lookup_by_phandle_args(struct device_node *np,
                                                     const char *property,
                                                     int arg_count,
                                                     unsigned int *out_args)
```

**ما تعمله:**
نفس `syscon_regmap_lookup_by_phandle()` بس مع قدرة إضافية — بتستخرج arguments من الـ phandle. يعني الـ DT property مش بس بتشير للـ syscon، لكن كمان بتحدد معلومات إضافية زي offset أو bit mask.

تخيلها زي مرجع مع إحداثيات — "ارجع لمستند X، صفحة 5، سطر 3".

**الـ Parameters:**
- `np` — الـ device node اللي عنده الـ property
- `property` — اسم الـ property في الـ DT
- `arg_count` — عدد الـ arguments المتوقعة بعد الـ phandle
- `out_args` — array يستقبل الـ arguments (الـ caller يحجزه)

**الـ Return Value:**
- `struct regmap *` صالح لو نجح والـ out_args اتملى
- `ERR_PTR(rc)` لو فشل الـ parse
- `ERR_PTR(-ENODEV)` لو الـ node مش موجود

**Key Details:**
- بتستخدم `of_parse_phandle_with_fixed_args()` — تتوقع عدداً محدداً من الـ args
- الـ args بتتنقل من `struct of_phandle_args` لـ `out_args` array
- بتستدعي `of_node_put()` دايماً لأن الـ node اتجاب من الـ parse

**مثال DT:**
```dts
/* property مع phandle + argument */
clock-controller = <&syscon 0x100>;  /* syscon node + offset */
```
```c
unsigned int offset;
regmap = syscon_regmap_lookup_by_phandle_args(np, "clock-controller", 1, &offset);
/* offset = 0x100 */
```

**من يستدعيها:**
الـ drivers اللي تحتاج معلومات إضافية مع الـ regmap زي base offset أو configuration bits.

---

#### `syscon_regmap_lookup_by_phandle_optional()`

```c
struct regmap *syscon_regmap_lookup_by_phandle_optional(struct device_node *np,
                                                         const char *property)
```

**ما تعمله:**
نفس `syscon_regmap_lookup_by_phandle()` بالضبط، بس مع فرق واحد بسيط وهام: لو الـ property مش موجودة (يعني -ENODEV)، بترجع NULL بدل error pointer. ده بيسهل الكود لما الـ syscon اختياري وليس إلزامي.

تخيلها زي سؤال "هل عندك بطاقة ولاء؟" — لو مش عندك، مفيش مشكلة، الطلب بيكمل عادي.

**الـ Parameters:**
- `np` — الـ device node
- `property` — اسم الـ property في الـ DT

**الـ Return Value:**
- `struct regmap *` صالح لو موجود
- `NULL` لو الـ property مش موجودة في الـ DT (بدل error)
- `ERR_PTR` لأي error تاني غير -ENODEV

**Key Details:**
- فقط -ENODEV بيتحول لـ NULL — أي error تاني (زي -EPROBE_DEFER) بيتمرر للـ caller
- ده الـ pattern الصح: الـ caller يقدر يعمل `if (regmap && !IS_ERR(regmap))` بدل الكود الأطول

**مثال استخدام:**
```c
/* بدل الكود الطويل */
regmap = syscon_regmap_lookup_by_phandle_optional(np, "optional-syscon");
if (IS_ERR(regmap))
    return PTR_ERR(regmap);  /* error حقيقي */
if (regmap) {
    /* استخدم الـ regmap */
}
/* لو NULL = مش موجود وده طبيعي */
```

**من يستدعيها:**
الـ drivers اللي الـ syscon فيها optional feature — موجودة في بعض الـ hardware variants بس مش كلها.

---

### ملخص العلاقات بين الـ Functions

```
Client Drivers
     │
     ├── syscon_regmap_lookup_by_compatible()
     │         └── syscon_node_to_regmap()
     │                   └── device_node_get_regmap(create=if_compatible, check_res=true)
     │                               └── of_syscon_register()  [if new]
     │
     ├── syscon_regmap_lookup_by_phandle()
     │         └── syscon_node_to_regmap()  [نفس السابق]
     │
     ├── syscon_regmap_lookup_by_phandle_args()
     │         └── syscon_node_to_regmap()  [نفس السابق]
     │
     ├── syscon_regmap_lookup_by_phandle_optional()
     │         └── syscon_regmap_lookup_by_phandle()  [مع تحويل -ENODEV لـ NULL]
     │
     ├── device_node_to_regmap()
     │         └── device_node_get_regmap(create=true, check_res=false)
     │
     └── of_syscon_register_regmap()  [تسجيل regmap خارجي]
               └── يضيف مباشرة في syscon_list
```

### الـ Locking Strategy

كل الـ functions تمر عبر `syscon_list_lock` (mutex) عند البحث أو الإنشاء أو التسجيل. هذا يضمن إن:

1. الـ `syscon_list` لا يتعدى عليه أكثر من thread في نفس الوقت
2. لا يتم إنشاء نفس الـ syscon مرتين
3. البحث والإنشاء atomic operation واحدة

الـ `of_syscon_register()` تتحقق من القفل بـ `WARN_ON(!mutex_is_locked(...))` — يعني لو استُدعيت بدون قفل، يطلع warning في الـ kernel log فوراً.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بالـ syscon

الـ **syscon** يعتمد على الـ **regmap** تحته، وبما إن الـ regmap عنده integration كاملة مع الـ debugfs، فكل الـ debugging يمشي عبره.

المسار الأساسي:
```
/sys/kernel/debug/regmap/
```

كل syscon مسجّل هيظهر بمجلد باسمه، والاسم يتكون من اسم الـ node في الـ DT + عنوانه:

```bash
# اعرض كل الـ syscon regmaps المسجّلة
ls /sys/kernel/debug/regmap/

# مثال للناتج:
# syscon@1c00000   syscon@fe010000   grf@ff770000
```

داخل كل مجلد، في ملفات مهمة:

| الملف | وظيفته | كيف تقرأه |
|-------|---------|-----------|
| `registers` | قيم كل الـ registers | `cat /sys/kernel/debug/regmap/<name>/registers` |
| `access` | أذونات القراءة والكتابة لكل register | `cat /sys/kernel/debug/regmap/<name>/access` |
| `name` | اسم الـ regmap | `cat /sys/kernel/debug/regmap/<name>/name` |
| `range` | نطاق العناوين المغطاة | `cat /sys/kernel/debug/regmap/<name>/range` |
| `cache_bypass` | هل الـ cache متجاوَز؟ | `cat /sys/kernel/debug/regmap/<name>/cache_bypass` |
| `cache_dirty` | هل في بيانات لم تُكتب بعد؟ | `cat /sys/kernel/debug/regmap/<name>/cache_dirty` |
| `cache_only` | هل الكتابة على الـ cache فقط؟ | `cat /sys/kernel/debug/regmap/<name>/cache_only` |

```bash
# اقرأ كل الـ registers للـ syscon
cat /sys/kernel/debug/regmap/syscon@ff770000/registers

# مثال للناتج:
# 00000000: 00000000
# 00000004: 0000ff00
# 00000008: deadbeef
# ...

# الصيغة: offset: value — كلها بالـ hex
```

---

#### 2. مدخلات الـ sysfs المتعلقة بالـ syscon

الـ syscon driver نفسه ما عنده sysfs خاص، لكن الـ platform device المرتبط به يظهر هنا:

```bash
# ابحث عن الـ syscon device في sysfs
ls /sys/bus/platform/devices/ | grep syscon

# أو ابحث عن طريق الـ compatible string
grep -r "syscon" /sys/bus/platform/devices/*/of_node/compatible 2>/dev/null
```

لو الـ device مربوط بـ hwspinlock:

```bash
# تحقق من حالة الـ hwspinlock
ls /sys/bus/platform/devices/ | grep hwspinlock
cat /sys/bus/platform/devices/<hwspinlock-dev>/of_node/compatible
```

للـ regmap نفسه:

```bash
# اعرض معلومات الـ regmap عبر sysfs (kernel 5.x+)
ls /sys/kernel/debug/regmap/
```

---

#### 3. استخدام الـ ftrace مع الـ syscon

الـ **ftrace** هو أداة الـ tracing الأقوى في الـ kernel. إليك كيف تتتبع كل عمليات الـ syscon:

##### تفعيل الـ function tracing على دوال الـ syscon:

```bash
# اذهب لمجلد الـ tracing
cd /sys/kernel/debug/tracing

# تفعيل الـ function_graph tracer
echo function_graph > current_tracer

# فلترة على دوال الـ syscon والـ regmap فقط
echo 'syscon*' > set_ftrace_filter
echo 'regmap*' >> set_ftrace_filter
echo 'of_syscon*' >> set_ftrace_filter
echo 'device_node_to_regmap' >> set_ftrace_filter

# ابدأ الـ tracing
echo 1 > tracing_on

# شغّل السيناريو اللي تريد تتبعه (مثلاً probe للـ driver)
# insmod mydriver.ko  أو  echo mydevice > /sys/bus/platform/drivers/mydrv/bind

# أوقف الـ tracing
echo 0 > tracing_on

# اقرأ النتيجة
cat trace
```

##### الـ tracepoints الخاصة بالـ regmap (اللي يستخدمها الـ syscon):

```bash
# اعرض كل tracepoints متعلقة بالـ regmap
ls /sys/kernel/debug/tracing/events/regmap/

# الـ events المتاحة عادةً:
# regmap_reg_read       — عند قراءة register
# regmap_reg_write      — عند كتابة register
# regmap_hw_read_start  — بداية القراءة من الـ hardware
# regmap_hw_read_done   — انتهاء القراءة من الـ hardware
# regmap_hw_write_start — بداية الكتابة للـ hardware
# regmap_hw_write_done  — انتهاء الكتابة للـ hardware

# فعّل كل events الـ regmap
echo 1 > /sys/kernel/debug/tracing/events/regmap/enable

# أو فعّل event واحد فقط
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/enable
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_read/enable

# شاهد الـ trace live
cat /sys/kernel/debug/tracing/trace_pipe
```

##### مثال على ناتج الـ regmap tracepoints:

```
# مثال للناتج:
          <...>-1234  [000] .... 1234.567890: regmap_reg_write: syscon@ff770000 (4) <= 0x00ff00ff
          <...>-1234  [000] .... 1234.567891: regmap_reg_read:  syscon@ff770000 (8) => 0xdeadbeef

# التفسير:
# syscon@ff770000 = اسم الـ regmap
# (4) = offset الـ register بالـ bytes
# <= 0x00ff00ff = القيمة المكتوبة
# => 0xdeadbeef = القيمة المقروءة
```

---

#### 4. الـ printk والـ dynamic debug

الـ syscon driver يستخدم `pr_err` لطباعة الأخطاء، لكن للحصول على رسائل أكثر تفصيلاً:

##### تفعيل الـ dynamic debug للـ syscon:

```bash
# تفعيل كل رسائل الـ debug في ملف syscon.c
echo 'file syscon.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل رسائل الـ regmap
echo 'module regmap +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع الـ stack trace لكل رسالة
echo 'file syscon.c +ps' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع timestamp
echo 'file syscon.c +pt' > /sys/kernel/debug/dynamic_debug/control

# تعطيل بعد الانتهاء
echo 'file syscon.c -p' > /sys/kernel/debug/dynamic_debug/control
```

##### من الـ kernel command line (في الـ bootloader):

```
dyndbg="file syscon.c +p; file regmap-mmio.c +p"
```

##### قراءة الـ kernel log:

```bash
# اقرأ الـ dmesg مع فلترة على syscon
dmesg | grep -i syscon

# اقرأ realtime
dmesg -w | grep -E "syscon|regmap"

# اعرض مستويات الـ log المختلفة
dmesg --level=err,warn | grep syscon
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| الـ Config Option | الوظيفة | متى تستخدمها |
|------------------|---------|-------------|
| `CONFIG_MFD_SYSCON` | تفعيل الـ syscon driver أصلاً | **دائماً** عند العمل مع syscon |
| `CONFIG_REGMAP` | دعم الـ regmap framework | **دائماً** — syscon يعتمد عليه |
| `CONFIG_REGMAP_DEBUGFS` | ظهور الـ registers في debugfs | مهم جداً للـ debugging |
| `CONFIG_REGMAP_MMIO` | دعم الـ MMIO (memory-mapped I/O) | **دائماً** — syscon يستخدم MMIO |
| `CONFIG_DEBUG_FS` | تفعيل الـ debugfs | شرط لكل أدوات الـ debugging |
| `CONFIG_DYNAMIC_DEBUG` | دعم الـ dynamic debug | لرسائل debug مفصّلة |
| `CONFIG_HWSPINLOCK` | دعم الـ hardware spinlock | لو الـ syscon مشترك بين معالجات |
| `CONFIG_RESET_CONTROLLER` | دعم الـ reset controller | لو الـ syscon عنده reset |
| `CONFIG_OF` | دعم الـ Device Tree | **دائماً** — syscon يعتمد على DT |
| `CONFIG_FTRACE` | تفعيل الـ function tracing | للـ tracing المتقدم |
| `CONFIG_TRACING` | البنية الأساسية للـ tracing | شرط للـ ftrace |
| `CONFIG_LOCKDEP` | كشف مشاكل الـ locks | لو تشك في deadlock على الـ mutex |
| `CONFIG_DEBUG_MUTEXES` | debug للـ mutex | لكشف مشاكل الـ syscon_list_lock |
| `CONFIG_PROVE_LOCKING` | إثبات صحة الـ locking | فحص WARN_ON في of_syscon_register |

للتحقق من الـ config الحالي:

```bash
# ابحث عن كل options متعلقة
zcat /proc/config.gz | grep -E "SYSCON|REGMAP|MFD"

# أو
cat /boot/config-$(uname -r) | grep -E "SYSCON|REGMAP|MFD"
```

---

#### 6. أدوات الـ devlink والأدوات الخاصة بالـ subsystem

الـ syscon لا يستخدم الـ devlink مباشرة، لكن هناك أدوات مفيدة:

##### أداة regmap-tools (لو متاحة):

```bash
# قراءة register معين من سطر الأوامر عبر debugfs
REG_OFFSET=0x4
cat /sys/kernel/debug/regmap/syscon@ff770000/registers | grep "^$(printf '%08x' $REG_OFFSET):"
```

##### استخدام /dev/mem مع devmem2:

```bash
# قراءة مباشرة من العنوان الفيزيائي (بدون kernel)
devmem2 0xff770004 w    # قراءة word من عنوان الـ syscon + offset 4
devmem2 0xff770004 w 0x0000ff00   # كتابة قيمة
```

##### فحص الـ device عبر sysfs:

```bash
# اعرض كل خصائص الـ device tree node
find /sys/firmware/devicetree/base -name "compatible" -exec grep -l "syscon" {} \;

# اقرأ خصائص node معين
cat /sys/firmware/devicetree/base/syscon@ff770000/reg | xxd
cat /sys/firmware/devicetree/base/syscon@ff770000/compatible
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|------------|---------|------|
| `regmap init failed` | فشل إنشاء الـ regmap أثناء `regmap_init_mmio` | تحقق من عنوان الـ `reg` في الـ DT، وتأكد أن الـ MMIO mapping صحيح |
| `Failed to retrieve valid hwlock: <N>` | فشل الحصول على الـ hardware spinlock المحدد في الـ DT | تأكد من تفعيل `CONFIG_HWSPINLOCK` وأن الـ hwlock ID صحيح في الـ DT |
| `-EPROBE_DEFER` (سيُعاد المحاولة لاحقاً) | الـ syscon لم يُسجَّل بعد عند محاولة client الوصول إليه | ترتيب الـ probe — تأكد أن الـ syscon يتم probing قبل الـ client driver |
| `-ENODEV` من `syscon_regmap_lookup_by_compatible` | لم يُعثر على node بهذا الـ compatible string في الـ DT | تحقق من أن الـ DT node موجود وأن الـ compatible string مطابق تماماً |
| `-ENODEV` من `syscon_regmap_lookup_by_phandle` | الـ phandle property غير موجودة في الـ DT node | أضف الـ property الصحيحة في الـ DT للـ client device |
| `-EEXIST` في `of_syscon_register_regmap` | محاولة تسجيل regmap لنفس الـ DT node مرتين | تحقق من أنك لا تستدعي `of_syscon_register_regmap` مرتين لنفس الـ node |
| `-ENOMEM` عند `of_iomap` | فشل الـ ioremap — لا توجد ذاكرة كافية أو العنوان خاطئ | تحقق من صحة عنوان الـ `reg` في الـ DT وأن النطاق لم يُحجز مسبقاً |
| `-EFAULT` عندما `res_size < reg_io_width` | حجم الـ register space أصغر من عرض الـ register | تحقق من `reg-io-width` في الـ DT — يجب أن يكون 1، 2، أو 4 |
| `WARNING: bad unlock balance detected!` | مشكلة في `syscon_list_lock` | يُشير إلى bug في الـ driver، فعّل `CONFIG_LOCKDEP` |
| `-EINVAL` من `of_syscon_register_regmap` | تم تمرير `np` أو `regmap` بقيمة NULL | تحقق من الـ pointers قبل الاستدعاء |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

بناءً على الكود، هذه النقاط الأهم للـ debugging المتعمق:

```c
/* نقطة 1: بعد محاولة lookup الـ syscon — لكشف سبب الفشل */
regmap = syscon_regmap_lookup_by_compatible("vendor,my-syscon");
if (IS_ERR(regmap)) {
    /* الـ WARN_ON موجود أصلاً في of_syscon_register للتحقق من الـ mutex */
    pr_err("syscon lookup failed: %ld\n", PTR_ERR(regmap));
    dump_stack(); /* اعرف من أين جاء الطلب */
}

/* نقطة 2: للتحقق من أن الـ mutex محجوز قبل التسجيل */
/* هذا WARN_ON موجود أصلاً في الكود: */
WARN_ON(!mutex_is_locked(&syscon_list_lock));
/* يُطلق WARNING لو أحد استدعى of_syscon_register خارج الـ lock */

/* نقطة 3: عند كتابة register حساس */
ret = regmap_write(regmap, REG_OFFSET, value);
WARN_ON(ret != 0); /* لو الكتابة فشلت — غير متوقع في MMIO */

/* نقطة 4: عند قراءة register للتحقق من الـ hardware state */
ret = regmap_read(regmap, REG_OFFSET, &val);
if (ret) {
    WARN_ON(1);
    dump_stack();
}
```

نقاط **استراتيجية** يمكن إضافة debugging فيها (بدون تعديل الكود الأصلي):

```
1. في device_node_get_regmap() — عند عدم إيجاد الـ syscon في القائمة
2. في of_syscon_register() — بعد of_iomap() لتأكيد العنوان
3. في syscon_regmap_lookup_by_phandle() — بعد of_parse_phandle()
4. في of_syscon_register_regmap() — للتحقق من عدم التكرار
```

---

### Hardware Level

---

#### 1. كيف تتحقق أن حالة الـ Hardware تطابق حالة الـ Kernel

الـ syscon يمثّل مجموعة registers في الـ SoC تتحكم في النظام. الخطوات:

```bash
# الخطوة 1: اعرف العنوان الفيزيائي للـ syscon من الـ DT
cat /sys/firmware/devicetree/base/syscon@ff770000/reg | xxd
# أو
hexdump -C /sys/firmware/devicetree/base/syscon@ff770000/reg

# مثال للناتج (big-endian cells):
# 00000000  00 00 00 00 ff 77 00 00  00 00 00 00 00 01 00 00  |.....w..........|
# العنوان: 0xff770000، الحجم: 0x10000

# الخطوة 2: اقرأ القيمة عبر kernel (regmap)
cat /sys/kernel/debug/regmap/syscon@ff770000/registers | head -20

# الخطوة 3: اقرأ نفس القيمة مباشرة من الـ hardware
devmem2 0xff770000 w   # register 0
devmem2 0xff770004 w   # register 1

# الخطوة 4: قارن النتيجتين — يجب أن تكونا متطابقتين
```

لو في فرق بين قراءة الـ kernel وقراءة devmem2 المباشرة:
- الـ regmap عنده cache مفعّل (غير متوقع للـ syscon لأنه MMIO بدون cache عادة)
- قد يكون هناك مشكلة في الـ endianness
- قد يكون الـ hardware يتغير بشكل غير متوقع

---

#### 2. تقنيات الـ Register Dump

##### باستخدام devmem2:

```bash
# تثبيت devmem2
apt-get install devmem2   # أو من المصدر

# قراءة register واحد
devmem2 0xff770000 w   # w = 32-bit word

# قراءة byte واحد (لو reg-io-width=1)
devmem2 0xff770000 b

# قراءة halfword (لو reg-io-width=2)
devmem2 0xff770000 h

# كتابة قيمة
devmem2 0xff770000 w 0x12345678
```

##### سكريبت لعمل dump كامل للـ syscon:

```bash
#!/bin/bash
# dump_syscon.sh — اعمل dump لكل registers الـ syscon

BASE_ADDR=0xff770000    # عنوان الـ syscon
SIZE=0x1000             # حجم نطاق الـ registers
STEP=4                  # عرض الـ register بالـ bytes

echo "=== Syscon Register Dump @ 0x$(printf '%x' $BASE_ADDR) ==="
for ((offset=0; offset<SIZE; offset+=STEP)); do
    addr=$((BASE_ADDR + offset))
    val=$(devmem2 $addr w 2>/dev/null | grep "Read" | awk '{print $NF}')
    echo "  0x$(printf '%08x' $offset): $val"
done
```

##### باستخدام /dev/mem مباشرة:

```bash
# قراءة 256 byte من العنوان الفيزيائي
dd if=/dev/mem bs=4 count=64 skip=$((0xff770000/4)) 2>/dev/null | xxd

# تحذير: يحتاج CONFIG_STRICT_DEVMEM=n أو يستخدم iomem=relaxed في kernel cmdline
```

##### عبر الـ debugfs (الطريقة الموصى بها):

```bash
# الأسهل والأأمن — يمر عبر الـ kernel
cat /sys/kernel/debug/regmap/syscon@ff770000/registers

# حفظ snapshot
cat /sys/kernel/debug/regmap/syscon@ff770000/registers > /tmp/syscon_before.txt
# ... شغّل السيناريو ...
cat /sys/kernel/debug/regmap/syscon@ff770000/registers > /tmp/syscon_after.txt
diff /tmp/syscon_before.txt /tmp/syscon_after.txt
```

---

#### 3. نصائح Logic Analyzer و Oscilloscope

الـ syscon يعمل على الـ APB bus أو AHB bus داخل الـ SoC — الوصول إليه بالـ logic analyzer صعب لأنه داخلي، لكن:

##### لو عندك JTAG debugger (مثل OpenOCD):

```
# في OpenOCD
# اقرأ register الـ syscon مباشرة عبر JTAG
> mdw phys 0xff770000 16
# يعرض 16 words بدءاً من العنوان

# اكتب قيمة
> mww phys 0xff770000 0x12345678
```

##### للـ logic analyzer الخارجي:

- ركّز على الـ pins الخارجية المتحكم بها بالـ syscon (مثل GPIO direction، clock outputs)
- راقب pin معين قبل وبعد كتابة الـ register المقابل

##### نصائح عملية:

```
1. استخدم GPIO pins كـ markers — اكتب في GPIO register قبل العملية وبعدها
2. راقب التغيير في الـ output clock عند تغيير clock divider register في الـ syscon
3. استخدم JTAG لقراءة الـ registers بدون تأثير على الـ system
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | نمط الـ kernel log | التفسير |
|---------|------------------|---------|
| عنوان خاطئ في الـ DT | `of_iomap: failed to iomap` أو `syscon@XXXX: resource size N is too small` | الـ `reg` property في الـ DT لا يطابق الـ hardware الفعلي |
| الـ hardware غير موجود | `regmap init failed` بعد `of_iomap` | العنوان غير مُعيَّن في الـ memory map للـ SoC |
| مشكلة endianness | القيم المقروءة معكوسة | تحتاج `big-endian` أو `little-endian` property في الـ DT |
| hwspinlock لم يُفعَّل | `Failed to retrieve valid hwlock: -ENODEV` | `CONFIG_HWSPINLOCK` غير مفعّل أو الـ ID خاطئ |
| reset controller مشكلة | `reset_control_deassert failed` | الـ reset signal لم يُرفع — الـ hardware في حالة reset |
| clock مفقود | `of_clk_get` يعيد error غير `-ENOENT` | الـ clock المحدد في الـ DT غير مسجَّل |
| تعارض في الـ MMIO region | `ioremap of XX size YY failed` | منطقة الـ memory محجوزة من driver آخر |
| الـ register read-only | كتابة ناجحة لكن القيمة لا تتغير | بعض registers الـ syscon read-only في الـ hardware — راجع الـ datasheet |

---

#### 5. Debugging الـ Device Tree

الـ syscon يعتمد **اعتماداً كلياً** على الـ Device Tree — أي خطأ في الـ DT = مشكلة في الـ syscon.

##### التحقق من الـ DT node:

```bash
# اعرض كل خصائص الـ syscon node
ls /sys/firmware/devicetree/base/syscon@ff770000/
# أو
cat /sys/firmware/devicetree/base/syscon@ff770000/compatible

# تحقق من الـ reg property (العنوان والحجم)
hexdump -C /sys/firmware/devicetree/base/syscon@ff770000/reg

# تحقق من الـ reg-io-width
hexdump -C /sys/firmware/devicetree/base/syscon@ff770000/reg-io-width 2>/dev/null \
    && echo "reg-io-width موجود" || echo "reg-io-width غير موجود (default=4)"

# تحقق من الـ endianness
ls /sys/firmware/devicetree/base/syscon@ff770000/ | grep -E "endian"
```

##### مثال DT node صحيح:

```dts
syscon: syscon@ff770000 {
    compatible = "rockchip,rk3399-grf", "syscon";
    reg = <0x0 0xff770000 0x0 0x1000>;   /* العنوان والحجم */
    reg-io-width = <4>;                   /* 32-bit access */
    /* big-endian; */                     /* uncomment لو الـ SoC big-endian */
};
```

##### التحقق من الـ phandle references:

```bash
# ابحث عن كل devices تشير للـ syscon عبر phandle
grep -r "syscon" /sys/firmware/devicetree/base/ 2>/dev/null | grep -v Binary

# تحقق من أن الـ phandle يشير للـ node الصحيح
# الـ phandle هو رقم، اعثر عليه في الـ DT blob
dtc -I dtb -O dts /boot/dtb-file.dtb | grep -A5 "syscon"
```

##### استخدام fdtdump:

```bash
# استخرج الـ DT blob واعرضه
dtc -I dtb -O dts /sys/firmware/fdt > /tmp/current.dts
grep -A 20 "syscon" /tmp/current.dts
```

##### التحقق من تطابق الـ DT مع الـ Hardware:

```bash
# 1. اعرف العنوان من الـ DT
REG=$(hexdump -e '"%08x\n"' /sys/firmware/devicetree/base/syscon@ff770000/reg | head -1)
echo "Physical address from DT: 0x$REG"

# 2. تحقق أن هذا العنوان موجود في /proc/iomem
grep -i "ff770000" /proc/iomem

# مثال للناتج الصحيح:
# ff770000-ff770fff : syscon@ff770000

# 3. لو غير موجود → الـ DT يشير لعنوان خاطئ أو الـ region غير مُعرَّف في الـ SoC
```

---

### Practical Commands

---

#### مجموعة أوامر جاهزة للنسخ

##### فحص شامل سريع للـ syscon:

```bash
#!/bin/bash
# quick_syscon_debug.sh

echo "=== 1. الـ syscon devices المسجلة في regmap debugfs ==="
ls /sys/kernel/debug/regmap/ 2>/dev/null || echo "debugfs غير متاح"

echo ""
echo "=== 2. الـ syscon في sysfs ==="
ls /sys/bus/platform/devices/ 2>/dev/null | grep -i syscon

echo ""
echo "=== 3. رسائل kernel المتعلقة بالـ syscon ==="
dmesg | grep -i "syscon" | tail -20

echo ""
echo "=== 4. الـ memory regions المحجوزة ==="
grep -i "syscon\|grf\|pmugrf" /proc/iomem

echo ""
echo "=== 5. الـ DT nodes للـ syscon ==="
find /sys/firmware/devicetree/base -name "compatible" 2>/dev/null | while read f; do
    if grep -q "syscon" "$f" 2>/dev/null; then
        echo "Found: $(dirname $f)"
        cat "$f" 2>/dev/null
        echo ""
    fi
done
```

##### قراءة register محدد من الـ syscon عبر debugfs:

```bash
# اقرأ register عند offset 0x10
SYSCON_NAME="syscon@ff770000"
OFFSET=0x10

grep "^$(printf '%08x' $OFFSET):" /sys/kernel/debug/regmap/$SYSCON_NAME/registers
```

##### تفعيل الـ regmap tracing الكامل:

```bash
# سكريبت لتفعيل كل الـ regmap events
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo 1 > /sys/kernel/debug/tracing/events/regmap/enable
echo 1 > tracing_on

# شغّل الـ driver أو العملية المطلوبة هنا
sleep 5

echo 0 > tracing_on
echo "=== Regmap Events ==="
cat trace | grep "syscon"
```

##### فحص الـ regmap config:

```bash
# اعرض معلومات الـ regmap
SYSCON_NAME="syscon@ff770000"
echo "=== Regmap Name ===" && cat /sys/kernel/debug/regmap/$SYSCON_NAME/name
echo "=== Regmap Range ===" && cat /sys/kernel/debug/regmap/$SYSCON_NAME/range 2>/dev/null
echo "=== Cache Status ==="
echo -n "cache_bypass: " && cat /sys/kernel/debug/regmap/$SYSCON_NAME/cache_bypass 2>/dev/null
echo -n "cache_only: "   && cat /sys/kernel/debug/regmap/$SYSCON_NAME/cache_only 2>/dev/null
echo -n "cache_dirty: "  && cat /sys/kernel/debug/regmap/$SYSCON_NAME/cache_dirty 2>/dev/null
```

##### مقارنة snapshot قبل وبعد عملية معينة:

```bash
SYSCON_NAME="syscon@ff770000"
SNAP="/tmp/syscon_snap"

# snapshot قبل
cp /sys/kernel/debug/regmap/$SYSCON_NAME/registers ${SNAP}_before.txt
echo "Snapshot before taken"

# ... شغّل العملية ...
read -p "اضغط Enter بعد تنفيذ العملية..."

# snapshot بعد
cp /sys/kernel/debug/regmap/$SYSCON_NAME/registers ${SNAP}_after.txt

# المقارنة
echo "=== الـ registers التي تغيرت ==="
diff ${SNAP}_before.txt ${SNAP}_after.txt
```

##### تفعيل dynamic debug لكل مكونات الـ syscon:

```bash
# فعّل debug للـ syscon وكل ما يتعلق به
for module in syscon regmap regmap-mmio; do
    echo "module $module +p" > /sys/kernel/debug/dynamic_debug/control 2>/dev/null \
        && echo "Enabled: $module" \
        || echo "Not available: $module"
done

# راقب الـ kernel log
dmesg -w
```

##### فحص كامل للـ DT vs Hardware:

```bash
#!/bin/bash
# verify_syscon_dt.sh

echo "=== التحقق من تطابق DT مع الـ Hardware ==="

for syscon_dir in /sys/firmware/devicetree/base/*/; do
    compat_file="$syscon_dir/compatible"
    if [ -f "$compat_file" ] && grep -q "syscon" "$compat_file" 2>/dev/null; then
        node_name=$(basename "$syscon_dir")
        echo ""
        echo "--- Node: $node_name ---"
        echo -n "Compatible: "
        strings "$compat_file" 2>/dev/null | tr '\n' ' '
        echo ""

        # اقرأ الـ reg property
        if [ -f "$syscon_dir/reg" ]; then
            echo -n "DT Address: 0x"
            hexdump -e '4/1 "%02x"' "$syscon_dir/reg" 2>/dev/null | cut -c9-16

            # تحقق من /proc/iomem
            addr=$(hexdump -e '4/1 "%02x"' "$syscon_dir/reg" 2>/dev/null | cut -c9-16)
            echo -n "In /proc/iomem: "
            grep -i "$addr" /proc/iomem | head -1 || echo "NOT FOUND!"
        fi

        # تحقق من وجود regmap
        echo -n "Regmap in debugfs: "
        ls /sys/kernel/debug/regmap/ 2>/dev/null | grep "${node_name%%@*}" | head -1 \
            || echo "NOT registered"
    fi
done
```

---

#### تفسير النتائج

##### مثال على ناتج `/sys/kernel/debug/regmap/syscon@ff770000/registers`:

```
00000000: 00000000    ← offset 0x00: قيمته 0x00000000
00000004: 0000ff00    ← offset 0x04: قيمته 0x0000ff00
00000008: deadbeef    ← offset 0x08: قيمته 0xdeadbeef
0000000c: 00000001    ← offset 0x0c: قيمته 0x00000001
```

**التفسير**: كل سطر = `<offset>: <value>` — كلها بالـ hex. الـ offset هو البُعد عن بداية الـ syscon region.

##### مثال على ناتج الـ ftrace للـ regmap:

```
          insmod-234   [001] .... 100.123456: regmap_reg_write: grf@ff770000 (0) <= 0x00000003
          driver-456   [000] .... 100.234567: regmap_reg_read:  grf@ff770000 (4) => 0x0000ff00
```

**التفسير**:
- `insmod-234` = اسم الـ process + PID
- `[001]` = رقم الـ CPU
- `100.123456` = الوقت بالثواني
- `grf@ff770000` = اسم الـ regmap
- `(0)` = offset الـ register
- `<= 0x00000003` = قيمة مكتوبة
- `=> 0x0000ff00` = قيمة مقروءة
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: بورد RK3562 — UART لا يشتغل في gateway صناعي

#### العنوان
**الـ UART controller على RK3562 يرفض probe بسبب syscon مش موجود في الـ DT**

#### السياق
شركة تعمل على industrial gateway بمعالج **RK3562**. البورد يحتوي على UART4 مربوط بـ RS-485 لقراءة بيانات من أجهزة قياس. الكيرنل يبوت، بس عند تشغيل الـ driver خاص بـ RS-485 controller، ما يشوف الـ port أبداً.

#### المشكلة
الـ driver الخاص بـ Rockchip UART يستخدم `syscon_regmap_lookup_by_phandle()` عشان يوصل لـ GRF (General Register Files) — السجلات العامة اللي تتحكم في الـ pinmux وخصائص الـ UART. الـ engineer الجديد نسي يضيف الـ `grf` phandle في الـ device tree node الخاص بـ UART4.

```
[    2.341209] rockchip-uart fe660000.serial: no syscon regmap
[    2.341215] rockchip-uart fe660000.serial: probe failed: -ENODEV
```

#### التحليل

الـ driver بيعمل كالتالي:

```c
/* داخل driver الـ UART على Rockchip */
regmap = syscon_regmap_lookup_by_phandle(dev->of_node, "rockchip,grf");
if (IS_ERR(regmap))
    return PTR_ERR(regmap); /* ترجع -ENODEV هنا */
```

لما ننظر داخل `syscon_regmap_lookup_by_phandle()` في `syscon.c`:

```c
struct regmap *syscon_regmap_lookup_by_phandle(struct device_node *np,
                                               const char *property)
{
    struct device_node *syscon_np;
    struct regmap *regmap;

    if (property)
        syscon_np = of_parse_phandle(np, property, 0);
    else
        syscon_np = np;

    if (!syscon_np)
        return ERR_PTR(-ENODEV); /* هنا المشكلة — phandle مش موجود */
    ...
}
```

الدالة `of_parse_phandle()` ترجع `NULL` لأن الـ property `rockchip,grf` مش موجودة في الـ DT node، فتُرجع `ERR_PTR(-ENODEV)` مباشرة.

#### الحل

**1. تعديل الـ Device Tree:**

```dts
/* قبل التعديل — ناقص الـ phandle */
&uart4 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&uart4_xfer>;
};

/* بعد التعديل — أضف الـ grf reference */
&uart4 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&uart4_xfer>;
    rockchip,grf = <&grf>; /* الإشارة للـ GRF syscon node */
};
```

**2. تأكد إن الـ GRF node موجود وصح:**

```dts
grf: syscon@ff000000 {
    compatible = "rockchip,rk3562-grf", "syscon";
    reg = <0x0 0xff000000 0x0 0x10000>;
};
```

**3. تحقق من النتيجة:**

```bash
# تحقق من إن الـ node اتسجل صح في syscon list
grep -r "grf" /sys/bus/platform/devices/ 2>/dev/null
dmesg | grep -i "uart4\|syscon"
```

#### الدرس المستفاد
الـ `syscon_regmap_lookup_by_phandle()` ترجع `-ENODEV` لما الـ phandle مش موجود في الـ DT، مش `-EINVAL`. دايماً افحص الـ DT node بشكل كامل وتأكد إن كل الـ references موجودة. استخدم `dtc -I fs /sys/firmware/devicetree/base` لتفريغ الـ DT الحالي وفحصه.

---

### السيناريو 2: STM32MP1 — I2C يتجمد بسبب hardware spinlock

#### العنوان
**الـ I2C على STM32MP1 يتجمد عند استخدام syscon مشترك بين الـ A7 core والـ M4 core**

#### السياق
منتج IoT sensor hub يعمل على **STM32MP1** اللي فيه معالجين: Cortex-A7 يشغّل Linux، وCortex-M4 يشغّل RTOS خفيف. الاثنين يحتاجون يكتبوا في نفس **SYSCFG** registers (System Configuration). المهندس استخدم الـ `hwspinlock` عشان يمنع التعارض، بس الـ I2C بدأ يتجمد بشكل عشوائي.

#### المشكلة
الـ `of_syscon_register()` في `syscon.c` تقرأ الـ hwspinlock config من الـ DT:

```c
ret = of_hwspin_lock_get_id(np, 0);
if (ret > 0 || (IS_ENABLED(CONFIG_HWSPINLOCK) && ret == 0)) {
    syscon_config.use_hwlock = true;
    syscon_config.hwlock_id = ret;
    syscon_config.hwlock_mode = HWLOCK_IRQSTATE; /* يحجب الـ interrupts */
}
```

الـ `HWLOCK_IRQSTATE` معناه إن الـ kernel يحجب الـ interrupts لما يأخذ الـ lock. المشكلة إن الـ M4 أحياناً يمسك الـ hwspinlock لفترة طويلة، فالـ A7 يضل ينتظر مع interrupts محجوبة، وده يسبب تجميد الـ I2C interrupt handler.

#### التحليل

الـ flow اللي يحصل:

```
A7 (Linux)                          M4 (RTOS)
────────────────────────────────    ─────────────────────────
regmap_write(syscon_regmap, ...)
  └─> regmap_lock() بداخله
      └─> hwspin_lock_irqsave()
          └─ ينتظر...              ← M4 مسك الـ lock (20ms!)

  [I2C interrupt وصل هنا]
  [ما قدر يشتغل لأن IRQ محجوب]
  [I2C timeout!]
```

#### الحل

**الحل 1: تقليل وقت الـ M4 لما يمسك الـ lock**

ده يتطلب تعديل firmware الـ M4 — خارج نطاق الـ kernel.

**الحل 2: استخدام `of_syscon_register_regmap()` مع regmap مخصص**

بدل ما نخلي الـ syscon يدير الـ lock بنفسه، نسجّل regmap مخصص بـ timeout:

```c
/* في driver مخصص */
static struct regmap_config custom_syscfg_config = {
    .reg_bits = 32,
    .val_bits = 32,
    .reg_stride = 4,
    .use_hwlock = true,
    .hwlock_id = 0,
    .hwlock_mode = HWLOCK_IRQSTATE,
    /* أضف timeout handling مخصص */
};

/* بعد إنشاء الـ regmap */
of_syscon_register_regmap(np, custom_regmap);
```

**الحل 3 (الأبسط): فصل الـ registers**

```dts
/* اعمل syscon منفصل لكل core */
syscfg_a7: syscon@50020000 {
    compatible = "st,stm32mp1-syscfg", "syscon";
    reg = <0x50020000 0x400>;
    /* بدون hwspinlock — للـ A7 فقط */
};

syscfg_shared: syscon@50020400 {
    compatible = "syscon";
    reg = <0x50020400 0x100>;
    hwlocks = <&hwspinlock 0>; /* فقط للـ registers المشتركة */
};
```

**فحص المشكلة:**

```bash
# شوف إذا في lock contention
cat /sys/kernel/debug/hwspinlock/lock0/owner
# أو راقب الـ timing
trace-cmd record -e hwspinlock:* sleep 10
trace-cmd report | grep "lock_id=0"
```

#### الدرس المستفاد
الـ `HWLOCK_IRQSTATE` mode في syscon خطير لو الـ lock ممكن يتمسك من core ثاني لفترة طويلة. الـ `of_syscon_register()` تضبط الـ hwlock mode تلقائياً من الـ DT بدون ما تعطيك control على الـ timeout. في الأنظمة asymmetric multiprocessing، فكّر مرتين قبل ما تشارك syscon بين cores مختلفة.

---

### السيناريو 3: i.MX8MQ — HDMI لا يعمل بسبب endianness خاطئ

#### العنوان
**شاشة HDMI على i.MX8MQ تظهر ألوان مقلوبة بسبب big-endian خاطئ في syscon**

#### السياق
Android TV box يعمل على **i.MX8MQ** من NXP. المهندس يعمل bring-up لـ HDMI controller. الشاشة تشتغل، بس الألوان غريبة جداً — الأحمر يظهر أزرق والعكس. المهندس يشك في الـ color space conversion، بس المشكلة أعمق من كده.

#### المشكلة

الـ HDMI على i.MX8MQ يستخدم **HDMIMIX** block اللي بيتعامل معه عبر syscon. المهندس بالغلط كتب `big-endian` في الـ DT node للـ HDMIMIX syscon، بينما الـ SoC little-endian أصلاً.

#### التحليل

داخل `of_syscon_register()`:

```c
/* الـ syscon.c يقرأ الـ endianness من الـ DT */
if (of_property_read_bool(np, "big-endian"))
    syscon_config.val_format_endian = REGMAP_ENDIAN_BIG;
else if (of_property_read_bool(np, "little-endian"))
    syscon_config.val_format_endian = REGMAP_ENDIAN_LITTLE;
else if (of_property_read_bool(np, "native-endian"))
    syscon_config.val_format_endian = REGMAP_ENDIAN_NATIVE;
/* لو ما في شيء → REGMAP_ENDIAN_DEFAULT (native) */
```

لما الـ driver يكتب قيمة للـ HDMIMIX:

```c
/* الـ driver يريد يكتب 0x00FF0000 (أحمر كامل في RGB) */
regmap_write(hdmimix_regmap, COLOR_REG, 0x00FF0000);
```

بسبب `REGMAP_ENDIAN_BIG`، الـ regmap يعكس الـ bytes قبل الكتابة:
- القيمة المطلوبة: `0x00 FF 00 00`
- اللي يتكتب فعلاً: `0x00 00 FF 00`

النتيجة: الألوان مقلوبة تماماً في الـ output.

**كيف تكتشف المشكلة:**

```bash
# اقرأ قيمة register مباشرة من memory وقارنها مع regmap
# عنوان HDMIMIX على i.MX8MQ
devmem2 0x32fc0000 w  # قراءة مباشرة من الذاكرة

# ثم اقرأ نفس الـ register عبر regmap debugfs
cat /sys/kernel/debug/regmap/hdmimix@32fc0000/registers | grep "^000:"
```

لو القيم مختلفة → فيه مشكلة endianness.

#### الحل

**تصحيح الـ DT:**

```dts
/* قبل — خاطئ */
hdmimix: syscon@32fc0000 {
    compatible = "fsl,imx8mq-hdmimix", "syscon";
    reg = <0x0 0x32fc0000 0x0 0x1000>;
    big-endian; /* خاطئ! */
};

/* بعد — صح */
hdmimix: syscon@32fc0000 {
    compatible = "fsl,imx8mq-hdmimix", "syscon";
    reg = <0x0 0x32fc0000 0x0 0x1000>;
    /* بدون endianness property = native = little-endian على ARM */
};
```

**التحقق:**

```bash
# بعد التعديل وإعادة البوت
dmesg | grep hdmimix
# تأكد إن الـ regmap اتسجل بشكل صحيح
ls /sys/kernel/debug/regmap/
```

#### الدرس المستفاد
الـ `of_syscon_register()` تأخذ الـ endianness مباشرة من الـ DT بدون أي validation. لو كتبت `big-endian` بالغلط على SoC اعتيادي little-endian ARM، الـ regmap يعكس كل القيم. في حالة الشك، اقرأ الـ register مباشرة بـ `devmem2` وقارنها مع قراءة الـ regmap.

---

### السيناريو 4: AM62x — driver لا يعمل بسبب -EPROBE_DEFER عشوائي

#### العنوان
**الـ USB driver على AM62x يفشل أحياناً في الـ boot بسبب syscon غير موجود وقت الـ probe**

#### السياق
لوح تطوير صناعي يعمل على **TI AM62x** (Texas Instruments). الـ USB subsystem يستخدم **CTRL_MMR** (Control Module Registers) عبر syscon لضبط الـ USB PHY. أحياناً في الـ boot الـ USB يشتغل، وأحياناً ما يشتغل — المشكلة عشوائية وتُربك الفريق.

#### المشكلة

الـ USB PHY driver يستخدم `syscon_regmap_lookup_by_phandle()` لجلب الـ CTRL_MMR regmap. أحياناً الـ CTRL_MMR node ما اتسجّل في الـ syscon_list بعد لما الـ USB PHY يبدأ probe.

داخل `device_node_get_regmap()`:

```c
static struct regmap *device_node_get_regmap(struct device_node *np,
                                             bool create_regmap,
                                             bool check_res)
{
    ...
    list_for_each_entry(entry, &syscon_list, list)
        if (entry->np == np) {
            syscon = entry; /* لو موجود، رجّعه */
            break;
        }

    if (!syscon) {
        if (create_regmap)
            syscon = of_syscon_register(np, check_res); /* إنشاء جديد */
        else
            syscon = ERR_PTR(-EPROBE_DEFER); /* انتظر */
    }
    ...
}
```

لما الـ USB PHY يستخدم `syscon_node_to_regmap()`:

```c
struct regmap *syscon_node_to_regmap(struct device_node *np)
{
    /* create_regmap = true فقط لو compatible مع "syscon" */
    return device_node_get_regmap(np,
        of_device_is_compatible(np, "syscon"), true);
}
```

المشكلة: لو الـ CTRL_MMR node مش compatible مع `"syscon"` الـ generic، الـ `create_regmap` = `false`، فيرجع `-EPROBE_DEFER`. لو الـ CTRL_MMR node فيه compatible `"ti,am62-ctrl-mmr"` بس مش `"syscon"`، ما ينشئ الـ regmap تلقائياً.

#### التحليل

```bash
# فحص الـ compatible strings للـ CTRL_MMR node
cat /proc/device-tree/bus@f0000/system-controller@100000/compatible | tr '\0' '\n'
# المتوقع: ti,am62-ctrl-mmr syscon
# لو ناقص "syscon" → المشكلة هنا
```

الـ flow الخاطئ:

```
USB PHY probe()
  └─> syscon_regmap_lookup_by_phandle(np, "ti,ctrl_mmr0")
      └─> syscon_node_to_regmap(ctrl_mmr_np)
          └─> of_device_is_compatible(np, "syscon") → false (ناقص!)
          └─> device_node_get_regmap(np, false, true)
              └─> !syscon && !create_regmap
              └─> return ERR_PTR(-EPROBE_DEFER) ← يرجع للـ USB دايماً!
```

#### الحل

**الحل 1: أضف `"syscon"` للـ compatible list في الـ DT:**

```dts
/* قبل — ناقص "syscon" */
ctrl_mmr0: system-controller@100000 {
    compatible = "ti,am62-ctrl-mmr";
    reg = <0x00 0x00100000 0x00 0x20000>;
};

/* بعد — مع "syscon" */
ctrl_mmr0: system-controller@100000 {
    compatible = "ti,am62-ctrl-mmr", "syscon"; /* أضف syscon */
    reg = <0x00 0x00100000 0x00 0x20000>;
};
```

**الحل 2: استخدام `device_node_to_regmap()` من الـ driver مباشرة:**

```c
/* في driver الـ USB PHY */
struct device_node *ctrl_np = of_parse_phandle(dev->of_node, "ti,ctrl_mmr0", 0);
/* device_node_to_regmap يستخدم create_regmap=true دايماً */
regmap = device_node_to_regmap(ctrl_np);
of_node_put(ctrl_np);
```

**الحل 3: تسجيل الـ regmap مسبقاً:**

```c
/* في driver الـ CTRL_MMR */
static int ti_ctrl_mmr_probe(struct platform_device *pdev)
{
    /* ... إنشاء regmap ... */
    of_syscon_register_regmap(pdev->dev.of_node, regmap);
    return 0;
}
```

#### الدرس المستفاد
`syscon_node_to_regmap()` تنشئ الـ regmap تلقائياً فقط لو الـ node compatible مع `"syscon"` الـ generic string. لو الـ vendor compatible بس موجود، يرجع `-EPROBE_DEFER`. الحل الأضمن هو إضافة `"syscon"` للـ compatible list، أو استخدام `device_node_to_regmap()` اللي تنشئ regmap دايماً بدون شرط.

---

### السيناريو 5: Allwinner H616 — SPI لا يعمل بسبب reg-io-width خاطئ

#### العنوان
**الـ SPI على Allwinner H616 يكتب بيانات خاطئة بسبب `reg-io-width` مش صح في الـ syscon**

#### السياق
Android TV box رخيص يعمل على **Allwinner H616**. الفريق يحاول يضيف دعم لـ SPI NOR flash. الـ SPI controller يحتاج يوصل لـ **CCU** (Clock Control Unit) و**PRCM** (Power, Reset, Clock Management) عبر syscon لضبط الـ SPI clocks. الـ SPI يشتغل، بس البيانات المقروءة من الـ flash خاطئة — كل 4 bytes فيها خلط.

#### المشكلة

الـ PRCM registers على H616 بعضها 8-bit access وبعضها 32-bit. المهندس نسخ الـ DT node من SoC ثاني كان فيه `reg-io-width = <1>` (8-bit)، بس الـ PRCM على H616 يحتاج 32-bit access.

#### التحليل

داخل `of_syscon_register()`:

```c
ret = of_property_read_u32(np, "reg-io-width", &reg_io_width);
if (ret)
    reg_io_width = 4; /* الافتراضي 4 bytes */

/* ...بعدين... */
syscon_config.reg_stride = reg_io_width;      /* خطوة الـ register */
syscon_config.val_bits = reg_io_width * 8;    /* عرض القيمة بالبت */
syscon_config.max_register = res_size - reg_io_width;
```

لما `reg_io_width = 1`:
- `val_bits = 8` → الـ regmap يكتب/يقرأ byte واحد فقط
- `reg_stride = 1` → كل register يبعد byte واحد

بس PRCM على H616 يحتاج:
- `val_bits = 32`
- `reg_stride = 4`

لما الـ SPI driver يطلب يكتب قيمة 32-bit:

```c
regmap_write(prcm_regmap, SPI_CLK_REG, 0x00000003);
```

الـ regmap بيكتب بالـ 8-bit mode:
- يكتب `0x03` في offset 0
- يتجاهل بقية الـ 3 bytes
- الـ register الحقيقي ما اتضبطش كامل

النتيجة: الـ SPI clock ما اتضبطش صح، فبيانات الـ flash تجي خاطئة أو corrupt.

**كيف تكتشف المشكلة:**

```bash
# افحص الـ regmap config للـ PRCM
cat /sys/kernel/debug/regmap/prcm@*/config
# لازم تشوف: val_bits: 32, reg_stride: 4

# لو شايف val_bits: 8 → المشكلة هنا

# قارن الـ register قبل وبعد الكتابة
cat /sys/kernel/debug/regmap/prcm@*/registers | grep "^0cc:"
devmem2 0x07010000+0xcc  # قراءة مباشرة
```

**فحص الـ DT:**

```bash
# افحص الـ reg-io-width في الـ PRCM node
cat /proc/device-tree/soc/prcm@7010000/reg-io-width | xxd
# لو القيمة 01 → 1 byte → خاطئ
# لو القيمة 04 → 4 bytes → صح
# لو مش موجود → الافتراضي 4 → صح
```

#### الحل

**تصحيح الـ DT:**

```dts
/* قبل — خاطئ، نسخ من SoC ثاني */
prcm: prcm@7010000 {
    compatible = "allwinner,sun50i-h616-prcm", "syscon";
    reg = <0x07010000 0x400>;
    reg-io-width = <1>; /* خاطئ! */
};

/* بعد — صح */
prcm: prcm@7010000 {
    compatible = "allwinner,sun50i-h616-prcm", "syscon";
    reg = <0x07010000 0x400>;
    /* reg-io-width مش موجود = الافتراضي 4 bytes = صح */
};
```

**أو صراحة:**

```dts
prcm: prcm@7010000 {
    compatible = "allwinner,sun50i-h616-prcm", "syscon";
    reg = <0x07010000 0x400>;
    reg-io-width = <4>; /* واضح وصريح */
};
```

**التحقق بعد التصحيح:**

```bash
# بعد reboot
cat /sys/kernel/debug/regmap/prcm@7010000/config
# المتوقع:
# name: prcm@7010000
# val_bits: 32
# reg_stride: 4

# اختبر الـ SPI
spidev_test -D /dev/spidev0.0 -v -l 16
```

**فحص إضافي عبر الـ regmap validation:**

لاحظ إن `of_syscon_register()` تتحقق من صحة الـ size:

```c
res_size = resource_size(&res);
if (res_size < reg_io_width) {
    ret = -EFAULT; /* الـ register block أصغر من reg-io-width */
    goto err_regmap;
}
```

لو `reg-io-width = 4` بس الـ `reg` في الـ DT يعطي size = 2، الـ kernel يرجع `-EFAULT` ويرفض الـ node — ده بيساعد يكتشف أخطاء الـ DT مبكراً.

#### الدرس المستفاد
الـ `reg-io-width` في الـ syscon DT node تتحكم مباشرة في `val_bits` و`reg_stride` للـ regmap اللي ينشئها `of_syscon_register()`. خطأ واحد هنا يسبب كل الـ register reads/writes تكون بعرض غلط بدون أي error واضح. دايماً راجع الـ datasheet للـ SoC وحدد الـ access width المطلوب لكل register block قبل ما تكتب الـ DT، وتحقق من الـ regmap config عبر debugfs بعد الـ boot.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ** [LWN.net](https://lwn.net) هو المرجع الأول لأخبار وتحليلات kernel Linux، وفيه مقالات تقنية عميقة عن كل subsystem تقريبًا.

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| **add syscon driver based on regmap for general registers access** — أول patch series أضاف الـ syscon driver عام 2012 | [lwn.net/Articles/514764](https://lwn.net/Articles/514764/) | ⭐⭐⭐ الأصل |
| **regmap: Generic I2C and SPI register map library** — مقال شرح دخول الـ regmap للـ kernel، وهو الأساس اللي بُني عليه syscon | [lwn.net/Articles/451789](https://lwn.net/Articles/451789/) | ⭐⭐⭐ أساسي |
| **add syscon driver based on regmap (النقاش الأول)** — نقاش المجتمع حول patch v3 | [lwn.net/Articles/513629](https://lwn.net/Articles/513629/) | ⭐⭐ مهم |
| **regmap: Add asynchronous I/O support** — تطور الـ regmap ليدعم async | [lwn.net/Articles/534893](https://lwn.net/Articles/534893/) | ⭐⭐ مفيد |
| **regmap: introduce fast_io busses** — تحسين أداء الـ regmap | [lwn.net/Articles/490704](https://lwn.net/Articles/490704/) | ⭐ تكميلي |
| **add reboot mode driver (syscon-reboot-mode)** — مثال عملي لاستخدام syscon | [lwn.net/Articles/674428](https://lwn.net/Articles/674428/) | ⭐⭐ مثال |
| **MediaTek MT6735 syscon clock/reset controller** — استخدام syscon في SoC حديثة | [lwn.net/Articles/994948](https://lwn.net/Articles/994948/) | ⭐ مثال |
| **Add PLL clocks and syscon for StarFive JH7110** — syscon في RISC-V SoC | [lwn.net/Articles/931720](https://lwn.net/Articles/931720/) | ⭐ مثال |

---

### التوثيق الرسمي داخل الـ kernel

**الـ** kernel نفسه فيه توثيق للـ syscon و regmap، ابحث عنها في مسارات `Documentation/`:

```
Documentation/devicetree/bindings/mfd/syscon.yaml
Documentation/devicetree/bindings/mfd/syscon.txt     (النسخ القديمة)
Documentation/driver-api/regmap.rst
```

**الملف الرئيسي للـ API:**
```
include/linux/mfd/syscon.h        ← الـ header اللي بنشرحه
drivers/mfd/syscon.c              ← الـ implementation الكاملة
```

**توثيق Device Tree Bindings على kernel.org:**
- [kernel.org — mfd/syscon.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/mfd/mfd.txt)
- [mjmwired.net — syscon.txt mirror](https://mjmwired.net/kernel/Documentation/devicetree/bindings/mfd/syscon.txt)

---

### Commits المهمة في تاريخ syscon

| الـ commit / الحدث | التفصيل |
|-------------------|---------|
| **أول إضافة syscon — 2012** | Dong Aisheng من Freescale/Linaro أضاف الـ `drivers/mfd/syscon.c` لأول مرة كـ MFD driver مبني على regmap |
| **bdb0066df96e** — "mfd: syscon: Decouple syscon interface from platform devices" | فصل الـ syscon عن الـ platform devices، يعني أصبح الـ `dev` field يساوي NULL في بعض الحالات |
| **c89c0114955a** — "mfd: syscon: Set regmap max_register in of_syscon_register" | تحسين في حساب حجم الـ register space |
| **إضافة `device_node_to_regmap()`** | [patchwork — v15,04/13](https://patchwork.kernel.org/project/linux-mips/patch/20190724171615.20774-5-paul@crapouillou.net/) |
| **إضافة `syscon_regmap_lookup_by_phandle_args()`** | لدعم phandle مع arguments متعددة |

**للبحث في تاريخ commits مباشرة:**
- [github.com — torvalds/linux — drivers/mfd/syscon.c](https://github.com/torvalds/linux/blob/master/drivers/mfd/syscon.c)
- [Linux Commits Search (Typesense)](https://linux-commits-search.typesense.org/)

---

### نقاشات المجتمع والـ Mailing Lists

| المصدر | الرابط |
|--------|--------|
| **[PATCH v2 1/7] mfd: add syscon driver based on regmap** — النقاش الأصلي على linux-arm-kernel | [lists.infradead.org](https://lists.infradead.org/pipermail/linux-arm-kernel/2012-August/116032.html) |
| **[PATCH v3 0/7] add syscon driver based on regmap** — المراجعة الثالثة للـ patch | [fa.linux.kernel.narkive.com](https://fa.linux.kernel.narkive.com/L4BKRYaB/patch-v3-0-7-add-syscon-driver-based-on-regmap-for-general-registers-access) |
| **[PATCH 1/7] mfd: add imx syscon driver based on regmap** — النسخة الأولى من Freescale | [lists.archive.carbon60.com](https://lists.archive.carbon60.com/linux/kernel/1585930) |
| **mfd: syscon: Decouple syscon from platform devices — نقاش LKML** | [lkml.iu.edu](https://lkml.iu.edu/hypermail/linux/kernel/1409.2/01638.html) |
| **regmap: introduce regmap_name to fix syscon regmap trace events** | [lkml.iu.edu](https://lkml.iu.edu/hypermail/linux/kernel/1503.0/05360.html) |
| **rtc: at91sam9: use syscon/regmap for GPBR** — مثال استخدام | [lore.kernel.org](https://lore.kernel.org/patchwork/patch/501853/) |
| **mfd: syscon: Consider platform data a regmap config name** | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1392138636-29240-8-git-send-email-pawel.moll@arm.com/) |

---

### محاضرات وعروض تقنية

**الـ** presentation الأفضل عن syscon هو محاضرة Alexandre Belloni من Free Electrons/Bootlin في مؤتمر ELC-E 2015:

> **"Supporting Multi-Function Devices in the Linux Kernel: A Tour of the mfd, regmap and syscon APIs"**

- [Slides (PDF) — events17.linuxfoundation.org](http://events17.linuxfoundation.org/sites/events/files/slides/belloni-mfd-regmap-syscon_0.pdf)
- [Source (LaTeX) — bootlin.com](https://bootlin.com/pub/conferences/2015/elce/belloni-mfd-regmap-syscon/src/belloni-mfd-regmap-syscon.tex)
- [PDF — bootlin.com](https://bootlin.com/pub/conferences/2015/elce/belloni-mfd-regmap-syscon/belloni-mfd-regmap-syscon.pdf)
- [ملخص المحاضرة على mindlinux.wordpress.com](https://mindlinux.wordpress.com/2015/10/07/supporting-multi-function-devices-in-the-linux-kernel-a-tour-of-the-mfd-regmap-and-syscon-apis-alexandre-belloni-free-electrons/)

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)

**الـ** LDD3 هو الكتاب الكلاسيكي لتطوير drivers في Linux. بما إن syscon ظهر بعد نشر LDD3، ما راح تلاقي فصلًا مخصصًا له، لكن الفصول الأساسية اللي تبني فهمك:

| الفصل | الموضوع | الصلة بـ syscon |
|-------|---------|----------------|
| Chapter 1 | An Introduction to Device Drivers | فهم البنية العامة |
| Chapter 9 | Communicating with Hardware | الـ ioread/iowrite اللي يستبدلها regmap |
| Chapter 14 | The Linux Device Model | الـ device_node و platform_device |

- [تحميل LDD3 مجانًا — lwn.net](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)

**الـ** Robert Love يشرح بنية الـ kernel بشكل سلس. الفصول المفيدة:

- **Chapter 17: Devices and Modules** — فهم كيف يُسجّل driver نفسه
- **Chapter 19: Portability** — التجريد عن الـ hardware (مبدأ syscon)

#### Mastering Linux Device Driver Development — John Madieu (Packt)

هذا الكتاب الأحدث فيه فصل مخصص:

> **Chapter 3: Delving into the MFD Subsystem and Syscon API**

وفيه قسم صريح اسمه:

> **"Understanding syscon and simple-mfd"**

- [packtpub.com — Chapter excerpt](https://subscription.packtpub.com/book/iot-and-hardware/9781789342048/4/ch04lvl1sec16/understanding-syscon-and-simple-mfd)

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- **Chapter 15: Kernel Initialization** — فهم ترتيب تهيئة الـ drivers
- **Chapter 16: Kernel Debugging** — أدوات debug مفيدة عند العمل مع syscon

---

### مصادر kernelnewbies.org

**الـ** kernelnewbies لا يوجد فيه صفحة مخصصة لـ syscon، لكن هذه الصفحات مفيدة للسياق:

| الصفحة | الرابط | ما تتعلمه |
|--------|--------|-----------|
| **IoMemoryAccess** — كيف يُقرأ الـ MMIO في الـ kernel | [kernelnewbies.org/IoMemoryAccess](https://kernelnewbies.org/IoMemoryAccess) | الأساس اللي يحل محله regmap |
| **Linux 5.2 changelog** — فيه ذكر "syscon: Add optional clock support" | [kernelnewbies.org/Linux_5.2](https://kernelnewbies.org/Linux_5.2) | تطور syscon |
| **نقاش regmap على mailing list** | [lists.kernelnewbies.org](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2020-April/020732.html) | سؤال عملي عن regmap |

---

### مصادر elinux.org

**الـ** elinux.org ركّزت أكثر على الجانب Embedded، وهذه الصفحات مفيدة:

| الصفحة | الرابط |
|--------|--------|
| **Linux Kernel Resources** — قائمة شاملة بمصادر kernel | [elinux.org/Linux_Kernel_Resources](https://elinux.org/Linux_Kernel_Resources) |
| **Kernel areas of focus for mainlining** — كيف تدخل driver للـ mainline | [elinux.org/Kernel_areas_of_focus_for_mainlining](https://elinux.org/Kernel_areas_of_focus_for_mainlining) |
| **CE Workgroup Device Mainlining Project** — مشاريع SoC mainlining (syscon شائع فيها) | [elinux.org/CE_Workgroup_Device_Mainlining_Project](https://elinux.org/CE_Workgroup_Device_Mainlining_Project) |

---

### مدونات تقنية مفيدة

| المدونة | الرابط | المحتوى |
|---------|--------|---------|
| **Linux syscon and regmap — Sherlock's blog** | [wangzhou.github.io](https://wangzhou.github.io/Linux-syscon-and-regmap/) | شرح عملي ممتاز بأمثلة كود |
| **regmap — pandysong.github.io** | [pandysong.github.io/blog/post/regmap](https://pandysong.github.io/blog/post/regmap/) | فهم الـ regmap من الأساس |
| **regmap: Reducing Redundancy in Linux Code** | [opensourceforu.com](https://www.opensourceforu.com/2017/01/regmap-reducing-redundancy-linux-code/) | مقال بسيط عن فائدة regmap |

---

### مصطلحات البحث

لو أردت البحث عن معلومات أكثر، استخدم هذه الـ search terms:

```
linux kernel syscon regmap mfd
syscon_regmap_lookup_by_phandle example
linux mfd syscon device tree binding
of_syscon_register kernel
syscon_node_to_regmap driver example
linux simple-mfd syscon
CONFIG_MFD_SYSCON kernel config
regmap mmio linux driver example
linux kernel mfd multi function device driver
device_node_to_regmap linux kernel
```

---

### ملخص سريع للمصادر حسب الأولوية

```
للمبتدئين:
  1. LDD3 الفصول 9 و 14        ← أساس lأساس
  2. wangzhou.github.io         ← شرح عملي بالعربي تقريبًا
  3. bootlin.com presentation   ← أفضل عرض للـ mfd+regmap+syscon

للمتوسطين:
  4. lwn.net/Articles/451789    ← regmap من البداية
  5. lwn.net/Articles/514764    ← patch syscon الأصلي
  6. Mastering LDD chapter 3    ← syscon بالتفصيل

للمتقدمين:
  7. drivers/mfd/syscon.c       ← الكود نفسه
  8. lore.kernel.org patches    ← تاريخ التطور
  9. linux-arm-kernel ML        ← النقاشات الأصلية
```
## Phase 8: Writing simple module

### الفكرة — ماذا سنراقب؟

سنضع **kprobe** على الدالة `syscon_regmap_lookup_by_compatible`، وهي واحدة من أكثر الدوال استخداماً في الـ syscon subsystem. كل ما يحدث هو: أي driver يريد يوصل لـ syscon معين بيبعت اسم الـ compatible string زي `"syscon"` أو `"ti,sysc"` — والـ kernel يرجعله الـ regmap المناسب.

بالـ kprobe، كل ما حدا بيستدعي هالدالة، موديولنا يصحى ويطبع: مين استدعاها + أيش الـ compatible string اللي طلبها.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * syscon_lookup_probe.c
 *
 * kprobe على syscon_regmap_lookup_by_compatible
 * يطبع كل مرة driver يبحث عن syscon بـ compatible string معينة
 */

/* ── الـ includes الأساسية ── */
#include <linux/kernel.h>      /* pr_info, pr_err */
#include <linux/module.h>      /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe, ... */
#include <linux/ptrace.h>      /* struct pt_regs — سجلات المعالج وقت الـ probe */

/* ── اسم الدالة اللي رح نراقبها ── */
#define TARGET_FUNC "syscon_regmap_lookup_by_compatible"

/*
 * pre_handler — بيتنفذ قبل ما الدالة الأصلية تشتغل
 *
 * @p   : pointer للـ kprobe نفسه (مفيد لو عندك أكثر من probe)
 * @regs: سجلات المعالج — منها نقرأ الـ arguments
 *
 * على معمارية x86_64، الـ argument الأول يجي في سجل rdi.
 * على ARM64 يجي في سجل x0.
 * الـ kernel يوفّر regs_get_kernel_argument(regs, n) كـ abstraction
 * مشترك بين المعماريات.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* نجيب الـ argument الأول (رقم 0) — هو الـ compatible string */
    const char *compat = (const char *)regs_get_kernel_argument(regs, 0);

    /*
     * نطبع اسم العملية الحالية + الـ compatible اللي طُلبت.
     * current->comm هو اسم الـ process اللي استدعى الدالة (مثلاً اسم الـ driver).
     */
    pr_info("[syscon_probe] %s() called by process='%s' (pid=%d), compat='%s'\n",
            TARGET_FUNC,
            current->comm,
            current->pid,
            compat ? compat : "<NULL>");

    /* إرجاع 0 = لا توقف التنفيذ، كمّل للدالة الأصلية */
    return 0;
}

/*
 * post_handler — بيتنفذ بعد ما الدالة الأصلية تنتهي
 *
 * @p    : pointer للـ kprobe
 * @regs : سجلات المعالج بعد تنفيذ الدالة
 * @flags: علامات داخلية للـ kprobe framework (عادةً 0)
 *
 * هنا نطبع فقط رسالة بسيطة إن الدالة انتهت — يمكن تستخدمه
 * لقراءة القيمة المُرجَعة من regs->ax (x86) أو regs->regs[0] (ARM64).
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    pr_info("[syscon_probe] %s() returned\n", TARGET_FUNC);
}

/* ── تعريف الـ kprobe struct ── */
static struct kprobe kp = {
    .symbol_name    = TARGET_FUNC,   /* الكيرنل يحل الـ symbol لعنوان الدالة */
    .pre_handler    = handler_pre,   /* callback قبل التنفيذ */
    .post_handler   = handler_post,  /* callback بعد التنفيذ */
};

/* ── module_init ── */
static int __init syscon_probe_init(void)
{
    int ret;

    /*
     * register_kprobe تحجز الـ probe وتضع breakpoint افتراضي
     * على أول تعليمة في الدالة المستهدفة.
     * لو الدالة مش موجودة أو inlined، ترجع error.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[syscon_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[syscon_probe] Planted kprobe at %s (%px)\n",
            TARGET_FUNC, kp.addr);
    return 0;
}

/* ── module_exit ── */
static void __exit syscon_probe_exit(void)
{
    /*
     * unregister_kprobe ضروري جداً:
     * لو ما شيلنا الـ breakpoint قبل ما يُنزَل الموديول،
     * الكيرنل رح يحاول يستدعي handler_pre اللي صارت عنوانها invalid
     * وهذا يسبب kernel panic أو memory corruption.
     */
    unregister_kprobe(&kp);
    pr_info("[syscon_probe] kprobe removed from %s\n", TARGET_FUNC);
}

module_init(syscon_probe_init);
module_exit(syscon_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("kprobe on syscon_regmap_lookup_by_compatible — monitors syscon lookups");
```

---

### شرح كل قسم

#### الـ includes

| Header | سبب وجوده |
|--------|-----------|
| `linux/kernel.h` | عشان `pr_info` و `pr_err` |
| `linux/module.h` | الماكروات الأساسية للموديول زي `module_init/exit` و `MODULE_LICENSE` |
| `linux/kprobes.h` | تعريف `struct kprobe` ودوال `register/unregister_kprobe` |
| `linux/ptrace.h` | تعريف `struct pt_regs` اللي بيحمل قيم سجلات المعالج |

الـ kprobe subsystem موجود في الكيرنل بشكل مستقل — ما نحتاج نضيف أي dependency على `MFD_SYSCON` لأننا بنراقب الدالة من بره.

---

#### لماذا `syscon_regmap_lookup_by_compatible` تحديداً؟

لأنها **exported بـ `EXPORT_SYMBOL_GPL`** وبالتالي رمزها موجود في الـ symbol table — الـ kprobe يقدر يلقيها بسهولة عن طريق الاسم. كمان هي نقطة دخول مركزية: كل driver يريد يوصل لـ syscon بالاسم يمر عليها، فهي نقطة مراقبة ثرية بالمعلومات.

---

#### الـ `pre_handler` والـ arguments

الفكرة زي التنصت على مكالمة تلفون قبل ما الطرف الثاني يرد — نشوف مين اتصل وبماذا طلب. الـ `regs_get_kernel_argument(regs, 0)` هي **abstraction** محمولة بين المعماريات (x86_64 / ARM64 / RISC-V) تعيد قيمة الـ argument رقم 0 من سجلات المعالج. الـ `current->comm` يعطينا اسم الـ process الحالي — مفيد جداً لمعرفة أي driver هو اللي يبحث.

---

#### الـ `post_handler`

نفّذ بعد انتهاء الدالة الأصلية، مفيد لو أردنا نقرأ القيمة المُرجَعة (الـ regmap pointer أو error code). حالياً بيطبع فقط رسالة إتمام — يمكن توسيعه لاحقاً لفحص `IS_ERR(retval)` وطباعة الـ error إن وُجد.

---

#### الـ `module_exit` وأهمية `unregister_kprobe`

تخيل إنك لصقت ملصقاً على باب — لو خلعت الباب بدون ما تشيل الملصق، الملصق رح يطير لمكان مجهول. نفس الفكرة: لو أُنزل الموديول دون إزالة الـ kprobe، الـ breakpoint رح يبقى في الكود وأي استدعاء لاحق للدالة رح يقفز لعنوان handler صار invalid، مسبباً **kernel panic**. لذا `unregister_kprobe` ليست اختيارية.

---

### كيف تبني وتشغّل الموديول

```bash
# Makefile بسيط
# obj-m += syscon_lookup_probe.o

make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الموديول
sudo insmod syscon_lookup_probe.ko

# مراقبة المخرجات
sudo dmesg -w | grep syscon_probe

# إزالة الموديول
sudo rmmod syscon_lookup_probe
```

**مثال على مخرج متوقع في `dmesg`:**

```
[syscon_probe] Planted kprobe at syscon_regmap_lookup_by_compatible (ffffffffc0ab1234)
[syscon_probe] syscon_regmap_lookup_by_compatible() called by process='kworker/0:1' (pid=45), compat='simple-mfd'
[syscon_probe] syscon_regmap_lookup_by_compatible() returned
[syscon_probe] syscon_regmap_lookup_by_compatible() called by process='modprobe' (pid=1234), compat='syscon'
[syscon_probe] syscon_regmap_lookup_by_compatible() returned
```
