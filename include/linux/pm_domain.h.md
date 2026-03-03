## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ Generic PM Domains (genpd)** — جزء من subsystem الـ Power Management في Linux kernel. الـ maintainer هو Ulf Hansson، والكود موجود في `drivers/pmdomain/` والـ header الرئيسي هو `include/linux/pm_domain.h`.

---

### المشكلة اللي بيحلها الملف ده

تخيل إنك بتبني موبايل. جوا الـ SoC (مثلاً Qualcomm Snapdragon أو Samsung Exynos) فيه عشرات الـ hardware blocks: GPU، ISP (كاميرا)، DSP، USB، Display engine، إلخ. كل block ده بياكل طاقة حتى لو مش بيشتغل.

الحل: بنقسم الـ SoC لـ **power domains** — كل domain عبارة عن "حي كهربي" فيه مجموعة devices. لما مفيش device شغال في الحي ده، بنطفي الكهرباء عنه بالكامل. لما device محتاج يشتغل، بنوقع الكهرباء على الحي كله الأول.

**الـ `pm_domain.h`** هو العقد (contract) اللي بيحدد إزاي الـ kernel بيتكلم مع الـ power domains دي.

---

### القصة كاملة: من البداية للنهاية

#### المشهد الأول: SoC بدون genpd

```
+------------------SoC------------------+
| GPU | ISP | DSP | USB | DISPLAY | CPU |
|  ON |  ON |  ON |  ON |    ON   |  ON |  ← كل حاجة ON طول الوقت = بطارية تخس
+----------------------------------------+
```

كل driver بيعمل power management بطريقته الخاصة — chaos.

#### المشهد الثاني: SoC مع genpd

```
+--SoC--+
|       |
| [PD-Media]---+--- ISP (camera driver)
|              +--- Video Encoder
|              └--- Display Engine
|
| [PD-Compute]-+--- GPU
|              └--- DSP
|
| [PD-Connectivity]-- USB
|                 +-- WiFi
+-------+
```

لما الكاميرا مش بتشتغل + الـ Video + الـ Display كلهم idle → الـ kernel يطفي `PD-Media` بالكامل ويوفر طاقة.

لما driver الكاميرا قال `pm_runtime_get()` → الـ genpd يشغل `PD-Media` الأول → بعدين يشغل الـ device.

---

### الـ genpd: الـ Generic Power Domain

**الـ `generic_pm_domain`** هو القلب — struct بيمثل domain واحد:

```c
struct generic_pm_domain {
    struct device dev;              /* domain نفسه كـ device في sysfs */
    struct dev_pm_domain domain;    /* PM callbacks (suspend/resume/...) */
    struct list_head parent_links;  /* روابط للـ parent domains */
    struct list_head child_links;   /* روابط للـ sub-domains */
    struct list_head dev_list;      /* الـ devices المنضمة للـ domain */
    struct dev_power_governor *gov; /* مين يقرر امتى نطفي؟ */
    enum gpd_status status;         /* ON أو OFF */
    int (*power_off)(struct generic_pm_domain *domain); /* callback لطفي الطاقة */
    int (*power_on)(struct generic_pm_domain *domain);  /* callback لتشغيل الطاقة */
    struct genpd_power_state *states; /* قائمة حالات idle المختلفة */
    unsigned int flags;             /* GENPD_FLAG_* */
    /* ... */
};
```

---

### الـ Hierarchy: Domains داخل Domains

الـ genpd بيدعم شجرة من الـ domains — domain ممكن يكون subdomain لـ domain أكبر:

```
[PD-TOP]  ← لازم يكون ON عشان أي حاجة تشتغل
   |
   +--[PD-Media]
   |      +-- ISP
   |      +-- Display
   |
   +--[PD-Compute]
          +-- GPU
```

لو الـ GPU طلب power → الـ kernel يشغل `PD-TOP` الأول، بعدين `PD-Compute`، بعدين الـ GPU. الإطفاء بالعكس — **last-man-standing algorithm**.

الـ struct اللي بيمثل الرابط بين domain وـ subdomain:

```c
struct gpd_link {
    struct generic_pm_domain *parent;
    struct generic_pm_domain *child;
    unsigned int performance_state;      /* الـ perf المطلوب من الـ parent */
    unsigned int prev_performance_state;
};
```

---

### الـ Governor: مين يقرر امتى نطفي؟

مش كل domain ممكن يتطفى في أي وقت. الـ **`dev_power_governor`** بيقرر:

```c
struct dev_power_governor {
    bool (*system_power_down_ok)(struct dev_pm_domain *domain); /* عند system suspend */
    bool (*power_down_ok)(struct dev_pm_domain *domain);        /* عند runtime PM */
    bool (*suspend_ok)(struct device *dev);                     /* لكل device على حدة */
};
```

فيه 3 governors جاهزة:
- **`simple_qos_governor`** — بيراعي الـ QoS latency constraints
- **`pm_domain_always_on_gov`** — domain دايماً ON
- **`pm_domain_cpu_gov`** — مخصص للـ CPU domains (idle states)

---

### الـ Performance States

الـ genpd مش بس ON/OFF — ممكن يدعم **performance states** (زي الـ voltage/frequency scaling على مستوى domain):

```c
struct genpd_power_state {
    const char *name;
    s64 power_off_latency_ns;  /* وقت الإطفاء */
    s64 power_on_latency_ns;   /* وقت التشغيل */
    s64 residency_ns;          /* الحد الأدنى للـ idle عشان يستحق الإطفاء */
    u64 usage;                 /* إحصائية: كام مرة اتخدمت؟ */
    u64 rejected;              /* إحصائية: كام مرة اترفض؟ */
};
```

الـ governor بيستخدم `residency_ns` يحسب: هل يستحق نطفي الـ domain لو هيتشغل تاني بعد 100 microsecond؟ لو الـ power-on latency أكبر من الوقت المتاح، الـ governor يقول "خليه ON".

---

### الـ Device Attachment

لما driver بيعمل probe، بيحتاج ينضم لـ domain بتاعه. الـ flow:

```
Device Tree / ACPI / Firmware
        |
        | "هذا الـ device ينتمي لـ pd-media"
        ↓
  genpd_dev_pm_attach(dev)
        |
        ↓
  dev_pm_domain_attach(dev, flags)
        |
        ↓
  يضيف الـ device لـ struct generic_pm_domain.dev_list
  وينشئ device_link بين الـ device والـ domain
```

الـ **`dev_pm_domain_attach_data`** بيسمح لـ device ينضم لأكتر من domain في نفس الوقت (multi-domain devices):

```c
struct dev_pm_domain_attach_data {
    const char * const *pd_names; /* أسماء الـ domains */
    const u32 num_pd_names;
    const u32 pd_flags;           /* PD_FLAG_* */
};
```

---

### الـ Flags المهمة

#### GENPD_FLAG_*  (للـ domain نفسه)

| Flag | المعنى |
|------|---------|
| `GENPD_FLAG_IRQ_SAFE` | الـ power_on/off callbacks مش بتنام — ممكن تتنفذ في atomic context |
| `GENPD_FLAG_ALWAYS_ON` | Domain دايماً ON — مش بيتطفى أبداً |
| `GENPD_FLAG_CPU_DOMAIN` | Domain بيحتوي CPUs — يفعّل CPU idle management |
| `GENPD_FLAG_MIN_RESIDENCY` | فعّل الـ governor يحسب next wakeup قبل ما يطفي |
| `GENPD_FLAG_OPP_TABLE_FW` | الـ OPP tables جاية من الـ firmware مش من DT |

#### PD_FLAG_* (عند attach device لـ domain)

| Flag | المعنى |
|------|---------|
| `PD_FLAG_NO_DEV_LINK` | ماتعملش device_link تلقائي |
| `PD_FLAG_DEV_LINK_ON` | شغّل الـ domain فوراً عند الـ attach |
| `PD_FLAG_ATTACH_POWER_ON` | Power on the domain during attach |

---

### الـ OF (Device Tree) Integration

الـ DT بيعرّف الـ domains ومين ينتمي لمين:

```
// مثال DT مبسط (Qualcomm)
power-domains {
    pd_gpu: gpu-domain {
        #power-domain-cells = <0>;
    };
};

gpu@... {
    power-domains = <&pd_gpu>;
};
```

الـ `genpd_onecell_data` بيسمح لـ provider driver يسجّل مجموعة domains دفعة واحدة:

```c
struct genpd_onecell_data {
    struct generic_pm_domain **domains; /* array of domains */
    unsigned int num_domains;
    genpd_xlate_t xlate; /* translator: phandle args → domain pointer */
};
```

---

### الـ Locking Strategy

مشكلة: الـ domain ممكن يتشغل/يتطفى من:
- Runtime PM context (sleepable)
- IRQ handler (atomic, can't sleep)
- Tasklet

الحل: union من 3 أنواع locks في `generic_pm_domain`:

```c
union {
    struct mutex mlock;           /* للـ domains العادية */
    struct {
        spinlock_t slock;         /* لـ GENPD_FLAG_IRQ_SAFE */
        unsigned long lock_flags;
    };
    struct {
        raw_spinlock_t raw_slock; /* للـ RT kernel */
        unsigned long raw_lock_flags;
    };
};
```

---

### الملفات المرتبطة

#### الـ Core

| الملف | الدور |
|-------|-------|
| `include/linux/pm_domain.h` | **الملف الحالي** — كل الـ structs والـ APIs |
| `drivers/pmdomain/core.c` | التنفيذ الكامل لكل الـ APIs المعلنة هنا |
| `drivers/pmdomain/governor.c` | تنفيذ الـ `simple_qos_governor` |
| `include/linux/pm.h` | يعرّف `struct dev_pm_domain` و `struct dev_pm_ops` |

#### الـ Hardware Drivers (أمثلة)

| الملف | الـ SoC |
|-------|---------|
| `drivers/pmdomain/qcom/rpmhpd.c` | Qualcomm — RPMh power domains |
| `drivers/pmdomain/imx/gpcv2.c` | NXP i.MX8 — GPC v2 |
| `drivers/pmdomain/mediatek/mtk-pm-domains.c` | MediaTek |
| `drivers/pmdomain/renesas/rcar-sysc.c` | Renesas R-Car |
| `drivers/pmdomain/bcm/raspberrypi-power.c` | Raspberry Pi |
| `drivers/pmdomain/arm/scmi_pm_domain.c` | ARM SCMI (firmware-based) |

#### الـ DT Bindings

| الملف |
|-------|
| `Documentation/devicetree/bindings/power/power-domain.yaml` |

---

### الخلاصة

**الـ `pm_domain.h`** هو الـ blueprint اللي بيقول:
1. إيه شكل الـ power domain (`generic_pm_domain`)
2. إزاي الـ device بينضم/ببتفصل من domain (`dev_pm_domain_attach/detach`)
3. إزاي الـ governor بيقرر امتى نطفي (`dev_power_governor`)
4. إزاي نربط كل ده بالـ Device Tree (`of_genpd_*`)

النتيجة: battery life أطول لأن hardware blocks بتتطفى أوتوماتيكياً لما مفيش شغل — من غير ما كل driver يعمل power management من الصفر.
## Phase 2: شرح الـ Generic Power Domain (genpd) Framework

---

### المشكلة: ليه الـ framework ده موجود أصلاً؟

في أي SoC حديث — خد مثلاً Qualcomm SM8550 أو TI AM62x أو Rockchip RK3588 — الـ hardware مش مجموعة devices مستقلة كل واحدة بتتحكم في power بتاعها لوحدها. الواقع أكثر تعقيدًا:

- الـ GPU و الـ display engine و الـ video decoder ممكن يكونوا داخل نفس **power island** واحدة في الـ PMIC.
- لو حبيت توقف الـ GPU، لازم أول تتأكد إن الـ display engine وقف كمان، عشان مش هيبقى فيه حاجة بتستخدم نفس الـ power rail.
- في نفس الوقت الـ CPU cluster ممكن يكون في power domain منفصل تمامًا.

**المشكلة بالتفصيل:**

كل driver كان بيتعامل مع power management بتاعه بشكل مستقل — كل واحد بيشيل كود خاص بيه يتكلم مع الـ PMIC أو PMSCU أو Power Controller. النتيجة:
1. **تكرار كود** ضخم.
2. **race conditions**: لو دريفرين حاولوا يغيروا نفس الـ power domain في نفس الوقت.
3. **مفيش tracking**: الكيرنل مش عارف مين شايل الـ domain صاحي.
4. **مفيش hierarchy**: لو domain A جوا domain B، مين المسؤول عن ترتيب إيقاف التشغيل؟

---

### الحل: Generic Power Domain (genpd)

الكيرنل حل المشكلة دي بـ **framework مركزي** اسمه `genpd` — اختصار لـ Generic PM Domain.

الفكرة الجوهرية:
- كل مجموعة hardware بتشترك في نفس الـ power supply تتمثل كـ **`struct generic_pm_domain`** واحدة.
- الـ framework يحتفظ بـ **reference count** للأجهزة اللي شايلة الـ domain صاحي.
- لما الـ count يوصل لصفر، الـ framework هو اللي يقرر إمتى يقفل الـ domain — مش الـ driver.
- الـ domains ممكن تتوضع في **شجرة هرمية**: parent domain و child domains.

---

### الـ Real-World Analogy: مبنى إداري بالطوابير

تخيل مبنى إداري فيه:
- **طابق كامل** = Power Domain.
- **الأوضة على الطابق** = الـ devices داخل الـ domain.
- **بريكر الكهرباء للطابق** = الـ `power_on/power_off` callbacks.
- **السكيوريتي على باب الطابق** = الـ genpd core (اللي بيحتفظ بالـ reference count).
- **المبنى الكامل** = parent domain.

القاعدة: السكيوريتي مش بيطفي النور في الطابق غير لما **آخر واحد** يخرج. لو حد لسه جوا — ولو أوضته اتقفلت — الطابق لازم يفضل مضيء. لما المبنى كمان بيتقفل، لازم أول ينتهي من كل الطوابير قبل إنه يقفل بريكر المبنى.

**الـ mapping الكامل للـ analogy:**

| الـ Analogy | الكود الحقيقي |
|---|---|
| الطابق | `struct generic_pm_domain` |
| الأوضة | `struct device` مرتبطة بالـ genpd |
| السكيوريتي | genpd core — `pm_genpd_runtime_suspend/resume` |
| بريكر الطابق | `genpd->power_off()` / `genpd->power_on()` |
| قائمة المستأجرين | `genpd->dev_list` + `genpd->device_count` |
| طابق جوا مبنى | child domain → parent domain عبر `gpd_link` |
| مدير المبنى | `dev_power_governor` — بيقرر هل تقفل التشغيل ولا لأ |
| ساعة الدوام | `genpd->on_time` و accounting |

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Linux Kernel                             │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  Device      │    │  Runtime PM  │    │  System Sleep    │  │
│  │  Drivers     │    │  (RPM) Core  │    │  (suspend/resume)│  │
│  │  (GPU, ISP,  │    │              │    │                  │  │
│  │   DSP, ...)  │    └──────┬───────┘    └────────┬─────────┘  │
│  └──────┬───────┘           │                     │            │
│         │ attach             │                     │            │
│         ▼                   ▼                     ▼            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │               genpd Core (pm_domain.c)                   │  │
│  │                                                          │  │
│  │  ┌─────────────────┐   ┌──────────────────────────────┐  │  │
│  │  │ Reference Count │   │  Governor (dev_power_governor)│  │  │
│  │  │ Tracking        │   │  - simple_qos_governor        │  │  │
│  │  │ (sd_count,      │   │  - pm_domain_always_on_gov    │  │  │
│  │  │  device_count)  │   │  - pm_domain_cpu_gov          │  │  │
│  │  └─────────────────┘   └──────────────────────────────┘  │  │
│  │                                                          │  │
│  │  ┌──────────────────────────────────────────────────┐    │  │
│  │  │   Domain Hierarchy (gpd_link tree)               │    │  │
│  │  │                                                  │    │  │
│  │  │   [Parent Domain]──────────────────────────┐    │    │  │
│  │  │        │ parent_links                       │    │    │  │
│  │  │        ▼                                   ▼    │    │  │
│  │  │   [Child Domain A]              [Child Domain B] │    │  │
│  │  │      │ dev_list                    │ dev_list    │    │  │
│  │  │      ▼                            ▼             │    │  │
│  │  │   [GPU dev] [ISP dev]         [DSP dev]         │    │  │
│  │  └──────────────────────────────────────────────── ┘    │  │
│  └──────────────────────────────────────────────────────────┘  │
│         │                                                       │
│         │ power_on() / power_off() callbacks                    │
│         ▼                                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │        genpd Backend Drivers (SoC-specific)              │  │
│  │  e.g. drivers/pmdomain/qcom/   drivers/pmdomain/ti/      │  │
│  │        drivers/pmdomain/rockchip/                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │    Hardware: PMIC / PMSCU / Power Controller             │  │
│  │    (via MMIO, I2C, or firmware interface like SCMI)      │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**الـ DT integration:**

```
┌────────────────────────────────────────────────────┐
│  Device Tree                                       │
│                                                    │
│  power-domain-controller@... {                     │
│      #power-domain-cells = <1>;   ← genpd provider │
│  };                                                │
│                                                    │
│  gpu@... {                                         │
│      power-domains = <&pd GPU_DOMAIN>;  ← consumer │
│  };                                                │
└────────────────────────────────────────────────────┘
          │ of_genpd_add_provider_onecell()
          │ genpd_dev_pm_attach()
          ▼
   genpd core يربط الـ device بالـ domain تلقائيًا
```

---

### الـ Core Abstraction: التجريد الأساسي

الـ genpd بيقدم **تجريد الـ shared power resource** — بدل ما كل driver يعرف إزاي يشغل/يوقف الـ power rail اللي بيشاركه مع غيره، الـ framework بيوفر واجهة موحدة.

الـ central idea في جملة واحدة: **"كل device تقول للـ domain إنها محتاجاه أو مش محتاجاه، والـ domain هو اللي يقرر إمتى يشغل أو يوقف الـ hardware فعلاً."**

---

### شرح الـ Structs المحورية

#### 1. `struct generic_pm_domain` — قلب الـ framework

```c
struct generic_pm_domain {
    struct device dev;           /* الـ domain نفسه له device في الكيرنل */
    struct dev_pm_domain domain; /* الـ PM ops اللي بتتعامل مع الـ runtime PM */

    struct list_head parent_links; /* روابطه كـ parent لـ child domains */
    struct list_head child_links;  /* روابطه كـ child لـ parent domain */
    struct list_head dev_list;     /* الأجهزة المرتبطة بالـ domain ده */

    struct dev_power_governor *gov; /* من يقرر: هل نطفي ولا لأ؟ */
    struct genpd_governor_data *gd; /* بيانات الـ governor (latency, wakeup) */

    atomic_t sd_count;     /* عدد الـ child domains اللي صاحية */
    enum gpd_status status;/* GENPD_STATE_ON أو GENPD_STATE_OFF */

    int (*power_off)(struct generic_pm_domain *domain); /* ← SoC specific */
    int (*power_on)(struct generic_pm_domain *domain);  /* ← SoC specific */

    struct genpd_power_state *states; /* الـ idle states المتاحة (زي C-states) */
    unsigned int state_count;
    unsigned int state_idx;   /* الـ state اللي هيروح فيه لما يتقفل */

    unsigned int performance_state; /* أعلى performance state مطلوب */
    int (*set_performance_state)(struct generic_pm_domain *genpd,
                                 unsigned int state); /* ← للـ OPP integration */
    unsigned int flags; /* GENPD_FLAG_* */

    /* الـ locking: mutex عادي أو spinlock حسب GENPD_FLAG_IRQ_SAFE */
    union {
        struct mutex mlock;
        struct { spinlock_t slock; unsigned long lock_flags; };
        struct { raw_spinlock_t raw_slock; unsigned long raw_lock_flags; };
    };
};
```

**مهم:** الـ `domain` field من نوع `struct dev_pm_domain` اللي فيها الـ `dev_pm_ops`. ده اللي بيخلي الـ runtime PM core يتكلم مع الـ genpd مباشرة لما device بيعمل `pm_runtime_put()`.

```
device->power.pm_domain ──► struct dev_pm_domain
                                   │
                                   │ container_of via pd_to_genpd()
                                   ▼
                            struct generic_pm_domain
```

---

#### 2. `struct gpd_link` — ربط الـ domains في شجرة

```c
struct gpd_link {
    struct generic_pm_domain *parent; /* الـ domain الأكبر */
    struct list_head parent_node;     /* node في parent->child_links */
    struct generic_pm_domain *child;  /* الـ domain الأصغر */
    struct list_head child_node;      /* node في child->parent_links */

    unsigned int performance_state;      /* ما يطلبه الـ child من الـ parent */
    unsigned int prev_performance_state; /* للـ tracking */
};
```

**الـ hierarchy بالتفصيل:**

```
         Parent Domain (e.g., "always-on" power island)
         ┌─────────────────────────────────────┐
         │  child_links list:                  │
         │   → gpd_link (child=GPU domain)     │
         │   → gpd_link (child=DSP domain)     │
         └─────────────────────────────────────┘
                 │                   │
                 ▼                   ▼
          GPU Domain            DSP Domain
          ┌──────────┐          ┌──────────┐
          │dev_list: │          │dev_list: │
          │ → GPU    │          │ → DSP    │
          │ → ISP    │          │          │
          └──────────┘          └──────────┘
```

لما GPU Domain يحاول يتقفل، الـ genpd core بيشوف: هل في child domains تانية صاحية جوا الـ parent? لأ؟ تمام، ممكن نوقف الـ parent.

**الـ `sd_count` في الـ parent domain** هو `atomic_t` بيعد عدد الـ child domains الـ ON. لما بيوصل لصفر، ممكن نوقف الـ parent.

---

#### 3. `struct generic_pm_domain_data` — بيانات كل device داخل الـ domain

```c
struct generic_pm_domain_data {
    struct pm_domain_data base;  /* فيه: list_node + pointer للـ device */
    struct gpd_timing_data *td;  /* suspend/resume latency للـ device ده */
    struct notifier_block nb;    /* للاستقبال notifications من الـ domain */
    int cpu;                     /* لو الـ device ده CPU */
    unsigned int performance_state; /* الـ pstate المطلوب من الـ device ده */
    unsigned int opp_token;      /* للـ OPP integration */
    bool hw_mode;                /* هل الـ domain شغال في HW-managed mode؟ */
    bool rpm_always_on;          /* الـ device مش بيسمح بـ power off عند RPM */
};
```

كل device بيتضاف للـ domain بيعمل `generic_pm_domain_data` خاص بيه — بيتخزن في `dev->power.subsys_data->domain_data`. للوصول للـ domain data:

```c
// من device للـ genpd data بتاعها:
struct generic_pm_domain_data *gpd_data = dev_gpd_data(dev);
// = container_of(dev->power.subsys_data->domain_data,
//                struct generic_pm_domain_data, base)
```

---

#### 4. `struct genpd_power_state` — الـ Idle States

```c
struct genpd_power_state {
    const char *name;               /* "off", "ret" (retention), etc. */
    s64 power_off_latency_ns;       /* وقت الـ power off */
    s64 power_on_latency_ns;        /* وقت الـ power on */
    s64 residency_ns;               /* الحد الأدنى للوقت عشان يستاهل */
    u64 usage;                      /* كام مرة اتاخد القرار ده */
    u64 rejected;                   /* كام مرة اترفض */
    struct fwnode_handle *fwnode;   /* الـ DT node بتاعها */
    void *data;                     /* SoC-specific data */
};
```

ده شبيه جدًا بـ **CPU idle states** (C-states) — كل state ليها tradeoff بين وفر الطاقة ووقت الـ wake-up latency. الـ governor هو اللي بيختار أنسب state حسب الـ next wakeup المتوقع.

---

#### 5. `struct dev_power_governor` — من يقرر؟

```c
struct dev_power_governor {
    bool (*system_power_down_ok)(struct dev_pm_domain *domain);
    bool (*power_down_ok)(struct dev_pm_domain *domain);
    bool (*suspend_ok)(struct device *dev);
};
```

**ملاحظة مهمة:** الـ governor بيشتغل مع `struct dev_pm_domain` مش `struct generic_pm_domain` — لأن الـ governor API مش مرتبط بـ genpd بس، ممكن يتستخدم مع أي power domain. الـ `pd_to_genpd()` هي اللي بتعمل الـ cast لما محتاجه.

الـ governors المتاحة:
- **`simple_qos_governor`**: بيستخدم الـ QoS latency constraints — لو في device بتطلب wake-up سريع، مش هيسمح بإطفاء الـ domain.
- **`pm_domain_always_on_gov`**: دايما يرفض الإطفاء — للـ domains الـ critical.
- **`pm_domain_cpu_gov`**: مخصص للـ CPU domains مع الـ cpuidle framework.

---

#### 6. `struct genpd_governor_data` — البيانات اللي بيشوفها الـ Governor

```c
struct genpd_governor_data {
    s64 max_off_time_ns;           /* أقصى وقت يقدر يفضل off */
    bool max_off_time_changed;     /* هل اتغير من آخر مرة؟ */
    ktime_t next_wakeup;           /* أقرب wakeup متوقع */
    ktime_t next_hrtimer;          /* أقرب hrtimer */
    ktime_t last_enter;            /* آخر مرة دخل idle */
    bool reflect_residency;
    bool cached_power_down_ok;     /* cache للقرار الأخير */
    bool cached_power_down_state_idx;
};
```

الـ governor بيحسب: هل الـ `max_off_time_ns` أكبر من الـ `power_off_latency_ns + power_on_latency_ns + residency_ns`؟ لو أيوه، يستاهل نطفي. لو لأ، وفرنا على نفسنا latency زيادة من غير فايدة.

---

#### 7. `struct gpd_timing_data` — latency كل device

```c
struct gpd_timing_data {
    s64 suspend_latency_ns;        /* وقت الـ suspend بتاع الـ device */
    s64 resume_latency_ns;         /* وقت الـ resume بتاع الـ device */
    s64 effective_constraint_ns;   /* الـ QoS constraint الفعلي */
    ktime_t next_wakeup;           /* متى الـ device هيصحى؟ */
    bool constraint_changed;
    bool cached_suspend_ok;
};
```

الـ genpd بيجمع الـ latency من كل الأجهزة في الـ domain — أعلى latency بتحدد إمتى يقدر الـ domain يتقفل.

---

### الـ Struct Relationship Graph الكامل

```
struct generic_pm_domain
├── struct dev_pm_domain domain ─────────────────────────────────┐
│       └── struct dev_pm_ops ops                                │
│               (runtime_suspend, runtime_resume, ...)           │
│                                                                │
├── struct list_head dev_list                                    │
│       │                                                        │
│       ▼ (each node is)                                         │
│   struct pm_domain_data ◄── embedded in:                      │
│   └── struct generic_pm_domain_data                           │
│           ├── struct gpd_timing_data *td                       │
│           ├── unsigned int performance_state                   │
│           └── bool hw_mode                                     │
│                                                                │
├── struct list_head parent_links / child_links                  │
│       │                                                        │
│       ▼ (each link is)                                         │
│   struct gpd_link                                              │
│   ├── *parent → struct generic_pm_domain                      │
│   └── *child  → struct generic_pm_domain                      │
│                                                                │
├── struct dev_power_governor *gov                               │
│       └── power_down_ok(), suspend_ok()                        │
│                                                                │
├── struct genpd_governor_data *gd                               │
│       └── next_wakeup, max_off_time_ns, ...                    │
│                                                                │
└── struct genpd_power_state *states[state_count]               │
        └── latency_ns, residency_ns, usage, ...                │
                                                                │
struct device                                                   │
└── struct dev_pm_info power                                    │
        └── struct dev_pm_domain *pm_domain ──────────────────►┘
                (points to domain field above)
```

---

### الـ OF (Device Tree) Integration

**سؤال:** إزاي الـ device بتعرف إنها منتمية لأي domain؟

الإجابة: من الـ Device Tree عبر `power-domains` property:

```dts
/* الـ genpd provider */
power_domains: power-controller@1c00000 {
    compatible = "rockchip,rk3588-power-controller";
    reg = <0x0 0x1c00000 0x0 0x1000>;
    #power-domain-cells = <1>;
};

/* الـ consumer */
gpu: gpu@fb000000 {
    compatible = "arm,mali-g610-m4";
    reg = <0x0 0xfb000000 0x0 0x200000>;
    power-domains = <&power_domains RK3588_PD_GPU>;
};
```

الـ flow:
1. الـ genpd backend driver بيعمل `of_genpd_add_provider_onecell()` → يسجل نفسه كـ provider.
2. لما الـ GPU driver يتـ probe، الكيرنل يلاقي `power-domains` في الـ DT.
3. بيعمل `genpd_dev_pm_attach()` → بيربط الـ GPU device بالـ genpd صح.
4. بيعمل `dev_pm_domain_set()` → بيخلي `dev->power.pm_domain` يشاور على الـ genpd ops.

**الـ `genpd_xlate_t`:**

```c
typedef struct generic_pm_domain *(*genpd_xlate_t)(
    const struct of_phandle_args *args, void *data);
```

ده callback بيترجم من الـ DT phandle argument (رقم الـ domain) للـ `generic_pm_domain` الصح. الـ `genpd_onecell_data` بيخزن مصفوفة الـ domains وأي `xlate` function هتستخدم.

---

### ما الـ genpd بيمتلكه vs ما بيفوضه للـ Driver

| المسؤولية | المالك |
|---|---|
| Reference counting للأجهزة | **genpd core** |
| ترتيب إيقاف/تشغيل الـ parent domains | **genpd core** |
| Locking وحل الـ race conditions | **genpd core** |
| قرار "هل نطفي دلوقتي؟" | **genpd Governor** |
| حساب latency constraints | **genpd core** (بيانات من الـ driver) |
| OPP aggregation عبر الـ domain | **genpd core** |
| الـ notification عند power on/off | **genpd core** |
| Write للـ hardware registers لتشغيل/إطفاء الـ island | **genpd Backend Driver** |
| تعريف الـ power states وإمتى تستخدم | **genpd Backend Driver + DT** |
| device-specific suspend sequence | **Device Driver** |
| تسجيل الـ domain في الـ DT | **genpd Backend Driver** |

---

### مفاهيم تانية محتاج تعرفها قبل التعمق

- **Runtime PM (CONFIG_PM):** الـ subsystem اللي بيدير تشغيل/إطفاء الـ device أثناء الـ runtime. الـ genpd بيخلل في الـ runtime PM ops بتاعة كل device. مش هينفعك تفهم genpd من غيره.
- **OPP (Operating Performance Points):** الـ subsystem اللي بيدير الـ frequency/voltage tables. الـ genpd بيستخدمه عشان `performance_state` — كل domain ممكن يكون ليه OPP table.
- **Device Links (`device_link`):** الـ genpd بيعمل device-links بين الـ device والـ virtual device بتاع الـ domain — عشان الـ runtime PM يعرف الـ dependency hierarchy.
- **QoS (Quality of Service):** الـ `simple_qos_governor` بيعتمد على latency constraints اللي بيسجلها كل device — من `include/linux/pm_qos.h`.

---

### الـ Hardware Mode (`hw_mode`)

في بعض الـ SoCs (خصوصًا Qualcomm وبعض ARM designs)، الـ power domain ممكن يتحكم فيها الـ hardware مباشرة بدون تدخل الـ CPU — الـ hardware بيحس إن الـ device خلاص idle وبيقفلها لوحده.

```c
int (*set_hwmode_dev)(struct generic_pm_domain *domain,
                      struct device *dev, bool enable);
bool (*get_hwmode_dev)(struct generic_pm_domain *domain,
                       struct device *dev);
```

لو `hw_mode = true` على الـ `generic_pm_domain_data`، الـ genpd بيفهم إن الـ power control بيحصل على مستوى الـ HW وبيعدل سلوكه.

---

### الـ Synced Poweroff

```c
bool synced_poweroff; /* A consumer needs a synced poweroff */
```

في حالات معينة، device محتاجة إن الـ domain يتقفل بشكل **synchronous** (مش في الـ work queue). الـ `dev_pm_genpd_synced_poweroff()` بيعمل set لـ flag ده. الـ genpd core بيلاحظه لما بيقرر يستخدم `power_off_work` (async) أو يتصل بـ `power_off()` مباشرة.

---

### الـ Performance States وعلاقتها بالـ Aggregation

كل device داخل الـ domain ممكن تطلب performance state معين. الـ genpd بيعمل **aggregation**: بياخد أعلى `performance_state` مطلوب من كل الأجهزة.

```
GPU  → requests pstate 3 (high freq)
ISP  → requests pstate 1 (low freq)
────────────────────────────────────
domain->performance_state = 3  ← max(3, 1)
```

لما آخر device تخلص، الـ `performance_state` بيتزيل. لو الـ domain ليه parent، بيـ propagate الـ aggregated state للـ parent بردو عبر `gpd_link->performance_state`.

---

### الـ Locking Strategy

الـ genpd لديه locking مرن حسب الـ context:

```c
union {
    struct mutex mlock;      // الحالة العادية - sleeping lock
    struct {
        spinlock_t slock;    // لو GENPD_FLAG_IRQ_SAFE معموله set
        unsigned long lock_flags;
    };
    struct {
        raw_spinlock_t raw_slock;  // للـ realtime kernels
        unsigned long raw_lock_flags;
    };
};
```

الـ `genpd_lock_ops` struct (forward declared في الـ header) بيحدد الـ ops المستخدمة — الـ genpd core بيستخدمه internally عشان يقدر يشتغل في الـ contexts المختلفة من غير ما يعمل `if/else` في كل مكان.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums — Cheatsheet

#### `PD_FLAG_*` — بيتحدد وقت الـ attach بتاع الـ device للـ PM domain

| Flag | Value | المعنى |
|------|-------|--------|
| `PD_FLAG_NO_DEV_LINK` | `BIT(0)` | متعملش device-link أوتوماتيك |
| `PD_FLAG_DEV_LINK_ON` | `BIT(1)` | فعّل `DL_FLAG_RPM_ACTIVE` عند إنشاء الـ device-link |
| `PD_FLAG_REQUIRED_OPP` | `BIT(2)` | اربط الـ required OPPs بالـ domains حسب الـ index |
| `PD_FLAG_ATTACH_POWER_ON` | `BIT(3)` | شغّل الـ domain أوتوماتيك عند الـ attach |
| `PD_FLAG_DETACH_POWER_OFF` | `BIT(4)` | أوقف الـ domain أوتوماتيك عند الـ detach |

#### `GENPD_FLAG_*` — بيتحدد قبل `pm_genpd_init()`، بيتخزن في `generic_pm_domain.flags`

| Flag | Value | المعنى |
|------|-------|--------|
| `GENPD_FLAG_PM_CLK` | `1U << 0` | استخدم PM clock framework |
| `GENPD_FLAG_IRQ_SAFE` | `1U << 1` | الـ callbacks مش بتنام → صالحة من atomic context |
| `GENPD_FLAG_ALWAYS_ON` | `1U << 2` | الـ domain يفضل شغال دايمًا |
| `GENPD_FLAG_ACTIVE_WAKEUP` | `1U << 3` | افضل ON لو فيه device wakeup path |
| `GENPD_FLAG_CPU_DOMAIN` | `1U << 4` | الـ domain بيشيل CPUs — يعمل CPU idle management |
| `GENPD_FLAG_RPM_ALWAYS_ON` | `1U << 5` | ON دايمًا إلا في system suspend |
| `GENPD_FLAG_MIN_RESIDENCY` | `1U << 6` | الـ governor يحسب minimum residency |
| `GENPD_FLAG_OPP_TABLE_FW` | `1U << 7` | جدول الـ OPP جاي من FW مش DT |
| `GENPD_FLAG_DEV_NAME_FW` | `1U << 8` | اعمل device name فريد باستخدام IDA |
| `GENPD_FLAG_NO_SYNC_STATE` | `1U << 9` | الـ provider بيدير sync_state بنفسه |
| `GENPD_FLAG_NO_STAY_ON` | `1U << 10` | اسمح بـ power-off قبل ما sync_state يُستدعى |

#### الـ Enums

| Enum | القيم | الاستخدام |
|------|-------|----------|
| `gpd_status` | `GENPD_STATE_ON=0`, `GENPD_STATE_OFF` | الحالة الحالية للـ domain |
| `genpd_notication` | `PRE_OFF`, `OFF`, `PRE_ON`, `ON` | أحداث الـ notifier chain |
| `genpd_sync_state` | `OFF=0`, `SIMPLE`, `ONECELL` | طريقة إدارة الـ sync_state |

#### الـ Config Options الأساسية

| Kconfig | الأثر |
|---------|-------|
| `CONFIG_PM_GENERIC_DOMAINS` | يفعّل كل الـ genpd API |
| `CONFIG_PM_GENERIC_DOMAINS_OF` | يفعّل الـ DT/OF integration |
| `CONFIG_PM_GENERIC_DOMAINS_SLEEP` | يفعّل `dev_pm_genpd_suspend/resume` |
| `CONFIG_CPU_IDLE` | يفعّل `pm_domain_cpu_gov` |

---

### الـ Structs المهمة

#### 1. `struct generic_pm_domain` — القلب

**الغرض:** يمثل الـ power domain نفسه. كل الـ devices اللي جوا الـ domain دي بتتوقف وبتشتغل مع بعض.

**الحقول الأساسية:**

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `dev` | `struct device` | الـ domain نفسه device في الـ kernel |
| `domain` | `struct dev_pm_domain` | الـ PM ops اللي الـ PM core بيكلمها |
| `gpd_list_node` | `list_head` | node في القائمة العالمية للـ domains |
| `parent_links` | `list_head` | الـ domains اللي هو parent ليها |
| `child_links` | `list_head` | الـ domains اللي هو child فيها |
| `dev_list` | `list_head` | قائمة الـ devices المرتبطة بيه |
| `gov` | `*dev_power_governor` | الـ governor اللي بيقرر امتى يوقف |
| `gd` | `*genpd_governor_data` | بيانات الـ governor الداخلية |
| `power_off_work` | `work_struct` | شغل الـ power-off في workqueue |
| `provider` | `*fwnode_handle` | مين اللي سجّل الـ domain دا في DT/ACPI |
| `sd_count` | `atomic_t` | عدد الـ subdomains اللي شغالة |
| `status` | `enum gpd_status` | ON ولا OFF حاليًا |
| `device_count` | `unsigned int` | عدد الـ devices المرفقة |
| `performance_state` | `unsigned int` | max performance state مجمّع من كل الـ devices |
| `cpus` | `cpumask_var_t` | CPUs المرتبطة بالـ domain ده |
| `power_off/on` | function pointers | الـ backend driver callbacks الفعلية |
| `power_notifiers` | `raw_notifier_head` | chain للـ notifiers على power events |
| `opp_table` | `*opp_table` | جدول الـ Operating Performance Points |
| `set_performance_state` | function pointer | callback لتغيير الـ performance state |
| `set_hwmode_dev` / `get_hwmode_dev` | function pointers | تحكم في HW autonomy mode لـ device |
| `attach_dev` / `detach_dev` | function pointers | backend hooks عند ربط/فصل device |
| `flags` | `unsigned int` | `GENPD_FLAG_*` بتحدد السلوك |
| `states` | `*genpd_power_state` | مصفوفة حالات الـ idle states |
| `state_count` / `state_idx` | `unsigned int` | عدد الـ states + الـ state المختار |
| `on_time` | `u64` | وقت آخر power-on (ns) |
| `lock_ops` | `*genpd_lock_ops` | vtable لعمليات الـ locking |
| `mlock` / `slock` / `raw_slock` | union | الـ lock الفعلي (mutex أو spinlock حسب IRQ safety) |

---

#### 2. `struct dev_pm_domain` (من `pm.h`)

**الغرض:** الـ interface اللي الـ PM core بيشوفه من الـ domain — set of callbacks.

```c
struct dev_pm_domain {
    struct dev_pm_ops ops;          /* suspend/resume/runtime callbacks */
    int  (*start)(struct device *);
    void (*detach)(struct device *, bool power_off);
    int  (*activate)(struct device *);
    void (*sync)(struct device *);
    void (*dismiss)(struct device *);
    int  (*set_performance_state)(struct device *, unsigned int state);
};
```

الـ `generic_pm_domain` بيضم الـ struct دا `domain` field، والـ `pd_to_genpd()` بترجعله بـ `container_of`.

---

#### 3. `struct generic_pm_domain_data` — بيانات الـ device جوا الـ domain

**الغرض:** بيانات خاصة بكل device مرتبطة بـ genpd.

| الحقل | الشرح |
|-------|-------|
| `base` | `struct pm_domain_data` — الـ base اللي فيه pointer للـ device والـ list node |
| `td` | `*gpd_timing_data` — latencies وـ wakeup times |
| `nb` | `notifier_block` — للاستماع لـ power events من الـ domain |
| `power_nb` | pointer لـ notifier مسجّل على الـ supplier domain |
| `cpu` | رقم الـ CPU لو الـ device دي CPU |
| `performance_state` | الـ state المطلوب من الـ device دي |
| `default_pstate` | الـ performance state الافتراضي |
| `rpm_pstate` | الـ performance state لحالة runtime PM |
| `opp_token` | token من OPP framework |
| `hw_mode` | هل الـ device شغال في HW autonomy mode |
| `rpm_always_on` | هل الـ device طلبت إن الـ domain يفضل ON runtime |

---

#### 4. `struct pm_domain_data` — الـ Base

**الغرض:** أصغر وحدة بتربط device بـ domain.

```c
struct pm_domain_data {
    struct list_head list_node; /* node in genpd->dev_list */
    struct device   *dev;       /* pointer to the device */
};
```

بيتوصل منه لـ `generic_pm_domain_data` بـ `to_gpd_data()` اللي بتعمل `container_of`.

---

#### 5. `struct gpd_link` — ربط الـ Parent/Child Domains

**الغرض:** يمثل العلاقة بين domain أب وـ domain ابن (subdomain hierarchy).

| الحقل | الشرح |
|-------|-------|
| `parent` | pointer للـ domain الأب |
| `parent_node` | node في `parent->parent_links` |
| `child` | pointer للـ domain الابن |
| `child_node` | node في `child->child_links` |
| `performance_state` | الـ performance state المطلوبة من الابن على الأب |
| `prev_performance_state` | القيمة السابقة (للمقارنة) |

---

#### 6. `struct genpd_power_state` — حالة idle واحدة

**الغرض:** بتوصف حالة idle واحدة للـ domain (زي C-states في CPUs).

| الحقل | الشرح |
|-------|-------|
| `name` | اسم الـ state (مثلًا "off") |
| `power_off_latency_ns` | وقت الدخول للـ state بالـ ns |
| `power_on_latency_ns` | وقت الخروج من الـ state بالـ ns |
| `residency_ns` | الحد الأدنى من الوقت عشان الـ state يستحق |
| `usage` | عدد المرات اللي اتدخل فيها الـ state |
| `rejected` | عدد المرات اللي اتفضل فيها |
| `above` / `below` | إحصائيات governor |
| `fwnode` | مرجع الـ DT node اللي وصفه |
| `idle_time` | الوقت الكلي المقضي في الـ state |
| `data` | private data للـ backend driver |

---

#### 7. `struct gpd_timing_data` — latency بيانات الـ device

| الحقل | الشرح |
|-------|-------|
| `suspend_latency_ns` | وقت suspend الـ device |
| `resume_latency_ns` | وقت resume الـ device |
| `effective_constraint_ns` | القيد الفعلي المحسوب من كل الـ devices |
| `next_wakeup` | موعد الـ wakeup القادم |
| `constraint_changed` | flag لإعادة حساب الـ constraint |
| `cached_suspend_ok` | نتيجة مكاشة لـ suspend_ok |

---

#### 8. `struct genpd_governor_data` — حالة الـ Governor

| الحقل | الشرح |
|-------|-------|
| `max_off_time_ns` | أقصى وقت ممكن يكون فيه الـ domain OFF |
| `max_off_time_changed` | flag لإعادة الحساب |
| `next_wakeup` | أقرب wakeup من أي device |
| `next_hrtimer` | أقرب hrtimer active |
| `last_enter` | وقت آخر دخول للـ idle |
| `reflect_residency` | هل نقيس residency فعلي |
| `cached_power_down_ok` | نتيجة مكاشة للـ power_down decision |
| `cached_power_down_state_idx` | الـ state المختار من الـ cache |

---

#### 9. `struct dev_power_governor` — سياسة الإيقاف

```c
struct dev_power_governor {
    bool (*system_power_down_ok)(struct dev_pm_domain *);
    bool (*power_down_ok)(struct dev_pm_domain *);
    bool (*suspend_ok)(struct device *dev);
};
```

الـ kernel بيوفر ثلاث تنفيذات:
- **`simple_qos_governor`** — بيشيل الـ QoS latency constraints
- **`pm_domain_always_on_gov`** — دايمًا `false` (الـ domain يفضل ON)
- **`pm_domain_cpu_gov`** — متخصص للـ CPUs (لما `CONFIG_CPU_IDLE`)

---

#### 10. `struct gpd_dev_ops` — عمليات Device داخل الـ Domain

```c
struct gpd_dev_ops {
    int (*start)(struct device *dev); /* شغّل الـ device */
    int (*stop)(struct device *dev);  /* وقّف الـ device */
};
```

---

#### 11. `struct genpd_onecell_data` — تسجيل OF Provider

```c
struct genpd_onecell_data {
    struct generic_pm_domain **domains; /* مصفوفة الـ domains */
    unsigned int               num_domains;
    genpd_xlate_t              xlate;   /* translator: phandle args → genpd */
};
```

---

#### 12. `struct dev_pm_domain_attach_data` و `struct dev_pm_domain_list`

```c
struct dev_pm_domain_attach_data {
    const char * const *pd_names;  /* أسماء الـ domains المطلوبة */
    const u32 num_pd_names;
    const u32 pd_flags;            /* PD_FLAG_* */
};

struct dev_pm_domain_list {
    struct device      **pd_devs;  /* الـ virtual devices للـ domains */
    struct device_link **pd_links; /* الـ device-links المنشأة */
    u32                 *opp_tokens;
    u32                  num_pds;
};
```

---

### رسم العلاقات بين الـ Structs

```
                    ┌─────────────────────────────────────────┐
                    │         generic_pm_domain                │
                    │  ┌────────────┐  ┌──────────────────┐   │
                    │  │    dev     │  │     domain       │   │
                    │  │ (device)   │  │ (dev_pm_domain)  │   │
                    │  └────────────┘  └──────────────────┘   │
                    │  *gov ──────────────► dev_power_governor │
                    │  *gd  ──────────────► genpd_governor_data│
                    │  *opp_table ────────► opp_table          │
                    │  *states ───────────► genpd_power_state[]│
                    │  *provider ─────────► fwnode_handle      │
                    │  parent_links ◄─────────────────────┐    │
                    │  child_links  ──────────────────┐   │    │
                    │  dev_list     ◄─────────────┐   │   │    │
                    └─────────────────────────────│───│───│────┘
                                                  │   │   │
              ┌───────────────────────────────────┘   │   │
              │  pm_domain_data (list_node + *dev)    │   │
              │    ▲                                   │   │
              │    │ base                              │   │
              │  generic_pm_domain_data                │   │
              │    *td ──► gpd_timing_data             │   │
              │    nb  ──► (notifier_block)            │   │
              └───────────────────────────────────────    │
                                                          │
              ┌───────────────────────────────────────────┘
              │         gpd_link
              │  *parent ──────────────► generic_pm_domain (parent)
              │  parent_node  (in parent->parent_links)
              │  *child  ──────────────► generic_pm_domain (child)
              │  child_node   (in child->child_links)
              └────────────────────────────────────────────

              genpd_onecell_data
                  **domains ──────────► generic_pm_domain[]
                  xlate()   ──────────► (phandle_args → *genpd)
```

---

### رسم هيكل الـ Domain Hierarchy (مثال عملي)

```
     ┌─────────────────────┐
     │   power-domain@0    │   ← parent genpd (SoC top-level)
     │   GENPD_FLAG_PM_CLK │
     │   parent_links: [─────────────────────────┐
     └─────────────────────┘                     │
              │ child_links                       │
              ▼                                   │
     ┌─────────────────────┐          ┌──────────┴──────────┐
     │  power-domain@1     │          │   power-domain@2    │
     │  (GPU domain)       │          │   (DSP domain)      │
     │  dev_list:          │          │   dev_list:         │
     │   [gpu_device]      │          │   [dsp_device]      │
     └─────────────────────┘          └─────────────────────┘
              │
         gpd_link
          .parent → power-domain@0
          .child  → power-domain@1
          .performance_state = 200 (MHz requested from parent)
```

---

### دورة حياة الـ Genpd (Lifecycle Diagram)

```
DRIVER/BOARD CODE                    GENPD CORE
─────────────────                    ──────────
alloc generic_pm_domain
set .flags = GENPD_FLAG_*
set .power_on / .power_off callbacks
         │
         ▼
pm_genpd_init(genpd, gov, is_off)
  → initializes lists, lock, state
  → adds to global gpd_list
  → sets status = ON or OFF
         │
         ▼
of_genpd_add_provider_simple/onecell()
  → registers with OF framework
  → consumers can now find this domain via DT phandle
         │
         ▼
pm_genpd_add_subdomain()     [optional]
  → creates gpd_link between parent & child
         │
         ▼
(device probe happens)
dev_pm_domain_attach() / genpd_dev_pm_attach()
  → calls attach_dev() backend callback
  → allocates generic_pm_domain_data
  → adds device to genpd->dev_list
  → creates device-link (unless PD_FLAG_NO_DEV_LINK)
         │
         ▼
     [RUNTIME: device active / idle cycles]
         │
         ▼
(device remove happens)
dev_pm_domain_detach()
  → calls detach_dev()
  → removes from dev_list
  → frees generic_pm_domain_data
  → breaks device-link
         │
         ▼
pm_genpd_remove(genpd)
  → removes from gpd_list
  → must have zero devices & subdomains
         │
         ▼
of_genpd_del_provider()
  → unregisters from OF framework
```

---

### دورة حياة الـ Power State (Runtime PM Flow)

```
device runtime idle
         │
         ▼
genpd governor: power_down_ok()?
  suspend_ok(dev)?   ← checks QoS latency constraints
  power_down_ok()?   ← checks all devices suspended + timing
         │
    YES  │
         ▼
genpd_power_off()
  → for each device in dev_list:
       gpd_dev_ops.stop(dev)
  → check sd_count == 0 (no active subdomains)
  → call governor again for state selection
  → genpd->power_off(genpd)   ← backend callback
  → status = GENPD_STATE_OFF
  → notify: GENPD_NOTIFY_OFF
         │
         ▼ (wakeup event)
genpd_power_on()
  → notify: GENPD_NOTIFY_PRE_ON
  → power on parent domain first (recursive)
  → genpd->power_on(genpd)    ← backend callback
  → status = GENPD_STATE_ON
  → notify: GENPD_NOTIFY_ON
  → for each device:
       gpd_dev_ops.start(dev)
```

---

### Call Flow Diagrams

#### 1. Device Attach Flow

```
driver calls dev_pm_domain_attach(dev, flags)
  → pm.c: looks up domain from DT phandle (of_genpd_add_device)
    → of_genpd.c: xlate() maps phandle → generic_pm_domain *
      → genpd_add_device(genpd, dev)
        → alloc generic_pm_domain_data (td, nb, ...)
        → list_add to genpd->dev_list
        → genpd->attach_dev(genpd, dev)   ← backend hook
        → genpd->domain.activate(dev)     ← PM core hook
        → device_link_add(dev, &genpd->dev)  [unless NO_DEV_LINK]
        → if PD_FLAG_ATTACH_POWER_ON:
            genpd_power_on(genpd, ...)
```

#### 2. Runtime Suspend Flow

```
rpm_suspend(dev)
  → dev->pm_domain->ops.runtime_suspend(dev)
    → genpd_runtime_suspend(dev)
      → gpd_dev_ops.stop(dev)           ← optional stop
      → update gpd_timing_data
      → check if all devs suspended
        → governor->power_down_ok()?
          → genpd_power_off_work / direct call
            → genpd->power_off(genpd)   ← HW backend (e.g. MMIO write)
              → notify power_notifiers (NOTIFY_OFF)
              → propagate to parent domain (recursive)
```

#### 3. Performance State Update Flow

```
dev_pm_genpd_set_performance_state(dev, state)
  → gpd_data->performance_state = state
  → genpd_update_performance_state(genpd)
    → aggregate max across all devices + subdomains
    → genpd->set_performance_state(genpd, agg_state)
      → backend: e.g. sets voltage/frequency via OPP
        → opp_set_rate() or regulator_set_voltage()
    → if parent domain:
        → update gpd_link->performance_state
        → propagate up to parent->set_performance_state()
```

#### 4. System Suspend Flow

```
kernel: pm_suspend()
  → dpm_suspend_noirq()
    → dev->pm_domain->ops.suspend_noirq(dev)
      → genpd_suspend_noirq(dev)
        → gpd_data->td->suspend_latency_ns recorded
        → genpd->suspended_count++
        → check if all devices suspended
          → genpd_power_off(genpd)  [if governor agrees]
```

---

### استراتيجية الـ Locking

#### نوع الـ Lock في `generic_pm_domain`

الـ `generic_pm_domain` بيستخدم **union** من ثلاث أنواع locks، وبيتحدد بواسطة `GENPD_FLAG_IRQ_SAFE`:

```
                ┌──────────────────────────────────┐
                │  generic_pm_domain.lock_ops      │
                │  (vtable for lock/unlock/trylock) │
                └──────────────────────────────────┘
                            │
               ─────────────────────────────
               │                           │
    IRQ_SAFE flag NOT set         IRQ_SAFE flag SET
               │                           │
        struct mutex mlock          spinlock_t slock
        (sleeping lock)           (or raw_spinlock_t)
        (normal process ctx)      (atomic/IRQ context OK)
```

**قاعدة مهمة:** لو domain فيه `GENPD_FLAG_IRQ_SAFE`، كل الـ master domains بتاعته لازم يكون عندهم نفس الـ flag، عشان ما يحصلش deadlock لما يقفل parent من atomic context.

#### ترتيب الـ Locks (Lock Ordering)

```
child genpd lock
    ↓ (acquired first)
parent genpd lock
    ↓ (acquired second)
grandparent genpd lock
    ↓ ...

NEVER reverse this order!
```

**الـ device->power.lock** (من RPM framework) بيتأخذ قبل الـ genpd lock:

```
rpm lock (dev->power.lock) → genpd lock → parent genpd lock
```

#### ما اللي بيحميه كل lock

| الـ Lock | ما بيحميه |
|----------|-----------|
| `genpd->mlock` / `slock` | `dev_list`, `status`, `sd_count`, `performance_state`, `state_idx`, `suspended_count`, `prepared_count` |
| `dev->power.lock` (RPM) | حالة الـ runtime PM للـ device نفسها |
| `gpd_list_lock` (global, في pm_genpd.c) | القائمة العالمية للـ domains |

#### الـ Notifier Chain

الـ `power_notifiers` في `generic_pm_domain` هي `raw_notifier_head` — بتُستدعى تحت الـ genpd lock نفسه، فالـ notifier callbacks **ممنوع تأخذ الـ genpd lock** تاني.

```c
/* ترتيب الأحداث */
GENPD_NOTIFY_PRE_OFF  → power_off()  → GENPD_NOTIFY_OFF
GENPD_NOTIFY_PRE_ON   → power_on()   → GENPD_NOTIFY_ON
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### مجموعة: Initialization & Registration

| Function | الغرض |
|---|---|
| `pm_genpd_init()` | تهيئة `generic_pm_domain` وربطه بـ governor |
| `pm_genpd_remove()` | إزالة genpd من الـ global list |
| `pm_genpd_add_device()` | إضافة device لـ genpd |
| `pm_genpd_remove_device()` | إزالة device من genpd |
| `pm_genpd_add_subdomain()` | ربط child domain بـ parent domain |
| `pm_genpd_remove_subdomain()` | فك الربط بين subdomain وparent |

#### مجموعة: OF Provider Registration

| Function | الغرض |
|---|---|
| `of_genpd_add_provider_simple()` | تسجيل node له domain واحد |
| `of_genpd_add_provider_onecell()` | تسجيل node له array من domains |
| `of_genpd_del_provider()` | حذف OF provider |
| `of_genpd_add_device()` | ربط device بـ genpd عن طريق phandle |
| `of_genpd_add_subdomain()` | ربط subdomain بـ phandle args |
| `of_genpd_remove_subdomain()` | فك ربط subdomain عن طريق phandle |
| `of_genpd_remove_last()` | إزالة آخر domain في node معين |
| `of_genpd_parse_idle_states()` | parse idle states من DT |
| `of_genpd_sync_state()` | مزامنة sync_state لـ node |

#### مجموعة: Device Attach / Detach

| Function | الغرض |
|---|---|
| `dev_pm_domain_attach()` | ربط device بـ PM domain بناءً على DT/ACPI |
| `dev_pm_domain_attach_by_id()` | ربط device بـ domain محدد بـ index |
| `dev_pm_domain_attach_by_name()` | ربط device بـ domain محدد بـ اسم |
| `dev_pm_domain_attach_list()` | ربط device بـ قائمة domains |
| `devm_pm_domain_attach_list()` | نفس السابق لكن بـ devres (auto cleanup) |
| `dev_pm_domain_detach()` | فصل device عن PM domain |
| `dev_pm_domain_detach_list()` | فصل device عن قائمة domains |
| `genpd_dev_pm_attach()` | attach via OF phandle (genpd internal) |
| `genpd_dev_pm_attach_by_id()` | attach عن طريق index مع إرجاع virtual device |
| `genpd_dev_pm_attach_by_name()` | attach عن طريق اسم مع إرجاع virtual device |

#### مجموعة: Runtime Helpers

| Function | الغرض |
|---|---|
| `dev_pm_domain_start()` | تشغيل device عبر domain |
| `dev_pm_domain_set()` | ضبط الـ `dev_pm_domain` يدوياً |
| `dev_pm_genpd_set_performance_state()` | طلب performance state جديدة |
| `dev_pm_domain_set_performance_state()` | نفس السابق عبر الـ domain ops |
| `dev_pm_genpd_add_notifier()` | تسجيل notifier على power on/off |
| `dev_pm_genpd_remove_notifier()` | إلغاء تسجيل notifier |
| `dev_pm_genpd_set_next_wakeup()` | إخبار genpd بـ wakeup القادم |
| `dev_pm_genpd_get_next_hrtimer()` | جلب أقرب hrtimer في الـ domain |
| `dev_pm_genpd_synced_poweroff()` | طلب synced power-off لـ domain |
| `dev_pm_genpd_set_hwmode()` | تفعيل/تعطيل hardware-managed power mode |
| `dev_pm_genpd_get_hwmode()` | قراءة حالة hw mode |
| `dev_pm_genpd_rpm_always_on()` | إجبار device على البقاء powered-on في runtime |
| `dev_pm_genpd_is_on()` | التحقق إذا كان domain مشغّل |
| `dev_pm_genpd_suspend()` | suspend genpd أثناء system sleep |
| `dev_pm_genpd_resume()` | resume genpd بعد system sleep |
| `pm_genpd_inc_rejected()` | زيادة عداد الـ rejected transitions |
| `dev_to_genpd_dev()` | جلب virtual device المرتبط بـ device |

#### مجموعة: Inline Helpers

| Function | الغرض |
|---|---|
| `pd_to_genpd()` | casting من `dev_pm_domain*` لـ `generic_pm_domain*` |
| `to_gpd_data()` | casting من `pm_domain_data*` لـ `generic_pm_domain_data*` |
| `dev_gpd_data()` | جلب `generic_pm_domain_data` مباشرةً من device |

---

### Group 1: Initialization & Core Registration

هذه المجموعة مسؤولة عن دورة حياة الـ `generic_pm_domain` نفسه — من التهيئة حتى الإزالة، وعن ربط الـ devices والـ subdomains به. بدون هذه الـ functions، الـ genpd لا يُعرف للـ kernel ولا يقدر يدير أي device.

---

#### `pm_genpd_init`

```c
int pm_genpd_init(struct generic_pm_domain *genpd,
                  struct dev_power_governor *gov, bool is_off);
```

**الـ function دي بتهيّئ الـ `generic_pm_domain` وبتضيفه لـ global list الخاص بـ genpd subsystem.** بتضبط الـ locking ops (mutex أو spinlock حسب الـ `GENPD_FLAG_IRQ_SAFE`)، وبتربط الـ governor المطلوب. لو الـ `GENPD_FLAG_ALWAYS_ON` متضبط، الـ domain بيفضل on دايماً وبيتجاهل الـ `is_off`.

| Parameter | الوصف |
|---|---|
| `genpd` | الـ domain المراد تهيئته — لازم يكون allocated ومُعبّي الـ callbacks قبل الاستدعاء |
| `gov` | الـ power governor — مثلاً `simple_qos_governor` أو `NULL` للـ default |
| `is_off` | حالة الـ domain الابتدائية: `true` = off، `false` = on |

**Return:** `0` عند النجاح، أو error code سالب.

**Key details:**
- لازم تضبط `genpd->flags` قبل الاستدعاء — الـ `lock_ops` بتتحدد هنا مرة واحدة.
- لو `GENPD_FLAG_IRQ_SAFE` متضبط، بيُستخدم `spinlock` بدل `mutex` — وده يعني كل الـ parent domains لازم يبقوا `IRQ_SAFE` برضو.
- بيتم `INIT_LIST_HEAD` لكل الـ lists الداخلية (dev_list، parent_links، child_links).
- بيُسجّل `genpd->dev` كـ device في الـ genpd bus لدعم الـ sysfs entries.
- **Caller context:** initialization code فقط، مش runtime.

---

#### `pm_genpd_remove`

```c
int pm_genpd_remove(struct generic_pm_domain *genpd);
```

**بتشيل الـ genpd من الـ global list وبتحذف الـ device المرتبط بيه.** بتفشل لو في devices أو subdomains لسا مرتبطة بالـ domain.

| Parameter | الوصف |
|---|---|
| `genpd` | الـ domain المراد إزالته |

**Return:** `0` عند النجاح، `-EBUSY` لو في devices أو subdomains لسا مسجّلين.

**Key details:**
- بتقفل الـ global `gpd_list_lock` قبل الإزالة.
- بعد الإزالة، الـ memory مش بتتحرر تلقائياً — ده مسؤولية الـ caller.

---

#### `pm_genpd_add_device`

```c
int pm_genpd_add_device(struct generic_pm_domain *genpd, struct device *dev);
```

**بتضيف device لـ genpd وبتضبط الـ `dev->power.subsys_data` اللي بيحمل الـ `generic_pm_domain_data`.** بتخلّي الـ genpd هو المسؤول عن إدارة الـ runtime PM للـ device.

| Parameter | الوصف |
|---|---|
| `genpd` | الـ domain المستهدف |
| `dev` | الـ device المراد إضافته |

**Return:** `0` عند النجاح، `-EINVAL` أو `-ENOMEM` عند الفشل.

**Key details:**
- بتستدعي الـ `genpd->attach_dev()` callback لو موجود.
- بتزوّد `genpd->device_count`.
- الـ device ممكن يكون في domain واحد بس في نفس الوقت.
- **Locking:** بتمسك الـ genpd lock أثناء العملية.

---

#### `pm_genpd_remove_device`

```c
int pm_genpd_remove_device(struct device *dev);
```

**بتشيل الـ device من الـ genpd اللي هو مضاف فيه، وبتنظّف الـ `generic_pm_domain_data` الخاص بيه.**

| Parameter | الوصف |
|---|---|
| `dev` | الـ device المراد إزالته |

**Return:** `0` عند النجاح، `-EINVAL` لو الـ device مش في أي domain.

**Key details:**
- بتستدعي `genpd->detach_dev()` callback.
- بتنقص `genpd->device_count`.
- لو الـ genpd فاضي وفي `power_off_work` pending، ممكن الـ domain ياخد قرار يطفش.

---

#### `pm_genpd_add_subdomain`

```c
int pm_genpd_add_subdomain(struct generic_pm_domain *genpd,
                           struct generic_pm_domain *subdomain);
```

**بتبني علاقة parent-child بين domain وsubdomain.** الـ parent domain مش هيتطفى إلا لما كل الـ children يتطفوا الأول (last-man-standing).

| Parameter | الوصف |
|---|---|
| `genpd` | الـ parent domain |
| `subdomain` | الـ child domain |

**Return:** `0` عند النجاح، `-EINVAL` / `-ENOMEM` / `-EEXIST` عند الفشل.

**Key details:**
- بيخلق `struct gpd_link` لربط الاتنين.
- الـ `sd_count` في الـ parent بيتزوّد لو الـ subdomain شغّال.
- لو أي domain فيهم `IRQ_SAFE`، الـ domain التاني لازم يبقى كده برضو.

---

#### `pm_genpd_remove_subdomain`

```c
int pm_genpd_remove_subdomain(struct generic_pm_domain *genpd,
                              struct generic_pm_domain *subdomain);
```

**بتفكّ الربط بين parent وchild وبتحذف الـ `gpd_link` المناظر.**

| Parameter | الوصف |
|---|---|
| `genpd` | الـ parent domain |
| `subdomain` | الـ child domain |

**Return:** `0` عند النجاح، `-EINVAL` لو العلاقة مش موجودة.

---

### Group 2: OF Provider Registration

الـ OF provider system بيخلّي الـ genpd backends تعرّف نفسها لـ Device Tree، عشان أي device يقدر يقول في DT بتاعه `power-domains = <&mypd>;` ويتلاقيه automatically.

---

#### `of_genpd_add_provider_simple`

```c
int of_genpd_add_provider_simple(struct device_node *np,
                                 struct generic_pm_domain *genpd);
```

**بتسجّل `device_node` كـ provider لـ domain واحد بس.** الـ xlate function الافتراضية بترجع نفس الـ `genpd` بصرف النظر عن الـ phandle args.

| Parameter | الوصف |
|---|---|
| `np` | الـ device node في DT اللي هيكون provider |
| `genpd` | الـ domain الوحيد اللي بيوفّره الـ node ده |

**Return:** `0` عند النجاح، error code سالب عند الفشل.

**Key details:**
- مناسبة للـ SoCs اللي فيها PM domain واحد بس لكل node — مثل Qualcomm RPM nodes.
- بتضبط `genpd->provider = &np->fwnode`.
- لازم تستدعي `of_genpd_del_provider()` عند الـ teardown.

---

#### `of_genpd_add_provider_onecell`

```c
int of_genpd_add_provider_onecell(struct device_node *np,
                                  struct genpd_onecell_data *data);
```

**بتسجّل node كـ provider لـ array من domains، بحيث الـ phandle arg الأول بيكون index في الـ array.** لو مفيش xlate مخصص في الـ `data`، بيستخدم الـ default الخاص بـ onecell.

| Parameter | الوصف |
|---|---|
| `np` | الـ device node في DT |
| `data` | struct فيه الـ domains array وعددهم واختياري custom xlate |

**Return:** `0` عند النجاح، error code سالب عند الفشل.

**Key details:**
- النمط الأشيع في platforms زي Rockchip وMediaTek وExynos — node واحد بيوفّر عشرات الـ domains.
- الـ `xlate` callback لو موجود بيقدر يعمل custom mapping بين الـ phandle args والـ domain.
- بتضبط `genpd->provider` لكل domain في الـ array.
- لو `GENPD_FLAG_DEV_NAME_FW`، بتولّد اسم unique باستخدام IDA.

**Pseudocode:**
```
for each domain in data->domains:
    set domain->provider = &np->fwnode
    set domain->has_provider = true

register np as OF provider with xlate = data->xlate or default_xlate
```

---

#### `of_genpd_del_provider`

```c
void of_genpd_del_provider(struct device_node *np);
```

**بتشيل الـ OF provider المسجّل لـ node معين من الـ provider list.** بتتأكد إن مفيش references على الـ provider قبل الحذف.

| Parameter | الوصف |
|---|---|
| `np` | الـ node اللي اتسجّل قبل كده |

---

#### `of_genpd_add_device`

```c
int of_genpd_add_device(const struct of_phandle_args *args, struct device *dev);
```

**بتحلّ الـ phandle args لـ genpd فعلي وبتضيف الـ device ليه.** بتستخدم الـ xlate function المسجّلة في الـ provider.

| Parameter | الوصف |
|---|---|
| `args` | الـ phandle args من DT (np + args array) |
| `dev` | الـ device المراد إضافته |

**Return:** `0` عند النجاح، error code سالب عند الفشل.

---

#### `of_genpd_parse_idle_states`

```c
int of_genpd_parse_idle_states(struct device_node *dn,
                               struct genpd_power_state **states, int *n);
```

**بتقرأ الـ `domain-idle-states` property من الـ DT وبتبني array من `genpd_power_state` structures.** كل state بيتم تحويله من الـ timing values الموجودة في DT subnode (entry-latency-us, exit-latency-us, min-residency-us) لـ nanoseconds.

| Parameter | الوصف |
|---|---|
| `dn` | الـ device node اللي فيه الـ idle states |
| `states` | pointer لـ pointer، بيتملي بـ allocated array |
| `n` | عدد الـ states اللي اتقرأت |

**Return:** `0` عند النجاح، error سالب عند الفشل.

**Key details:**
- الـ caller مسؤول عن تحرير الـ `states` array.
- ممكن تُمرر كـ `genpd->states` مباشرةً بعد الاستدعاء.

---

#### `of_genpd_sync_state`

```c
void of_genpd_sync_state(struct device_node *np);
```

**بتطلق الـ sync_state لـ genpd providers المرتبطة بـ node معين.** بتُستدعى عادةً لما يكون كل الـ consumers اللي بيستخدموا هذا الـ domain اتوصلوا لـ kernel، فيقدر الـ genpd يقرر هيطفش power domain ولا لأ.

---

### Group 3: Device Attach / Detach (High-Level API)

ده الـ API اللي بيستخدمه الـ bus drivers والـ device drivers للربط بـ PM domains. بتعمل abstract فوق الـ OF/ACPI layer.

---

#### `dev_pm_domain_attach`

```c
int dev_pm_domain_attach(struct device *dev, u32 flags);
```

**بتربط الـ device بـ PM domain الخاص بيه عن طريق الـ firmware (DT أو ACPI).** لو الـ device في DT وعنده `power-domains` property، بتلاقي الـ domain وبتربطهم.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device المراد ربطه |
| `flags` | `PD_FLAG_*` بتتحكم في السلوك (مثلاً `PD_FLAG_NO_DEV_LINK`) |

**Return:** `0` عند النجاح، `-EPROBE_DEFER` لو الـ provider لسا مسجّلش، error سالب تاني عند الفشل.

**Key details:**
- الـ device بيتربط بـ domain واحد بس (أول domain في DT).
- لو في أكتر من domain، استخدم `dev_pm_domain_attach_list()`.
- بيخلق device-link بين الـ device والـ domain provider ما لم يكن `PD_FLAG_NO_DEV_LINK`.

---

#### `dev_pm_domain_attach_by_id`

```c
struct device *dev_pm_domain_attach_by_id(struct device *dev,
                                          unsigned int index);
```

**بتربط الـ device بـ PM domain محدد بـ index في الـ `power-domains` list في DT.** بترجع virtual device يمثّل الـ domain — ده مفيد لما الـ device بيربط بـ multiple domains.

| Parameter | الوصف |
|---|---|
| `dev` | الـ consumer device |
| `index` | رقم الـ domain في الـ list الـ DT |

**Return:** `struct device*` للـ domain virtual device عند النجاح، `ERR_PTR` عند الفشل، `NULL` لو مش supported.

**Key details:**
- الـ virtual device اللي بترجعه لازم تعمله `dev_pm_domain_detach()` وقت الـ cleanup.
- استخدمها لو محتاج تتحكم في كل domain على حدة.

---

#### `dev_pm_domain_attach_by_name`

```c
struct device *dev_pm_domain_attach_by_name(struct device *dev,
                                            const char *name);
```

**نفس `dev_pm_domain_attach_by_id` لكن بتحدد الـ domain بـ اسم من الـ `power-domain-names` property في DT.**

| Parameter | الوصف |
|---|---|
| `dev` | الـ consumer device |
| `name` | اسم الـ domain كما هو في DT |

**Return:** `struct device*` عند النجاح، `ERR_PTR` أو `NULL` عند الفشل.

---

#### `dev_pm_domain_attach_list`

```c
int dev_pm_domain_attach_list(struct device *dev,
                              const struct dev_pm_domain_attach_data *data,
                              struct dev_pm_domain_list **list);
```

**بتربط device بكل الـ PM domains المذكورة في `data->pd_names` (أو كلهم لو `pd_names = NULL`).** بترجع `dev_pm_domain_list` جاهز للـ detach لاحقاً.

| Parameter | الوصف |
|---|---|
| `dev` | الـ consumer device |
| `data` | بيانات الـ attach: أسماء الـ domains، عددهم، والـ flags |
| `list` | output pointer — بتتملي بـ allocated list |

**Return:** عدد الـ domains المتصلة عند النجاح (>= 0)، error code سالب عند الفشل.

**Key details:**
- لو `PD_FLAG_ATTACH_POWER_ON` متضبط، بيشغّل كل domain بعد الـ attach.
- لو `PD_FLAG_REQUIRED_OPP` متضبط، بيعمل assign للـ required OPPs.
- الـ `list` struct بتتحلّص تلقائياً بـ `dev_pm_domain_detach_list()`.

**Pseudocode:**
```
allocate dev_pm_domain_list
for i in range(num_domains):
    if pd_names:
        attach by name pd_names[i]
    else:
        attach by index i
    store virtual device in list->pd_devs[i]
    if not PD_FLAG_NO_DEV_LINK:
        create device-link
    if PD_FLAG_REQUIRED_OPP:
        assign required OPP
    if PD_FLAG_ATTACH_POWER_ON:
        pm_runtime_get_sync(pd_dev)
return num_attached
```

---

#### `devm_pm_domain_attach_list`

```c
int devm_pm_domain_attach_list(struct device *dev,
                               const struct dev_pm_domain_attach_data *data,
                               struct dev_pm_domain_list **list);
```

**نفس `dev_pm_domain_attach_list` لكن مسجّلة في الـ devres system، فبتتنظّف تلقائياً لما الـ device يتشال.**

| Parameter | الوصف |
|---|---|
| `dev` | الـ consumer device |
| `data` | attach data |
| `list` | output list pointer |

**Return:** نفس `dev_pm_domain_attach_list`.

**Key details:**
- الـ cleanup callback بيستدعي `dev_pm_domain_detach_list()` تلقائياً.
- ده الـ preferred API في الـ drivers الحديثة — لا manual cleanup.

---

#### `dev_pm_domain_detach`

```c
void dev_pm_domain_detach(struct device *dev, bool power_off);
```

**بتفصل الـ device عن الـ PM domain المرتبط بيه.** لو `power_off = true`، بتحاول تطفي الـ domain بعد الفصل.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device المراد فصله |
| `power_off` | هل نطفّي الـ domain بعد الـ detach؟ |

**Key details:**
- بتستدعي `domain->detach()` callback لو موجود.
- بتُستدعى عادةً من `remove()` callback للـ driver.
- لو الـ device اتربط بـ `dev_pm_domain_attach_by_id/name`، لازم تعمل detach على الـ virtual device مش على الـ consumer.

---

#### `dev_pm_domain_detach_list`

```c
void dev_pm_domain_detach_list(struct dev_pm_domain_list *list);
```

**بتفصل كل الـ domains في الـ list وبتحرر الـ `dev_pm_domain_list` نفسه.**

| Parameter | الوصف |
|---|---|
| `list` | الـ list اللي رجعت من `dev_pm_domain_attach_list()` |

**Key details:**
- لو `PD_FLAG_DETACH_POWER_OFF`، بتعمل `pm_runtime_put_sync` على كل domain.
- بتعمل `device_unlink` لكل device-link اتعمل وقت الـ attach.

---

#### `genpd_dev_pm_attach`

```c
int genpd_dev_pm_attach(struct device *dev);
```

**الـ internal function اللي بتعمل actual attach للـ device من خلال الـ OF phandle.** بتُستخدم من `dev_pm_domain_attach()` ومن الـ bus drivers مباشرةً.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device المراد ربطه |

**Return:** `1` لو الـ attach نجح، `0` لو مفيش domain في DT، error سالب عند الفشل.

**Key details:**
- بتبحث عن `power-domains` في DT بتاع الـ device.
- `1` كـ return يعني "نجح الـ attach" — مش خطأ!
- `-EPROBE_DEFER` يعني الـ provider لسا مش موجود — ده normal في الـ probe sequencing.

---

#### `genpd_dev_pm_attach_by_id`

```c
struct device *genpd_dev_pm_attach_by_id(struct device *dev,
                                         unsigned int index);
```

**بتربط الـ device بـ domain بـ index محدد، وبتخلق virtual device يمثّل هذا الـ domain.** الـ virtual device ده بيكون له `dev_pm_domain` خاص بيه يعكس الـ genpd.

| Parameter | الوصف |
|---|---|
| `dev` | الـ consumer device |
| `index` | index الـ domain في الـ phandle list |

**Return:** `struct device*` للـ virtual domain device، `NULL` لو مفيش domains، `ERR_PTR` عند الفشل.

---

#### `genpd_dev_pm_attach_by_name`

```c
struct device *genpd_dev_pm_attach_by_name(struct device *dev,
                                           const char *name);
```

**نفس `genpd_dev_pm_attach_by_id` لكن بالاسم.** بتبحث في `power-domain-names` عن الـ index المناظر ثم بتستدعي `genpd_dev_pm_attach_by_id`.

| Parameter | الوصف |
|---|---|
| `dev` | الـ consumer device |
| `name` | اسم الـ domain |

**Return:** `struct device*` عند النجاح، `NULL` / `ERR_PTR` عند الفشل.

---

### Group 4: Performance State Management

الـ performance states بتخلي الـ genpd يتحكم في voltage/frequency لـ domain بالكامل بدل ما كل device يتحكم بنفسه.

---

#### `dev_pm_genpd_set_performance_state`

```c
int dev_pm_genpd_set_performance_state(struct device *dev, unsigned int state);
```

**بتطلب performance state معينة للـ genpd اللي الـ device ده ضمنه.** الـ genpd بيعمل aggregation لأعلى state مطلوبة من كل الـ devices قبل ما يبعتها للـ `set_performance_state` callback.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device الطالب |
| `state` | الـ state المطلوبة (platform-specific value) |

**Return:** `0` عند النجاح، error سالب عند الفشل.

**Key details:**
- `state = 0` معناه "مش محتاج performance"، وبيشيل طلب الـ device.
- الـ aggregation بيختار `max(all device states)` — last-man-standing approach.
- الـ `generic_pm_domain_data->performance_state` بيتحدّث لكل device.
- بيُستخدم مع OPP framework عادةً.

---

#### `dev_pm_domain_set_performance_state`

```c
int dev_pm_domain_set_performance_state(struct device *dev, unsigned int state);
```

**نفس السابق لكن بيمشي عبر `dev->pm_domain->set_performance_state` callback مباشرةً** بدل ما يكون genpd-specific.

---

### Group 5: Power Notifications & Wakeup Hints

---

#### `dev_pm_genpd_add_notifier`

```c
int dev_pm_genpd_add_notifier(struct device *dev, struct notifier_block *nb);
```

**بتسجّل notifier على الـ power on/off events للـ genpd اللي الـ device ده فيه.** الـ notifier بياخد events من `enum genpd_notication`: `GENPD_NOTIFY_PRE_OFF`، `GENPD_NOTIFY_OFF`، `GENPD_NOTIFY_PRE_ON`، `GENPD_NOTIFY_ON`.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device اللي جواه الـ genpd |
| `nb` | الـ notifier block اللي هيتعمله call |

**Return:** `0` عند النجاح، error سالب عند الفشل.

**Key details:**
- بيستخدم `raw_notifier_chain` — يعني بيشتغل في أي context.
- مفيد للـ drivers اللي محتاجة تعمل save/restore بعض الـ registers لو الـ domain هيتطفى.
- الـ notifier بيكون مربوط بالـ domain مش بالـ device — كل حد مسجّل بياخد events الـ domain كلها.

---

#### `dev_pm_genpd_remove_notifier`

```c
int dev_pm_genpd_remove_notifier(struct device *dev);
```

**بتشيل الـ notifier المسجّل للـ device من الـ genpd notifier chain.**

| Parameter | الوصف |
|---|---|
| `dev` | الـ device اللي سجّل الـ notifier |

**Return:** `0` عند النجاح، error سالب عند الفشل.

---

#### `dev_pm_genpd_set_next_wakeup`

```c
void dev_pm_genpd_set_next_wakeup(struct device *dev, ktime_t next);
```

**بتخبر الـ genpd governor بـ wakeup event القادم للـ device.** الـ governor بيستخدم المعلومة دي عشان يقرر هيدخل idle state عميقة ولا مش يستاهل.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device صاحب الـ wakeup |
| `next` | وقت الـ wakeup القادم كـ `ktime_t` (absolute) — `KTIME_MAX` لو مش معروف |

**Key details:**
- بتضبط `gpd_timing_data->next_wakeup` للـ device.
- الـ governor بعدين بيحسب `max_off_time_ns` بناءً على أقرب wakeup في الـ domain.
- بيشتغل مع `GENPD_FLAG_MIN_RESIDENCY`.

---

#### `dev_pm_genpd_get_next_hrtimer`

```c
ktime_t dev_pm_genpd_get_next_hrtimer(struct device *dev);
```

**بترجع أقرب hrtimer expiry في الـ genpd اللي الـ device ده فيه.** الـ governor بيحتفظ بالمعلومة دي في `genpd_governor_data->next_hrtimer`.

| Parameter | الوصف |
|---|---|
| `dev` | أي device في الـ domain |

**Return:** `ktime_t` للـ next hrtimer، أو `KTIME_MAX` لو مفيش.

---

#### `dev_pm_genpd_synced_poweroff`

```c
void dev_pm_genpd_synced_poweroff(struct device *dev);
```

**بتشعر الـ genpd إن consumer معين محتاج الـ poweroff يحصل synchronously** (مش في background work). بتضبط `genpd->synced_poweroff = true`.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device الطالب |

**Key details:**
- مهمة لـ CPUs وأي device حساس للـ timing في الـ poweroff.
- بتمنع إن الـ genpd يحط الـ power-off في work queue ويرجع — بيعملها sync.

---

### Group 6: Hardware Mode & Runtime Control

---

#### `dev_pm_genpd_set_hwmode`

```c
int dev_pm_genpd_set_hwmode(struct device *dev, bool enable);
```

**بتفعّل أو تعطّل الـ hardware-managed power mode للـ device.** في الـ hw mode، الـ hardware نفسه بيتحكم في الـ power domain بدون تدخل الـ software.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device |
| `enable` | `true` = فعّل hw mode، `false` = software mode |

**Return:** `0` عند النجاح، `-EOPNOTSUPP` لو الـ genpd مش بيدعم `set_hwmode_dev`، error سالب تاني عند الفشل.

**Key details:**
- بتستدعي `genpd->set_hwmode_dev()` callback.
- بتضبط `generic_pm_domain_data->hw_mode`.
- شائع في ARM SoCs مع hardware power controllers مثل PSCI.

---

#### `dev_pm_genpd_get_hwmode`

```c
bool dev_pm_genpd_get_hwmode(struct device *dev);
```

**بترجع حالة الـ hw mode للـ device.** بتستدعي `genpd->get_hwmode_dev()` callback لو موجود، وإلا بترجع `generic_pm_domain_data->hw_mode`.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device |

**Return:** `true` لو hw mode مفعّل، `false` لو لأ.

---

#### `dev_pm_genpd_rpm_always_on`

```c
int dev_pm_genpd_rpm_always_on(struct device *dev, bool on);
```

**بتضبط الـ device بحيث الـ genpd مش هيقدر يطفيه أثناء runtime PM.** بتضبط `generic_pm_domain_data->rpm_always_on`.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device |
| `on` | `true` = ابقى on دايماً في runtime، `false` = normal behavior |

**Return:** `0` عند النجاح، error سالب عند الفشل.

**Key details:**
- الـ domain ممكن يتطفى في system suspend — الـ flag ده runtime PM فقط.
- مختلف عن `GENPD_FLAG_RPM_ALWAYS_ON` اللي بينطبق على الـ domain كله.

---

#### `dev_pm_genpd_is_on`

```c
bool dev_pm_genpd_is_on(struct device *dev);
```

**بترجع حالة الـ domain اللي الـ device فيه — هل هو on أم off.**

| Parameter | الوصف |
|---|---|
| `dev` | الـ device |

**Return:** `true` لو `genpd->status == GENPD_STATE_ON`، `false` لو off.

**Key details:**
- snapshot قراءة بس — الـ status ممكن يتغير على طول.
- بيمسك الـ genpd lock عشان يقرأ الـ status بشكل آمن.

---

#### `dev_pm_domain_start`

```c
int dev_pm_domain_start(struct device *dev);
```

**بتبدأ تشغيل الـ device عبر الـ `domain->start()` callback لو موجود.**

| Parameter | الوصف |
|---|---|
| `dev` | الـ device |

**Return:** `0` عند النجاح أو لو مفيش domain، error سالب عند الفشل.

---

#### `dev_pm_domain_set`

```c
void dev_pm_domain_set(struct device *dev, struct dev_pm_domain *pd);
```

**بتضبط الـ `dev->pm_domain` يدوياً.** بتُستخدم لما الـ bus driver محتاج يربط device بـ domain بدون أوتوماتيك DT detection.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device |
| `pd` | الـ `dev_pm_domain` المراد تعيينه، أو `NULL` للإزالة |

---

### Group 7: System Sleep Helpers

---

#### `dev_pm_genpd_suspend`

```c
void dev_pm_genpd_suspend(struct device *dev);
```

**بتعمل suspend للـ genpd في سياق system sleep.** بتُستدعى من الـ bus suspend path لما الـ system بيروح في system suspend mode.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device المرتبط بالـ genpd |

**Key details:**
- `CONFIG_PM_GENERIC_DOMAINS_SLEEP` لازم يكون enabled.
- بتعمل save للـ state الحالي وبتطلب من الـ domain يدخل في الـ lowest idle state المناسبة.

---

#### `dev_pm_genpd_resume`

```c
void dev_pm_genpd_resume(struct device *dev);
```

**بتعمل resume للـ genpd من system sleep.** بتعيد الـ domain لحالته قبل الـ suspend.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device المرتبط بالـ genpd |

---

### Group 8: Inline Helpers & Casting Utilities

هذه الـ functions خفيفة جداً وبتُستخدم بكثرة في كل أنحاء الـ genpd subsystem لتسهيل الـ casting بين الـ struct types المتداخلة.

---

#### `pd_to_genpd`

```c
static inline struct generic_pm_domain *pd_to_genpd(struct dev_pm_domain *pd)
{
    return container_of(pd, struct generic_pm_domain, domain);
}
```

**بتحوّل `dev_pm_domain*` لـ `generic_pm_domain*` باستخدام `container_of`.** الـ `struct generic_pm_domain` بيحمل `domain` field من نوع `struct dev_pm_domain` — وده الـ entry point اللي الـ PM core بيشوف.

| Parameter | الوصف |
|---|---|
| `pd` | pointer للـ `dev_pm_domain` embedded |

**Return:** الـ enclosing `generic_pm_domain*`.

**Key details:**
- بتُستخدم في كل الـ genpd PM callbacks لأنها بتاخد `dev_pm_domain*` وبتحتاج الـ full genpd.
- لا memory allocation، لا locking — zero overhead.

---

#### `to_gpd_data`

```c
static inline struct generic_pm_domain_data *to_gpd_data(struct pm_domain_data *pdd)
{
    return container_of(pdd, struct generic_pm_domain_data, base);
}
```

**بتحوّل `pm_domain_data*` لـ `generic_pm_domain_data*`.** الـ `generic_pm_domain_data` بيحمل الـ `base` field من نوع `pm_domain_data`.

| Parameter | الوصف |
|---|---|
| `pdd` | pointer للـ base `pm_domain_data` |

**Return:** الـ `generic_pm_domain_data*`.

---

#### `dev_gpd_data`

```c
static inline struct generic_pm_domain_data *dev_gpd_data(struct device *dev)
{
    return to_gpd_data(dev->power.subsys_data->domain_data);
}
```

**بتجيب الـ `generic_pm_domain_data` مباشرةً من الـ device.** shortcut لسلسلة الـ pointers: `dev -> power.subsys_data -> domain_data -> generic_pm_domain_data`.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device |

**Return:** `generic_pm_domain_data*` للـ device.

**Key details:**
- لازم الـ device يكون مرتبط بـ genpd قبل الاستدعاء — وإلا `subsys_data` هيكون `NULL` وبيحصل crash.
- بتُستخدم كتير في الـ genpd internal functions للوصول لـ per-device timing data والـ performance state.

---

### Group 9: Statistics Helper

---

#### `pm_genpd_inc_rejected`

```c
void pm_genpd_inc_rejected(struct generic_pm_domain *genpd,
                           unsigned int state_idx);
```

**بتزوّد عداد الـ rejected transitions لـ power state معينة.** الـ governor لما يرفض دخول state معينة (لأن الـ wakeup قريب مثلاً)، بيستدعي الـ function دي لتسجيل الإحصائية في `genpd->states[state_idx].rejected`.

| Parameter | الوصف |
|---|---|
| `genpd` | الـ domain |
| `state_idx` | index الـ idle state اللي اترفضت |

**Key details:**
- الـ data دي ظاهرة في `/sys/kernel/debug/pm_genpd/` لـ debugging وتحسين الـ governor.
- مفيشة return value — مجرد counter increment.

---

#### `dev_to_genpd_dev`

```c
struct device *dev_to_genpd_dev(struct device *dev);
```

**بترجع الـ virtual device اللي بيمثّل الـ genpd اللي الـ device ده مرتبط بيه.** الـ genpd نفسه عنده `struct device` embedded — والـ function دي بتجيبه.

| Parameter | الوصف |
|---|---|
| `dev` | الـ consumer device |

**Return:** `struct device*` للـ genpd device عند النجاح، `ERR_PTR` عند الفشل.

**Key details:**
- مفيدة للوصول لـ sysfs attributes الخاصة بالـ domain.
- الـ genpd device ده هو نفسه `&genpd->dev`.

---

### ملاحظات على الـ Locking

```
genpd lock (mutex أو spinlock):
  - كل عمليات الـ read/write على genpd state
  - محدودة بنوع الـ GENPD_FLAG_IRQ_SAFE

gpd_list_lock (global):
  - يحمي الـ global list من كل الـ genpd instances
  - held فقط أثناء add/remove

device's pm lock:
  - held من الـ PM core أثناء الـ suspend/resume callbacks
```

### علاقة الـ APIs ببعض — ASCII Diagram

```
Driver Code
    │
    ├──► dev_pm_domain_attach()          ─── high-level, auto-detect
    ├──► dev_pm_domain_attach_by_id()    ─── multi-domain, by index
    ├──► dev_pm_domain_attach_by_name()  ─── multi-domain, by name
    └──► dev_pm_domain_attach_list()     ─── attach all at once
              │
              ▼
    genpd_dev_pm_attach()  (OF layer)
              │
              ▼
    of_genpd_add_device()
              │
              ▼
    pm_genpd_add_device()  (core)
              │
              ▼
    struct generic_pm_domain_data allocated & linked

Genpd Provider (SoC driver):
    pm_genpd_init()
    of_genpd_add_provider_simple() / of_genpd_add_provider_onecell()
    pm_genpd_add_subdomain()
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المسارات المهمة وطريقة قراءتها

**الـ Generic PM Domains (genpd)** بتعرض بياناتها في `/sys/kernel/debug/pm_genpd/`.

```bash
# اعرض كل الـ PM domains الموجودة في النظام
ls /sys/kernel/debug/pm_genpd/

# اقرأ حالة domain معين (مثلاً domain اسمه "pd_gpu")
cat /sys/kernel/debug/pm_genpd/pd_gpu/summary

# مثال على الـ output وتفسيره
# domain              status       children                           devices
# pd_gpu              off                                             gpu0
#   ↑                   ↑                                              ↑
#   اسم الـ domain    الحالة الحالية                           الأجهزة المتعلقة
```

```bash
# عرض إحصائيات الـ power states (latency, residency, usage)
cat /sys/kernel/debug/pm_genpd/pd_gpu/power_on_latency_ns
cat /sys/kernel/debug/pm_genpd/pd_gpu/power_off_latency_ns

# summary ملف شامل بيحتوي على:
# - اسم الـ domain + حالته (on/off)
# - عدد الأجهزة المتعلقة
# - الـ subdomains
# - وقت الـ accounting
cat /sys/kernel/debug/pm_genpd/summary
```

الـ `summary` بيطلع جدول زي ده:
```
domain              status       children                           devices
-------------------------------------------------------------------------------------
power-domain-0      on           power-domain-1                    spi@..., i2c@...
power-domain-1      off                                            gpu@...
```

#### 2. sysfs — المسارات المهمة

```bash
# حالة الـ runtime PM لجهاز معين
cat /sys/bus/platform/devices/MY_DEVICE/power/runtime_status
# output: active | suspended | error | unsupported

# عدد مرات الـ suspend والـ resume
cat /sys/bus/platform/devices/MY_DEVICE/power/runtime_suspended_time
cat /sys/bus/platform/devices/MY_DEVICE/power/runtime_active_time

# الـ autosuspend delay (بالـ milliseconds)
cat /sys/bus/platform/devices/MY_DEVICE/power/autosuspend_delay_ms

# هل الـ wakeup مفعّل؟
cat /sys/bus/platform/devices/MY_DEVICE/power/wakeup

# الـ PM domain اللي الجهاز ده متعلق بيه
cat /sys/bus/platform/devices/MY_DEVICE/power/pm_domain

# قوائم الـ device links (supplier/consumer)
ls /sys/bus/platform/devices/MY_DEVICE/consumer:*
ls /sys/bus/platform/devices/MY_DEVICE/supplier:*
```

```bash
# لو عايز تشوف performance state للـ domain
cat /sys/class/power_domain/pd_gpu/performance_state   # لو موجود
```

#### 3. ftrace — الـ tracepoints والـ events

```bash
# اعرض كل الـ PM-related trace events المتاحة
ls /sys/kernel/debug/tracing/events/power/

# الـ events الأساسية للـ genpd
cat /sys/kernel/debug/tracing/events/power/genpd_power_off/format
cat /sys/kernel/debug/tracing/events/power/genpd_power_on/format
```

```bash
# فعّل الـ genpd tracing كله
echo 1 > /sys/kernel/debug/tracing/events/power/genpd_power_off/enable
echo 1 > /sys/kernel/debug/tracing/events/power/genpd_power_on/enable
echo 1 > /sys/kernel/debug/tracing/events/power/genpd_governor_power_off/enable

# فعّل الـ runtime PM events بالكامل
echo 1 > /sys/kernel/debug/tracing/events/rpm/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل العملية اللي عايز تراقبها (suspend جهاز مثلاً)
echo 1 > /sys/bus/platform/devices/MY_DEVICE/power/control

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

مثال على الـ output:
```
# TASK-PID    CPU# |||||  TIMESTAMP  FUNCTION
# |           |    |||||     |         |
kworker/0:1-23 [000] d..1  123.456: genpd_power_off: pd_gpu state=1 latency=100000
kworker/0:1-23 [000] d..1  123.567: genpd_power_on:  pd_gpu state=0 latency=80000
```
- **latency**: وقت الـ power transition بالـ nanoseconds
- **state**: index الـ power state اللي اتحول ليه

```bash
# استخدم function_graph tracer لتتبع مسار الكود
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo genpd_power_off_work > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

#### 4. printk / dynamic debug

```bash
# فعّل الـ dynamic debug لكل ملفات الـ genpd
echo 'file drivers/base/power/domain.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/base/power/domain_governor.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/base/power/runtime.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل messages في الـ pm subsystem
echo 'module pm +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ debug messages في الـ dmesg
dmesg -w | grep -E 'genpd|pm_domain|runtime'

# أو فلتر بـ loglevel
dmesg --level=debug | grep genpd
```

```bash
# من kernel command line (في /etc/default/grub أو bootloader)
# GRUB_CMDLINE_LINUX="... pm_debug_messages"

# أو عن طريق sysctl
sysctl -w kernel.printk=8
```

#### 5. Kernel Config Options للـ Debugging

| Config | الوصف |
|--------|-------|
| `CONFIG_PM_GENERIC_DOMAINS` | الـ genpd subsystem الأساسي — لازم يكون مفعّل |
| `CONFIG_PM_GENERIC_DOMAINS_OF` | دعم الـ Device Tree للـ genpd |
| `CONFIG_PM_GENERIC_DOMAINS_SLEEP` | دعم الـ system sleep في الـ genpd |
| `CONFIG_PM_DEBUG` | يفعّل الـ PM debugging interface في sysfs |
| `CONFIG_PM_ADVANCED_DEBUG` | يضيف معلومات تفصيلية أكثر في sysfs (runtime counts) |
| `CONFIG_PM_SLEEP_DEBUG` | debugging لدورة الـ system suspend/resume |
| `CONFIG_PM_TRACE_RTC` | يستخدم الـ RTC لتتبع سبب crash أثناء الـ suspend |
| `CONFIG_PM_SLEEP` | دعم الـ system-wide sleep states |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل الـ dynamic debug framework |
| `CONFIG_TRACING` | البنية الأساسية للـ ftrace |
| `CONFIG_CPU_IDLE` | لازم للـ pm_domain_cpu_gov |
| `CONFIG_DEBUG_OBJECTS_WORK` | يكتشف الـ use-after-free في الـ work_struct |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في الـ genpd mutex/spinlock |
| `CONFIG_PROVE_LOCKING` | يتحقق من صحة ترتيب الـ locks |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'PM_GENERIC|PM_DEBUG|PM_ADVANCED'
# أو
grep -E 'PM_GENERIC|PM_DEBUG' /boot/config-$(uname -r)
```

#### 6. أدوات خاصة بالـ Subsystem

**الـ pm-utils و powertop:**
```bash
# powertop: يعرض الـ wakeups والـ power consumers
powertop --auto-tune  # للـ tuning التلقائي
powertop --csv=/tmp/power_report.csv  # تقرير مفصّل

# rtcwake: اختبار الـ suspend/resume
rtcwake -m mem -s 10  # suspend لـ RAM، صحّى بعد 10 ثواني

# pm-suspend: اختبار يدوي
pm-suspend  # suspend للـ system
```

**الـ cpupower لـ CPU domains:**
```bash
# عرض حالة الـ idle states (لو GENPD_FLAG_CPU_DOMAIN مفعّل)
cpupower idle-info
cpupower monitor

# تعطيل idle state معين للـ debugging
cpupower idle-set -d 2
```

**الـ devlink** (للـ network domains):
```bash
# لو الـ genpd مرتبط بـ network device
devlink dev info
devlink dev param show
```

**فحص الـ OPP table:**
```bash
# عرض الـ OPP table للـ domain
cat /sys/bus/platform/devices/MY_DEVICE/power/pm_qos_resume_latency_us
ls /sys/bus/platform/devices/MY_DEVICE/  # ابحث عن opp_table
```

#### 7. رسائل الـ Errors الشائعة — المعنى والحل

| رسالة الـ Kernel Log | المعنى | الحل |
|----------------------|--------|------|
| `genpd: Failed to power off: -ETIMEDOUT` | الـ power_off callback استغرق وقت أطول من المتوقع | افحص الـ hardware، زوّد الـ timeout في الـ driver |
| `genpd: Failed to power on: -EIO` | فشل في تشغيل الـ domain، غالباً hardware error | افحص الـ register values، تأكد من الـ clock/voltage |
| `WARNING: CPU: X PID: Y at drivers/base/power/domain.c` | انتهاك لـ assumption في الـ genpd code | اقرأ الـ WARN_ON condition، افحص الـ call stack |
| `genpd: parent domain not off` | حاول تطفي subdomain لكن الـ parent لسه شغّال | تأكد من ترتيب الـ shutdown، راجع الـ domain hierarchy |
| `IRQ safe domain not allowed on non-IRQ safe` | domain بـ `GENPD_FLAG_IRQ_SAFE` متعلق بـ parent بدون الـ flag ده | ضيف الـ flag للـ parent domain |
| `rpm_resume() failed: -EACCES` | الـ runtime PM مش مسموح له يشتغل | افحص `pm_runtime_enable()` وتسلسل الـ init |
| `Device is not allowed to suspend` | الـ device link بيمنع الـ suspend | افحص الـ DL_FLAG_RPM_ACTIVE وحالة الـ supplier |
| `could not attach device to domain` | فشل الـ attach — ممكن الـ domain مش موجود في الـ DT | تحقق من الـ `power-domains` property في الـ DT |
| `Failed to find power domain: -EPROBE_DEFER` | الـ genpd provider لسه ما اتسجّلش | الـ driver هيعيد الـ probe تاني — عادي، بس لو استمر افحص الـ provider |
| `genpd sync_state: domain not yet ready` | الـ sync_state اتكلم قبل ما كل الـ consumers يتعلقوا | افحص الـ `GENPD_FLAG_NO_SYNC_STATE` ومنطق الـ provider |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في الـ power_on callback — تحقق إن الـ domain مش شغّال أصلاً */
int my_pd_power_on(struct generic_pm_domain *domain)
{
    struct my_pd *pd = container_of(domain, struct my_pd, genpd);

    /* تحقق إن الحالة في الـ hardware تتوافق مع الـ software */
    WARN_ON(my_hw_is_powered_on(pd) && domain->status == GENPD_STATE_ON);

    if (my_hw_power_on(pd) < 0) {
        dump_stack(); /* اطبع الـ call chain كاملاً عشان تعرف مين طلب الـ power on */
        return -EIO;
    }
    return 0;
}
```

```c
/* في الـ attach_dev — تحقق إن الجهاز مش متعلق بالفعل */
int my_pd_attach_dev(struct generic_pm_domain *domain, struct device *dev)
{
    /* لو الجهاز عنده data هنا بالفعل، في مشكلة */
    WARN_ON(dev->power.subsys_data &&
            dev->power.subsys_data->domain_data);

    return 0;
}
```

```c
/* تحقق من الـ performance_state consistency */
int my_set_performance_state(struct generic_pm_domain *genpd, unsigned int state)
{
    /* الـ state مش المفروض يعدى الـ state_count */
    WARN_ON(state >= genpd->state_count);

    /* لو الـ domain طافي، مش المفروض تشتغل */
    WARN_ON(genpd->status == GENPD_STATE_OFF && state > 0);

    return my_hw_set_perf(genpd, state);
}
```

**أفضل نقاط لوضع `dump_stack()`:**
- في بداية `power_on` / `power_off` callbacks لو بتحاول تفهم مين بيطلبهم
- بعد فشل أي `pm_genpd_init()` أو `of_genpd_add_provider_*()`
- في الـ `set_performance_state` لو اتبعت request غير متوقع
- في الـ `attach_dev` / `detach_dev` لتتبع دورة حياة الأجهزة

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware بتتوافق مع الـ Kernel

```bash
# قرأ حالة الـ genpd من الـ kernel
cat /sys/kernel/debug/pm_genpd/MY_DOMAIN/summary
# لازم تتطابق مع الـ register اللي بيحدد حالة الـ power في الـ hardware

# مثال: Qualcomm GDSC (Global Distributed Switch Controller)
# الـ kernel بيقول domain off → الـ GDSCR register bit[0] لازم = 0

# مثال: Raspberry Pi (BCM2711 power domains)
# اقرأ حالة الـ domain من الـ VC4 firmware
vcgencmd get_throttled
vcgencmd measure_clock arm
```

```bash
# تحقق من الـ clock state (لو GENPD_FLAG_PM_CLK مفعّل)
# الـ domain المطفي لازم تبقى ساعاته مفصولة
cat /sys/kernel/debug/clk/MY_CLK/clk_enable_count
cat /sys/kernel/debug/clk/MY_CLK/clk_rate
```

#### 2. قراءة الـ Registers — devmem2 و /dev/mem و io

```bash
# devmem2: أسهل طريقة لقراءة register
# (محتاج CONFIG_DEVMEM=y وممكن يتعطّل بـ CONFIG_STRICT_DEVMEM)
devmem2 0xFD440000 w  # قرأ word (32-bit) من address معين

# مثال: قرأ GDSC register في Snapdragon (CAMSS power domain)
# GCC_CAMSS_TOP_GDSCR = 0x00058004 (relative to GCC base)
devmem2 $((0x00100000 + 0x00058004)) w
# الـ bit[0] = 1 → domain on, bit[0] = 0 → domain off

# /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((0xFD440000 / 4)) 2>/dev/null | hexdump -C

# io tool (من الـ package io أو busybox)
io -4 -r 0xFD440000  # قرأ 32-bit register
```

```bash
# استخدم /proc/iomem الأول عشان تعرف الـ base addresses
cat /proc/iomem | grep -i power
cat /proc/iomem | grep -i gdsc
```

#### 3. Logic Analyzer / Oscilloscope

**نقاط القياس:**
- **Power rails**: قيس الـ voltage على خط الطاقة بتاع الـ domain (VDDCX, VDDMX, VDDA, إلخ)
- **Enable signals**: الـ pin اللي بيتحكم في الـ LDO أو DC-DC converter
- **Clock lines**: تأكد إن الـ clock بيتوقف فعلاً لما الـ domain يتطفى
- **Reset lines**: بعض الـ domains بتحتاج reset قبل ما تتشغّل

**طريقة الاستخدام مع الـ genpd:**

```
زمن →
────────────────────────────────────────────────
VDDGPU (rail)  ████████▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄████████
                                          ↑ 2.5ms  ← latency القياسية لازم تتوافق
PM_ENABLE (pin)████████▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄████████
GPU_CLK         ████████▄▄▄▄▄▄▄▄▄▄▄▄████████████
                        ↑power_off     ↑power_on
```

- **الـ latency المقاسة** لازم تتطابق مع `power_off_latency_ns` و `power_on_latency_ns` في `struct genpd_power_state`
- لو في فرق كبير → اضبط القيم في الـ DT أو الـ driver

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | الأعراض في الـ Kernel Log | الأسلوب |
|---------|--------------------------|--------|
| الـ voltage rail ما بيجيش لما الـ domain يتفتح | `power_on: -EIO` بعد timeout | راقب الـ power rail بـ oscilloscope |
| الـ clock ما بيتوقفش لما الـ domain يتطفى | ارتفاع في استهلاك الطاقة في `powertop` | افحص `clk_enable_count` |
| الـ domain بيتطفى قبل ما الجهاز يخلّص شغله | `rpm_suspend` fails with `-EBUSY` | زوّد الـ autosuspend_delay_ms |
| مشكلة في تسلسل الـ power (parent قبل child) | `parent domain not off` warning | راجع الـ DT hierarchy |
| الـ isolation cells مش شغّالة صح | بيانات غلط بترجع من جهاز مطفي | افحص الـ hardware isolation logic |

#### 5. Device Tree Debugging

```bash
# اعرض كل الـ DT nodes اللي بتستخدم power-domains
grep -r "power-domains" /sys/firmware/devicetree/base/ 2>/dev/null | head -20

# أو الأوضح
find /sys/firmware/devicetree/base -name "power-domains" | while read f; do
    echo "=== Node: $(dirname $f) ==="
    hexdump -C "$f"
done
```

```bash
# تحقق إن الـ genpd provider سجّل نفسه صح
cat /sys/kernel/debug/pm_genpd/summary | grep "MY_DOMAIN"

# لو ما لقيتوش → الـ provider ما اتسجّلش (ابحث عن EPROBE_DEFER في dmesg)
dmesg | grep -E 'power.domain|genpd|EPROBE_DEFER'
```

**مثال DT صح:**
```dts
/* الـ provider */
power: power-controller@fd440000 {
    compatible = "vendor,my-power-ctrl";
    reg = <0xfd440000 0x1000>;
    #power-domain-cells = <1>;  /* رقم واحد = index */
};

/* الـ consumer */
gpu@fc400000 {
    compatible = "vendor,my-gpu";
    reg = <0xfc400000 0x10000>;
    power-domains = <&power 0>;  /* domain index 0 */
    power-domain-names = "core"; /* اسم اختياري */
};
```

```bash
# تحقق إن الـ phandle بيشاور على الـ provider الصح
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "power-domains"

# استخدم fdtdump لو عندك الـ dtb file
fdtdump /boot/dtbs/$(uname -r)/my-board.dtb | grep -A10 power-domain
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**الحالة العامة للـ genpd:**
```bash
#!/bin/bash
# genpd_status.sh — عرض شامل لحالة كل الـ PM domains

echo "=== PM Domain Summary ==="
cat /sys/kernel/debug/pm_genpd/summary

echo ""
echo "=== Runtime PM Status per Device ==="
for dev in /sys/bus/platform/devices/*/power; do
    name=$(basename $(dirname $dev))
    status=$(cat $dev/runtime_status 2>/dev/null)
    usage=$(cat $dev/runtime_usage 2>/dev/null)
    printf "%-40s status=%-12s usage=%s\n" "$name" "$status" "$usage"
done
```

**تفعيل الـ tracing الكامل:**
```bash
#!/bin/bash
# enable_genpd_trace.sh

TRACE=/sys/kernel/debug/tracing

echo nop > $TRACE/current_tracer
echo > $TRACE/trace

# فعّل كل PM events
for event in $TRACE/events/power/*/enable; do
    echo 1 > "$event"
done

# فعّل RPM events
echo 1 > $TRACE/events/rpm/enable 2>/dev/null

echo 1 > $TRACE/tracing_on
echo "Tracing enabled. Run your test, then: cat $TRACE/trace"
```

**قراءة وتفسير الـ trace:**
```bash
# بعد تشغيل اختبارك
cat /sys/kernel/debug/tracing/trace | grep -E 'genpd|rpm' | head -50

# مثال على الـ output وتفسيره:
# kworker/u8:2-89  [001] ....  45.123456: genpd_power_off: name=pd_gpu state=1 latency=120000
#   ↑ الـ task اللي نفّذ الـ power off
#                                          ↑ اسم الـ domain
#                                                       ↑ رقم الـ state المستهدف
#                                                              ↑ الوقت بالـ ns
```

**فحص حالة الـ device links:**
```bash
#!/bin/bash
# show_device_links.sh — عرض الـ device links المرتبطة بالـ PM domains

echo "=== Device Links (PM-related) ==="
for link in /sys/bus/platform/devices/*/supplier:*/; do
    consumer=$(basename $(dirname $(dirname $link)))
    supplier=$(basename $link | sed 's/supplier://')
    flags=$(cat ${link}d_flags 2>/dev/null)
    status=$(cat ${link}status 2>/dev/null)
    echo "$consumer → $supplier | flags=$flags status=$status"
done
```

**فحص الـ performance states:**
```bash
# عرض الـ performance state الحالي لكل domain
cat /sys/kernel/debug/pm_genpd/summary | awk 'NR>2 {print $1, "perf_state:", $4}'

# ضبط الـ performance state يدوياً (للـ testing)
# dev_pm_genpd_set_performance_state() من الـ kernel API
# من الـ userspace عبر OPP:
cat /sys/bus/platform/devices/MY_DEV/opp_table  # لو موجود
```

**فحص الـ genpd power states statistics:**
```bash
#!/bin/bash
# genpd_stats.sh — إحصائيات الـ power states

for domain_dir in /sys/kernel/debug/pm_genpd/*/; do
    domain=$(basename $domain_dir)
    echo "=== Domain: $domain ==="

    # الحالة الحالية
    status=$(grep -w "$domain" /sys/kernel/debug/pm_genpd/summary 2>/dev/null | awk '{print $2}')
    echo "Status: $status"

    # لو في stats files
    for stat in power_on_latency_ns power_off_latency_ns residency_ns; do
        val=$(cat ${domain_dir}${stat} 2>/dev/null)
        [ -n "$val" ] && echo "$stat: $val"
    done
    echo ""
done
```

**اختبار سريع للـ suspend/resume:**
```bash
# اختبر الـ runtime suspend يدوياً لجهاز معين
echo 0 > /sys/bus/platform/devices/MY_DEVICE/power/autosuspend_delay_ms
echo auto > /sys/bus/platform/devices/MY_DEVICE/power/control

# انتظر الـ autosuspend ثم افحص الحالة
sleep 2
cat /sys/bus/platform/devices/MY_DEVICE/power/runtime_status
# المفروض يطلع: suspended

# أجبر الـ resume
echo on > /sys/bus/platform/devices/MY_DEVICE/power/control
cat /sys/bus/platform/devices/MY_DEVICE/power/runtime_status
# المفروض يطلع: active
```

**dynamic debug لمتابعة الـ genpd بالتفصيل:**
```bash
# فعّل كل debug messages في الـ genpd core
echo 'file drivers/base/power/domain.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/base/power/runtime.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# p = print, f = function name, l = line number, m = module name, t = thread ID

# راقب الـ output في الـ real-time
dmesg -w

# أوقف الـ debug بعد ما تخلّص
echo 'file drivers/base/power/domain.c -p' > /sys/kernel/debug/dynamic_debug/control
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART مش بيصحى من الـ suspend على RK3562

#### العنوان
**الـ UART بيتعطل بعد الـ system suspend على industrial gateway بـ RK3562**

#### السياق
شركة بتبني industrial gateway بـ RK3562 — الجهاز بيتكلم مع حساسات عبر UART. الـ gateway بيدخل `suspend-to-RAM` كل فترة لتوفير الطاقة. بعد الـ resume، الـ UART مش بيستجاوبش خالص.

#### المشكلة
الـ UART driver بيعمل `pm_runtime_put()` قبل الـ suspend، فالـ genpd بتاعته (`pd_uart` مثلاً) بتتقفل. لما بيصحى، الـ domain مش بيرجع يتشغل قبل ما الـ driver يحاول يكتب في الـ registers — فبيحصل garbage أو deadlock.

#### التحليل
في `pm_domain.h`، الـ `generic_pm_domain` عنده:

```c
/* pm_domain.h - line 218 */
int (*power_off)(struct generic_pm_domain *domain);
int (*power_on)(struct generic_pm_domain *domain);
```

الـ genpd بتستخدم `power_on` لما device محتاجه. لو الـ timing مش صح (latency عالية في `genpd_power_state`):

```c
/* pm_domain.h - lines 177-189 */
struct genpd_power_state {
    s64 power_off_latency_ns;
    s64 power_on_latency_ns;
    s64 residency_ns;   /* minimum time to justify power-off */
    ...
};
```

لو `residency_ns` صغيرة أوي، الـ genpd بتقفل الـ domain حتى لو هيتشغل تاني بسرعة. وبعدين الـ `power_on_latency_ns` كانت مضبوطة بـ 0 في الـ DT — مش عاطياش وقت كافي للـ clock يستقر.

الـ governor (`simple_qos_governor`) بيتخذ قرار power-off بناءً على:
```c
/* pm_domain.h - lines 155-159 */
struct dev_power_governor {
    bool (*power_down_ok)(struct dev_pm_domain *domain);
    bool (*suspend_ok)(struct device *dev);
    ...
};
```

**الـ `gpd_timing_data`** بتوضح المشكلة:
```c
/* pm_domain.h - lines 271-278 */
struct gpd_timing_data {
    s64 suspend_latency_ns;
    s64 resume_latency_ns;
    s64 effective_constraint_ns;
    ktime_t next_wakeup;
    bool constraint_changed;
    bool cached_suspend_ok;
};
```

`effective_constraint_ns` اتحسبت غلط لأن `resume_latency_ns` في الـ DT كانت 0.

#### الحل

**1. صلح الـ DT لـ idle states:**
```dts
/* rk3562-gateway.dts */
power-domain@RK3562_PD_UART {
    domain-idle-states = <&UART_PD_OFF>;
};

UART_PD_OFF: domain-idle-state {
    compatible = "domain-idle-state";
    entry-latency-us = <500>;   /* was 0 */
    exit-latency-us  = <1000>;  /* was 0 */
    min-residency-us = <10000>;
};
```

**2. تحقق من الـ states عبر debugfs:**
```bash
# شوف الـ genpd states والـ latency
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary

# شوف الـ power state للـ UART domain
cat /sys/kernel/debug/pm_genpd/pd_uart/current_state
```

**3. لو الـ driver نفسه محتاج ينتظر:**
```c
/* في الـ UART driver resume */
dev_pm_genpd_resume(dev);  /* force genpd resume أول */
```

#### الدرس المستفاد
الـ `genpd_power_state` بتحكم متى يستاهل الـ domain يتقفل. `residency_ns = 0` معناها الـ genpd ممكن تقفل الـ domain حتى لو هييجي request بعد ميكروثانية — دايماً اضبط latency values صح في الـ DT.

---

### السيناريو 2: أداء SPI بيتدهور على Allwinner H616 في Android TV Box

#### العنوان
**الـ SPI بيشتغل بـ performance state منخفض على Allwinner H616 أثناء تشغيل الـ video**

#### السياق
Android TV box بـ Allwinner H616. الـ SPI بيتكلم مع flash chip للـ firmware updates. أثناء تشغيل 4K video، السرعة بتاعة الـ SPI بتنزل بشكل ملحوظ وبيحصل timeout.

#### المشكلة
الـ SPI موجود في power domain مع devices تانية. الـ `performance_state` بتاع الـ domain بيتحدد بأعلى طلب من الـ devices. لما الـ video decoder شغال، هو بيطلب performance state عالي — بس الـ SPI driver مش بيطلبش أي state، فالـ aggregation بتاعة الـ genpd ممكن تكون مش بتشمله صح.

#### التحليل
في `generic_pm_domain`:

```c
/* pm_domain.h - line 213 */
unsigned int performance_state; /* Aggregated max performance state */
```

الـ genpd بتعمل aggregate للـ max performance state من كل device في الـ domain:

```c
/* pm_domain.h - lines 285-298 */
struct generic_pm_domain_data {
    ...
    unsigned int performance_state; /* ده اللي الـ device طلبه */
    unsigned int default_pstate;    /* الـ default لو مطلبش */
    unsigned int rpm_pstate;        /* الـ state أثناء runtime PM */
    unsigned int opp_token;
    ...
};
```

المشكلة: الـ SPI driver مش بيستدعي `dev_pm_genpd_set_performance_state()` — فـ `performance_state` بتاعته = 0. والـ `default_pstate` مش اتضبطش في الـ DT. الـ genpd بتاخد الـ max وهو performance state بتاع الـ decoder — مش مشكلة في حد ذاتها — لكن الـ `set_performance_state` callback مش موجود في الـ H616 genpd driver.

```c
/* pm_domain.h - lines 222-223 */
int (*set_performance_state)(struct generic_pm_domain *genpd,
                             unsigned int state);
```

الـ callback = NULL — فالـ genpd بتعمل aggregate بس مش بتطبقه على الـ hardware.

#### الحل

**1. اتحقق إن الـ SoC genpd driver بيimplementوا الـ callback:**
```bash
# شوف لو fi performance states في الـ genpd
cat /sys/kernel/debug/pm_genpd/pd_spi/performance_state
grep -r "set_performance_state" /drivers/clk/sunxi-ng/
```

**2. في الـ SPI driver، اطلب الـ performance state صح:**
```c
/* في الـ probe أو resume */
ret = dev_pm_genpd_set_performance_state(dev, SPI_PERF_STATE_HIGH);
if (ret && ret != -EOPNOTSUPP)
    dev_warn(dev, "failed to set perf state: %d\n", ret);
```

**الـ API من `pm_domain.h`:**
```c
/* pm_domain.h - line 323 */
int dev_pm_genpd_set_performance_state(struct device *dev, unsigned int state);
```

**3. ضيف required-opps في الـ DT:**
```dts
&spi0 {
    required-opps = <&spi_opp_high>;
    power-domains = <&pd H616_PD_SPI>;
};
```

#### الدرس المستفاد
الـ `performance_state` في الـ `generic_pm_domain` هو aggregation فقط — لازم الـ SoC genpd driver يimplementوا `set_performance_state` callback عشان يطبقه على الـ hardware فعلاً. لو الـ callback NULL، الـ aggregation بلا فايدة.

---

### السيناريو 3: kernel panic أثناء الـ boot على STM32MP1 بسبب IRQ-safe domain

#### العنوان
**Kernel panic: "Trying to use non-IRQ-safe domain from IRQ context" على STM32MP1 IoT sensor**

#### السياق
IoT sensor board بـ STM32MP1. الـ I2C controller بيقرا بيانات من sensor في interrupt context (IRQ handler). الـ kernel بيـ panic أثناء الـ boot مع رسالة تتكلم عن IRQ-safe domains.

#### المشكلة
الـ I2C driver بيستخدم `pm_runtime_get()` داخل الـ interrupt handler. الـ genpd بتاعة الـ I2C مش `GENPD_FLAG_IRQ_SAFE` — فلما الـ genpd تحاول تشغل الـ domain من الـ IRQ context، الـ kernel يكتشف إن الـ mutex الـ genpd بيستخدمه مش مناسب للـ atomic context.

#### التحليل
الـ flag system في `pm_domain.h`:

```c
/* pm_domain.h - lines 73-81 */
/*
 * GENPD_FLAG_IRQ_SAFE: This informs genpd that its backend callbacks,
 *   ->power_on|off(), doesn't sleep. Hence, these can be invoked from
 *   within atomic context...
 *   Note that, a genpd having this flag set, requires its
 *   masterdomains to also have it set.
 */
#define GENPD_FLAG_IRQ_SAFE  (1U << 1)
```

الـ genpd بتستخدم lock_ops مختلفة حسب الـ flag:

```c
/* pm_domain.h - lines 241-252 */
const struct genpd_lock_ops *lock_ops;
union {
    struct mutex mlock;         /* used when NOT IRQ_SAFE */
    struct {
        spinlock_t slock;       /* used when IRQ_SAFE */
        unsigned long lock_flags;
    };
    struct {
        raw_spinlock_t raw_slock;
        unsigned long raw_lock_flags;
    };
};
```

لما الـ driver بيعمل `pm_runtime_get()` من IRQ context والـ domain مش IRQ_SAFE، الـ genpd تحاول تاخد الـ `mlock` (mutex) — وده مستحيل في atomic context.

**ملاحظة مهمة من الـ header:**
```
Note that, a genpd having this flag set, requires its masterdomains to also have it set.
```
يعني لو ضفت الـ flag على الـ I2C domain بس مش على الـ parent، هيحصل panic تاني.

#### الحل

**خيار 1: فعّل الـ GENPD_FLAG_IRQ_SAFE في الـ STM32MP1 genpd driver:**
```c
/* في drivers/soc/st/stm32mp1_power.c */
static struct generic_pm_domain stm32_i2c_domain = {
    .name  = "i2c_pd",
    .flags = GENPD_FLAG_IRQ_SAFE,   /* add this */
    .power_off = stm32_pd_power_off,
    .power_on  = stm32_pd_power_on,
};

/* لازم الـ parent كمان */
static struct generic_pm_domain stm32_core_domain = {
    .name  = "core_pd",
    .flags = GENPD_FLAG_IRQ_SAFE,   /* required by child */
    ...
};
```

**خيار 2 (أبسط): خلي الـ I2C driver مييجيش يعمل rpm_get في IRQ context:**
```c
/* في الـ I2C ISR */
/* BAD: pm_runtime_get_sync(dev);  */

/* GOOD: استخدم threaded IRQ */
return IRQ_WAKE_THREAD;
```

**للتشخيص:**
```bash
# شوف flags بتاعة الـ domains
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary | grep -A2 i2c

# شوف لو الـ domain IRQ-safe
cat /sys/kernel/debug/pm_genpd/i2c_pd/flags
```

#### الدرس المستفاد
`GENPD_FLAG_IRQ_SAFE` مش بس option — ضروري لو أي device في الـ domain بيشتغل من IRQ context. والأهم: لازم كل الـ parent domains تحمل نفس الـ flag، وإلا الـ kernel هيرفض الـ operation.

---

### السيناريو 4: USB بيقطع على i.MX8 بعد الـ system suspend بسبب sync_state

#### العنوان
**الـ USB controller بيختفي من الـ sysfs بعد الـ resume على i.MX8 automotive ECU**

#### السياق
Automotive ECU بـ i.MX8MQ. الـ USB بيتستخدم لتحديث الـ firmware. بعد الـ system suspend والـ resume، الـ USB controller مش بييجي في `/sys/bus/usb/devices` — المشكلة هتأخر الـ production.

#### المشكلة
الـ USB power domain عنده `stay_on = true` أثناء الـ boot (لأنه كان on وقت init). الـ genpd بتحافظ عليه on لحد ما `sync_state` يتنادى. لكن بعد الـ resume، الـ genpd شوفت إن مفيش consumer active وعملت power-off للـ domain — والـ USB controller مش عنده proper `power_on` re-initialization.

#### التحليل
في `generic_pm_domain`:

```c
/* pm_domain.h - lines 215-217 */
bool synced_poweroff;   /* A consumer needs a synced poweroff */
bool stay_on;           /* Stay powered-on during boot. */
enum genpd_sync_state sync_state; /* How sync_state is managed. */
```

الـ `sync_state` enum:
```c
/* pm_domain.h - lines 149-153 */
enum genpd_sync_state {
    GENPD_SYNC_STATE_OFF = 0,
    GENPD_SYNC_STATE_SIMPLE,
    GENPD_SYNC_STATE_ONECELL,
};
```

لما الـ system بييجي من الـ suspend، الـ genpd `sync_state` اتشغل مرة تانية وقرر إن الـ USB domain مش محتاج يفضل on — وعمل power-off. الـ USB driver مش لاحظ ده لأن الـ `power_notifiers` مش اشتركت فيه.

الـ notifier system:
```c
/* pm_domain.h - line 220 */
struct raw_notifier_head power_notifiers; /* Power on/off notifiers */
```

الـ USB driver مش عمل `dev_pm_genpd_add_notifier()` — فما اتبلغش بالـ power-off.

```c
/* pm_domain.h - line 324 */
int dev_pm_genpd_add_notifier(struct device *dev, struct notifier_block *nb);
```

#### الحل

**1. اشترك في الـ power notifier في الـ USB driver:**
```c
/* في الـ USB driver probe */
static int usb_genpd_notify(struct notifier_block *nb,
                             unsigned long action, void *data)
{
    struct my_usb_dev *udev = container_of(nb, struct my_usb_dev, nb);

    switch (action) {
    case GENPD_NOTIFY_PRE_ON:
        /* جهز الـ clocks قبل الـ power-on */
        break;
    case GENPD_NOTIFY_ON:
        /* reinitialize الـ controller */
        usb_reinit_controller(udev);
        break;
    }
    return NOTIFY_OK;
}

/* في probe */
udev->nb.notifier_call = usb_genpd_notify;
dev_pm_genpd_add_notifier(dev, &udev->nb);
```

**2. لو المشكلة في الـ sync_state نفسه، استخدم الـ flag:**
```c
/* pm_domain.h - lines 119-123 */
/*
 * GENPD_FLAG_NO_STAY_ON: allow power-off without waiting for sync_state.
 */
#define GENPD_FLAG_NO_STAY_ON  (1U << 10)
```

أو العكس، ضيف `GENPD_FLAG_NO_SYNC_STATE` لو الـ driver نفسه بيتعامل مع sync_state:
```c
#define GENPD_FLAG_NO_SYNC_STATE (1U << 9)
```

**3. debug السيناريو:**
```bash
# شوف sync_state للـ USB domain
cat /sys/kernel/debug/pm_genpd/usb_pd/sync_state

# شوف لو في notifiers مسجلة
cat /sys/kernel/debug/pm_genpd/usb_pd/devices
```

#### الدرس المستفاد
الـ `stay_on` flag في `generic_pm_domain` بتحمي الـ domain أثناء الـ boot بس — مش بعد الـ resume. الـ driver اللي محتاج re-initialization بعد كل power cycle لازم يستخدم `dev_pm_genpd_add_notifier()` عشان يعرف بـ `GENPD_NOTIFY_ON` ويعيد تهيئة الـ hardware.

---

### السيناريو 5: HDMI مش بيظهر على AM62x بعد إضافة subdomain

#### العنوان
**الـ HDMI بيتعطل بعد إضافة display subdomain في الـ DT على AM62x custom board**

#### السياق
Custom board للـ digital signage بـ AM62x (Texas Instruments). Engineer أضاف power domain جديدة للـ display subsystem كـ subdomain تحت الـ main `pd_main`. بعد التغيير، الـ HDMI مش بيطلع صورة وبيظهر في الـ dmesg: `failed to power on domain`.

#### المشكلة
Engineer استخدم `pm_genpd_add_subdomain()` لكن نسي إنه لو الـ parent domain على `GENPD_FLAG_IRQ_SAFE`، لازم الـ child كمان يكون `IRQ_SAFE`. كمان، الـ `gpd_link` بين الـ parent والـ child اتعمل بعد ما الـ HDMI device اتـ attach — فالـ performance state aggregation مش شغالة صح.

#### التحليل
الـ `gpd_link` structure:

```c
/* pm_domain.h - lines 260-269 */
struct gpd_link {
    struct generic_pm_domain *parent;
    struct list_head parent_node;
    struct generic_pm_domain *child;
    struct list_head child_node;

    /* Sub-domain's per-master domain performance state */
    unsigned int performance_state;
    unsigned int prev_performance_state;
};
```

الـ genpd لما تحاول تشغل الـ child domain، بتحاول تشغل الـ parent أولاً (last-man-standing algorithm). الـ parent على `GENPD_FLAG_IRQ_SAFE` لكن الـ HDMI child مش كده — الـ genpd ترفض الـ operation.

من الـ header:
```c
/* pm_domain.h - lines 89-97 */
/*
 * GENPD_FLAG_CPU_DOMAIN: ...backend driver must comply with the
 *   last-man-standing algorithm, for the CPUs in the PM domain.
 */
#define GENPD_FLAG_CPU_DOMAIN  (1U << 4)
```

كمان، الـ `pm_genpd_add_subdomain()` API:
```c
/* pm_domain.h - line 313 */
int pm_genpd_add_subdomain(struct generic_pm_domain *genpd,
                            struct generic_pm_domain *subdomain);
```

لازم يتنادى قبل أي device يتـ attach للـ child domain. لو اتنادى بعدين، الـ `gpd_link` بيتعمل لكن الـ parent mبيعرفش يعمل aggregate للـ performance state صح.

#### الحل

**1. ضبط تسلسل الـ initialization في الـ AM62x genpd driver:**
```c
/* في drivers/soc/ti/ti_sci_pm_domains.c أو مشابه */

/* أولاً: عرّف الـ domains */
static struct generic_pm_domain am62x_main_pd = {
    .name  = "pd_main",
    .flags = 0,   /* NOT IRQ_SAFE */
    .power_off = am62x_pd_power_off,
    .power_on  = am62x_pd_power_on,
};

static struct generic_pm_domain am62x_display_pd = {
    .name  = "pd_display",
    .flags = 0,   /* match parent */
    .power_off = am62x_pd_power_off,
    .power_on  = am62x_pd_power_on,
};

/* ثانياً: init كل domain */
pm_genpd_init(&am62x_main_pd, &simple_qos_governor, true);
pm_genpd_init(&am62x_display_pd, &simple_qos_governor, true);

/* ثالثاً: اربط الـ subdomain قبل أي device attach */
pm_genpd_add_subdomain(&am62x_main_pd, &am62x_display_pd);

/* رابعاً: سجل الـ OF provider */
of_genpd_add_provider_onecell(np, &am62x_genpd_data);
```

**2. الـ DT:**
```dts
/* am62x-signage.dts */
power-domains-controller {
    pd_main: pd@0 {
        #power-domain-cells = <0>;
    };
    pd_display: pd@1 {
        #power-domain-cells = <0>;
        power-domains = <&pd_main>;  /* declares parent-child */
    };
};

&hdmi {
    power-domains = <&pd_display>;  /* child domain only */
};
```

**3. تشخيص المشكلة:**
```bash
# شوف hierarchy الـ domains
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary

# المفروض يظهر:
# pd_main      on  1 subdomain
#   pd_display on  1 device
#     hdmi     ...

# شوف الـ device count والـ sd_count
cat /sys/kernel/debug/pm_genpd/pd_main/devices
cat /sys/kernel/debug/pm_genpd/pd_display/devices
```

**4. تحقق من الـ power state عبر sysfs:**
```bash
# لو الـ HDMI domain مش شغال
cat /sys/kernel/debug/pm_genpd/pd_display/current_state

# شوف الـ power_on latency
cat /sys/kernel/debug/pm_genpd/pd_display/power_on_delay_ns
```

#### الدرس المستفاد
الـ subdomain relationship عبر `pm_genpd_add_subdomain()` لازم تتعمل **قبل** أي `of_genpd_add_provider_*()` وقبل أي device يتـ probe. الترتيب في الـ `init` function بيفرق جداً. كمان، لما بتضيف child domain، check إن flags بتاعتها متوافقة مع الـ parent — خصوصاً `GENPD_FLAG_IRQ_SAFE`.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأول لتتبع تطور الـ Linux kernel، والمقالات دي بتغطي تطور الـ `pm_domain` / `genpd` من البداية لحد المرحلة الحالية:

| المقال | الموضوع |
|--------|---------|
| [PM / Domains: Support for generic I/O PM domains](https://lwn.net/Articles/449302/) | أول patchset رسمي أضاف الـ generic I/O PM domains |
| [Dealing with complexity: power domains and asymmetric multiprocessing](https://lwn.net/Articles/449585/) | شرح البنية الهرمية للـ power domains والـ AMP |
| [Generic Device Tree based power domain look-up](https://lwn.net/Articles/580148/) | إضافة DT bindings للـ genpd وربط الـ devices بالـ providers |
| [PM / Domains: Add support for explicit control of PM domains](https://lwn.net/Articles/718263/) | دعم ربط أكتر من PM domain بنفس الـ device |
| [CPU PM domains](https://lwn.net/Articles/716300/) | تطبيق الـ genpd على الـ CPUs والـ clusters |
| [ARM: PM / Domains: Generic PM domains for CPUs/Clusters](https://lwn.net/Articles/653579/) | تفاصيل دعم الـ last-man-standing algorithm للـ CPUs |
| [PM / Domains: Performance state support](https://lwn.net/Articles/736017/) | إضافة مفهوم الـ performance states للـ genpd |
| [PM /Domain/OPP: Add support to get performance state from DT](https://lwn.net/Articles/742136/) | ربط الـ OPP framework بالـ genpd performance states |
| [Active state management of power domains](https://lwn.net/Articles/744047/) | إدارة الـ active states والـ performance states في الـ genpd |
| [pmdomain: Add generic sync_state() support to genpd](https://lwn.net/Articles/1017931/) | إضافة دعم الـ sync_state لكل الـ genpd providers |
| [PM: QoS: Introduce a system-wakeup QoS limit for s2idle and genpd](https://lwn.net/Articles/1030130/) | ربط الـ genpd بالـ system-wide QoS constraints |
| [Linux power management: The documentation I wanted to read](https://lwn.net/Articles/505683/) | نظرة شاملة على الـ Linux PM infrastructure |

---

### توثيق Kernel الرسمي

الملفات دي موجودة جوه الـ kernel source مباشرةً:

```
Documentation/power/runtime_pm.rst          ← الـ Runtime PM framework الأساسي
Documentation/power/admin-guide/            ← دليل المشرف للـ PM
Documentation/devicetree/bindings/power/power-domain.yaml  ← DT bindings للـ genpd
Documentation/devicetree/bindings/power/power_domain.txt   ← النسخة القديمة من الـ DT bindings
Documentation/driver-api/pm/               ← API documentation للـ driver writers
```

**الـ DT binding الرسمي** على kernel.org:
- [power-domain.yaml](https://www.kernel.org/doc/Documentation/devicetree/bindings/power/power-domain.yaml)
- [power_domain.txt (legacy)](https://www.kernel.org/doc/Documentation/devicetree/bindings/power/power_domain.txt)

**الـ Runtime PM documentation** الرسمي:
- [Runtime PM Framework](https://docs.kernel.org/power/runtime_pm.html)
- [Device PM Basics](https://docs.kernel.org/driver-api/pm/devices.html)

---

### Kernel Commits المهمة

الـ commits دي مثّلت نقاط تحول في تطور الـ subsystem:

| الحدث | الوصف |
|-------|--------|
| **Linux 3.1 (2011)** | Rafael J. Wysocki أضاف أول implementation للـ generic PM domains |
| **Linux 3.8** | إضافة الـ OF (Device Tree) provider support عبر `of_genpd_add_provider_*` |
| **Linux 4.7** | دعم ربط أكتر من PM domain بنفس الـ device |
| **Linux 4.17** | إضافة الـ performance states API (`set_performance_state`) |
| **Linux 5.0** | دمج الـ OPP table مع الـ genpd performance states |
| **Linux 5.9** | إضافة الـ `GENPD_FLAG_MIN_RESIDENCY` لتحسين الـ governor |
| **Linux 6.6** | إعادة تسمية الـ subsystem من `genpd` إلى `pmdomain` بعد نقد Linus Torvalds |

**الـ renaming discussion** على Phoronix:
- [Following Criticism By Linus Torvalds, GenPD Subsystem Renamed To "pmdomain"](https://www.phoronix.com/news/GenPD-Renamed-pmdomain)

---

### نقاشات Mailing List

**الـ LKML** فيه نقاشات تقنية عميقة حول الـ genpd/pmdomain:

- [RFC: PM domains: Reverse the order of performance and enabling ops](https://lkml.kernel.org/lkml/20220720110246.762939-1-abel.vesa@linaro.org/T/) — نقاش حول ترتيب عمليات الـ power-on وتغيير الـ performance state
- [PSCI: Add support for PM domains using genpd](https://lkml.org/lkml/2019/6/7/570) — تطبيق الـ genpd على الـ PSCI firmware للـ ARM
- [PM Domains for CPUs (RFC)](https://lore.kernel.org/linux-arm-kernel/55740D00.1030203@samsung.com/T/) — أول patch لدعم الـ CPU PM domains

**الـ linux-pm mailing list** هو المكان الرئيسي للنقاشات الحالية:

```
https://lore.kernel.org/linux-pm/
```

---

### BayLibre و Linaro Resources

مصادر تقنية ممتازة من الشركات اللي تشتغل على الـ embedded Linux:

- [An Overview of Generic Power Domains (genpd) on Linux — BayLibre](https://baylibre.com/generic-power-domains/) — شرح شامل ومصور للـ genpd architecture والـ API
- [Introduction to Kernel Power Management — Kevin Hilman, Linaro](http://events17.linuxfoundation.org/sites/events/files/slides/Intro_Kernel_PM.pdf) — slides من Linux Foundation بتغطي الـ PM domains
- [Generic PM Domains and Platform Device Drivers — Rafael Wysocki, LC-JP 2012](https://events.static.linuxfound.org/images/stories/pdf/lcjp2012_wysocki.pdf) — شرح من مصمم الـ subsystem نفسه

---

### eLinux.org

- [Power Management Using PM Domains on SH7372 (PDF)](https://elinux.org/images/e/e6/Elce11_wysocki.pdf) — عرض من ELCE 2011 بيشرح أول implementation للـ PM domains على Renesas SH7372
- [OMAP Power Management](https://elinux.org/OMAP_Power_Management) — تطبيق عملي للـ PM domains على OMAP من TI
- [Power Management Presentations](https://elinux.org/Power_Management_Presentations) — مجموعة presentations حول الـ power domains والـ clock domains
- [Power Management Framework](https://elinux.org/Power_Management_Framework) — نظرة عامة على الـ Linux PM framework

---

### KernelNewbies.org

الـ pages دي بتوضح متى اتضافت features مهمة في كل kernel version:

- [Linux 2.6.39 — genpd initial support](https://kernelnewbies.org/Linux_2_6_39) — أول إصدار يحتوي على الـ generic PM domains
- [Linux 4.12 — TI SCI PM domains](https://kernelnewbies.org/Linux_4.12) — إضافة الـ `ti_sci_pm_domains` driver
- [Linux 5.14 — Tegra PMC power domain](https://kernelnewbies.org/Linux_5.14) — دعم الـ sync_state للـ Tegra
- [Linux 5.17 — pmdomain enhancements](https://kernelnewbies.org/Linux_5.17) — تحسينات على الـ genpd subsystem

---

### كتب مرجعية

#### Linux Device Drivers (LDD3) — Corbet, Rubini, Kroah-Hartman
- **الفصل 14**: *The Linux Device Model* — بيشرح الـ `device`, `bus_type`, وكيف الـ PM domain بيتكامل مع الـ device model
- **الفصل 14**: قسم *Power Management* — يغطي الـ `dev_pm_ops` callbacks اللي الـ genpd بيستخدمها
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 14**: *The Block I/O Layer* — يشرح الـ device model الأساسي
- **الفصل 17**: *Devices and Modules* — يغطي الـ device registration وكيف الـ PM domain بيرتبط بالـ driver model
- الـ PM domain مش مغطي بشكل مباشر لأن الطبعة قبل إضافته، بس الـ fundamentals مهمة

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 16**: *Kernel Debugging Techniques* — فيه أمثلة على debug الـ PM domains
- الـ power management على الـ embedded systems بيشرح ليه الـ genpd موجود

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 1**: *Introduction and Overview* — يشرح الـ kernel subsystems والعلاقة بينهم بما فيها الـ PM

---

### Search Terms للبحث عن معلومات أكتر

```
# للبحث في الـ kernel source
"generic_pm_domain"          ← الـ main struct
"pm_genpd_init"              ← initialization function
"genpd_xlate_t"              ← DT translation callback
"GENPD_FLAG_"                ← flags and their uses
"dev_pm_domain_attach"       ← device attach API
"of_genpd_add_provider"      ← DT provider registration
"genpd_power_state"          ← idle state definitions
"dev_power_governor"         ← power governor interface
"pmdomain:"                  ← مستخدم في commit messages بعد Linux 6.6

# للبحث على Google/DuckDuckGo
"linux kernel genpd tutorial"
"generic pm domain embedded linux"
"pm_domain device tree binding example"
"linux power domain OPP performance state"
"genpd governor simple_qos"
"linux pm domain hierarchical"
"pmdomain driver implementation"
"linux power gating SoC genpd"
```

---

### Source Code المرجعي في الـ Kernel

الملفات دي هي القلب الحقيقي للـ subsystem:

```
drivers/pmdomain/core.c              ← الـ genpd core implementation
drivers/pmdomain/of/pm_domain_of.c  ← الـ DT provider logic
drivers/pmdomain/governor/          ← الـ power governors (simple_qos, etc.)
include/linux/pm_domain.h           ← الـ public API (الملف اللي بندرسه)
include/linux/pm.h                  ← الـ dev_pm_ops definitions
drivers/base/power/domain.c         ← (قبل Linux 6.6 كانت هنا)

# أمثلة على genpd drivers حقيقية
drivers/pmdomain/renesas/           ← Renesas R-Car power domains
drivers/pmdomain/qcom/              ← Qualcomm RPM power domains
drivers/pmdomain/tegra/             ← NVIDIA Tegra power domains
drivers/pmdomain/ti/                ← TI SCI power domains
drivers/pmdomain/apple/             ← Apple Silicon power domains
```
## Phase 8: Writing simple module

### الفكرة

**`generic_pm_domain`** فيها `power_notifiers` من نوع `raw_notifier_head` — ده chain بيستخدمه genpd لما domain بيتحرك بين states (ON/OFF). الـ kernel بيبعت notifications بـ `genpd_notication` enum: `GENPD_NOTIFY_PRE_OFF`، `GENPD_NOTIFY_OFF`، `GENPD_NOTIFY_PRE_ON`، `GENPD_NOTIFY_ON`.

الـ API المتاح عشان نسجل notifier على domain معين هو `dev_pm_genpd_add_notifier()` / `dev_pm_genpd_remove_notifier()` — بس دول بيتطلبوا device مرتبط بالـ domain. الأبسط والأكثر وضوحاً كـ module تعليمي هو استخدام **kprobe** على `pm_genpd_init()` — وهي الـ function اللي بيتسجل فيها كل generic PM domain في النظام. ده بيديك مشاهدة لكل domain بيتعمل initialize، اسمه، وflags الخاصة بيه.

دي function مُصدَّرة (`EXPORT_SYMBOL_GPL`) وآمنة للـ probe عليها.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * genpd_probe.c — kprobe on pm_genpd_init() to log every
 * Generic PM Domain as it gets registered in the kernel.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* kprobe struct and API */
#include <linux/pm_domain.h>    /* generic_pm_domain, GENPD_FLAG_* */
#include <linux/printk.h>       /* pr_info, pr_err */

/*
 * pre_handler is called right before pm_genpd_init() executes.
 * At this point the CPU registers still hold the original arguments:
 *   arg0 (di/x0) = struct generic_pm_domain *genpd
 *   arg1 (si/x1) = struct dev_power_governor *gov
 *   arg2 (dx/x2) = bool is_off
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct generic_pm_domain *genpd;

#if defined(CONFIG_X86_64)
    /* On x86-64 first arg lives in RDI */
    genpd = (struct generic_pm_domain *)regs->di;
#elif defined(CONFIG_ARM64)
    /* On ARM64 first arg lives in x0 */
    genpd = (struct generic_pm_domain *)regs->regs[0];
#else
    /* Fallback: skip logging on unsupported arch */
    return 0;
#endif

    /* Basic sanity — genpd must not be NULL */
    if (!genpd)
        return 0;

    pr_info("[genpd_probe] domain='%s' flags=0x%x state=%s\n",
            genpd->name ? genpd->name : "(null)",
            genpd->flags,
            /* show initial power state based on flags */
            (genpd->flags & GENPD_FLAG_ALWAYS_ON) ? "ALWAYS_ON" :
            (genpd->flags & GENPD_FLAG_RPM_ALWAYS_ON) ? "RPM_ALWAYS_ON" :
            "normal");

    /* Log interesting optional flags when present */
    if (genpd->flags & GENPD_FLAG_IRQ_SAFE)
        pr_info("[genpd_probe]   -> IRQ_SAFE (atomic callbacks)\n");
    if (genpd->flags & GENPD_FLAG_CPU_DOMAIN)
        pr_info("[genpd_probe]   -> CPU_DOMAIN\n");
    if (genpd->flags & GENPD_FLAG_MIN_RESIDENCY)
        pr_info("[genpd_probe]   -> MIN_RESIDENCY governor hint\n");

    return 0; /* 0 = let the original function run normally */
}

/* kprobe descriptor — we target pm_genpd_init by symbol name */
static struct kprobe genpd_kp = {
    .symbol_name = "pm_genpd_init",
    .pre_handler = handler_pre,
};

/* ---------- init ---------- */
static int __init genpd_probe_init(void)
{
    int ret;

    ret = register_kprobe(&genpd_kp);
    if (ret < 0) {
        pr_err("[genpd_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[genpd_probe] kprobe planted on pm_genpd_init @ %px\n",
            genpd_kp.addr);
    return 0;
}

/* ---------- exit ---------- */
static void __exit genpd_probe_exit(void)
{
    unregister_kprobe(&genpd_kp);
    pr_info("[genpd_probe] kprobe removed\n");
}

module_init(genpd_probe_init);
module_exit(genpd_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("PM Domain Observer");
MODULE_DESCRIPTION("kprobe on pm_genpd_init to trace Generic PM Domain registration");
```

---

### Makefile

```makefile
obj-m += genpd_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# تجميع وتحميل الـ module
make
sudo insmod genpd_probe.ko
dmesg | grep genpd_probe

# إزالته
sudo rmmod genpd_probe
```

---

### شرح كل جزء

#### الـ includes

```c
#include <linux/kprobes.h>
#include <linux/pm_domain.h>
```

**الـ `kprobes.h`** بيجيب `struct kprobe` و `register_kprobe` / `unregister_kprobe`. **الـ `pm_domain.h`** بيجيب `generic_pm_domain` والـ `GENPD_FLAG_*` constants اللي بنستخدمها عشان نفسر الـ flags بشكل مقروء.

---

#### الـ `handler_pre` — الـ callback

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

الـ kprobe بيستدعي الـ `pre_handler` قبل ما `pm_genpd_init()` تتنفذ — يعني الـ arguments لسه موجودة في registers. بنسحب `genpd` من `regs->di` (x86-64) أو `regs->regs[0]` (ARM64) عشان نوصل لـ `generic_pm_domain` ونطبع اسمه وflags الخاصة بيه بدون ما نعدل أي حاجة في تدفق الـ kernel.

الـ return value `0` معناها "امشي، نفّذ الـ function الأصلية زي ما هي".

---

#### الـ `genpd_kp` struct

```c
static struct kprobe genpd_kp = {
    .symbol_name = "pm_genpd_init",
    .pre_handler = handler_pre,
};
```

الـ kernel بيحل اسم الـ symbol لـ address وقت الـ registration. استخدمنا `symbol_name` بدل address ثابت عشان يشتغل على أي kernel بدون ما نحتاج نعرف العنوان.

---

#### الـ `module_init`

```c
ret = register_kprobe(&genpd_kp);
```

**الـ `register_kprobe`** بتزرع breakpoint في `pm_genpd_init()`. لو فشلت (مثلاً الـ symbol مش موجود أو الـ kprobes disabled) بنرجع الـ error code ونفشل الـ insmod بدل ما نكمل بـ module مش شغال.

---

#### الـ `module_exit`

```c
unregister_kprobe(&genpd_kp);
```

لازم نشيل الـ kprobe قبل ما يتم unload الـ module — لو ماعملناش كده الـ kernel هيحاول يستدعي `handler_pre` اللي بقى عنوانه invalid وهيحصل kernel panic. الـ `unregister_kprobe` بترجع الـ instruction الأصلية في مكانها وتضمن إن مفيش callback شغالة وقت الـ unload.

---

### مثال على الـ output

```
[genpd_probe] kprobe planted on pm_genpd_init @ ffffffff819a3c40
[genpd_probe] domain='pd_gpu' flags=0x0 state=normal
[genpd_probe] domain='pd_dsp' flags=0x2 state=normal
[genpd_probe]   -> IRQ_SAFE (atomic callbacks)
[genpd_probe] domain='pd_always_on' flags=0x4 state=ALWAYS_ON
[genpd_probe] kprobe removed
```

---

### ملاحظات مهمة

| نقطة | التفاصيل |
|------|-----------|
| **`pm_genpd_init` exported** | مُصدَّرة بـ `EXPORT_SYMBOL_GPL` فـ kprobe بيلاقيها بسهولة |
| **لا تعديل للبيانات** | الـ `pre_handler` بيقرأ فقط — لو حبيت تعدل استخدم `post_handler` أو `jprobe` (deprecated) أو `kretprobe` |
| **early boot** | لو الـ domains بيتسجلوا قبل `insmod` مش هتشوفهم — الـ kprobe بيشتغل من وقت الـ registration بس |
| **CONFIG_KPROBES** | لازم يكون `=y` في kernel config — معظم distros بتشغّله by default |
