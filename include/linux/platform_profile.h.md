## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

**الـ** `include/linux/platform_profile.h` هو الـ header الرئيسي لـ subsystem اسمه **Platform Profile**، وهو جزء من نظام **ACPI** في الـ Linux kernel. مهمته: توحيد الطريقة التي يتحكم بها المستخدم (أو التطبيقات) في **وضع أداء الجهاز** — من توفير الطاقة حتى أقصى أداء — بغض النظر عن العلامة التجارية للجهاز (Lenovo، Dell، ASUS، Samsung، إلخ).

---

### القصة: لماذا وُجد هذا الـ subsystem؟

تخيّل أنك تملك لابتوب. عندما تفتح فيلماً تريد الهدوء والبطارية تدوم طويلاً، وعندما تلعب لعبة تريد أقصى أداء. الـ hardware نفسه (المعالج، المروحة، شريحة التحكم) يستطيع أن يعمل بأوضاع مختلفة.

**المشكلة القديمة:** كل شركة كانت تعمل بطريقتها الخاصة:
- Lenovo لديها `thinkpad_acpi` driver بـ API خاص بها.
- Dell لديها `dell-pc` driver بـ API مختلف تماماً.
- ASUS لديها `asus-wmi` driver بـ API ثالث.

فتطبيقات مثل **GNOME Power Profiles** أو **TLP** كانت مضطرة أن تكتب كود مختلف لكل جهاز — فوضى كاملة.

**الحل:** في عام 2020، جاء **Hans de Goede** من Red Hat وأنشأ الـ `platform_profile` subsystem كـ abstraction layer موحّد:

```
[ تطبيق المستخدم ]
       |
       | يكتب "performance" إلى sysfs
       v
/sys/firmware/acpi/platform_profile   ← واجهة موحّدة واحدة
       |
       v
[ platform_profile core - drivers/acpi/platform_profile.c ]
       |
       +---> thinkpad_acpi (Lenovo)
       +---> dell-pc (Dell)
       +---> asus-wmi (ASUS)
       +---> amd-pmf (AMD)
       +---> ...
```

الآن أي تطبيق يكتب كلمة واحدة في مكان واحد، والـ kernel يترجمها للـ driver المناسب.

---

### ماذا يعرّف هذا الـ Header تحديداً؟

#### 1. الـ `enum platform_profile_option` — قائمة الأوضاع المتاحة

```c
enum platform_profile_option {
    PLATFORM_PROFILE_LOW_POWER,          /* توفير الطاقة القصوى */
    PLATFORM_PROFILE_COOL,               /* تبريد الجهاز */
    PLATFORM_PROFILE_QUIET,              /* تقليل ضوضاء المروحة */
    PLATFORM_PROFILE_BALANCED,           /* توازن بين الأداء والطاقة */
    PLATFORM_PROFILE_BALANCED_PERFORMANCE, /* ميل طفيف نحو الأداء */
    PLATFORM_PROFILE_PERFORMANCE,        /* أداء عالٍ */
    PLATFORM_PROFILE_MAX_POWER,          /* أقصى طاقة ممكنة */
    PLATFORM_PROFILE_CUSTOM,             /* وضع مخصص (قرّاءة فقط) */
    PLATFORM_PROFILE_LAST,               /* marker داخلي */
};
```

هذه ليست أرقاماً تُرسَل للـ hardware مباشرة — إنها أسماء رمزية موحّدة يترجمها كل driver لأوامره الخاصة.

#### 2. الـ `struct platform_profile_ops` — عقد الـ driver

```c
struct platform_profile_ops {
    /* ما الأوضاع التي يدعمها هذا الجهاز؟ */
    int (*probe)(void *drvdata, unsigned long *choices);

    /* أوضاع مخفية عن المستخدم لكن يمكن للـ driver ضبطها */
    int (*hidden_choices)(void *drvdata, unsigned long *choices);

    /* اقرأ الوضع الحالي من الـ hardware */
    int (*profile_get)(struct device *dev, enum platform_profile_option *profile);

    /* اضبط وضعاً جديداً على الـ hardware */
    int (*profile_set)(struct device *dev, enum platform_profile_option profile);
};
```

كل driver يملأ هذا الـ struct ويسجّل نفسه في الـ core — وهذا كل ما يحتاجه.

#### 3. الـ API العام للـ drivers

```c
/* تسجيل driver جديد */
struct device *platform_profile_register(struct device *dev, const char *name,
                                         void *drvdata,
                                         const struct platform_profile_ops *ops);

/* نسخة مُدارة تلقائياً بـ devres */
struct device *devm_platform_profile_register(...);

/* إلغاء التسجيل */
void platform_profile_remove(struct device *dev);

/* إشعار الـ sysfs بتغيّر الوضع (من hotkey مثلاً) */
void platform_profile_notify(struct device *dev);

/* التنقل بين الأوضاع المتاحة دورياً */
int platform_profile_cycle(void);
```

---

### سيناريو حقيقي — كيف يعمل كل شيء معاً؟

**المستخدم يضغط Fn+Q على لابتوب Lenovo:**

```
1. زر Fn+Q يُطلق حدث ACPI في الـ firmware
2. thinkpad_acpi driver يستقبل الحدث
3. يستدعي platform_profile_cycle() للانتقال للوضع التالي
4. platform_profile core يجد الوضع التالي من قائمة الأوضاع المتاحة
5. يستدعي ops->profile_set() في thinkpad_acpi
6. thinkpad_acpi يرسل الأمر للـ embedded controller
7. يستدعي platform_profile_notify() لإبلاغ sysfs
8. GNOME يقرأ التغيير من /sys/firmware/acpi/platform_profile
9. يعرض إشعاراً للمستخدم: "تم التبديل إلى وضع Performance"
```

---

### دعم متعدد الـ Drivers على نفس الجهاز

إذا كان الجهاز يحتوي على أكثر من driver (مثلاً: AMD PMF وchip تحكم مستقل)، يعرض الـ core **تقاطع** الأوضاع المشتركة فقط في الواجهة الموحّدة. إذا اختلفت الأوضاع بين الـ drivers، يظهر الوضع `custom`.

```
Driver A يدعم: [low-power, balanced, performance]
Driver B يدعم: [balanced, performance, max-power]
                         ↓
الواجهة الموحّدة تعرض: [balanced, performance]  ← التقاطع فقط
```

---

### الـ Subsystem الذي ينتمي إليه

**الـ** `platform_profile` جزء من **ACPI subsystem** في الـ kernel. يتكامل مع:
- الـ **sysfs** لعرض الأوضاع وقبول الكتابة من userspace.
- الـ **ACPI kobject** (`/sys/firmware/acpi/`) كواجهة legacy.
- الـ **device class** `platform-profile` كواجهة جديدة لكل driver.

---

### الملفات التي يجب معرفتها

| الملف | الدور |
|---|---|
| `include/linux/platform_profile.h` | الـ header الرئيسي — enum + ops + API |
| `drivers/acpi/platform_profile.c` | الـ core implementation — sysfs + class management |
| `Documentation/userspace-api/sysfs-platform_profile.rst` | توثيق الـ userspace API |
| `Documentation/ABI/testing/sysfs-platform_profile` | توثيق الـ ABI الرسمي |

**Drivers التي تستخدم هذا الـ API (hardware drivers):**

| الملف | الجهاز |
|---|---|
| `drivers/platform/x86/lenovo/thinkpad_acpi.c` | Lenovo ThinkPad |
| `drivers/platform/x86/lenovo/ideapad-laptop.c` | Lenovo IdeaPad |
| `drivers/platform/x86/lenovo/wmi-gamezone.c` | Lenovo Gaming |
| `drivers/platform/x86/dell/dell-pc.c` | Dell |
| `drivers/platform/x86/dell/alienware-wmi-wmax.c` | Alienware |
| `drivers/platform/x86/asus-wmi.c` | ASUS |
| `drivers/platform/x86/acer-wmi.c` | Acer |
| `drivers/platform/x86/hp/hp-wmi.c` | HP |
| `drivers/platform/x86/samsung-galaxybook.c` | Samsung |
| `drivers/platform/x86/amd/pmf/sps.c` | AMD PMF |
| `drivers/platform/x86/inspur_platform_profile.c` | Inspur |
| `drivers/platform/surface/surface_platform_profile.c` | Microsoft Surface |
| `drivers/thermal/intel/int340x_thermal/processor_thermal_soc_slider.c` | Intel SoC |
## Phase 2: شرح الـ Platform Profile Framework

### المشكلة — لماذا يوجد هذا الـ Subsystem؟

الأجهزة الحديثة — لاب توبات، تابلت بـ ARM، أجهزة embedded تعمل بـ Linux — تملك **firmware أو embedded controller (EC)** يتحكم في السلوك الحراري والطاقة للجهاز بالكامل. هذا الـ EC يفهم مفاهيم مثل:
- "وضع الأداء العالي" → رفع تردد CPU، رفع سرعة المراوح
- "وضع الهدوء" → تقليل الضجيج على حساب الأداء
- "وضع توفير الطاقة" → تمديد عمر البطارية

**المشكلة الجوهرية**: كل صانع hardware (Lenovo ThinkPad، Dell، ASUS، HP) يُعبّر عن هذه المفاهيم بطريقة مختلفة تماماً في driver الخاص به. النتيجة:

- كل driver يُصدّر `/sys/devices/platform/...` بمسار وصيغة مختلفة
- لا يوجد واجهة موحدة لـ userspace tools مثل `power-profiles-daemon`
- التطبيقات مثل GNOME Power Profiles لا تستطيع التعامل مع كل جهاز باستقلالية

### الحل — نهج الـ Kernel

الـ **Platform Profile Framework** يحل هذه المشكلة بفرض **abstraction layer موحّدة**:

1. يُعرّف مجموعة ثابتة من الـ profiles العالمية (`BALANCED`, `PERFORMANCE`, `QUIET`...)
2. كل driver يُسجّل نفسه عبر `platform_profile_register()` ويُعطي الـ framework مجموعة callbacks
3. الـ framework يُصدّر واجهة sysfs موحّدة تحت:
   - `/sys/class/platform-profile/<name>/profile` — القراءة/الكتابة
   - `/sys/class/platform-profile/<name>/choices` — الـ profiles المدعومة

بهذا يُصبح `power-profiles-daemon` قادراً على التحكم في أي جهاز بنفس الكود.

---

### التشبيه الواقعي — الترجمة الفورية في الأمم المتحدة

تخيّل قاعة اجتماعات الأمم المتحدة:

| عنصر في الأمم المتحدة | المقابل في الـ Kernel |
|---|---|
| كل دولة تتكلم لغتها الأصلية | كل driver يفهم لغة الـ EC الخاص به |
| المترجم الفوري في الكابينة | الـ Platform Profile Framework |
| اللغة الرسمية (إنجليزي/فرنسي) | `enum platform_profile_option` |
| المندوب الأمريكي يطلب "رفع الصوت" | `power-profiles-daemon` يكتب `performance` |
| المترجم يُحوّل الطلب للدولة المعنية | الـ framework يستدعي `profile_set()` callback |
| الدولة تُنفّذ بطريقتها الخاصة | الـ EC driver يُرسل الأمر لـ hardware بـ ACPI/I2C/... |
| المترجم لا يعرف تفاصيل السياسة الداخلية | الـ framework لا يعرف كيف يُغيّر الـ EC الـ clock |

النقطة المحورية: المترجم (الـ framework) لا يتدخل في **كيفية** تنفيذ القرار — فقط يضمن **وحدة اللغة**.

---

### البنية العامة — ASCII Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Userspace                                │
│                                                                 │
│   power-profiles-daemon     GNOME Shell     custom scripts      │
│         │                      │                │              │
│         └──────────────────────┴────────────────┘              │
│                               │                                 │
│              write("performance") to sysfs                      │
└───────────────────────────────┼─────────────────────────────────┘
                                │
                    ╔═══════════▼═══════════════╗
                    ║     sysfs interface        ║
                    ║  /sys/class/              ║
                    ║  platform-profile/        ║
                    ║    <name>/profile         ║
                    ║    <name>/choices         ║
                    ╚═══════════╤═══════════════╝
                                │
                    ╔═══════════▼═══════════════╗
                    ║  Platform Profile Core     ║
                    ║  (platform_profile.c)      ║
                    ║                            ║
                    ║  - manages class devices   ║
                    ║  - validates profiles      ║
                    ║  - calls driver callbacks  ║
                    ║  - sends uevent/notify     ║
                    ╚═══════════╤═══════════════╝
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
    ┌─────────▼──────┐ ┌────────▼───────┐ ┌──────▼──────────┐
    │  thinkpad_acpi  │ │  asus-wmi      │ │  amd-pmc        │
    │  driver         │ │  driver        │ │  driver         │
    │                 │ │                │ │                 │
    │ profile_set():  │ │ profile_set(): │ │ profile_set():  │
    │  ACPI call to   │ │  WMI method    │ │  SMU mailbox    │
    │  ThinkPad EC    │ │  to ASUS EC    │ │  to AMD PSP     │
    └────────┬────────┘ └───────┬────────┘ └──────┬──────────┘
             │                 │                  │
    ┌────────▼────────┐ ┌───────▼────────┐ ┌──────▼──────────┐
    │  ThinkPad EC    │ │  ASUS WMI EC   │ │  AMD SMU        │
    │  (Embedded Ctrl)│ │                │ │                 │
    └─────────────────┘ └────────────────┘ └─────────────────┘
```

---

### الـ Core Abstraction — التجريد المحوري

الـ **central idea** هو: **فصل "ماذا" عن "كيف"**.

- الـ framework يعرف **ماذا** يريد المستخدم: `PERFORMANCE`, `BALANCED`, `QUIET`...
- الـ driver يعرف **كيف** يُنفذ ذلك على hardware محدد

هذا الفصل يُحقَّق عبر ثلاثة عناصر رئيسية:

#### 1. الـ `enum platform_profile_option` — اللغة المشتركة

```c
enum platform_profile_option {
    PLATFORM_PROFILE_LOW_POWER,          /* توفير أقصى طاقة */
    PLATFORM_PROFILE_COOL,               /* تبريد على حساب أداء */
    PLATFORM_PROFILE_QUIET,              /* هدوء صوتي */
    PLATFORM_PROFILE_BALANCED,           /* توازن (الافتراضي) */
    PLATFORM_PROFILE_BALANCED_PERFORMANCE, /* توازن مع ميل للأداء */
    PLATFORM_PROFILE_PERFORMANCE,        /* أداء عالٍ */
    PLATFORM_PROFILE_MAX_POWER,          /* أقصى طاقة */
    PLATFORM_PROFILE_CUSTOM,             /* مخصص من الـ driver */
    PLATFORM_PROFILE_LAST,               /* sentinel — دائماً في النهاية */
};
```

هذه الـ enum تُمثّل **عقد ثابت** بين userspace والـ drivers. إضافة قيمة جديدة تتطلب تحديث `profile_names[]` في `platform_profile.c` وتوثيق sysfs.

**الـ `PLATFORM_PROFILE_LAST`** هو pattern شائع في الـ kernel لتمكين iteration عبر الـ enum بدون magic numbers:
```c
/* مثال على استخدامه للـ iteration */
for (i = 0; i < PLATFORM_PROFILE_LAST; i++) {
    if (test_bit(i, choices))
        /* هذا الـ profile مدعوم */
}
```

#### 2. الـ `struct platform_profile_ops` — الـ vtable

```c
struct platform_profile_ops {
    /* probe: يُحدد الـ profiles المدعومة عند التسجيل */
    int (*probe)(void *drvdata, unsigned long *choices);

    /* hidden_choices: profiles يستطيع الـ driver ضبطها لكن لا تظهر للمستخدم */
    int (*hidden_choices)(void *drvdata, unsigned long *choices);

    /* profile_get: قراءة الـ profile الحالي من الـ hardware */
    int (*profile_get)(struct device *dev, enum platform_profile_option *profile);

    /* profile_set: كتابة profile جديد إلى الـ hardware */
    int (*profile_set)(struct device *dev, enum platform_profile_option profile);
};
```

هذا هو **vtable** بالمعنى الكلاسيكي — مثل C++ virtual functions لكن بـ C خالص. كل driver يُعبئ هذا الـ struct بـ function pointers خاصة بـ hardware الخاص به.

#### 3. الـ `unsigned long *choices` — الـ Bitmap للـ Capabilities

الـ `choices` parameter في `probe()` و`hidden_choices()` هو **bitmap** حيث كل bit يُمثل `platform_profile_option` واحد.

```
bit 0 → PLATFORM_PROFILE_LOW_POWER
bit 1 → PLATFORM_PROFILE_COOL
bit 2 → PLATFORM_PROFILE_QUIET
bit 3 → PLATFORM_PROFILE_BALANCED       ← مضبوط = مدعوم
bit 4 → PLATFORM_PROFILE_BALANCED_PERFORMANCE
bit 5 → PLATFORM_PROFILE_PERFORMANCE    ← مضبوط = مدعوم
...
```

هنا يأتي دور `linux/bitops.h` — الـ macros مثل `__set_bit()` و`test_bit()` تُستخدم للتعامل مع هذا الـ bitmap:

```c
/* مثال داخل probe() callback في driver */
int my_driver_probe(void *drvdata, unsigned long *choices)
{
    /* إعلان دعم BALANCED و PERFORMANCE فقط */
    __set_bit(PLATFORM_PROFILE_BALANCED, choices);
    __set_bit(PLATFORM_PROFILE_PERFORMANCE, choices);
    return 0;
}
```

---

### العلاقة بين الـ Structs — Data Flow Diagram

```
platform_profile_register() تستدعيها
         │
         ▼
┌────────────────────────────────────────────────────┐
│  platform_profile_register(dev, name, drvdata, ops)│
│                                                    │
│  المُدخلات:                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │  dev     │  │  name    │  │  drvdata         │ │
│  │(parent   │  │("asus-  │  │(driver's private │ │
│  │ device)  │  │ wmi")   │  │ data pointer)    │ │
│  └──────────┘  └──────────┘  └──────────────────┘ │
│                                                    │
│  ┌─────────────────────────────────────────────┐   │
│  │ ops (struct platform_profile_ops *)         │   │
│  │  ├─ probe         → يُستدعى فوراً           │   │
│  │  ├─ hidden_choices → يُستدعى فوراً          │   │
│  │  ├─ profile_get   → عند قراءة sysfs        │   │
│  │  └─ profile_set   → عند كتابة sysfs        │   │
│  └─────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────┐
│  الـ Framework يُنشئ:                               │
│                                                    │
│  struct device (class device)                      │
│  ├─ parent: dev (الـ platform device الأصلي)       │
│  ├─ class: &platform_profile_class                 │
│  ├─ groups: [profile, choices sysfs attrs]        │
│  └─ drvdata: يحتوي ops + drvdata + choices bitmap  │
└────────────────────────────────────────────────────┘
         │
         ▼
  /sys/class/platform-profile/<name>/
  ├── profile    (rw) ← يستدعي profile_get/set
  └── choices    (ro) ← مبني من bitmap
```

---

### ما يمتلكه الـ Framework وما يُفوّضه للـ Driver

| المسؤولية | الـ Framework (core) | الـ Driver |
|---|---|---|
| تعريف الـ profiles المتاحة | يوفر الـ `enum` الثابت | يختار منها ما يدعمه عبر `probe()` |
| إنشاء sysfs entries | يُنشئها تلقائياً | لا يتدخل |
| التحقق من صحة الـ profile | يرفض ما ليس في `choices` | لا يحتاج validation |
| استدعاء الـ callbacks | يستدعيها عند قراءة/كتابة | يُنفّذ الـ callback |
| إرسال uevent عند التغيير | عبر `platform_profile_notify()` | يستدعي notify يدوياً إذا تغيّر خارجياً |
| إدارة دورة حياة الـ device | عبر `devm_` variant | يستدعي register/remove |
| التواصل مع الـ hardware | لا يعلم شيئاً | ACPI/WMI/I2C/SMU... |
| التعامل مع تعدد الـ profiles | يدعم تعدد devices | كل driver يُسجّل device واحد |

---

### الـ API بالتفصيل

#### التسجيل اليدوي

```c
struct device *platform_profile_register(
    struct device *dev,           /* الـ parent device */
    const char *name,             /* اسم الـ profile device */
    void *drvdata,                /* بيانات Driver الخاصة */
    const struct platform_profile_ops *ops  /* الـ callbacks */
);
```

تُعيد `struct device *` جديداً — هذا هو الـ **class device** المُنشأ. الـ driver مسؤول عن استدعاء `platform_profile_remove()` عند unload.

#### التسجيل المُدار بـ devm

```c
struct device *devm_platform_profile_register(
    struct device *dev,
    const char *name,
    void *drvdata,
    const struct platform_profile_ops *ops
);
```

الفرق: الـ `devm_` variant يستخدم **Device Resource Management** (subsystem مستقل يجب معرفته: يربط عمر المورد بعمر الـ device، فعند `device_unregister` يُحرر تلقائياً). هذا يُلغي الحاجة لاستدعاء `platform_profile_remove()` يدوياً.

#### الـ Cycling بين الـ Profiles

```c
int platform_profile_cycle(void);
```

تنتقل إلى الـ profile التالي في قائمة الـ choices المدعومة — مفيد لـ hardware buttons (Fn+P في بعض الـ laptops).

#### الإشعار عند تغيير خارجي

```c
void platform_profile_notify(struct device *dev);
```

عندما يُغيّر الـ firmware الـ profile بنفسه (مثل الضغط على زر مادي في الجهاز)، يُستدعى هذا لإرسال `uevent` لـ userspace حتى يُحدّث واجهته.

---

### مثال كامل — Driver مبسّط

```c
#include <linux/platform_profile.h>
#include <linux/platform_device.h>

struct my_ec_data {
    void __iomem *ec_base;   /* base address of EC registers */
};

/* callback: إعلان الـ profiles المدعومة */
static int my_probe(void *drvdata, unsigned long *choices)
{
    __set_bit(PLATFORM_PROFILE_BALANCED, choices);
    __set_bit(PLATFORM_PROFILE_PERFORMANCE, choices);
    __set_bit(PLATFORM_PROFILE_LOW_POWER, choices);
    return 0;
}

/* callback: قراءة الـ profile الحالي من الـ EC */
static int my_get(struct device *dev, enum platform_profile_option *profile)
{
    struct my_ec_data *ec = dev_get_drvdata(dev);
    u8 val = readb(ec->ec_base + EC_PROFILE_REG);

    switch (val) {
    case 0x01: *profile = PLATFORM_PROFILE_LOW_POWER;  break;
    case 0x02: *profile = PLATFORM_PROFILE_BALANCED;   break;
    case 0x03: *profile = PLATFORM_PROFILE_PERFORMANCE; break;
    default:   return -EIO;
    }
    return 0;
}

/* callback: ضبط profile جديد في الـ EC */
static int my_set(struct device *dev, enum platform_profile_option profile)
{
    struct my_ec_data *ec = dev_get_drvdata(dev);
    u8 val;

    switch (profile) {
    case PLATFORM_PROFILE_LOW_POWER:   val = 0x01; break;
    case PLATFORM_PROFILE_BALANCED:    val = 0x02; break;
    case PLATFORM_PROFILE_PERFORMANCE: val = 0x03; break;
    default: return -EOPNOTSUPP;
    }

    writeb(val, ec->ec_base + EC_PROFILE_REG);
    return 0;
}

static const struct platform_profile_ops my_ops = {
    .probe       = my_probe,
    .profile_get = my_get,
    .profile_set = my_set,
};

static int my_driver_probe(struct platform_device *pdev)
{
    struct my_ec_data *ec;
    struct device *profile_dev;

    ec = devm_kzalloc(&pdev->dev, sizeof(*ec), GFP_KERNEL);
    /* ... map EC registers ... */

    /* devm: سيُحرّر تلقائياً عند remove */
    profile_dev = devm_platform_profile_register(
        &pdev->dev, "my-ec", ec, &my_ops);

    return PTR_ERR_OR_ZERO(profile_dev);
}
```

---

### صلة هذا الـ Subsystem بـ Subsystems أخرى

- **Driver Model / `struct device`**: (يجب فهمه أولاً — البنية الأساسية التي تُمثّل كل جهاز في الـ kernel، تحمل parent/child relationships، pm_ops، وغيرها) — الـ Platform Profile يُنشئ `struct device` جديداً كـ child لـ device الـ driver.

- **sysfs**: (نظام ملفات افتراضي يُصدّر kernel objects للـ userspace) — الـ framework يُسجّل attributes تحت `/sys/class/platform-profile/`.

- **Device Resource Management (devres)**: يُستخدم في `devm_platform_profile_register()` لربط عمر الـ profile device بعمر الـ parent device.

- **uevent / kobject**: الـ `platform_profile_notify()` تُطلق uevent يصل لـ `udev` و`power-profiles-daemon` في userspace.

- **ACPI / WMI / I2C**: ليست جزءاً من الـ framework — لكنها الـ transport layer التي تستخدمها الـ drivers لتنفيذ الـ callbacks.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Config Options — Cheatsheet

#### `enum platform_profile_option`

| القيمة | القيمة العددية | المعنى |
|--------|---------------|--------|
| `PLATFORM_PROFILE_LOW_POWER` | 0 | توفير الطاقة القصوى — أداء منخفض جداً |
| `PLATFORM_PROFILE_COOL` | 1 | تبريد أولوية — تقليل الحرارة على حساب الأداء |
| `PLATFORM_PROFILE_QUIET` | 2 | صمت أولوية — تقليل دوران المراوح |
| `PLATFORM_PROFILE_BALANCED` | 3 | توازن بين الأداء والطاقة — الوضع الافتراضي |
| `PLATFORM_PROFILE_BALANCED_PERFORMANCE` | 4 | متوازن مائل نحو الأداء |
| `PLATFORM_PROFILE_PERFORMANCE` | 5 | أداء أولوية — استهلاك طاقة أعلى |
| `PLATFORM_PROFILE_MAX_POWER` | 6 | أقصى أداء ممكن بلا قيود |
| `PLATFORM_PROFILE_CUSTOM` | 7 | إعدادات مخصصة يحددها الـ driver |
| `PLATFORM_PROFILE_LAST` | 8 | sentinel — دائماً الأخير، يُستخدم كحد أعلى |

> **ملاحظة:** الـ `choices` bitmap يُبنى باستخدام `BIT(option)` من `<linux/bitops.h>`، حيث كل bit تمثّل profile متاح.

#### استخدام الـ Bitmap

```c
/* تحديد الـ profiles المتاحة عند الـ probe */
unsigned long choices = 0;
set_bit(PLATFORM_PROFILE_BALANCED, &choices);
set_bit(PLATFORM_PROFILE_PERFORMANCE, &choices);
set_bit(PLATFORM_PROFILE_LOW_POWER, &choices);
/* choices = 0b00101001 = BIT(0) | BIT(3) | BIT(5) */
```

---

### 1. الـ Structs الأساسية

#### `struct platform_profile_ops`

**الغرض:** جدول العمليات (vtable) الذي يربط الـ subsystem بالـ driver الفعلي. كل driver يُسجّل نسخة منه مع callbacks تُنفّذ عمليات القراءة والكتابة على العتاد.

| الحقل | النوع | الوصف |
|-------|-------|--------|
| `probe` | `int (*)(void *drvdata, unsigned long *choices)` | يُستدعى عند التسجيل — يملأ `choices` بالـ profiles المتاحة للمستخدم |
| `hidden_choices` | `int (*)(void *drvdata, unsigned long *choices)` | مثل `probe` لكن للـ profiles المخفية (قابلة للضبط من الـ driver فقط) |
| `profile_get` | `int (*)(struct device *dev, enum platform_profile_option *profile)` | يقرأ الـ profile الحالي من العتاد |
| `profile_set` | `int (*)(struct device *dev, enum platform_profile_option profile)` | يكتب profile جديد إلى العتاد |

**العلاقات:**
- الـ `drvdata` هو بيانات خاصة بالـ driver تُمرَّر عند التسجيل وتُعاد في كل callback
- الـ `dev` في `profile_get/set` هو الـ class device الذي أنشأه الـ subsystem (ليس device الـ driver الأصلي)
- الـ `choices` هو `unsigned long` يُستخدم كـ bitmap (حد أقصى `PLATFORM_PROFILE_LAST` bits)

#### `struct device` (من `<linux/device.h>`)

الـ subsystem يُنشئ `struct device` داخلياً لكل profile handler ويُعيده لمن يُسجّل. هذا الـ device:
- ينتمي لـ `platform_profile` class
- يظهر في `/sys/class/platform-profile/<name>/`
- يحمل الـ sysfs attributes (`profile`, `choices`)

---

### 2. مخطط علاقات الـ Structs (ASCII)

```
Driver Side                         Subsystem Side
-----------                         --------------

struct my_driver_data               platform_profile subsystem
  ├── hw_regs                              │
  ├── mutex lock                           │
  └── (drvdata ptr) ─────────────►  [internal handler]
                                           │
struct platform_profile_ops                │
  ├── .probe ──────────────────────► probe(drvdata, choices)
  ├── .hidden_choices ─────────────► hidden_choices(drvdata, choices)
  ├── .profile_get ───────────────► profile_get(class_dev, &option)
  └── .profile_set ───────────────► profile_set(class_dev, option)
          │
          │ passed to
          ▼
platform_profile_register(parent_dev, name, drvdata, ops)
          │
          │ returns
          ▼
    struct device  ◄──── class_device (owned by subsystem)
          │
          │ appears as
          ▼
  /sys/class/platform-profile/<name>/
          ├── profile        (rw)
          └── choices        (ro)
```

---

### 3. مخطط دورة الحياة (Lifecycle)

```
┌─────────────────────────────────────────────────────────┐
│                   CREATION / REGISTRATION               │
│                                                         │
│  driver probe()                                         │
│      │                                                  │
│      ├─► allocate drvdata (my_driver_data)              │
│      │                                                  │
│      ├─► define platform_profile_ops {                  │
│      │       .probe = my_probe_cb,                      │
│      │       .profile_get = my_get_cb,                  │
│      │       .profile_set = my_set_cb,                  │
│      │   }                                              │
│      │                                                  │
│      └─► platform_profile_register(dev, name,          │
│              drvdata, &ops)                             │
│               │                                         │
│               ├─► ops->probe(drvdata, &choices)         │
│               │     (fills available profile bitmap)    │
│               │                                         │
│               ├─► ops->hidden_choices(drvdata, &hid)   │
│               │     (optional, fills hidden bitmap)     │
│               │                                         │
│               └─► creates sysfs class device           │
│                   returns struct device *               │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                      USAGE (Runtime)                    │
│                                                         │
│  User reads /sys/.../profile                            │
│      └─► ops->profile_get(class_dev, &option)          │
│              └─► driver reads HW register               │
│              └─► returns enum value                     │
│                                                         │
│  User writes /sys/.../profile                           │
│      └─► subsystem validates option ∈ choices           │
│      └─► ops->profile_set(class_dev, option)           │
│              └─► driver writes HW register              │
│                                                         │
│  platform_profile_cycle()                               │
│      └─► cycles through available profiles              │
│      └─► calls profile_set internally                   │
│                                                         │
│  platform_profile_notify(class_dev)                     │
│      └─► driver calls this after HW-triggered change    │
│      └─► triggers uevent → userspace notified           │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                      TEARDOWN                           │
│                                                         │
│  driver remove() / unbind                               │
│      │                                                  │
│      ├─► platform_profile_remove(class_dev)            │
│      │     └─► unregisters sysfs device                 │
│      │     └─► frees internal handler                   │
│      │                                                  │
│      └─► OR (if devm): automatic via devm cleanup      │
│            devm_platform_profile_register()             │
│            └─► cleanup registered with devres           │
│            └─► called automatically at device release   │
└─────────────────────────────────────────────────────────┘
```

---

### 4. مخطط تدفق الاستدعاءات (Call Flow)

#### قراءة الـ Profile (sysfs read)

```
userspace: cat /sys/class/platform-profile/asus-wmi/profile
    │
    ▼
sysfs show() callback (platform_profile.c)
    │
    ├─► handler->ops->profile_get(class_dev, &profile)
    │       │
    │       └─► [ASUS WMI driver]
    │               ├─► acpi_evaluate_integer(WMI_METHOD)
    │               ├─► maps HW value → enum platform_profile_option
    │               └─► returns 0 on success
    │
    ├─► profile_names[profile]  →  "balanced"
    │
    └─► returns string to userspace
```

#### كتابة الـ Profile (sysfs write)

```
userspace: echo performance > /sys/class/platform-profile/.../profile
    │
    ▼
sysfs store() callback
    │
    ├─► match input string → enum value (PLATFORM_PROFILE_PERFORMANCE)
    │
    ├─► test_bit(option, handler->choices)  ← validates option allowed
    │       └─► EINVAL if not in choices bitmap
    │
    ├─► handler->ops->profile_set(class_dev, PLATFORM_PROFILE_PERFORMANCE)
    │       │
    │       └─► [driver]
    │               ├─► acquires driver mutex
    │               ├─► writes to ACPI/EC/WMI/hardware register
    │               └─► releases mutex
    │
    └─► platform_profile_notify(class_dev)
            └─► kobject_uevent → KOBJ_CHANGE
```

#### الـ Cycle (تدوير الـ Profiles)

```
platform_profile_cycle()   ← called from hotkey driver / ACPI event
    │
    ├─► profile_get()  →  current_profile
    │
    ├─► find next set bit in choices bitmap
    │       └─► next_profile = find_next_bit(choices, LAST, current+1)
    │           └─► wraps around to lowest if at end
    │
    └─► profile_set(next_profile)
            └─► driver applies new profile to HW
```

#### الإشعار عند التغيير من العتاد (HW-triggered)

```
Hardware interrupt / ACPI event
    │
    ▼
driver interrupt handler / ACPI notify callback
    │
    ├─► reads new profile from HW
    ├─► (optionally updates internal state)
    │
    └─► platform_profile_notify(class_dev)
            │
            └─► kobject_uevent(kobj, KOBJ_CHANGE)
                    └─► udev / systemd sees event
                    └─► userspace re-reads /sys/.../profile
```

---

### 5. استراتيجية الـ Locking

#### مستويات الـ Locking

| المستوى | المسؤول | الحماية |
|---------|---------|--------|
| **Subsystem level** | `platform_profile.c` (mutex داخلي) | تسجيل/إلغاء تسجيل الـ handlers، قراءة/كتابة الـ sysfs |
| **Driver level** | mutex خاص بكل driver | الوصول لمتغيرات الـ driver الداخلية وسجلات العتاد |

#### قواعد الـ Locking

```
┌──────────────────────────────────────────────────┐
│  subsystem_mutex  (في platform_profile.c)        │
│     │                                            │
│     ├─ يُقفل قبل استدعاء أي ops callback        │
│     ├─ يحمي قائمة الـ registered handlers       │
│     └─ يحمي الـ sysfs read/write operations     │
│                                                  │
│     WHILE HELD: يستدعي ops->profile_get/set()   │
│         │                                        │
│         └─► driver يأخذ mutex خاص به             │
│               (nested lock — ترتيب ثابت)        │
└──────────────────────────────────────────────────┘
```

#### ملاحظات مهمة للـ Driver

- **`profile_get` و `profile_set`** تُستدعى تحت قفل الـ subsystem → يجب ألّا تحاول أخذ نفس القفل (deadlock).
- **`platform_profile_notify()`** يُستدعى من context الـ driver (قد يكون interrupt context) — `kobject_uevent` آمن في هذا السياق.
- **`platform_profile_cycle()`** تأخذ قفل الـ subsystem داخلياً → لا تُستدعى من داخل callback.

#### مثال صحيح لبنية الـ Locking في الـ Driver

```c
struct my_driver_data {
    struct mutex lock;      /* يحمي hw_state */
    int hw_state;
};

static int my_profile_set(struct device *dev,
                           enum platform_profile_option profile)
{
    struct my_driver_data *data = dev_get_drvdata(dev);
    int ret;

    /* subsystem_mutex already held by caller */
    mutex_lock(&data->lock);          /* driver-level lock (nested) */
    ret = write_hw_register(data, profile);
    mutex_unlock(&data->lock);

    return ret;
}
```

---

### ملخص تدفق التكامل الكامل

```
[Driver] ──register──► [platform_profile subsystem] ──sysfs──► [Userspace]
   │                           │                                    │
   │   ops->probe()            │   /sys/class/                      │
   │◄──────────────────────────│   platform-profile/name/           │
   │   fills choices bitmap    │     ├── profile (rw)               │
   │                           │     └── choices (ro)               │
   │   ops->profile_get()      │                          cat profile│
   │◄──────────────────────────│◄──────────────────────────────────│
   │   reads HW → enum         │   returns string                   │
   │──────────────────────────►│──────────────────────────────────►│
   │                           │                                    │
   │   ops->profile_set()      │                    echo performance│
   │◄──────────────────────────│◄──────────────────────────────────│
   │   validates + writes HW   │   validate → set → notify uevent  │
   │──────────────────────────►│──────────────────────────────────►│
   │                           │                         KOBJ_CHANGE│
   │   platform_profile_notify │                                    │
   │──────────────────────────►│──────────────────────────────────►│
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ API — Cheatsheet

#### Registration & Lifecycle

| Function | Signature (simplified) | الغرض |
|---|---|---|
| `platform_profile_register` | `struct device *f(dev, name, drvdata, ops)` | تسجيل handler جديد مع class |
| `platform_profile_remove` | `void f(struct device *dev)` | إلغاء تسجيل handler وتحرير الموارد |
| `devm_platform_profile_register` | `struct device *f(dev, name, drvdata, ops)` | نسخة managed تُنظَّف تلقائياً عند `device_release` |

#### Runtime / Control

| Function | Signature (simplified) | الغرض |
|---|---|---|
| `platform_profile_cycle` | `int f(void)` | التنقل للـ profile التالي المتاح على جميع الـ handlers |
| `platform_profile_notify` | `void f(struct device *dev)` | إشعار sysfs بتغيير الـ profile |

#### Ops Callbacks (في `platform_profile_ops`)

| Callback | Signature | الغرض |
|---|---|---|
| `probe` | `int f(void *drvdata, unsigned long *choices)` | تهيئة bitmap الخيارات المتاحة |
| `hidden_choices` | `int f(void *drvdata, unsigned long *choices)` | خيارات مخفية عن المستخدم |
| `profile_get` | `int f(struct device *dev, enum *profile)` | قراءة الـ profile الحالي من الـ hardware |
| `profile_set` | `int f(struct device *dev, enum profile)` | كتابة profile جديد إلى الـ hardware |

#### Internal Helpers (static في platform_profile.c)

| Function | الغرض |
|---|---|
| `_commmon_choices_show` | طباعة bitmap الخيارات كنص sysfs |
| `_store_class_profile` | تطبيق profile على device واحد |
| `_notify_class_profile` | إرسال `sysfs_notify` + `kobject_uevent` لـ device واحد |
| `get_class_profile` | قراءة profile من device واحد مع validation |
| `_aggregate_choices` | تجميع الخيارات المشتركة بين جميع الـ handlers بـ AND |
| `_remove_hidden_choices` | إزالة الخيارات المخفية من الـ aggregate |
| `_aggregate_profiles` | قراءة profile من جميع الـ handlers والتحقق من التوافق |
| `_store_and_notify` | store + notify لـ device واحد في خطوة واحدة |

---

### المجموعة الأولى: التسجيل والـ Lifecycle

هذه المجموعة مسؤولة عن إنشاء `platform_profile_handler` وتسجيله كـ class device ضمن class `platform-profile`، وكذلك إلغاء تسجيله عند انتهاء الحاجة. تعتمد على `IDA` لتخصيص minor numbers وعلى `device_register` لإضافة الـ device إلى device model.

---

#### `platform_profile_register`

```c
struct device *platform_profile_register(struct device *dev, const char *name,
                                         void *drvdata,
                                         const struct platform_profile_ops *ops);
```

تُنشئ `platform_profile_handler` جديداً وتسجّله كـ class device تحت class `platform-profile`. تستدعي `ops->probe` لملء bitmap الخيارات المتاحة، ثم تُخصّص minor number عبر `IDA`، ثم تستدعي `device_register`. بعد النجاح تُحدّث legacy sysfs group على `acpi_kobj`.

**Parameters:**
- `dev` — الـ parent device (عادةً `pdev->dev` للـ platform driver)
- `name` — اسم الـ handler (يظهر في sysfs attribute `name`)
- `drvdata` — pointer يُمرَّر للـ callbacks، يُخزَّن عبر `dev_set_drvdata`
- `ops` — pointer لـ `platform_profile_ops`، يجب أن تكون `probe`, `profile_get`, `profile_set` غير NULL

**Return:** pointer لـ `struct device` الجديد عند النجاح، أو `ERR_PTR(-errno)` عند الفشل.

**Key details:**
- يتحقق أولاً من صحة المعاملات بـ `WARN_ON_ONCE`.
- يستدعي `ops->probe` **قبل** الحصول على `profile_lock` لأنه قد يستغرق وقتاً.
- يحصل على `profile_lock` (mutex) بـ `guard(mutex)` قبل `ida_alloc` وتسجيل الـ device، مما يضمن atomicity مقابل طلبات sysfs متزامنة.
- إذا كان `bitmap_empty(choices)` يرفض التسجيل بـ `-EINVAL` (handler بلا خيارات غير مجدٍ).
- يستخدم cleanup scopes (`__free(kfree)`, `no_free_ptr`) لضمان عدم تسريب الذاكرة في مسارات الخطأ.
- عند الفشل في `device_register`: يستدعي `put_device` ثم `ida_free`.
- **Caller context:** process context فقط، قابل للنوم.

**Pseudocode flow:**
```
sanity_check(dev, name, ops)  → -EINVAL
alloc pprof (kzalloc)         → -ENOMEM
ops->probe(drvdata, choices)  → ERR on failure
bitmap_empty(choices)?        → -EINVAL
ops->hidden_choices(...)      [optional]
lock(profile_lock)
  minor = ida_alloc()
  init pprof fields
  device_register(ppdev)      → put_device + ida_free on error
  sysfs_notify(acpi_kobj)
  sysfs_update_group(acpi_kobj)
return ppdev
```

---

#### `platform_profile_remove`

```c
void platform_profile_remove(struct device *dev);
```

تُلغي تسجيل الـ `platform_profile_handler` المرتبط بـ `dev` وتحرر الـ minor number من الـ IDA. تُحدّث legacy sysfs group بعد الإزالة.

**Parameters:**
- `dev` — الـ class device المُعاد من `platform_profile_register`

**Return:** void.

**Key details:**
- تتحقق من `IS_ERR_OR_NULL(dev)` وتعود صامتةً إذا كان NULL أو ERR.
- تحصل على `profile_lock` قبل `ida_free` و`device_register` لضمان عدم التزامن مع قراءات sysfs.
- الـ `pprof_device_release` callback ينفذ `kfree(pprof)` عند آخر `put_device`.
- **Caller context:** process context فقط.

---

#### `devm_platform_profile_register`

```c
struct device *devm_platform_profile_register(struct device *dev, const char *name,
                                              void *drvdata,
                                              const struct platform_profile_ops *ops);
```

نسخة **device-managed** من `platform_profile_register`. تُخصّص `devres` resource يستدعي `platform_profile_remove` تلقائياً عند `device_release` للـ parent device.

**Parameters:** نفس `platform_profile_register`.

**Return:** pointer لـ `struct device` عند النجاح، أو `ERR_PTR(-errno)`.

**Key details:**
- تستخدم `devres_alloc` + `devres_add` للربط بـ parent device lifecycle.
- عند فشل `platform_profile_register` تستدعي `devres_free` لتجنب تسريب الـ devres node.
- مناسبة للـ drivers التي تتبع نمط `devm_*` للتنظيف التلقائي في `probe/remove`.
- **Caller context:** process context فقط.

---

### المجموعة الثانية: Runtime Control

تتحكم هذه المجموعة في قراءة وكتابة والتنقل بين الـ profiles أثناء تشغيل النظام، سواء من خلال sysfs أو من خلال واجهة kernel مباشرة (مثل hotkey handler).

---

#### `platform_profile_cycle`

```c
int platform_profile_cycle(void);
```

ينتقل إلى الـ profile التالي المتاح بين جميع الـ handlers المسجّلة. يُستخدم عادةً من hotkey driver (مثل Fn+profile على laptops Lenovo/HP) لتدوير الـ profiles بضغطة واحدة.

**Parameters:** لا يوجد.

**Return:** `0` عند النجاح، `-EINVAL` إذا كان الـ profile الحالي `MAX_POWER`/`CUSTOM`/`LAST` أو لا يوجد handler، `-ERESTARTSYS` إذا انقطع انتظار الـ mutex، `-errno` من `profile_get`/`profile_set`.

**Key details:**
- يحصل على `profile_lock` بـ `scoped_cond_guard(mutex_intr)` — يعود بـ `-ERESTARTSYS` إذا جاءت signal قبل الحصول على الـ lock.
- يجمع الـ profile الحالي من جميع الـ handlers عبر `_aggregate_profiles`.
- يرفض التنقل إذا كان الـ profile الحالي `MAX_POWER` أو `CUSTOM` (تصميم متعمد: هذه profiles نهائية أو مخصصة).
- يحذف `PLATFORM_PROFILE_MAX_POWER` و`PLATFORM_PROFILE_CUSTOM` من bitmap الخيارات قبل البحث عن التالي — لضمان عدم التنقل إليها تلقائياً.
- يستخدم `find_next_bit_wrap` للبحث عن أول bit مضبوط **بعد** الـ profile الحالي مع **wrap-around** للبداية.
- يُطبّق الـ profile الجديد على جميع الـ handlers ويُرسل uevent عبر `_store_and_notify`.
- يُعلم legacy sysfs بـ `sysfs_notify(acpi_kobj, NULL, "platform_profile")` خارج الـ lock.

**Pseudocode flow:**
```
lock(profile_lock) [interruptible]
  _aggregate_profiles() → profile
  if profile in {MAX_POWER, CUSTOM, LAST} → -EINVAL
  _aggregate_choices() → data.aggregate
  clear_bit(MAX_POWER, aggregate)
  clear_bit(CUSTOM, aggregate)
  next = find_next_bit_wrap(aggregate, LAST, profile+1)
  class_for_each_device(_store_and_notify, next)
sysfs_notify(acpi_kobj, "platform_profile")
return 0
```

---

#### `platform_profile_notify`

```c
void platform_profile_notify(struct device *dev);
```

تُرسل إشعاراً لـ sysfs بأن الـ profile تغيّر على الـ `dev` المُحدد. تُستخدم من الـ driver عندما يتغير الـ profile خارجياً (مثل تغييره من الـ firmware/BIOS دون تدخل userspace).

**Parameters:**
- `dev` — الـ class device الذي تغيّر profile-ه

**Return:** void.

**Key details:**
- تحصل على `profile_lock` بـ `scoped_cond_guard(mutex_intr)` — إذا جاءت signal تعود صامتةً (دون خطأ، وهو سلوك متعمد لأنها void).
- تستدعي `_notify_class_profile` التي تنفذ `sysfs_notify` على attribute `profile` الخاصة بـ class device + `kobject_uevent(KOBJ_CHANGE)`.
- تُعلم أيضاً legacy sysfs عبر `sysfs_notify(acpi_kobj, NULL, "platform_profile")`.
- **Caller context:** يمكن استدعاؤها من process context أو من interrupt context إذا كانت `GFP_ATOMIC`، لكن الـ mutex يجعلها process-only عملياً.

---

### المجموعة الثالثة: الـ Ops Callbacks (واجهة الـ Driver)

هذه ليست functions في الـ subsystem بل هي **callback pointers** يجب أن يُوفّرها الـ driver. تمثّل العقد بين الـ subsystem والـ hardware-specific driver.

---

#### `ops->probe`

```c
int (*probe)(void *drvdata, unsigned long *choices);
```

يُستدعى مرة واحدة أثناء `platform_profile_register`. يجب على الـ driver أن يملأ `choices` بـ bitmap يمثّل الـ profiles التي تدعمها الـ hardware.

**Parameters:**
- `drvdata` — الـ driver private data المُمرَّر عند التسجيل
- `choices` — bitmap بحجم `BITS_TO_LONGS(PLATFORM_PROFILE_LAST)` يجب ملؤه

**Return:** `0` عند النجاح، `-errno` عند الفشل (يُوقف التسجيل).

**مثال:**
```c
static int my_probe(void *drvdata, unsigned long *choices)
{
    /* Hardware supports: low-power, balanced, performance */
    set_bit(PLATFORM_PROFILE_LOW_POWER, choices);
    set_bit(PLATFORM_PROFILE_BALANCED, choices);
    set_bit(PLATFORM_PROFILE_PERFORMANCE, choices);
    return 0;
}
```

---

#### `ops->hidden_choices`

```c
int (*hidden_choices)(void *drvdata, unsigned long *choices);
```

**اختياري** (يُفحص بـ `if (ops->hidden_choices)`). يُعرّف profiles يمكن للـ driver ضبطها داخلياً لكنها **غير مرئية** في sysfs للمستخدم. تُستخدم مثلاً لـ profile `CUSTOM` الذي يعكس حالة firmware مخصصة.

**Parameters:** نفس `probe`.

**Return:** `0` أو `-errno`.

**Key detail:** في حالة وجود handler واحد فقط، تُحذف `hidden_choices` من الـ aggregate المُعروض في legacy sysfs عبر `_remove_hidden_choices`.

---

#### `ops->profile_get`

```c
int (*profile_get)(struct device *dev, enum platform_profile_option *profile);
```

يُستدعى عند قراءة sysfs attribute `profile`. يجب على الـ driver قراءة الـ profile الحالي من الـ hardware وإعادته.

**Parameters:**
- `dev` — الـ class device (يمكن الوصول لـ drvdata بـ `dev_get_drvdata(dev)`)
- `profile` — pointer لتخزين الـ profile المقروء

**Return:** `0` عند النجاح، `-errno` عند فشل التواصل مع الـ hardware.

**Key detail:** يُستدعى **داخل** `profile_lock`. يجب أن يكون الـ callback خفيفاً وغير blocking لتجنب inversion.

---

#### `ops->profile_set`

```c
int (*profile_set)(struct device *dev, enum platform_profile_option profile);
```

يُستدعى عند كتابة sysfs attribute `profile` أو من `platform_profile_cycle`. يكتب الـ profile الجديد إلى الـ hardware.

**Parameters:**
- `dev` — الـ class device
- `profile` — الـ profile المطلوب تطبيقه (تم التحقق من وجوده في `choices` أو `hidden_choices` قبل الاستدعاء)

**Return:** `0` عند النجاح، `-errno` عند الفشل.

**Key detail:** يُستدعى داخل `profile_lock` أيضاً. الـ subsystem يتحقق من أن الـ profile موجود في `choices | hidden_choices` قبل الاستدعاء عبر `_store_class_profile`.

---

### المجموعة الرابعة: Internal Helpers (Static)

هذه دوال داخلية لا تُصدَّر للـ drivers لكنها تُشكّل العمود الفقري لمنطق الـ subsystem.

---

#### `_commmon_choices_show`

```c
static ssize_t _commmon_choices_show(unsigned long *choices, char *buf);
```

تحوّل bitmap الخيارات إلى نص مفصول بمسافات لـ sysfs. تُستخدم من `choices_show` (per-device) و`platform_profile_choices_show` (legacy aggregate).

**Parameters:**
- `choices` — bitmap الخيارات
- `buf` — sysfs page buffer (حجمه `PAGE_SIZE`)

**Return:** عدد البايتات المكتوبة.

---

#### `_store_class_profile`

```c
static int _store_class_profile(struct device *dev, void *data);
```

تتحقق أن الـ `bit` المطلوب (الـ profile) موجود في `choices` أو `hidden_choices` للـ handler، ثم تستدعي `ops->profile_set`. تُستخدم كـ callback في `class_for_each_device`.

**Key detail:** `lockdep_assert_held(&profile_lock)` — يجب أن يكون المُستدعي يحمل الـ lock. يعيد `-EOPNOTSUPP` إذا لم يدعم الـ handler الـ profile المطلوب.

---

#### `_notify_class_profile`

```c
static int _notify_class_profile(struct device *dev, void *data);
```

تُرسل `sysfs_notify` على attribute `profile` الخاصة بـ class device + `kobject_uevent(KOBJ_CHANGE)` لإعلام userspace (مثل `udev`، `inotify`، أو polling على sysfs).

**Key detail:** `lockdep_assert_held(&profile_lock)`. الـ `data` غير مستخدم (NULL دائماً).

---

#### `get_class_profile`

```c
static int get_class_profile(struct device *dev,
                             enum platform_profile_option *profile);
```

تُغلّف `ops->profile_get` مع validation: تتحقق أن القيمة المُعادة `< PLATFORM_PROFILE_LAST` بـ `WARN_ON`. تُستدعى من `profile_show` و`_aggregate_profiles`.

**Key detail:** `lockdep_assert_held(&profile_lock)`.

---

#### `_aggregate_choices`

```c
static int _aggregate_choices(struct device *dev, void *arg);
```

تُستخدم مع `class_for_each_device`. تحسب **تقاطع** (AND) الخيارات المتاحة عبر جميع الـ handlers — أي الخيارات التي **يدعمها الجميع**. تبدأ بـ bitmap مملوء بـ `~0UL` وتطبق `bitmap_and` تراكمياً.

**الخوارزمية:**
```
tmp = handler->choices | handler->hidden_choices
aggregate = aggregate AND tmp
count++
```

---

#### `_remove_hidden_choices`

```c
static int _remove_hidden_choices(struct device *dev, void *arg);
```

**تُستخدم فقط** عندما يكون هناك handler واحد (`data->count == 1`). تحذف `hidden_choices` من الـ aggregate لأنه لا معنى لإظهارها للمستخدم في legacy sysfs عند وجود driver واحد.

**الخوارزمية:**
```
aggregate = choices ANDNOT hidden_choices
```

---

#### `_aggregate_profiles`

```c
static int _aggregate_profiles(struct device *dev, void *data);
```

تُستخدم مع `class_for_each_device` لقراءة الـ profile من كل handler وتجميعها. إذا اختلف الـ profile بين الـ handlers يُعيّن `PLATFORM_PROFILE_CUSTOM` كنتيجة للـ aggregate.

**منطق الـ Aggregation:**
```
if *profile == LAST:           *profile = val      (أول handler)
elif *profile == val:          *profile = val      (متطابق)
else:                          *profile = CUSTOM   (تعارض)
```

---

#### `_store_and_notify`

```c
static int _store_and_notify(struct device *dev, void *data);
```

تجمع `_store_class_profile` + `_notify_class_profile` في callback واحد لـ `class_for_each_device`. تُستخدم في `platform_profile_cycle` و`platform_profile_store` (legacy).

---

### المجموعة الخامسة: Module Init/Exit

---

#### `platform_profile_init`

```c
static int __init platform_profile_init(void);
```

تُنفَّذ عند تحميل الـ module. تتحقق من أن ACPI غير معطَّل (`acpi_disabled`)، ثم تُسجّل `platform_profile_class` وتُنشئ legacy sysfs group على `acpi_kobj` (`/sys/firmware/acpi/`).

**Key detail:** إذا كان `acpi_disabled == true` يعيد `-EOPNOTSUPP` ولا يتم تحميل الـ module — الـ subsystem مرتبط بـ ACPI firmware interface.

---

#### `platform_profile_exit`

```c
static void __exit platform_profile_exit(void);
```

تُزيل legacy sysfs group وتُلغي تسجيل الـ class. يجب أن تكون جميع الـ handlers قد أُزيلت قبل هذه النقطة (عبر `platform_profile_remove` أو `devm_*`).

---

### ملاحظات معمارية

**الـ Locking:**

```
profile_lock (mutex)
    ├── platform_profile_register  (guard(mutex))
    ├── platform_profile_remove    (guard(mutex))
    ├── platform_profile_cycle     (scoped_cond_guard + interruptible)
    ├── platform_profile_notify    (scoped_cond_guard + interruptible)
    ├── profile_show               (scoped_cond_guard + interruptible)
    └── profile_store              (scoped_cond_guard + interruptible)
```

الـ `scoped_cond_guard(mutex_intr)` يسمح للـ signal بمقاطعة الانتظار — ضروري لـ sysfs handlers لتجنب blocking المستخدم إلى أجل غير مسمى.

**تدفق البيانات في sysfs:**

```
userspace read  (/sys/class/platform-profile/platform-profile-0/profile)
    → profile_show
        → scoped_cond_guard(profile_lock)
        → get_class_profile
            → ops->profile_get     [driver callback]
        → sysfs_emit(buf, profile_names[profile])

userspace write (/sys/.../profile)
    → profile_store
        → sysfs_match_string → index
        → scoped_cond_guard(profile_lock)
        → _store_class_profile
            → test_bit(index, choices | hidden_choices)
            → ops->profile_set     [driver callback]
        → sysfs_notify(acpi_kobj, "platform_profile")  [legacy compat]
```

**Legacy vs Per-device sysfs:**

| الجانب | Legacy (`/sys/firmware/acpi/`) | Per-device (`/sys/class/platform-profile/`) |
|---|---|---|
| الموقع | `acpi_kobj` | class device kobj |
| دعم multi-handler | aggregate (AND/CUSTOM) | per-device مباشر |
| متى يظهر | عند وجود أي handler واحد على الأقل (`is_visible`) | دائماً لكل handler |
| الغرض | backward compat مع kernels قديمة | الواجهة الجديدة الموصى بها |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs ذات الصلة

الـ **platform_profile** subsystem لا يُسجِّل مدخلات في debugfs بشكل مباشر، لكن يمكن الوصول إلى معلومات الـ class device من خلال:

```bash
# عرض جميع devices المسجَّلة تحت platform_profile class
ls /sys/class/platform-profile/

# مثال مخرج:
# platform-profile0

# قراءة الـ kobject الداخلي عبر debugfs (إن كانت مفعَّلة)
ls /sys/kernel/debug/
```

> ملاحظة: الـ subsystem يعتمد أساساً على sysfs لا debugfs.

---

#### 2. مدخلات الـ sysfs

الـ **sysfs entries** الرئيسية تحت `/sys/class/platform-profile/<name>/`:

| المسار | الوصف | مثال القراءة |
|---|---|---|
| `choices` | القيم المتاحة للـ profile | `cat /sys/class/platform-profile/platform-profile0/choices` |
| `profile` | الـ profile الحالي (قراءة/كتابة) | `cat /sys/class/platform-profile/platform-profile0/profile` |
| `name` | اسم الجهاز | `cat /sys/class/platform-profile/platform-profile0/name` |

```bash
# قراءة الـ profile الحالي
cat /sys/class/platform-profile/platform-profile0/profile
# مخرج مثال: balanced

# قراءة الخيارات المتاحة
cat /sys/class/platform-profile/platform-profile0/choices
# مخرج مثال: low-power cool quiet balanced balanced-performance performance

# تغيير الـ profile
echo "performance" > /sys/class/platform-profile/platform-profile0/profile

# التحقق من التغيير
cat /sys/class/platform-profile/platform-profile0/profile
# المخرج: performance
```

---

#### 3. استخدام الـ ftrace

تفعيل الـ **tracepoints** والـ **events** المتعلقة بالـ platform_profile:

```bash
# الوصول إلى مجلد tracing
cd /sys/kernel/debug/tracing

# البحث عن events ذات صلة
grep -r "platform_profile" available_events

# تفعيل أحداث الـ power management العامة (تشمل profile changes)
echo 1 > events/power/enable

# تفعيل حدث pm_qos_update_request إن وُجد
echo 1 > events/power/pm_qos_update_request/enable

# تتبع استدعاءات دوال platform_profile عبر function tracer
echo function > current_tracer
echo "platform_profile_*" > set_ftrace_filter
echo 1 > tracing_on

# تنفيذ تغيير profile ثم قراءة النتيجة
echo "performance" > /sys/class/platform-profile/platform-profile0/profile
cat trace | grep platform_profile

# إيقاف التتبع
echo 0 > tracing_on
echo nop > current_tracer
```

```bash
# استخدام trace-cmd لتسهيل العملية
trace-cmd record -e power -p function --func-stack \
    -l "platform_profile_*" \
    sh -c 'echo performance > /sys/class/platform-profile/platform-profile0/profile'

trace-cmd report | grep -E "platform_profile|profile_set|profile_get"
```

---

#### 4. الـ printk والـ dynamic debug

تفعيل **dynamic debug** للـ platform_profile subsystem:

```bash
# تفعيل جميع رسائل debug في ملف platform_profile.c
echo "file platform_profile.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل رسائل debug مع معلومات إضافية (function name + line number)
echo "file platform_profile.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# التحقق من التفعيل
grep "platform_profile" /sys/kernel/debug/dynamic_debug/control

# قراءة الرسائل في الوقت الفعلي
dmesg -w | grep -i "platform.profile"

# أو عبر journalctl
journalctl -f -k | grep -i "platform.profile"
```

لزيادة مستوى الـ **printk** لـ kernel messages:

```bash
# رفع مستوى الـ loglevel لرؤية debug messages
echo 8 > /proc/sys/kernel/printk

# إعادة للوضع الطبيعي
echo 4 > /proc/sys/kernel/printk
```

---

#### 5. خيارات الـ Kernel Config للـ debugging

| الخيار | الوصف | الاستخدام |
|---|---|---|
| `CONFIG_ACPI_PLATFORM_PROFILE` | دعم platform profile عبر ACPI | أساسي للـ debugging |
| `CONFIG_DEBUG_FS` | تفعيل debugfs | لازم لكثير من أدوات الـ debug |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug | لتفعيل `pr_debug()` بشكل ديناميكي |
| `CONFIG_TRACING` | تفعيل نظام الـ tracing | للـ ftrace |
| `CONFIG_FUNCTION_TRACER` | تتبع استدعاءات الدوال | لـ function tracing |
| `CONFIG_DEBUG_OBJECTS` | اكتشاف استخدام objects المحررة | لـ device lifecycle bugs |
| `CONFIG_SYSFS` | تفعيل sysfs | أساسي للـ subsystem |
| `CONFIG_LEDS_TRIGGERS` | إن كان الـ driver يتحكم بـ LEDs | للأجهزة ذات LED indicators |
| `CONFIG_ACPI_DEBUG` | رسائل ACPI debug التفصيلية | لـ ACPI-based drivers |

```bash
# التحقق من الخيارات في الـ kernel الحالي
grep -E "CONFIG_(ACPI_PLATFORM_PROFILE|DYNAMIC_DEBUG|DEBUG_FS|TRACING)" /boot/config-$(uname -r)

# أو من داخل kernel source
grep -E "platform.profile" kernel/configs/ -r
```

---

#### 6. أدوات خاصة بالـ subsystem

**الـ platform_profile** يتفاعل مع userspace عبر أدوات متعددة:

```bash
# استخدام upower لقراءة الـ profile (يتفاعل مع platform_profile)
upower --dump | grep -i profile

# استخدام power-profiles-daemon (PPD) الذي يعتمد على platform_profile
systemctl status power-profiles-daemon

# قراءة الـ profile عبر PPD dbus API
gdbus call --system \
    --dest net.hadess.PowerProfiles \
    --object-path /net/hadess/PowerProfiles \
    --method org.freedesktop.DBus.Properties.Get \
    net.hadess.PowerProfiles ActiveProfile

# تغيير الـ profile عبر PPD
gdbus call --system \
    --dest net.hadess.PowerProfiles \
    --object-path /net/hadess/PowerProfiles \
    --method org.freedesktop.DBus.Properties.Set \
    net.hadess.PowerProfiles ActiveProfile \
    "<'performance'>"

# مراقبة تغييرات الـ profile عبر udev
udevadm monitor --kernel --subsystem-match=platform-profile

# التحقق من تسجيل الـ device في udev
udevadm info /sys/class/platform-profile/platform-profile0
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `platform_profile: ops->probe failed` | فشل الـ `probe` callback عند التسجيل | التحقق من منطق الـ probe وإرجاع قيمة choices صحيحة |
| `platform_profile: choices is empty` | الـ driver لم يُحدِّد أي profile متاح | التأكد من تفعيل bit واحد على الأقل في `choices` bitmap |
| `platform_profile: profile_set failed` | فشل تطبيق الـ profile على الـ hardware | التحقق من منطق الـ `profile_set` callback والـ hardware state |
| `platform_profile: profile not in choices` | محاولة تعيين profile غير مدعوم | القراءة من `choices` أولاً والاختيار منها فقط |
| `platform_profile: already registered` | تسجيل مزدوج لنفس الـ device | التأكد من عدم استدعاء `platform_profile_register` مرتين |
| `devm_platform_profile_register: -ENOMEM` | نقص الذاكرة عند التسجيل | فحص استهلاك الذاكرة، إعادة تشغيل النظام |
| `platform_profile: invalid profile option` | قيمة الـ profile خارج نطاق `PLATFORM_PROFILE_LAST` | التحقق من أن القيمة ضمن `platform_profile_option` enum |
| `platform_profile: ops->profile_get returned error` | فشل قراءة الـ profile من الـ hardware | فحص اتصال الـ hardware والـ firmware |

```bash
# مراقبة رسائل الـ kernel المتعلقة بالـ platform_profile
dmesg --follow | grep -iE "platform.profile|profile_set|profile_get"

# البحث في سجلات الـ kernel عن أخطاء سابقة
journalctl -k --since "1 hour ago" | grep -i "platform.profile"
```

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

نقاط حرجة يُنصح بإضافة **WARN_ON()** أو **dump_stack()** عندها:

```c
/* في platform_profile_register() — التحقق من صحة الـ ops */
int platform_profile_register(...)
{
    /* التحقق من أن profile_get و profile_set غير NULL */
    if (WARN_ON(!ops->profile_get || !ops->profile_set))
        return ERR_PTR(-EINVAL);

    ret = ops->probe(drvdata, &choices);
    if (WARN_ON(ret < 0)) {
        /* probe failed — dump call stack لتحديد المسبب */
        dump_stack();
        return ERR_PTR(ret);
    }

    /* التحقق من أن الـ choices bitmap غير فارغ */
    if (WARN_ON(bitmap_empty(&choices, PLATFORM_PROFILE_LAST)))
        return ERR_PTR(-EINVAL);
}

/* في profile_set callback — التحقق من صحة الـ profile المطلوب */
static int my_driver_profile_set(struct device *dev,
                                  enum platform_profile_option profile)
{
    /* يجب ألا يُستدعى بقيمة خارج النطاق */
    if (WARN_ON(profile >= PLATFORM_PROFILE_LAST))
        return -EINVAL;

    /* إن كان الـ hardware في حالة غير متوقعة */
    hw_state = read_hw_register();
    if (WARN_ON(hw_state == HW_STATE_ERROR)) {
        dump_stack(); /* لمعرفة من استدعى profile_set في هذا الوقت */
        return -EIO;
    }
}

/* في platform_profile_notify() — التحقق من أن dev صالح */
void platform_profile_notify(struct device *dev)
{
    if (WARN_ON(!dev))
        return;
    /* ... */
}
```

---

### Hardware Level

---

#### 1. التحقق من تطابق حالة الـ Hardware مع حالة الـ Kernel

```bash
# قراءة الـ profile المُفعَّل من الـ kernel
KERNEL_PROFILE=$(cat /sys/class/platform-profile/platform-profile0/profile)
echo "Kernel profile: $KERNEL_PROFILE"

# للأجهزة ASUS (عبر ACPI WMI)
# قراءة سجل ACPI المتحكم في الـ profile
cat /sys/bus/platform/devices/asus-nb-wmi/throttle_thermal_policy
# 0 = performance, 1 = balanced, 2 = silent

# للأجهزة Lenovo (ThinkPad)
cat /sys/bus/platform/devices/thinkpad_acpi/dytc_lapmode

# للأجهزة Dell
cat /sys/bus/wmi/drivers/dell-wmi-ddv/*/thermal_mode

# مقارنة القيمة المقروءة من ACPI مع ما يُظهره الـ kernel
# يجب أن تكون متطابقة بعد أي عملية profile_set
```

---

#### 2. تقنيات الـ Register Dump

```bash
# استخدام devmem2 لقراءة ACPI registers (إن كانت العناوين معروفة)
# مثال: قراءة سجل ACPI EC (Embedded Controller)
modprobe ec_sys write_support=1

# قراءة جميع سجلات الـ EC
dd if=/sys/kernel/debug/ec/ec0/io bs=1 count=256 2>/dev/null | xxd

# قراءة سجل محدد (byte 0x93 مثلاً للـ thermal mode في بعض الأجهزة)
dd if=/sys/kernel/debug/ec/ec0/io bs=1 skip=$((0x93)) count=1 2>/dev/null | xxd

# استخدام /dev/mem (يتطلب CONFIG_STRICT_DEVMEM=n أو CAP_SYS_RAWIO)
# قراءة ACPI FADT لمعرفة عنوان EC
acpidump -n FACP | acpixtract -l
iasl -d FACP.dat

# استخدام acpi_call لاستدعاء ACPI methods مباشرة
modprobe acpi_call
echo '\_SB.PCI0.LPCB.EC0._Q0E' > /proc/acpi/call
cat /proc/acpi/call
```

---

#### 3. نصائح Logic Analyzer / Oscilloscope

للـ **platform_profile** على مستوى الـ hardware، أهم الإشارات للمراقبة:

```
+------------------+------------------------------------------+
| الإشارة          | ما تُظهره                                |
+------------------+------------------------------------------+
| LPC Bus          | ACPI EC transactions عند تغيير الـ profile|
| SMBus/I2C        | اتصال الـ MCU أو BIOS عبر SMBus          |
| Fan PWM Signal   | تغير duty cycle مع تغيير الـ profile      |
| CPU Vcore        | انخفاض الجهد في low-power mode            |
| PROCHOT# pin     | تفعيله في performance mode عند الحمل     |
+------------------+------------------------------------------+
```

```
Logic Analyzer Setup (مثال sigrok/PulseView):
─────────────────────────────────────────────
Channel 0: LPC_CLK  (33 MHz)
Channel 1: LPC_AD0..3 (multiplexed)
Channel 2: LPC_FRAME#
Channel 3: Fan_PWM

Trigger: Rising edge on LPC_FRAME# عند كتابة إلى EC port 0x66
Capture: 10ms window
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | نمط الـ kernel log | السبب | الحل |
|---|---|---|---|
| الـ BIOS يُعيد تعيين الـ profile | `platform_profile: profile changed unexpectedly` | الـ BIOS يتجاهل تعيين الـ OS | تفعيل "OS control" في BIOS settings |
| الـ EC لا يستجيب | `ACPI: EC timeout` | تعارض في الـ EC protocol | إضافة `ec_no_wakeup` لـ kernel cmdline |
| الـ fan لا يتغير مع الـ profile | لا رسائل خطأ لكن الأداء ثابت | الـ driver لا يتحكم بالـ fan مباشرة | فحص BIOS fan control settings |
| crash عند التسجيل | `BUG: unable to handle kernel NULL pointer` | الـ ops struct تحتوي NULL pointers | التحقق من تهيئة جميع callbacks |
| الـ profile يعود لـ balanced بعد S3 | لا رسائل في الـ log | الـ BIOS يُعيد تهيئة EC عند الاستيقاظ | إضافة resume callback لإعادة تطبيق الـ profile |

```bash
# مراقبة أحداث الطاقة (suspend/resume) وتأثيرها على الـ profile
journalctl -k | grep -E "(platform.profile|PM:|ACPI:|suspend|resume)"

# التحقق من أن الـ profile يُحفظ ويُستعاد عند الـ resume
cat /sys/class/platform-profile/platform-profile0/profile  # قبل suspend
systemctl suspend
cat /sys/class/platform-profile/platform-profile0/profile  # بعد resume
```

---

#### 5. تصحيح أخطاء الـ Device Tree

الـ **platform_profile** يُستخدم أساساً على أجهزة ACPI (x86)، لكن على ARM/embedded platforms يعتمد على DT:

```bash
# التحقق من وجود الـ compatible string في DT
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "platform-profile"

# فحص الـ DT المُجمَّع المُحمَّل فعلياً
ls /sys/firmware/devicetree/base/

# التحقق من تطابق الـ compatible مع الـ driver
cat /sys/bus/platform/drivers/*/modalias | grep profile

# عرض الـ DT node الكامل لجهاز platform profile (ARM مثال)
cat /sys/firmware/devicetree/base/thermal-zones/platform-profile/compatible 2>/dev/null
```

مثال DT node صحيح:

```dts
/* مثال تعريف platform_profile في Device Tree */
thermal_profile: platform-profile {
    compatible = "vendor,thermal-profile";
    /* ربط بـ cooling devices */
    cooling-maps {
        map0 {
            trip = <&cpu_alert>;
            cooling-device = <&cpu0 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>;
        };
    };
};
```

```bash
# التحقق من أن الـ kernel يرى الـ DT node
dmesg | grep -i "platform.profile\|thermal.profile"

# مقارنة الـ DT المُترجَم مع توقعات الـ driver
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null > /tmp/current.dts
grep -A 20 "platform-profile" /tmp/current.dts
```

---

### Practical Commands

---

#### سكريبت تشخيص شامل

```bash
#!/bin/bash
# platform_profile_debug.sh — سكريبت تشخيص شامل

echo "=== Platform Profile Debug Report ==="
echo "Date: $(date)"
echo "Kernel: $(uname -r)"
echo ""

echo "--- Available Platform Profile Devices ---"
ls /sys/class/platform-profile/ 2>/dev/null || echo "No platform-profile devices found"
echo ""

for dev in /sys/class/platform-profile/*/; do
    name=$(basename "$dev")
    echo "--- Device: $name ---"
    echo "Current Profile : $(cat $dev/profile 2>/dev/null || echo 'N/A')"
    echo "Available Choices: $(cat $dev/choices 2>/dev/null || echo 'N/A')"
    echo "Device Name     : $(cat $dev/name 2>/dev/null || echo 'N/A')"
    echo ""
done

echo "--- Kernel Messages (last 50 lines) ---"
dmesg | grep -iE "platform.profile|power.profile" | tail -50
echo ""

echo "--- power-profiles-daemon Status ---"
systemctl is-active power-profiles-daemon 2>/dev/null && \
    gdbus call --system \
        --dest net.hadess.PowerProfiles \
        --object-path /net/hadess/PowerProfiles \
        --method org.freedesktop.DBus.Properties.GetAll \
        net.hadess.PowerProfiles 2>/dev/null || echo "PPD not running"
echo ""

echo "--- ACPI EC Info ---"
ls /sys/kernel/debug/ec/ 2>/dev/null || echo "EC debug not available (try: modprobe ec_sys)"
echo ""

echo "--- udev Events ---"
udevadm info /sys/class/platform-profile/platform-profile0 2>/dev/null
```

---

#### تفعيل الـ ftrace خطوة بخطوة

```bash
#!/bin/bash
# ftrace_platform_profile.sh

TRACEDIR=/sys/kernel/debug/tracing

# إعداد الـ tracer
echo 0 > $TRACEDIR/tracing_on
echo function > $TRACEDIR/current_tracer
echo "platform_profile_*" > $TRACEDIR/set_ftrace_filter 2>/dev/null || \
    echo "platform_profile_register platform_profile_remove platform_profile_notify platform_profile_cycle" \
    > $TRACEDIR/set_ftrace_filter
echo 1 > $TRACEDIR/tracing_on

echo "Tracing platform_profile functions... Press Ctrl+C to stop"

# تنفيذ عملية تغيير الـ profile
echo "performance" > /sys/class/platform-profile/platform-profile0/profile
sleep 1
echo "balanced" > /sys/class/platform-profile/platform-profile0/profile

# إيقاف وعرض النتائج
echo 0 > $TRACEDIR/tracing_on
echo "--- Trace Results ---"
cat $TRACEDIR/trace | grep -v "^#" | head -50

# تنظيف
echo nop > $TRACEDIR/current_tracer
echo > $TRACEDIR/set_ftrace_filter
```

---

#### قراءة وتفسير المخرجات

```bash
# مخرج cat /sys/class/platform-profile/platform-profile0/choices
low-power cool quiet balanced balanced-performance performance
# التفسير: 6 خيارات متاحة، الـ CUSTOM و MAX_POWER غير مدعومَين على هذا الجهاز

# مخرج udevadm info
P: /devices/platform/platform-profile0/platform-profile/platform-profile0
E: DEVPATH=/devices/platform/platform-profile0/platform-profile/platform-profile0
E: SUBSYSTEM=platform-profile
E: DEVTYPE=platform-profile
# التفسير: الـ device مسجَّل تحت platform-profile subsystem
# يمكن إنشاء udev rules بناءً على هذه المعلومات

# مخرج dmesg عند تغيير الـ profile بنجاح (مع dynamic debug مُفعَّل)
[  123.456789] platform_profile: setting profile to performance
[  123.457001] asus-nb-wmi: thermal policy set to 0 (performance)
# التفسير: الـ kernel تلقى الطلب، الـ driver طبَّقه على الـ hardware

# مخرج عند فشل الـ profile_set
[  456.789012] platform_profile: profile_set failed: -EIO
[  456.789100] asus-nb-wmi: EC write timeout at port 0x66
# التفسير: تعذَّر على الـ driver الكتابة إلى الـ EC، مشكلة في الـ hardware أو الـ firmware
```

---

#### monitoring مستمر للـ profile changes

```bash
# مراقبة تغييرات الـ profile في الوقت الفعلي عبر inotifywait
inotifywait -m -e close_write /sys/class/platform-profile/platform-profile0/profile 2>/dev/null &

# أو عبر udevadm monitor
udevadm monitor --kernel --property --subsystem-match=platform-profile &

# مراقبة دورية كل ثانية
watch -n 1 'echo "Profile: $(cat /sys/class/platform-profile/platform-profile0/profile)"; \
            echo "CPU Governor: $(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor)"; \
            echo "CPU Freq: $(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq) kHz"'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: جهاز IoT صناعي على AM62x — التوصيف الخاطئ للـ profile

#### العنوان
**Industrial IoT Gateway** لا يقبل تغيير **platform profile** عبر sysfs

#### السياق
شركة تبني **industrial gateway** قائمة على معالج **AM62x** من Texas Instruments، يعمل بنظام Linux مخصص. الجهاز يُستخدم في محطات قياس صناعية ويحتاج التبديل بين وضع `PLATFORM_PROFILE_LOW_POWER` أثناء الخمول و`PLATFORM_PROFILE_PERFORMANCE` عند معالجة البيانات. المهندس يكتب driver مخصص ويسجله عبر `platform_profile_register()`.

#### المشكلة
عند الكتابة إلى `/sys/bus/platform/devices/.../platform_profile`:
```bash
echo "performance" > /sys/bus/platform/devices/power_mgr/platform_profile
# bash: echo: write error: Invalid argument
```
رغم أن الـ driver يبدو مكتوباً بشكل صحيح.

#### التحليل
الـ driver يُعرِّف `probe` callback:
```c
static int am62x_pm_probe_choices(void *drvdata, unsigned long *choices)
{
    /* BUG: المهندس نسي تفعيل LOW_POWER */
    set_bit(PLATFORM_PROFILE_PERFORMANCE, choices);
    set_bit(PLATFORM_PROFILE_BALANCED, choices);
    /* PLATFORM_PROFILE_LOW_POWER غير مُفعَّل */
    return 0;
}
```

**الـ** `struct platform_profile_ops` المُسجَّلة:
```c
static const struct platform_profile_ops am62x_profile_ops = {
    .probe          = am62x_pm_probe_choices,
    .profile_get    = am62x_profile_get,
    .profile_set    = am62x_profile_set,
};
```

عند استدعاء `platform_profile_set()` من kernel، يتحقق النظام من أن الـ profile المطلوب موجود في `choices` bitmap الذي أعاده `probe`. لأن `PLATFORM_PROFILE_LOW_POWER` غير مُعيَّن في الـ bitmap، أي محاولة لكتابته ترفض بـ `-EINVAL`.

المشكلة ليست في الـ `profile_set` callback مباشرةً، بل في أن الـ `probe` لم يُعلن عن الـ profile كخيار مدعوم. **الـ** `enum platform_profile_option` يحدد كل الأوضاع الممكنة، لكن الـ driver يجب أن يُفعِّل فقط ما يدعمه فعلاً عبر `set_bit()`.

#### الحل
```c
static int am62x_pm_probe_choices(void *drvdata, unsigned long *choices)
{
    /* Enable all supported profiles */
    set_bit(PLATFORM_PROFILE_LOW_POWER,   choices);
    set_bit(PLATFORM_PROFILE_BALANCED,    choices);
    set_bit(PLATFORM_PROFILE_PERFORMANCE, choices);
    return 0;
}
```

للتحقق بعد الإصلاح:
```bash
# عرض الـ profiles المدعومة
cat /sys/bus/platform/devices/power_mgr/platform_profile_choices
# low-power balanced performance

# تغيير الـ profile
echo "low-power" > /sys/bus/platform/devices/power_mgr/platform_profile
echo $?  # 0
```

#### الدرس المستفاد
**الـ** `probe` callback في `struct platform_profile_ops` هو **بوابة التحقق** — كل profile لم يُعلَن فيه عبر `set_bit()` مرفوض تلقائياً بمعزل عن منطق `profile_set`. يجب توثيق الـ bitmap المدعوم عند كتابة الـ driver.

---

### السيناريو 2: Android TV Box على Allwinner H616 — تعارض في تسجيل الـ profile

#### العنوان
**Kernel panic** عند إدراج module ثانٍ يحاول تسجيل **platform profile** إضافي

#### السياق
منتج **Android TV box** يستخدم **Allwinner H616**. المهندسون يطوّرون driver لإدارة الطاقة الحرارية. في البداية يوجد driver واحد (`h616_thermal_profile`). لاحقاً يُضاف driver جديد للـ GPU (`h616_gpu_profile`) يستدعي هو الآخر `platform_profile_register()` بـ `name` مختلف.

#### المشكلة
```bash
insmod h616_gpu_profile.ko
# Kernel: WARNING: ... platform_profile class already registered
# أو في حالات أخرى: duplicate sysfs entry
```

في kernels أقدم، قد يؤدي إلى `BUG_ON()`.

#### التحليل
**الـ** `platform_profile_register()` تُنشئ `struct device` جديداً تحت **platform_profile class**. منذ kernel 6.x، الـ subsystem يدعم **multiple profile devices** (كل device بـ `name` مستقل). لكن المشكلة أن الـ driver القديم استدعى النسخة القديمة من الـ API التي كانت تسمح بجهاز واحد فقط.

```c
/* driver قديم — API مرحلية */
int platform_profile_register(struct platform_profile_ops *ops);  /* قديم */

/* API جديد — يعيد struct device* */
struct device *platform_profile_register(struct device *dev,
                                          const char *name,
                                          void *drvdata,
                                          const struct platform_profile_ops *ops);
```

**الـ** `name` في الـ API الجديد يسمح بتمييز كل profile device:
```c
/* في h616_thermal_profile driver */
pdev->profile_dev = platform_profile_register(&pdev->dev,
                                               "thermal",
                                               pdev,
                                               &h616_thermal_ops);

/* في h616_gpu_profile driver */
pdev->profile_dev = platform_profile_register(&pdev->dev,
                                               "gpu",
                                               pdev,
                                               &h616_gpu_ops);
```

#### الحل
الانتقال إلى `devm_platform_profile_register()` لكل driver بشكل مستقل مع أسماء مختلفة:

```c
/* h616_thermal_profile.c */
static int h616_thermal_profile_probe(struct platform_device *pdev)
{
    struct h616_thermal_data *data = ...;

    data->profile_dev = devm_platform_profile_register(&pdev->dev,
                                                        "h616-thermal",
                                                        data,
                                                        &h616_thermal_profile_ops);
    if (IS_ERR(data->profile_dev))
        return PTR_ERR(data->profile_dev);
    return 0;
}
```

```bash
# التحقق من وجود كلا الـ devices
ls /sys/class/platform-profile/
# h616-thermal  h616-gpu
```

#### الدرس المستفاد
**الـ** `name` parameter في `platform_profile_register()` ليس مجرد label — هو **مُعرِّف فريد** للـ sysfs device. استخدام `devm_platform_profile_register()` يضمن cleanup تلقائي عند `remove()`، مما يمنع memory leak إذا فشل الـ probe.

---

### السيناريو 3: STM32MP1 في نظام طبي — الـ hidden_choices لإدارة سرية

#### العنوان
جهاز طبي على **STM32MP1** يحتاج **hidden profile** للمعايرة لا يظهر للمستخدم

#### السياق
شركة طبية تبني جهاز مراقبة مرضى على **STM32MP1**. الجهاز له وضعان مرئيان للمستخدم: `balanced` و`low-power`. لكن الـ firmware يحتاج وضعاً ثالثاً `PLATFORM_PROFILE_CUSTOM` لعمليات المعايرة (calibration) التي يجريها الفنيون — هذا الوضع يجب ألا يظهر في `platform_profile_choices`.

#### المشكلة
إذا أُضيف `PLATFORM_PROFILE_CUSTOM` إلى `probe` choices، يظهر للمستخدم العادي في:
```bash
cat /sys/class/platform-profile/stm32-pm/platform_profile_choices
# low-power balanced custom   <-- غير مرغوب
```

#### التحليل
**الـ** `struct platform_profile_ops` يوفر callback منفصلاً:

```c
struct platform_profile_ops {
    int (*probe)(void *drvdata, unsigned long *choices);           /* الأوضاع المرئية */
    int (*hidden_choices)(void *drvdata, unsigned long *choices);  /* الأوضاع المخفية */
    int (*profile_get)(...);
    int (*profile_set)(...);
};
```

**الـ** `hidden_choices` callback يُسجِّل profiles يمكن **تفعيلها** من kernel/driver لكنها **لا تظهر** في `platform_profile_choices` المقروءة من userspace. هذا يعني:
- قراءة `/sys/.../platform_profile_choices` لا تُظهرها
- لكن `profile_set` تقبلها إذا طُلبت برمجياً من kernel

```c
static int stm32_probe_visible(void *drvdata, unsigned long *choices)
{
    set_bit(PLATFORM_PROFILE_LOW_POWER, choices);
    set_bit(PLATFORM_PROFILE_BALANCED,  choices);
    return 0;
}

static int stm32_hidden_choices(void *drvdata, unsigned long *choices)
{
    /* CUSTOM للمعايرة فقط — غير مرئي */
    set_bit(PLATFORM_PROFILE_CUSTOM, choices);
    return 0;
}

static const struct platform_profile_ops stm32_medical_ops = {
    .probe          = stm32_probe_visible,
    .hidden_choices = stm32_hidden_choices,
    .profile_get    = stm32_profile_get,
    .profile_set    = stm32_profile_set,
};
```

#### الحل
تفعيل وضع المعايرة من kernel code مباشرةً (مثلاً عبر ioctl خاص أو debugfs):

```c
/* في calibration module */
static void enter_calibration_mode(struct device *profile_dev)
{
    /* يُفعِّل CUSTOM مباشرةً عبر profile_set callback */
    struct platform_profile_ops *ops = dev_get_drvdata(profile_dev);
    ops->profile_set(profile_dev, PLATFORM_PROFILE_CUSTOM);

    /* إشعار userspace بالتغيير */
    platform_profile_notify(profile_dev);
}
```

```bash
# المستخدم العادي لا يرى custom
cat /sys/class/platform-profile/stm32-pm/platform_profile_choices
# low-power balanced

# لكن الـ profile الفعلي قد يكون custom
cat /sys/class/platform-profile/stm32-pm/platform_profile
# custom
```

#### الدرس المستفاد
**الـ** `hidden_choices` مصمم للـ profiles التي يتحكم فيها الـ driver داخلياً أو عبر واجهات مميزة. `platform_profile_notify()` ضرورية بعد أي تغيير programmatic حتى تحصل userspace applications على `uevent` صحيح.

---

### السيناريو 4: i.MX8 في Automotive ECU — الـ profile لا يتزامن مع الواقع

#### العنوان
**ECU** على **i.MX8** يُبلِّغ عن `performance` في sysfs لكن المعالج فعلياً في وضع توفير الطاقة

#### السياق
شركة تبني **automotive ECU** على **i.MX8MP** لتحليل الفيديو من كاميرات الرؤية الليلية. الـ driver يسجّل `platform_profile` وعند تعيين `performance`، يُفترض رفع تردد المعالج. لكن المهندسون يلاحظون أن المعالج يبقى على تردد منخفض رغم أن sysfs يعرض `performance`.

#### المشكلة
```bash
cat /sys/class/platform-profile/imx8-ecu/platform_profile
# performance   <-- ما يقوله sysfs

cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 800000        <-- 800 MHz فقط، ليس 1.8 GHz المتوقع
```

#### التحليل
**الـ** `profile_get` callback يُعيد الـ profile **المُخزَّن** في ذاكرة الـ driver، لا الحالة **الفعلية** للـ hardware:

```c
struct imx8_pm_data {
    enum platform_profile_option current_profile;  /* مُخزَّن في RAM */
    /* ... */
};

static int imx8_profile_get(struct device *dev,
                             enum platform_profile_option *profile)
{
    struct imx8_pm_data *data = dev_get_drvdata(dev);
    /* BUG: يقرأ من cache، لا من hardware */
    *profile = data->current_profile;
    return 0;
}

static int imx8_profile_set(struct device *dev,
                             enum platform_profile_option profile)
{
    struct imx8_pm_data *data = dev_get_drvdata(dev);
    data->current_profile = profile;

    /* BUG: نسي استدعاء الـ cpufreq policy */
    /* imx8_apply_cpufreq_policy(profile); -- مفقود */
    return 0;
}
```

**الـ** `profile_get` يعيد `current_profile` الذي يُحدَّث في `profile_set`، لكن `profile_set` نفسها لا تُطبِّق أي تغيير على الـ hardware فعلياً.

#### الحل
```c
static int imx8_profile_set(struct device *dev,
                             enum platform_profile_option profile)
{
    struct imx8_pm_data *data = dev_get_drvdata(dev);
    int ret;

    /* تطبيق الـ profile على الـ hardware أولاً */
    ret = imx8_apply_cpufreq_policy(data, profile);
    if (ret)
        return ret;

    /* ثم تحديث الـ cache */
    data->current_profile = profile;

    /* إشعار userspace */
    platform_profile_notify(dev);
    return 0;
}

static int imx8_profile_get(struct device *dev,
                             enum platform_profile_option *profile)
{
    struct imx8_pm_data *data = dev_get_drvdata(dev);

    /* قراءة الحالة الفعلية من hardware */
    *profile = imx8_read_hw_profile(data);
    return 0;
}
```

```bash
# التحقق بعد الإصلاح
echo "performance" > /sys/class/platform-profile/imx8-ecu/platform_profile
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 1800000   -- 1.8 GHz صحيح
```

#### الدرس المستفاد
**الـ** `profile_get` يجب أن يعكس **حالة الـ hardware الفعلية** وليس مجرد ما طُلِب آخر مرة. `platform_profile_notify()` تُرسل `uevent` لـ userspace — يجب استدعاؤها فقط بعد التأكد من تطبيق التغيير فعلاً على الـ hardware.

---

### السيناريو 5: RK3562 في Tablet — استخدام platform_profile_cycle في Hardware Button

#### العنوان
زر فيزيائي على لوحة **RK3562** يُدوِّر بين **platform profiles** عبر `platform_profile_cycle()`

#### السياق
شركة تبني **industrial tablet** على **RK3562** مزود بزر خاص "Performance Mode" على اللوحة الأم. عند الضغط على الزر، يُفترض التنقل الدوري بين `low-power → balanced → performance → low-power`. الفريق يقرر استخدام `platform_profile_cycle()` الموجود في الـ API.

#### المشكلة
عند الضغط على الزر، تعمل الدورة بشكل عشوائي — أحياناً تتخطى بعض الـ profiles، وأحياناً لا تتحرك أصلاً.

```bash
# أول ضغطة
cat /sys/class/platform-profile/rk3562-pm/platform_profile
# performance   <-- صحيح

# ضغطة ثانية
cat /sys/class/platform-profile/rk3562-pm/platform_profile
# performance   <-- لم تتغير!
```

#### التحليل
**الـ** `platform_profile_cycle()` تُدوِّر بين **كل الـ profiles المُسجَّلة عبر probe** في النظام. المشكلة أن `platform_profile_cycle()` تعمل على **كل devices** المُسجَّلة، لا على device واحد محدد. إذا كان هناك أكثر من device مُسجَّل، قد تُدوِّر على غير المتوقع.

```c
/* في driver الزر */
static irqreturn_t rk3562_btn_irq(int irq, void *data)
{
    /* platform_profile_cycle() تُدوِّر على كل devices */
    platform_profile_cycle();
    return IRQ_HANDLED;
}
```

المشكلة الأعمق: الـ driver لم يُفعِّل `PLATFORM_PROFILE_BALANCED` في `probe`:

```c
static int rk3562_probe_choices(void *drvdata, unsigned long *choices)
{
    set_bit(PLATFORM_PROFILE_LOW_POWER,   choices);
    /* نسي BALANCED */
    set_bit(PLATFORM_PROFILE_PERFORMANCE, choices);
    return 0;
}
```

**الـ** `platform_profile_cycle()` تتحقق من الـ bitmap وتتخطى الـ profiles غير المُعلَنة، فتنتقل مباشرةً من `low-power` إلى `performance` وتتخطى `balanced`. في بعض الحالات إذا كان الـ current profile غير موجود في الـ choices (مثلاً بعد resume من sleep)، لا تتحرك.

```c
/* للتشخيص: فحص choices الفعلية */
static ssize_t show_choices(struct device *dev, ...)
{
    /* قراءة platform_profile_choices */
}
```

```bash
cat /sys/class/platform-profile/rk3562-pm/platform_profile_choices
# low-power performance   <-- balanced مفقود
```

#### الحل
```c
static int rk3562_probe_choices(void *drvdata, unsigned long *choices)
{
    /* تفعيل كل الـ profiles المطلوبة بالترتيب */
    set_bit(PLATFORM_PROFILE_LOW_POWER,             choices);
    set_bit(PLATFORM_PROFILE_BALANCED,              choices);
    set_bit(PLATFORM_PROFILE_PERFORMANCE,           choices);
    return 0;
}
```

وإذا كان التحكم مطلوباً على device محدد بدلاً من `platform_profile_cycle()`:

```c
static irqreturn_t rk3562_btn_irq(int irq, void *data)
{
    struct rk3562_pm_data *pmdata = data;
    enum platform_profile_option current, next;

    /* قراءة الـ profile الحالي */
    pmdata->ops->profile_get(pmdata->profile_dev, &current);

    /* حساب التالي يدوياً */
    switch (current) {
    case PLATFORM_PROFILE_LOW_POWER:
        next = PLATFORM_PROFILE_BALANCED;
        break;
    case PLATFORM_PROFILE_BALANCED:
        next = PLATFORM_PROFILE_PERFORMANCE;
        break;
    default:
        next = PLATFORM_PROFILE_LOW_POWER;
        break;
    }

    /* تطبيق الـ profile الجديد */
    pmdata->ops->profile_set(pmdata->profile_dev, next);
    platform_profile_notify(pmdata->profile_dev);

    return IRQ_HANDLED;
}
```

```bash
# بعد الإصلاح
cat /sys/class/platform-profile/rk3562-pm/platform_profile_choices
# low-power balanced performance

# الدورة صحيحة
# ضغطة 1: low-power → balanced
# ضغطة 2: balanced → performance
# ضغطة 3: performance → low-power
```

#### الدرس المستفاد
**الـ** `platform_profile_cycle()` دالة مساعدة تعمل على مستوى النظام الكامل وتعتمد على الـ `choices` bitmap المُعلَن. لتحكم دقيق في sequence التنقل أو عند استهداف device واحد محدد، يُفضَّل تنفيذ منطق cycle يدوياً مع استدعاء `platform_profile_notify()` لضمان وصول الـ uevent إلى userspace (مثل `power-profiles-daemon`).
## Phase 7: مصادر ومراجع

### توثيق النواة الرسمي

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| **sysfs-platform_profile** | [docs.kernel.org](https://docs.kernel.org/userspace-api/sysfs-platform_profile.html) | التوثيق الرسمي لـ ABI الخاص بـ `platform_profile` في sysfs |
| **Platform Devices and Drivers** | [docs.kernel.org](https://docs.kernel.org/driver-api/driver-model/platform.html) | الدليل الرسمي لـ platform device model |
| **ACPI Support** | [kernel.org](https://www.kernel.org/doc/html/v6.12/admin-guide/acpi/index.html) | توثيق دعم ACPI في Linux |
| **ABI Testing Doc** | [kernel.org](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-platform_profile) | توثيق ABI الاختباري لـ sysfs-platform_profile |
| **CONFIG_ACPI_PLATFORM_PROFILE** | [kernelconfig.io](https://www.kernelconfig.io/config_acpi_platform_profile) | خيار Kconfig الخاص بتفعيل الدعم |

### مسارات التوثيق داخل شجرة المصدر

```
Documentation/userspace-api/sysfs-platform_profile.rst   ← الوثيقة الرئيسية
Documentation/ABI/testing/sysfs-platform_profile         ← ABI وصف الواجهة
drivers/acpi/platform_profile.c                           ← التنفيذ الفعلي
include/linux/platform_profile.h                          ← الترويسة العامة
```

---

### مقالات LWN.net

| المقال | الرابط |
|--------|--------|
| **Platform Profile Selection — kernel doc** | [static.lwn.net](https://static.lwn.net/kerneldoc/userspace-api/sysfs-platform_profile.html) |
| **The platform device API** | [lwn.net/Articles/448499](https://lwn.net/Articles/448499/) |
| **Platform devices and device trees** | [lwn.net/Articles/448502](https://lwn.net/Articles/448502/) |
| **inspur-wmi: Add platform profile support** | [lwn.net/Articles/948106](https://lwn.net/Articles/948106/) |
| **Profile-guided optimization for the kernel** | [lwn.net/Articles/830300](https://lwn.net/Articles/830300/) |

> ملاحظة: لا تتناول مقالات LWN الأربع الأولى `platform_profile` بشكل مباشر، لكنها توضح بنية `platform device` التي يعتمد عليها هذا النظام الفرعي.

---

### Commits مهمة في تاريخ platform_profile

| الحدث | الإصدار | الرابط |
|-------|---------|--------|
| **إدخال ACPI Platform Profile لأول مرة** | Linux 5.12 | [Phoronix](https://www.phoronix.com/news/Linux-ACPI-Platform-Profile) |
| **إضافة `balanced-performance`** | Linux 5.12-rc1 | ضمن شجرة linux-pm |
| **دعم Inspur عبر inspur-wmi** | Linux 6.7 | [kernelnewbies.org/Linux_6.7](https://kernelnewbies.org/Linux_6.7) |
| **تنفيذ Dell platform_profile** | Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| **Dell AWCC platform_profile** | Linux 6.13 | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) |
| **إضافة مفهوم `custom` profile** | مراجعة نشطة | [spinics.net patch v8](https://www.spinics.net/lists/kernel/msg5463406.html) |
| **دعم معالجات متعددة (multiple handlers)** | Linux 6.x+ | [spinics.net patch v5](https://www.spinics.net/lists/linux-acpi/msg128703.html) |

---

### نقاشات القائمة البريدية

الـ **mailing list** الرئيسية المتعلقة بـ `platform_profile`:

- **linux-acpi@vger.kernel.org** — للتغييرات الجوهرية في ACPI subsystem
- **platform-driver-x86@vger.kernel.org** — لدريفرات x86 المستخدِمة للـ profile

أبرز نقاشات Spinics/LKML:

| الموضوع | الرابط |
|---------|--------|
| **PATCH v8: Add "custom" profile** | [spinics.net](https://www.spinics.net/lists/kernel/msg5463406.html) |
| **PATCH v5: Multiple drivers support** | [spinics.net](https://www.spinics.net/lists/linux-acpi/msg128703.html) |
| **PATCH v10: Add name member to handlers** | [spinics.net](https://www.spinics.net/lists/platform-driver-x86/msg49147.html) |
| **Register class device for handlers** | [lkml.rescloud.iu.edu](https://lkml.rescloud.iu.edu/2411.0/01937.html) |

---

### kernelnewbies.org — ملاحظات الإصدارات ذات الصلة

| الإصدار | الرابط | ما يخص platform_profile |
|---------|--------|--------------------------|
| Linux 6.7 | [kernelnewbies.org/Linux_6.7](https://kernelnewbies.org/Linux_6.7) | إضافة `inspur-wmi` platform profile |
| Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) | Dell تنفّذ `platform_profile` |
| Linux 6.13 | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) | Dell AWCC platform_profile |
| Linux 6.12 | [kernelnewbies.org/Linux_6.12](https://kernelnewbies.org/Linux_6.12) | تحسينات عامة على ACPI |

> لا تتوفر صفحة مخصصة لـ `platform_profile` على **elinux.org** حالياً — النتائج المتعلقة بـ "platform" تشير إلى منصات LeapFrog المدمجة فقط.

---

### مصادر إضافية

| المصدر | الرابط |
|--------|--------|
| **LKDDB: CONFIG_ACPI_PLATFORM_PROFILE** | [cateee.net](https://cateee.net/lkddb/web-lkddb/ACPI_PLATFORM_PROFILE.html) |
| **Patchew — custom profile series** | [patchew.org](https://patchew.org/linux/20240926025955.1728766-1-superm1@kernel.org/) |
| **Google Git — platform_profile.h** | [kernel.googlesource.com](https://kernel.googlesource.com/pub/scm/linux/kernel/git/rdma/rdma/+/for-rc/include/linux/platform_profile.h) |
| **ArchWiki: ACPI modules** | [wiki.archlinux.org](https://wiki.archlinux.org/title/ACPI_modules) |
| **Fedora: platform_profile auto-reset** | [discussion.fedoraproject.org](https://discussion.fedoraproject.org/t/platform-profile-changes-back-to-balanced-automatically/75844) |

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
> متاح مجاناً على [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

- **الفصل 14** — The Linux Device Model: فهم `struct device`، `class`، و `kobject`
- **الفصل 3** — Char Drivers: أساس كيفية تعامل sysfs مع الواجهات
- **الفصل 13** — USB Drivers: نمط callbacks مشابه لـ `platform_profile_ops`

#### Linux Kernel Development — Robert Love (3rd Ed.)
- **الفصل 17** — Devices and Modules: شرح platform bus وعلاقته بـ sysfs
- **الفصل 18** — Debugging: أدوات مفيدة لاختبار sysfs nodes

#### Embedded Linux Primer — Christopher Hallinan (2nd Ed.)
- **الفصل 15** — Debugging Embedded Linux: مراقبة thermal/power profiles
- **الفصل 16** — Kernel Debugging Techniques: قراءة sysfs في بيئات مدمجة

---

### مصطلحات البحث الموصى بها

```
platform_profile linux kernel
ACPI_PLATFORM_PROFILE sysfs
/sys/firmware/acpi/platform_profile
platform_profile_ops probe profile_get profile_set
devm_platform_profile_register
platform_profile_notify
linux power profile thermal management sysfs
platform_profile multiple handlers class device
linux laptop performance profile balanced low-power
```

---

### ملخص سريع للمصادر حسب الأولوية

```
1. docs.kernel.org/userspace-api/sysfs-platform_profile.html  ← ابدأ هنا
2. drivers/acpi/platform_profile.c                             ← تنفيذ كامل
3. spinics.net (patch series v8/v10)                           ← أحدث التغييرات
4. kernelnewbies.org/Linux_6.7 / 6.11 / 6.13                  ← متابعة الإضافات
5. LDD3 Chapter 14 + Robert Love Chapter 17                    ← الأساس النظري
```
## Phase 8: Writing simple module

### الهدف

سنستخدم **kprobe** لاعتراض الدالة `platform_profile_notify` — وهي الدالة التي تُرسل إشعاراً لـ sysfs في كل مرة يتغير فيها **platform profile** (مثل التبديل بين وضع Performance ووضع Low Power). هذا الاعتراض آمن تماماً لأنه للقراءة فقط ولا يُعدّل أي بيانات.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe hook on platform_profile_notify()
 *
 * Intercepts every platform profile change notification
 * and logs the device pointer that triggered it.
 */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/device.h>
#include <linux/platform_profile.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe on platform_profile_notify — log profile change events");

/* ------------------------------------------------------------------
 * pre_handler: runs just before platform_profile_notify() executes.
 *
 * platform_profile_notify(struct device *dev)
 *   arg0 (regs->di on x86_64) = pointer to the profile class device
 * ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* On x86_64 the first argument is in RDI register */
    struct device *dev = (struct device *)regs->di;

    if (dev)
        pr_info("platform_profile: notify fired — device: %s\n",
                dev_name(dev));
    else
        pr_info("platform_profile: notify fired — device ptr is NULL\n");

    return 0; /* 0 = let the original function continue normally */
}

/* ------------------------------------------------------------------
 * post_handler: runs immediately after platform_profile_notify()
 * returns. Useful to confirm the sysfs notification completed.
 * ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    pr_info("platform_profile: notify completed\n");
}

/* ------------------------------------------------------------------
 * kprobe descriptor — we name the symbol we want to hook
 * ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "platform_profile_notify",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------
 * module_init: register the kprobe
 * ------------------------------------------------------------------ */
static int __init pp_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("platform_profile kprobe: register failed (%d)\n", ret);
        return ret;
    }

    pr_info("platform_profile kprobe: planted at %p\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------
 * module_exit: MUST unregister before the module is unloaded,
 * otherwise the kernel will call a handler that no longer exists
 * and cause a panic (use-after-free on code pages).
 * ------------------------------------------------------------------ */
static void __exit pp_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("platform_profile kprobe: removed\n");
}

module_init(pp_probe_init);
module_exit(pp_probe_exit);
```

---

### شرح كل قسم

#### الـ includes

```c
#include <linux/kprobes.h>
#include <linux/platform_profile.h>
```

**الـ** `kprobes.h` يوفر `struct kprobe` ودوال التسجيل، بينما `platform_profile.h` يُعرّف `enum platform_profile_option` و`struct device` الخاص بالـ profile — نحتاجه لطباعة اسم الجهاز بدقة.

---

#### الـ `handler_pre`

```c
struct device *dev = (struct device *)regs->di;
```

**الـ** `pt_regs` يحمل حالة السجلات لحظة الاعتراض؛ على x86_64 المعامل الأول دائماً في `rdi`، لذا نستخرج مؤشر `struct device` مباشرةً منه دون الحاجة لتعديل أي كود في الـ kernel.

---

#### الـ `handler_post`

يُنفَّذ بعد عودة `platform_profile_notify`، ويُثبت في الـ log أن إشعار sysfs اكتمل بنجاح — مفيد للـ debugging عند الاشتباه في أن الإشعار يُطلق لكنه لا يصل.

---

#### الـ `kprobe` descriptor

```c
static struct kprobe kp = {
    .symbol_name = "platform_profile_notify",
    ...
};
```

نُحدد الدالة بالاسم الرمزي فقط؛ الـ kernel يحلّ العنوان تلقائياً عبر `kallsyms` وقت التسجيل، مما يجعل الكود مستقلاً عن إصدار الـ kernel.

---

#### الـ `module_init` / `module_exit`

`register_kprobe` تزرع **breakpoint** (أو `ftrace` tramopline) عند عنوان الدالة. `unregister_kprobe` في `exit` ضرورية حتماً: إذا أُزيل المودول دون إلغاء التسجيل، سيظل الـ kernel يقفز إلى كود handler المحذوف وسيحدث **kernel panic**.

---

### Makefile للبناء

```makefile
obj-m += pp_notify_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل المودول

```bash
# بناء
make

# تحميل
sudo insmod pp_notify_probe.ko

# تغيير الـ profile لإطلاق platform_profile_notify
echo "performance" | sudo tee \
    /sys/firmware/acpi/platform_profile

# مشاهدة الـ log
sudo dmesg | grep platform_profile

# إزالة المودول
sudo rmmod pp_notify_probe
```

---

### مثال مخرجات `dmesg`

```
[  142.331201] platform_profile kprobe: planted at ffffffffc08a1f30
[  145.882014] platform_profile: notify fired — device: platform_profile:00
[  145.882021] platform_profile: notify completed
[  151.004512] platform_profile kprobe: removed
```

---

### ملاحظات أمان

| النقطة | التفصيل |
|--------|---------|
| القراءة فقط | الـ handler لا يُعدّل `regs` ولا بيانات الـ profile |
| `return 0` | يعني "تابع تنفيذ الدالة الأصلية" — لا تعطيل لأي وظيفة |
| `CONFIG_KPROBES` | يجب أن يكون مُفعَّلاً في الـ kernel config |
| `kallsyms_lookup` | إذا كانت الدالة `inline` لن يجد الـ kprobe رمزها — لكن `platform_profile_notify` معرَّفة في `platform_profile.c` وقابلة للاعتراض |
