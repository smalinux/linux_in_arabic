## Phase 1: الصورة الكبيرة ببساطة

### ما هو RAPL؟

**RAPL** — Running Average Power Limit — هو آلية من Intel بتخلي الـ CPU يراقب ويتحكم في استهلاك الطاقة بشكل مباشر. مش بس مراقبة — إنت بتقدر تقوله "يا CPU، ماتاخدش أكتر من 45 واط" وهو بيلتزم.

الـ CPU من Intel مش قطعة واحدة من ناحية الطاقة — هو مقسم لـ **domains** (مناطق) كل منطقة ليها استهلاكها الخاص:

```
┌─────────────────────────────────────────┐
│            Package (socket)             │
│  ┌──────────────┐   ┌─────────────────┐ │
│  │  PP0 (Cores) │   │  PP1 (GPU/UNC)  │ │
│  └──────────────┘   └─────────────────┘ │
│  ┌──────────────┐                       │
│  │     DRAM     │                       │
│  └──────────────┘                       │
│  ┌──────────────────────────────────────┤
│  │  Platform (PSys) — كل الـ SoC       │
│  └──────────────────────────────────────┘
└─────────────────────────────────────────┘
```

---

### القصة بالتفصيل — ليه RAPL موجود أصلاً؟

تخيل إنك عندك laptop وبتشتغل على بطارية. الـ CPU ممكن يسحب 95 واط في لحظة — البطارية هتفضل ساعة. المفروض تشتغل 4 ساعات.

من غير RAPL، الـ OS مش عارف يقول للـ CPU "اشتغل بالكتير بـ 15 واط". الوحيد اللي يعرف يعمل ده هو الـ CPU نفسه أو firmware — وده بيعمل مشاكل لأن الـ OS مش عارف إيه اللي بيحصل.

RAPL جه يحل ده: بيوفر **registers** (MSR أو MMIO أو TPMI) الـ OS يقدر يقرأ منها استهلاك الطاقة الحالي، ويكتب فيها حدود (limits) ما يتجاوزهاش الـ CPU.

الـ kernel بيعمل ده تحت إطار اسمه **powercap** — framework عام بيوفر sysfs interface ليوزر الـ space يتحكم فيه من `/sys/class/powercap/`.

---

### الـ `intel_rapl.h` — إيه دوره؟

الملف ده هو **العقد المشترك** بين:
1. الـ RAPL common layer (`intel_rapl_common.c`)
2. الـ interface-specific drivers (MSR, MMIO, TPMI)
3. أي subsystem تاني عايز يتكلم مع RAPL (زي perf events)

بيعرّف:
- أنواع الـ interfaces (MSR / MMIO / TPMI)
- أنواع الـ domains (Package, PP0, PP1, DRAM, Platform)
- أنواع الـ registers لكل domain
- الـ primitives — القيم الخام اللي بتتقرأ من الـ hardware
- الـ structs الأساسية: `rapl_domain`, `rapl_package`, `rapl_if_priv`
- الـ API functions اللي بيستخدمها كل driver

---

### الفكرة ELI5

فكر في RAPL زي **عداد الكهرباء في البيت** بس smarter:
- بيقيس الاستهلاك لكل غرفة (domain) على حدى
- بيسمحلك تحدد سقف للاستهلاك
- لو الاستهلاك اتجاوز السقف، بيتدخل ويخفض

الـ `intel_rapl.h` هو **المخطط الهندسي** (blueprint) للعداد ده — بيحدد إزاي شكل كل حاجة، من غير ما يبني أي حاجة فعلياً.

---

### الـ Interfaces الثلاثة

| Interface | وصفه | متى يُستخدم |
|-----------|------|-------------|
| **MSR** | Model Specific Registers — بتتكلم مع الـ CPU core مباشرة | الأجيال القديمة والحديثة من Intel |
| **MMIO** | Memory-Mapped I/O — registers في memory space | بعض chipsets |
| **TPMI** | Topology-Aware Register and PM Capsule Interface | Intel Granite Rapids وما بعدها |

---

### الـ Domains الخمسة

| Domain | المعنى |
|--------|--------|
| **PACKAGE** | كامل الـ CPU socket |
| **PP0** | الـ CPU cores فقط |
| **PP1** | الـ GPU uncore أو graphics |
| **DRAM** | ذاكرة الـ RAM |
| **PLATFORM (PSys)** | كامل الـ SoC شامل كل حاجة |

---

### الـ Power Limits

كل domain بيدعم أكتر من نوع من الـ limits:

- **PL1** — Long-term limit: الحد التاني على مدى فترة طويلة (عادةً ثواني)
- **PL2** — Short-term limit: بيسمح بـ burst لفترة قصيرة
- **PL4** — Peak power limit: الحد الأقصى المطلق — لا تتجاوزه أبداً

---

### الـ Primitives

الـ `enum rapl_primitives` بيعدد كل القيم اللي ممكن تتقرأ أو تتكتب في الـ hardware:

```c
enum rapl_primitives {
    POWER_LIMIT1,      /* قيمة الحد الأول للطاقة */
    POWER_LIMIT2,      /* قيمة الحد التاني */
    ENERGY_COUNTER,    /* عداد الطاقة المستهلكة (joules) */
    PL1_ENABLE,        /* تفعيل الحد الأول */
    TIME_WINDOW1,      /* النافذة الزمنية للـ PL1 */
    THROTTLED_TIME,    /* الوقت اللي الـ CPU اتعاقب فيه */
    AVERAGE_POWER,     /* متوسط الاستهلاك المحسوب */
    /* ... */
};
```

---

### الـ rapl_if_priv — قلب الـ abstraction

```c
struct rapl_if_priv {
    enum rapl_if_type type;          /* MSR أو MMIO أو TPMI */
    struct powercap_control_type *control_type; /* ربط بالـ powercap framework */
    union rapl_reg regs[RAPL_DOMAIN_MAX][RAPL_DOMAIN_REG_MAX]; /* جدول الـ registers */
    int (*read_raw)(int id, struct reg_action *ra, bool pmu_ctx);  /* callback للقراءة */
    int (*write_raw)(int id, struct reg_action *ra);               /* callback للكتابة */
};
```

كل driver بـ interface مختلف (MSR/MMIO/TPMI) بيملأ الـ struct ده بـ registers وـ callbacks الخاصة بيه، والـ common layer بيشتغل فوقيهم كلهم بنفس الكود.

---

### الـ PMU Integration

لو الـ `CONFIG_PERF_EVENTS` مفعّل، الـ RAPL بيتكامل مع الـ **perf subsystem** — يعني تقدر تستخدم `perf stat -e power/energy-pkg/` وتجيب استهلاك الطاقة زي ما بتجيب cycles أو instructions.

الـ `rapl_package_pmu_data` بيدير ده: بيشغّل hrtimer يقرأ الـ energy counter بانتظام عشان يمنع الـ overflow (الـ counter 32-bit وبيلف بسرعة).

---

### الملفات المكوّنة للـ Subsystem

**Core Logic:**
- `/include/linux/intel_rapl.h` — الـ header الرئيسي (الملف ده)
- `/drivers/powercap/intel_rapl_common.c` — الـ common layer، بيحتوي على كل المنطق المشترك

**Interface Drivers:**
- `/drivers/powercap/intel_rapl_msr.c` — الـ driver للـ MSR interface
- `/drivers/powercap/intel_rapl_tpmi.c` — الـ driver للـ TPMI interface (الأجيال الجديدة)

**Framework الأم:**
- `/include/linux/powercap.h` — الـ powercap framework اللي RAPL بيجلس فوقه
- `/drivers/powercap/powercap_sys.c` — تطبيق الـ powercap framework

**الـ Kconfig والـ Makefile:**
- `/drivers/powercap/Kconfig` — CONFIG_INTEL_RAPL و CONFIG_INTEL_RAPL_CORE
- `/drivers/powercap/Makefile`

**sysfs entries (runtime, مش ملفات كود):**
```
/sys/class/powercap/intel-rapl/
  intel-rapl:0/          ← Package 0
    constraint_0_power_limit_uw
    constraint_0_time_window_us
    energy_uj
    intel-rapl:0:0/      ← PP0 (cores)
    intel-rapl:0:1/      ← PP1 (uncore)
```
## Phase 2: شرح الـ Intel RAPL Framework

### المشكلة اللي بيحلها الـ RAPL

الـ CPU الحديث بيشتغل في بيئات متنوعة جداً — من server rack فيه تبريد ممتاز، لـ ultrabook بطارية بتخلص بسرعة. المشكلة الأساسية: **إزاي تحدد سقف لاستهلاك الطاقة للـ CPU/DRAM/GPU بطريقة dynamic بدون ما تقفل الـ frequency manually؟**

قبل RAPL، المختارات كانت:
- **cpufreq governors**: بتتحكم في الـ frequency مش الـ power مباشرة — تقريبي وبطيء.
- **thermal throttling**: passive، بتحصل بعد ما الحرارة تعدي حد — reactive مش proactive.
- **custom firmware**: حل vendor-specific مش قابل للتعميم.

الـ Intel لاقت إن الـ hardware نفسه أقدر على قياس ومنع تجاوز الطاقة بدقة أكبر بكتير من الـ OS، فأدخلت **RAPL (Running Average Power Limit)** — آلية hardware تسمح للـ software بضبط حدود طاقة لمناطق مختلفة في الـ chip.

---

### الحل: RAPL كـ Hardware Power Accounting

الـ RAPL مش مجرد throttling — هو **energy accounting + enforcement** في نفس الوقت:

1. الـ hardware بيقيس الطاقة الفعلية المستهلكة في كل domain (Package, Core, Uncore, DRAM).
2. الـ software بيضبط حدود (power limits) في registers خاصة.
3. الـ hardware نفسه بيطبق الحدود دي بشكل autonomic عن طريق تخفيض الـ frequency أو voltage.

الـ kernel بيوفر abstraction layer فوق hardware RAPL يخلي الـ userspace (مثلاً `powercap` tool أو `thermald`) يتعامل مع الـ power limits بدون ما يعرف إيه نوع الـ register (MSR, MMIO, أو TPMI).

---

### التشبيه الحقيقي: عداد الكهرباء الذكي

تخيل شقة فيها **عداد كهرباء ذكي** بيقيس كل غرفة بشكل منفصل:

| الكهرباء الحقيقية | RAPL |
|---|---|
| العداد الرئيسي للشقة كلها | `RAPL_DOMAIN_PACKAGE` |
| عداد فرعي لأوض النوم (CPU cores) | `RAPL_DOMAIN_PP0` |
| عداد فرعي للصالة (GPU uncore) | `RAPL_DOMAIN_PP1` |
| عداد فرعي للـ AC (DRAM) | `RAPL_DOMAIN_DRAM` |
| حد أقصى شهري متفق عليه مع الشركة | `POWER_LIMIT1` (long-term, PL1) |
| حد طوارئ لو الاستهلاك زاد فجأة | `POWER_LIMIT2` (short-term, PL2) |
| حد مطلق لا يُتجاوز أبداً (fuse) | `POWER_LIMIT4` (peak, PL4) |
| قراءة العداد كل شهر | `ENERGY_COUNTER` |
| شركة الكهرباء (hardware enforcer) | الـ CPU نفسه |
| شاشة العرض في الشقة (sysfs) | `powercap` framework |
| فني الكهرباء اللي بيعدل الحدود | الـ driver (MSR/MMIO/TPMI) |

**الأهم**: الـ hardware (CPU) هو اللي بيطبق الحد، مش الـ OS — الـ OS بس بيكتب الحد في register ويقرأ القياسات.

---

### Big Picture Architecture

```
User Space
  ├── thermald / powertop / power-profiles-daemon
  └── /sys/devices/virtual/powercap/...
         │  (sysfs read/write)
         ▼
┌─────────────────────────────────────────────────┐
│            powercap Framework (generic)          │
│  powercap_control_type → powercap_zone           │
│                        → powercap_zone_constraint│
└────────────────┬────────────────────────────────┘
                 │  (registers zones & constraints)
                 ▼
┌─────────────────────────────────────────────────┐
│          Intel RAPL Core (intel_rapl_common.c)   │
│                                                  │
│  rapl_package  →  rapl_domain[]                  │
│                   ├── rapl_power_limit[PL1/PL2/PL4]│
│                   ├── rapl_domain_data (cache)   │
│                   └── rapl_if_priv (callbacks)   │
└────────┬──────────────┬──────────────┬──────────┘
         │              │              │
         ▼              ▼              ▼
  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ MSR I/F  │   │ MMIO I/F │   │ TPMI I/F │
  │(intel_   │   │(intel_   │   │(intel_   │
  │rapl_msr) │   │rapl_mmio)│   │rapl_tpmi)│
  └────┬─────┘   └────┬─────┘   └────┬─────┘
       │              │              │
       ▼              ▼              ▼
  ┌─────────────────────────────────────────┐
  │         Hardware RAPL Registers          │
  │  MSR_PKG_POWER_LIMIT / MMIO / TPMI regs │
  │  ┌─────────┐ ┌──────┐ ┌──────┐ ┌──────┐│
  │  │ Package │ │ PP0  │ │ PP1  │ │ DRAM ││
  │  │ Domain  │ │Domain│ │Domain│ │Domain││
  │  └─────────┘ └──────┘ └──────┘ └──────┘│
  └─────────────────────────────────────────┘
```

---

### الـ Core Abstraction: Domain + Primitive

الفكرة المحورية في RAPL هي **الـ domain**: منطقة hardware ليها استهلاكها المنفصل وحدودها الخاصة.

```c
enum rapl_domain_type {
    RAPL_DOMAIN_PACKAGE,   /* الـ socket كله */
    RAPL_DOMAIN_PP0,       /* cores فقط (PP = Power Plane) */
    RAPL_DOMAIN_PP1,       /* GPU uncore (مش في كل CPUs) */
    RAPL_DOMAIN_DRAM,      /* memory controllers */
    RAPL_DOMAIN_PLATFORM,  /* PSys: المنصة كلها زي Surface */
    RAPL_DOMAIN_MAX,
};
```

كل domain فيها **primitives** — قيم atomic بتقدر تقراها أو تكتبها:

```c
enum rapl_primitives {
    POWER_LIMIT1,      /* PL1: long-term avg power (watts) */
    POWER_LIMIT2,      /* PL2: short burst power (watts) */
    POWER_LIMIT4,      /* PL4: instantaneous peak (watts) */
    ENERGY_COUNTER,    /* monotonic energy counter (joules raw) */
    PL1_ENABLE,        /* تفعيل PL1 enforcement */
    PL1_CLAMP,         /* يسمح للـ hardware يخفض تحت الـ OS request */
    TIME_WINDOW1,      /* نافذة الزمن لحساب PL1 (ثواني) */
    TIME_WINDOW2,      /* نافذة الزمن لحساب PL2 */
    THROTTLED_TIME,    /* وقت التقييد الكلي */
    AVERAGE_POWER,     /* محسوبة من الـ energy counter */
    /* ... */
};
```

---

### بنية الـ Structs وعلاقاتها

```
rapl_package (per socket/die)
├── id                      ← logical die ID
├── nr_domains              ← عدد الـ domains المفعّلة
├── domain_map              ← bitmask للـ domains
├── domains[]               ← array of rapl_domain
│   └── rapl_domain
│       ├── name            ← "package", "core", "dram" ...
│       ├── id              ← rapl_domain_type
│       ├── regs[]          ← union rapl_reg لكل register
│       │   └── union rapl_reg
│       │       ├── mmio    ← void __iomem* (MMIO interface)
│       │       ├── msr     ← u32 MSR address
│       │       └── val     ← u64 generic value
│       ├── power_zone      ← embedded powercap_zone (الواجهة للـ sysfs)
│       ├── rdd             ← rapl_domain_data (cache للقيم)
│       │   └── primitives[NR_RAPL_PRIMITIVES]
│       ├── rpl[]           ← rapl_power_limit[NR_POWER_LIMITS]
│       │   └── rapl_power_limit
│       │       ├── constraint ← powercap_zone_constraint*
│       │       ├── domain     ← back-pointer
│       │       ├── name       ← "long_term", "short_term"
│       │       └── locked     ← firmware locked?
│       └── rp              ← back-pointer to rapl_package
├── power_zone*             ← parent powercap_zone
├── priv*                   ← rapl_if_priv (interface callbacks)
│   └── rapl_if_priv
│       ├── type            ← RAPL_IF_MSR / MMIO / TPMI
│       ├── control_type*   ← powercap_control_type
│       ├── regs[][]        ← register map per domain per reg_id
│       ├── read_raw()      ← callback: قراءة register خام
│       └── write_raw()     ← callback: كتابة register خام
└── pmu_data (CONFIG_PERF_EVENTS)
    ├── scale[]             ← energy scale per domain
    ├── hrtimer             ← منع overflow للـ counter
    └── active_list         ← perf events نشطة
```

---

### الـ union rapl_reg: التجريد الذكي

الـ register في RAPL ممكن يكون MSR أو MMIO أو TPMI — الثلاثة بيتعاملوا مع نفس الـ union:

```c
union rapl_reg {
    void __iomem *mmio;  /* pointer لـ memory-mapped I/O */
    u32 msr;             /* رقم الـ MSR register */
    u64 val;             /* قيمة generic */
};
```

الـ `rapl_if_priv` بيحدد `type` وبناءً عليه الـ `read_raw`/`write_raw` callbacks بتفسر الـ union صح. هذا **Strategy Pattern** في kernel C.

---

### الـ reg_action: وحدة العملية الذرية

```c
struct reg_action {
    union rapl_reg reg;  /* الـ register المستهدف */
    u64 mask;            /* الـ bits المهمة */
    u64 value;           /* القيمة المطلوب كتابتها/قراءتها */
    int err;             /* نتيجة العملية */
};
```

الـ `read_raw`/`write_raw` callbacks بتاخد `reg_action` — ده بيسمح للـ interface-specific code يتعامل مع read-modify-write بشكل آمن، خصوصاً إن كتير من الـ RAPL registers بتحتوي fields متعددة في نفس الـ register.

---

### Interface Types: MSR vs MMIO vs TPMI

| الخاصية | MSR | MMIO | TPMI |
|---|---|---|---|
| الوصول | `rdmsr`/`wrmsr` instructions | memory read/write | MMIO عبر VSEC |
| الـ context | per-CPU | any | any |
| الاستخدام | CPUs قديمة (Sandy Bridge → Raptor Lake) | Granite Rapids | Birch Stream+ |
| الـ `id` parameter | CPU logical number | package/die ID | package/die ID |

الـ `pmu_ctx` parameter في `read_raw` بيفرق بين قراءة من CPU context وقراءة من PMU interrupt context — مهم لـ MSR لأن `rdmsr` lazim تتنفذ على الـ CPU الصح.

---

### علاقة RAPL بالـ powercap Framework

الـ **powercap framework** (مذكور في `include/linux/powercap.h`) هو الطبقة العامة اللي RAPL بيستخدمها. الـ powercap مش عارف حاجة عن Intel أو MSRs — هو بس بيوفر:

- **`powercap_control_type`**: container للـ technology (مثلاً "intel-rapl").
- **`powercap_zone`**: منطقة طاقة واحدة مع callbacks لقراءة الـ energy/power.
- **`powercap_zone_constraint`**: قيد واحد (مثلاً PL1 أو PL2) مع callbacks لـ get/set power limit.

الـ RAPL framework بيـ**embed** الـ `powercap_zone` جوه `rapl_domain`:

```c
struct rapl_domain {
    /* ... */
    struct powercap_zone power_zone;  /* embedded, مش pointer */
    /* ... */
};
```

ده بيسمح بـ `container_of()` للتحويل بين `powercap_zone*` و `rapl_domain*` بكفاءة.

---

### الـ PMU Integration (CONFIG_PERF_EVENTS)

الـ RAPL بيدعم كمان الـ **perf subsystem** عبر `rapl_package_pmu_data`. ده يسمح بـ:

```bash
perf stat -e power/energy-pkg/ ./my_program
```

المشكلة إن الـ energy counter بـ 32-bit وبيـ overflow بعد كذا ثانية (حسب الـ TDP). الـ `hrtimer` بيتنبه قبل الـ overflow يحصل ويقرأ الـ counter ويجمع القيمة في accumulator — ده اللي الـ `scale[]` و `timer_interval` بيخدموه.

---

### ملكية الـ Framework مقابل الـ Drivers

**الـ RAPL framework بيمتلك:**
- تعريف وتسجيل الـ `rapl_package` و `rapl_domain`.
- ربط الـ domains بالـ powercap zones.
- حساب الـ average power من الـ energy counter.
- إدارة الـ CPU hotplug (عبر `pcap_rapl_online` state).
- الـ PMU timer وحساب الـ scale.

**الـ Framework بيفوض للـ Interface Drivers (MSR/MMIO/TPMI):**
- الـ register addresses الفعلية (`regs[][]` في `rapl_if_priv`).
- كيفية قراءة/كتابة الـ register (`read_raw` / `write_raw`).
- اكتشاف الـ domains المتاحة على هذا النوع من الـ hardware.
- الـ unit conversion constants (`reg_unit`).

**الـ Framework بيفوض للـ powercap Framework:**
- تعريف الـ sysfs hierarchy.
- permissions وأحداث الـ uevent.
- الـ constraint IDs الـ numeric.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### جدول الـ Enums والـ Flags والـ Config Options

#### `enum rapl_if_type` — نوع الـ Interface

| القيمة | القيمة العددية | المعنى |
|--------|----------------|--------|
| `RAPL_IF_MSR` | 0 | الوصول عبر الـ Model-Specific Registers |
| `RAPL_IF_MMIO` | 1 | الوصول عبر الـ Memory-Mapped I/O |
| `RAPL_IF_TPMI` | 2 | الوصول عبر الـ Topology-aware Power Management Interface |

---

#### `enum rapl_domain_type` — نوع الـ Power Domain

| القيمة | المعنى |
|--------|--------|
| `RAPL_DOMAIN_PACKAGE` | الـ Socket بالكامل (كل الـ cores + uncore) |
| `RAPL_DOMAIN_PP0` | الـ Core Power Plane (ALUs, execution units) |
| `RAPL_DOMAIN_PP1` | الـ GPU uncore / Graphics |
| `RAPL_DOMAIN_DRAM` | الـ DRAM / Memory Controller |
| `RAPL_DOMAIN_PLATFORM` | الـ PSys — المنصة كاملة (Skylake+) |
| `RAPL_DOMAIN_MAX` | Sentinel — عدد الـ Domains |

---

#### `enum rapl_domain_reg_id` — أنواع الـ Registers لكل Domain

| القيمة | المعنى |
|--------|--------|
| `RAPL_DOMAIN_REG_LIMIT` | Power Limit register (PL1, PL2) |
| `RAPL_DOMAIN_REG_STATUS` | Energy Status register (counter) |
| `RAPL_DOMAIN_REG_PERF` | Performance Status |
| `RAPL_DOMAIN_REG_POLICY` | Policy register |
| `RAPL_DOMAIN_REG_INFO` | Info: Thermal Spec Power, Max/Min |
| `RAPL_DOMAIN_REG_PL4` | Power Limit 4 (Peak Power) |
| `RAPL_DOMAIN_REG_UNIT` | Unit register للـ energy/power/time |
| `RAPL_DOMAIN_REG_PL2` | Power Limit 2 (Short-term) |
| `RAPL_DOMAIN_REG_MAX` | Sentinel |

---

#### `enum rapl_primitives` — الـ Primitives الأساسية (Cheatsheet)

| الـ Primitive | النوع | المعنى |
|---------------|-------|--------|
| `POWER_LIMIT1` | r/w | الـ Long-term power limit (Watts) |
| `POWER_LIMIT2` | r/w | الـ Short-term power limit |
| `POWER_LIMIT4` | r/w | الـ Peak power limit |
| `ENERGY_COUNTER` | r | عداد الطاقة (يتراكم) |
| `FW_LOCK` | r | Firmware قفل الـ limit |
| `FW_HIGH_LOCK` | r | Firmware قفل بمستوى عالي |
| `PL1_LOCK` | r/w | قفل الـ Power Limit 1 من OS |
| `PL2_LOCK` | r/w | قفل الـ Power Limit 2 |
| `PL4_LOCK` | r/w | قفل الـ Power Limit 4 |
| `PL1_ENABLE` | r/w | تفعيل الـ Long-term limit |
| `PL1_CLAMP` | r/w | السماح بالنزول تحت OS request |
| `PL2_ENABLE` | r/w | تفعيل الـ Short-term limit |
| `PL2_CLAMP` | r/w | الـ Clamp لـ PL2 |
| `PL4_ENABLE` | r/w | تفعيل الـ Peak power limit |
| `TIME_WINDOW1` | r/w | نافذة الوقت لـ PL1 |
| `TIME_WINDOW2` | r/w | نافذة الوقت لـ PL2 |
| `THERMAL_SPEC_POWER` | r | الـ TDP المقنن |
| `MAX_POWER` | r | أقصى قدرة |
| `MIN_POWER` | r | أدنى قدرة |
| `MAX_TIME_WINDOW` | r | أقصى نافذة زمنية |
| `THROTTLED_TIME` | r | الوقت الذي تم فيه الـ Throttle |
| `PRIORITY_LEVEL` | r/w | أولوية الـ PP1 domain |
| `PSYS_POWER_LIMIT1/2` | r/w | Power limits للمنصة كاملة |
| `PSYS_PL1/2_ENABLE` | r/w | تفعيل الـ PSys limits |
| `PSYS_TIME_WINDOW1/2` | r/w | نوافذ الوقت للـ PSys |
| `AVERAGE_POWER` | calculated | يُحسب من الـ energy counter |
| `NR_RAPL_PRIMITIVES` | - | إجمالي عدد الـ Primitives |

---

#### Config Options

| الـ Config | التأثير |
|------------|---------|
| `CONFIG_PERF_EVENTS` | يُفعّل الـ PMU support في `rapl_package_pmu_data` وإضافة/إزالة الـ PMU |

---

### الـ Structs الأساسية

---

#### 1. `union rapl_reg`

**الغرض:** تمثيل موحّد لعنوان الـ Register بغض النظر عن نوع الـ Interface.

```c
union rapl_reg {
    void __iomem *mmio;  /* MMIO pointer — for MMIO/TPMI interfaces */
    u32 msr;             /* MSR number — for MSR interface */
    u64 val;             /* generic 64-bit view */
};
```

الـ Union ده بيخلي الـ code فوق ما يحتاجش يعرف نوع الـ interface — بيمرر الـ `union rapl_reg` وبيسأل الـ callback يعمل الـ read/write.

---

#### 2. `struct rapl_domain_data`

**الغرض:** تخزين snapshot من قيم الـ primitives لـ domain معين مع الـ timestamp.

```c
struct rapl_domain_data {
    u64 primitives[NR_RAPL_PRIMITIVES]; /* cached values from HW */
    unsigned long timestamp;            /* jiffies of last read */
};
```

- بيُستخدم لحساب `AVERAGE_POWER` من الفرق بين قراءتين.
- **الـ timestamp** بيحدد لو الـ cache قديمة محتاجة تتحدث.

---

#### 3. `struct rapl_power_limit`

**الغرض:** تمثيل Power Limit واحد (PL1, PL2, أو PL4) لـ domain معين، ومربوطه بالـ powercap framework.

```c
struct rapl_power_limit {
    struct powercap_zone_constraint *constraint; /* link to powercap layer */
    struct rapl_domain *domain;                  /* parent domain */
    const char *name;                            /* "long_term", "short_term", "peak_power" */
    bool locked;                                 /* HW/FW lock status */
    u64 last_power_limit;                        /* last written value (uW) */
};
```

- `NR_POWER_LIMITS = 3` (PL1, PL2, PL4).
- الـ `locked` بيمنع الـ userspace من الكتابة.
- الـ `constraint` هو الجسر مع الـ `/sys/class/powercap/`.

---

#### 4. `struct reg_action`

**الغرض:** وصف عملية read/write واحدة على register معين — يُمرَّر للـ callbacks.

```c
struct reg_action {
    union rapl_reg reg;  /* which register to access */
    u64 mask;            /* bitmask for the field of interest */
    u64 value;           /* value to write / value read back */
    int err;             /* error code if operation failed */
};
```

الـ mask بيسمح للـ core يعزل حقل معين من الـ register (مثلاً bits 14:0 للـ power limit).

---

#### 5. `struct rapl_domain`

**الغرض:** يمثل Power Domain واحد (مثلاً: package, core, DRAM) مع كل registers وبيانات وحدوده.

```c
struct rapl_domain {
    char name[RAPL_DOMAIN_NAME_LENGTH]; /* "package", "core", "dram", etc. */
    enum rapl_domain_type id;           /* RAPL_DOMAIN_PACKAGE, PP0, etc. */
    union rapl_reg regs[RAPL_DOMAIN_REG_MAX]; /* all registers for this domain */
    struct powercap_zone power_zone;    /* embedded powercap zone (sysfs node) */
    struct rapl_domain_data rdd;        /* cached primitive values + timestamp */
    struct rapl_power_limit rpl[NR_POWER_LIMITS]; /* PL1, PL2, PL4 */
    u64 attr_map;                       /* bitmask: which primitives exist on HW */
    unsigned int state;                 /* domain state flags */
    unsigned int power_unit;            /* scaling: raw -> uW */
    unsigned int energy_unit;           /* scaling: raw -> uJ */
    unsigned int time_unit;             /* scaling: raw -> us */
    struct rapl_package *rp;            /* back-pointer to parent package */
};
```

**نقاط مهمة:**
- الـ `power_zone` **embedded** مش pointer — يعني الـ `rapl_domain` هو نفسه الـ powercap zone.
- الـ `attr_map` بيتحدد وقت الـ probe عشان يعرف إيه الـ features الموجودة.
- الـ units (power/energy/time) بتيجي من `RAPL_DOMAIN_REG_UNIT` register.

---

#### 6. `struct rapl_if_priv`

**الغرض:** البيانات الخاصة بكل نوع Interface (MSR, MMIO, TPMI) — يعرّف الـ registers والـ callbacks الخاصة بيه.

```c
struct rapl_if_priv {
    enum rapl_if_type type;                        /* MSR / MMIO / TPMI */
    struct powercap_control_type *control_type;    /* parent powercap control */
    enum cpuhp_state pcap_rapl_online;             /* CPU hotplug state */
    union rapl_reg reg_unit;                       /* unit register address */
    union rapl_reg regs[RAPL_DOMAIN_MAX][RAPL_DOMAIN_REG_MAX]; /* register map */
    int limits[RAPL_DOMAIN_MAX];                   /* #power limits per domain */
    int (*read_raw)(int id, struct reg_action *ra, bool pmu_ctx); /* HW read */
    int (*write_raw)(int id, struct reg_action *ra);              /* HW write */
    void *defaults;  /* interface-specific default settings (opaque) */
    void *rpi;       /* interface-specific primitive info (opaque) */
};
```

- الـ `regs[RAPL_DOMAIN_MAX][RAPL_DOMAIN_REG_MAX]` هو الـ register map الكامل للـ interface.
- الـ `read_raw`/`write_raw` هما الـ abstraction layer للـ HW access.
- الـ `pcap_rapl_online` بيسجّل الـ CPU hotplug callbacks.

---

#### 7. `struct rapl_package_pmu_data` (CONFIG_PERF_EVENTS)

**الغرض:** بيانات الـ PMU الخاصة بكل package — لدعم `perf stat -e power/`.

```c
struct rapl_package_pmu_data {
    u64 scale[RAPL_DOMAIN_MAX]; /* 2^-32 Joules per counter increment, per domain */
    raw_spinlock_t lock;        /* protects n_active and active_list */
    int n_active;               /* number of active perf events */
    struct list_head active_list; /* list of active perf_event structs */
    ktime_t timer_interval;     /* hrtimer period to prevent counter wrap */
    struct hrtimer hrtimer;     /* periodic timer for counter overflow handling */
};
```

- الـ `scale` مختلف لكل domain — بيتحسب من الـ energy unit register.
- الـ `raw_spinlock_t` بدل `mutex` لأن الـ PMU callbacks بتتعمل في interrupt context.
- الـ `hrtimer` بيضمن إن الـ 32-bit energy counter ما يoverflow من غير ما حد يلاحظ.

---

#### 8. `struct rapl_package`

**الغرض:** تمثيل Package (Socket/Die) كاملة — أعلى مستوى في الـ RAPL hierarchy.

```c
struct rapl_package {
    unsigned int id;             /* logical die ID */
    unsigned int nr_domains;     /* number of active domains */
    unsigned long domain_map;    /* bitmap: bit N set = domain N active */
    struct rapl_domain *domains; /* array of rapl_domain, allocated at runtime */
    struct powercap_zone *power_zone; /* top-level powercap zone */
    unsigned long power_limit_irq;   /* bitmask: PL notify IRQ enable */
    struct list_head plist;      /* node in global package list */
    int lead_cpu;                /* one CPU per package for MSR access */
    struct cpumask cpumask;      /* all CPUs belonging to this package */
    char name[PACKAGE_DOMAIN_NAME_LENGTH]; /* "package-0-die-0" */
    struct rapl_if_priv *priv;   /* pointer to interface private data */
#ifdef CONFIG_PERF_EVENTS
    bool has_pmu;
    struct rapl_package_pmu_data pmu_data; /* embedded PMU data */
#endif
};
```

- الـ `domain_map` بـ bitmap بيخلي الـ iteration على الـ domains سريعة باستخدام `for_each_set_bit`.
- الـ `lead_cpu` مهم لأن MSR access لازم يتعمل على CPU معين من الـ package.
- الـ `domains` array بيتعمله `kzalloc` في الـ runtime بعد ما الـ probe يعرف عدد الـ domains.

---

### Struct Relationship Diagram

```
                    ┌─────────────────────────┐
                    │   rapl_if_priv           │
                    │  (MSR / MMIO / TPMI)    │
                    │                          │
                    │  control_type ─────────► powercap_control_type
                    │  regs[][]                │       │
                    │  read_raw()              │       │ (sysfs root)
                    │  write_raw()             │       │
                    └──────────┬───────────────┘       │
                               │ priv                  │
                    ┌──────────▼───────────────┐       │
                    │     rapl_package          │       │
                    │   (one per die/socket)   │       │
                    │                          │       │
                    │  domains[] ─────────────►│       │
                    │  domain_map (bitmap)      │       │
                    │  lead_cpu                 │       │
                    │  plist ──► [global list] │       │
                    │  pmu_data (if PMU)        │       │
                    └──────────┬───────────────┘       │
                               │ domains[N]            │
                    ┌──────────▼───────────────┐       │
                    │     rapl_domain           │       │
                    │  (package/PP0/PP1/DRAM..)│       │
                    │                          │       │
                    │  power_zone ─────────────┼───────►powercap_zone
                    │  (embedded, not pointer) │           │
                    │  regs[REG_MAX]           │           │ constraints[]
                    │  rdd (domain_data)        │           │
                    │  rpl[NR_POWER_LIMITS]────┼──┐        ▼
                    │  rp ──► rapl_package      │  │  powercap_zone_constraint
                    └──────────────────────────┘  │       ▲
                                                   │       │
                    ┌──────────────────────────┐   │       │
                    │   rapl_power_limit        │◄─┘       │
                    │  (PL1, PL2, PL4)         │           │
                    │                          │           │
                    │  constraint ─────────────┼───────────┘
                    │  domain ──► rapl_domain  │
                    │  locked                  │
                    │  last_power_limit        │
                    └──────────────────────────┘

                    ┌──────────────────────────┐
                    │    reg_action             │
                    │  (transient — per call)  │
                    │                          │
                    │  reg (union rapl_reg)     │
                    │  mask                    │
                    │  value                   │
                    │  err                     │
                    └──────────────────────────┘
                         ↑ used by read_raw/write_raw callbacks
```

---

### Lifecycle Diagram

```
╔══════════════════════════════════════════════════════════════╗
║               RAPL Driver Lifecycle                          ║
╚══════════════════════════════════════════════════════════════╝

  [Module Init / Platform Probe]
          │
          ▼
  powercap_register_control_type()
    → allocates powercap_control_type
    → creates /sys/class/powercap/intel-rapl/
          │
          ▼
  rapl_add_package()   ← called per CPU / per die
    → rapl_find_package_domain()  [check if exists]
    → kzalloc(rapl_package)
    → discover domains via rapl_if_priv->regs[]
    → kzalloc(domains array, nr_domains)
    → for each domain:
        powercap_register_zone()
          → creates /sys/.../intel-rapl:0/
          → sets up constraints (PL1, PL2, PL4)
    → register CPU hotplug notifier (pcap_rapl_online)
    → list_add(&rp->plist, &rapl_packages)
          │
          ▼  [CONFIG_PERF_EVENTS]
  rapl_package_add_pmu()
    → calculate scale[] per domain
    → setup hrtimer
    → perf_pmu_register()
          │
          ▼
  ┌─────────────────────────────────────────┐
  │         RUNTIME / OPERATIONAL           │
  │                                         │
  │  sysfs read  → powercap_zone_ops        │
  │    → rapl_get_power_uw()               │
  │      → read_raw(id, &reg_action, false) │
  │        → MSR/MMIO/TPMI register read   │
  │        → apply unit scaling            │
  │                                         │
  │  sysfs write → powercap_constraint_ops │
  │    → rapl_set_power_limit_uw()         │
  │      → check locked flag               │
  │      → write_raw(id, &reg_action)      │
  │        → MSR/MMIO/TPMI register write  │
  │                                         │
  │  CPU hotplug → update lead_cpu         │
  │              → update cpumask          │
  └─────────────────────────────────────────┘
          │
          ▼  [Module Exit / Driver Remove]
  rapl_remove_package()
    → [CONFIG_PERF_EVENTS] rapl_package_remove_pmu()
        → hrtimer_cancel()
        → perf_pmu_unregister()
    → for each domain (reverse order):
        powercap_unregister_zone()
    → list_del(&rp->plist)
    → kfree(domains)
    → kfree(rapl_package)
          │
          ▼
  powercap_unregister_control_type()
    → removes /sys/class/powercap/intel-rapl/
```

---

### Call Flow Diagrams

#### قراءة الـ Power (sysfs read)

```
user: cat /sys/.../intel-rapl:0/energy_uj
  │
  ▼
powercap_zone_ops->get_energy_uj()        [powercap framework]
  │
  ▼
rapl_get_energy_counter()                 [intel_rapl_common.c]
  │
  ├─ check: (jiffies - rdd.timestamp) > threshold?
  │    yes → need fresh read
  │
  ▼
rapl_read_data_raw(domain, ENERGY_COUNTER, false)
  │
  ▼
rapl_if_priv->read_raw(cpu_id, &reg_action, pmu_ctx=false)
  │
  ├─ [MSR path]  rdmsrl_on_cpu(lead_cpu, msr_num, &val)
  ├─ [MMIO path] readq(mmio_addr)
  └─ [TPMI path] tpmi_read64(offset)
  │
  ▼
reg_action.value = raw_bits
apply mask → extract energy field
scale: raw × energy_unit → microjoules
store in rdd.primitives[ENERGY_COUNTER]
update rdd.timestamp = jiffies
  │
  ▼
return value to sysfs / userspace
```

---

#### كتابة الـ Power Limit (sysfs write)

```
user: echo 15000000 > .../constraint_0_power_limit_uw
  │
  ▼
powercap_zone_constraint_ops->set_power_limit_uw()
  │
  ▼
rapl_set_power_limit(domain, PL1, value_uw)
  │
  ├─ check: rapl_power_limit.locked == true? → return -EROFS
  │
  ▼
rapl_write_data_raw(domain, POWER_LIMIT1, value_uw)
  │
  ▼
convert uW → raw HW units (÷ power_unit)
build reg_action { .reg, .mask, .value = raw }
  │
  ▼
rapl_if_priv->write_raw(cpu_id, &reg_action)
  │
  ├─ [MSR]  wrmsrl_on_cpu(lead_cpu, msr, val)
  ├─ [MMIO] writeq(val, mmio_addr)
  └─ [TPMI] tpmi_write64(offset, val)
  │
  ▼
store rpl[PL1].last_power_limit = value_uw
```

---

#### الـ PMU Counter Overflow Prevention

```
hrtimer fires (every timer_interval)
  │
  ▼
rapl_hrtimer_handle()
  │
  ▼
raw_spin_lock(&pmu_data.lock)
  │
  ▼
for each event in pmu_data.active_list:
  │
  ▼
  read_raw(id, &reg_action, pmu_ctx=true)   ← reads energy counter
    │
    ▼
  delta = current_raw - last_raw
  scaled_delta = delta × scale[domain]      ← convert to Joules (2^-32 J units)
  local64_add(scaled_delta, &event->count)
  │
  ▼
raw_spin_unlock(&pmu_data.lock)
  │
  ▼
hrtimer_forward_now(timer, timer_interval)
```

---

### Locking Strategy

#### جدول الـ Locks

| الـ Lock | النوع | أين يعيش | بيحمي إيه |
|----------|-------|-----------|-----------|
| `rapl_package_pmu_data.lock` | `raw_spinlock_t` | داخل `rapl_package` | `n_active`, `active_list` في الـ PMU |
| `powercap_control_type.lock` | `mutex` | داخل `powercap_control_type` | تسجيل وإلغاء تسجيل الـ zones |
| CPU hotplug lock (implicit) | `cpus_read_lock()` | kernel global | الـ `_cpuslocked` variants تتسمى بعد الـ lock |

---

#### تفاصيل الـ Locking

**الـ `raw_spinlock_t` في `rapl_package_pmu_data`:**
- بيتستخدم في الـ hrtimer interrupt context — لازم `raw_spinlock` مش `mutex`.
- الـ `rapl_package_add_pmu_locked()` بيُستدعى وهو محمل الـ lock بالفعل.
- الـ `rapl_package_add_pmu()` بيمسك الـ lock بنفسه — مناسب لـ normal context.

**الـ CPU Hotplug Lock:**
- الـ kernel بيستخدم `cpus_read_lock()` قبل ما يستدعي الـ functions الـ `_cpuslocked` variants.
- الـ `rapl_find_package_domain_cpuslocked()` و `rapl_add_package_cpuslocked()` بيفترضوا إن الـ caller مسك الـ lock.
- الـ non-locked variants (`rapl_add_package()`) بيمسكوا الـ CPU hotplug lock داخلياً.

**الـ `powercap_control_type.lock` (mutex):**
- بيحمي الـ `idr` وعدد الـ zones.
- بيتمسك خلال `powercap_register_zone()` و `powercap_unregister_zone()`.
- **لا يُمسك** خلال الـ sysfs read/write callbacks — الـ powercap framework بيعمل `device_lock()` بدلاً منه.

**ترتيب الـ Locks (Lock Ordering) — لتجنب الـ Deadlock:**

```
CPU Hotplug Lock (cpus_read_lock)
    └── powercap_control_type.lock (mutex)
            └── device_lock (per zone, during sysfs ops)
                    └── rapl_package_pmu_data.lock (raw_spinlock — IRQ-safe)
```

القاعدة: ما تعكسش الترتيب ده أبداً — خصوصاً لا تحاول تمسك الـ `raw_spinlock` وبعدين تطلب الـ `mutex`.

---

### ملاحظة على الـ `union rapl_reg` كـ Abstraction Layer

الـ union ده هو جوهر الـ portability في الكود:

```c
/* MSR interface sets: */
reg.msr = 0x610;  /* MSR_PKG_POWER_LIMIT */

/* MMIO interface sets: */
reg.mmio = base + 0x58;

/* The read_raw callback knows which to use based on rapl_if_type */
priv->read_raw(cpu_id, &reg_action, pmu_ctx);
```

الـ `rapl_domain.regs[RAPL_DOMAIN_REG_MAX]` array بيخزّن عنوان كل register للـ domain، والـ `rapl_if_priv.regs[RAPL_DOMAIN_MAX][RAPL_DOMAIN_REG_MAX]` هو الـ template اللي منه بيتنسخ للـ domain عند الـ initialization.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Package Management API

| Function | Locking | الغرض |
|---|---|---|
| `rapl_find_package_domain_cpuslocked()` | CPU hotplug lock held by caller | ابحث عن `rapl_package` موجود |
| `rapl_add_package_cpuslocked()` | CPU hotplug lock held by caller | أضف `rapl_package` جديد |
| `rapl_remove_package_cpuslocked()` | CPU hotplug lock held by caller | احذف `rapl_package` |
| `rapl_find_package_domain()` | يأخذ CPU hotplug lock داخلياً | wrapper آمن لـ `_find_cpuslocked` |
| `rapl_add_package()` | يأخذ CPU hotplug lock داخلياً | wrapper آمن لـ `_add_cpuslocked` |
| `rapl_remove_package()` | يأخذ CPU hotplug lock داخلياً | wrapper آمن لـ `_remove_cpuslocked` |

#### PMU API (CONFIG_PERF_EVENTS)

| Function | Locking | الغرض |
|---|---|---|
| `rapl_package_add_pmu()` | يأخذ lock داخلياً | سجّل PMU للـ package |
| `rapl_package_add_pmu_locked()` | lock held by caller | سجّل PMU (fast path) |
| `rapl_package_remove_pmu()` | يأخذ lock داخلياً | أزل PMU من الـ package |
| `rapl_package_remove_pmu_locked()` | lock held by caller | أزل PMU (fast path) |

---

### Group 1: Package Discovery & Registration

الـ **`rapl_package`** هو الوحدة الأساسية في RAPL — يمثل socket/die فيزيائي. كل package بيحتوي على مجموعة من الـ domains (Package، PP0، PP1، DRAM، PSys). الـ functions دي مسؤولة عن إنشاء وإيجاد وحذف الـ packages داخل الـ linked list اللي بتمسك كل الـ packages النشطة.

في الـ RAPL subsystem، الـ CPU hotplug lock بيحمي الـ package list لأن إضافة/إزالة الـ packages بتحصل في سياق CPU online/offline events. الـ `_cpuslocked` variants بتفترض إن الـ caller معاه الـ lock بالفعل (زي في hotplug callbacks)، والـ non-locked variants بتأخذ وبترخي الـ lock بنفسها.

---

#### `rapl_find_package_domain_cpuslocked()`

```c
struct rapl_package *rapl_find_package_domain_cpuslocked(
    int id,
    struct rapl_if_priv *priv,
    bool id_is_cpu);
```

بتدور في الـ global package list على `rapl_package` موجود يطابق الـ `id` والـ `priv` اللي اتبعتلها. لو `id_is_cpu` كان `true`، بتحوّل الـ CPU id لـ die/package id أولاً باستخدام topology APIs زي `topology_die_id()`. بترجع pointer للـ `rapl_package` لو لاقته، أو NULL لو مش موجود.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `id` | `int` | إما CPU number أو logical die ID حسب `id_is_cpu` |
| `priv` | `struct rapl_if_priv *` | الـ interface الخاص (MSR/MMIO/TPMI) — بيستخدم للتمييز بين packages من interfaces مختلفة |
| `id_is_cpu` | `bool` | لو `true` → `id` هو CPU number، لو `false` → `id` هو die ID مباشرة |

**Return:** `struct rapl_package *` — pointer للـ package الموجود، أو `NULL` لو مش موجود.

**Key Details:**
- **Locking:** يجب إن الـ CPU hotplug read lock يكون محجوز من الـ caller.
- بتدور بـ `list_for_each_entry` على الـ global package list مع مقارنة `rp->priv == priv` عشان تعزل packages من interfaces مختلفة.
- لو `id_is_cpu` كان `true`، بتعمل CPU-to-die mapping باستخدام `topology_die_id(cpu)` — ده مهم لأن الـ RAPL Package domain بيكون per-die مش per-CPU.

**Caller Context:** CPU hotplug callbacks، أو أي كود محجوز CPU hotplug lock.

---

#### `rapl_add_package_cpuslocked()`

```c
struct rapl_package *rapl_add_package_cpuslocked(
    int id,
    struct rapl_if_priv *priv,
    bool id_is_cpu);
```

بتنشئ `rapl_package` جديد، بتحدد domains المتاحة من الـ hardware، وبتسجلها كـ powercap zones تحت الـ `control_type` الخاص بالـ interface. ده الـ entry point الرئيسي لإضافة socket جديد للـ RAPL subsystem.

**Parameters:** نفس `rapl_find_package_domain_cpuslocked()`.

**Return:** pointer للـ `rapl_package` الجديد على النجاح، أو `ERR_PTR(-errno)` على الفشل.

**Pseudocode Flow:**
```
rapl_add_package_cpuslocked(id, priv, id_is_cpu):
    die_id = id_is_cpu ? topology_die_id(id) : id
    lead_cpu = id_is_cpu ? id : cpumask_any_in_die(die_id)

    rp = kzalloc(sizeof(*rp))
    rp->id = die_id
    rp->lead_cpu = lead_cpu
    rp->priv = priv

    /* اكتشف الـ domains من الـ hardware */
    rapl_detect_domains(rp)

    /* سجل مع powercap framework */
    rp->power_zone = powercap_register_zone(
        NULL, priv->control_type, rp->name,
        NULL, &zone_ops, 0, NULL)

    /* سجل كل domain كـ child zone */
    for each domain in rp->domains:
        rapl_register_domain(domain)

    list_add(&rp->plist, &rapl_packages)
    return rp
```

**Key Details:**
- **Locking:** CPU hotplug lock يجب يكون محجوز.
- بعد الـ registration مع powercap، الـ sysfs entries بتظهر تحت `/sys/class/powercap/intel-rapl/`.
- بتستخدم `priv->read_raw` callback لقراءة unit registers وتحديد power/energy/time units.
- لو الـ registration فشل، بتعمل cleanup كامل (unregister domains، kfree).

**Caller Context:** CPU hotplug online callbacks أو driver probe.

---

#### `rapl_remove_package_cpuslocked()`

```c
void rapl_remove_package_cpuslocked(struct rapl_package *rp);
```

بتعمل reverse لكل اللي عملته `rapl_add_package_cpuslocked` — بتسجّل الـ domains من الـ powercap framework، بتحذف الـ package من الـ global list، وبتحرر الـ memory.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `rp` | `struct rapl_package *` | الـ package المراد حذفه، يجب ألا يكون NULL |

**Return:** `void`

**Key Details:**
- **Locking:** CPU hotplug lock يجب يكون محجوز.
- بتعمل `powercap_unregister_zone()` لكل domain بالترتيب العكسي (children قبل parent).
- بتعمل `list_del(&rp->plist)` لإزالة الـ package من الـ global list.
- بتعمل `kfree(rp->domains)` ثم `kfree(rp)`.
- **Side Effect:** الـ sysfs entries بتختفي فور انتهاء الـ unregister.

**Caller Context:** CPU hotplug offline callbacks أو driver remove.

---

### Group 2: Package Management Wrappers (Non-locked)

الـ functions دي wrapper بسيطة حول الـ `_cpuslocked` variants. الهدف منها إنها تستخدم في contexts مش بيكون فيها CPU hotplug lock محجوز — زي `module_init` أو explicit driver initialization code.

---

#### `rapl_find_package_domain()`

```c
struct rapl_package *rapl_find_package_domain(
    int id,
    struct rapl_if_priv *priv,
    bool id_is_cpu);
```

بتأخذ CPU hotplug read lock، بتستدعي `rapl_find_package_domain_cpuslocked()`، وبترخي الـ lock. ده الـ safe wrapper للاستخدام في contexts عامة.

**Parameters:** نفس `rapl_find_package_domain_cpuslocked()`.

**Return:** `struct rapl_package *` أو `NULL`.

**Key Details:**
- بتستخدم `cpus_read_lock()` / `cpus_read_unlock()` داخلياً.
- مناسبة لـ one-shot lookups زي في `probe()` functions.

---

#### `rapl_add_package()`

```c
struct rapl_package *rapl_add_package(
    int id,
    struct rapl_if_priv *priv,
    bool id_is_cpu);
```

**الـ** wrapper الآمن لـ `rapl_add_package_cpuslocked()` — بتأخذ CPU hotplug lock وبترخيه.

**Return:** `struct rapl_package *` أو `ERR_PTR(-errno)`.

**Caller Context:** Driver `probe()` functions، module init.

---

#### `rapl_remove_package()`

```c
void rapl_remove_package(struct rapl_package *rp);
```

**الـ** wrapper الآمن لـ `rapl_remove_package_cpuslocked()` — بتأخذ CPU hotplug lock وبترخيه.

**Caller Context:** Driver `remove()` functions، module exit.

---

### Group 3: PMU Integration (CONFIG_PERF_EVENTS)

الـ RAPL PMU بيسمح لـ `perf stat` يقرأ energy counters كـ hardware performance events. الـ `rapl_package_pmu_data` بيحتوي على hrtimer بيشتغل بشكل دوري عشان يمنع الـ 32-bit energy counter من الـ overflow (الـ counter بيلف كل بضع ثواني حسب الـ TDP).

الـ PMU functions موجودة في نسختين: واحدة بتأخذ lock (للـ external callers)، وواحدة تانية بتفترض الـ lock محجوز (fast path للـ callers اللي عندهم الـ lock بالفعل).

---

#### `rapl_package_add_pmu()`

```c
int rapl_package_add_pmu(struct rapl_package *rp);
```

بتسجل الـ RAPL energy counters كـ perf PMU events للـ package ده. بتهيئ الـ `rapl_package_pmu_data` (الـ scale values والـ hrtimer)، وبتسجل الـ PMU مع الـ perf subsystem.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `rp` | `struct rapl_package *` | الـ package المراد تسجيل PMU ليه |

**Return:** `0` على النجاح، `errno` على الفشل.

**Key Details:**
- بتحسب `pmu_data.scale[domain]` كـ `2^-32 Joules` per counter increment لكل domain.
- بتهيئ `pmu_data.hrtimer` باستخدام `hrtimer_init()` بـ `CLOCK_MONOTONIC`.
- الـ `timer_interval` بيتحسب بناءً على أصغر `energy_unit` عشان نضمن إن الـ counter ميلفش قبل ما الـ timer يشوفه.
- بتضبط `rp->has_pmu = true` على النجاح.
- **Only available when** `CONFIG_PERF_EVENTS=y`.

**Caller Context:** Driver probe أو package initialization، بعد `rapl_add_package()`.

---

#### `rapl_package_add_pmu_locked()`

```c
int rapl_package_add_pmu_locked(struct rapl_package *rp);
```

نفس `rapl_package_add_pmu()` بس بتفترض إن الـ caller عنده الـ relevant lock محجوز بالفعل. Fast path للـ hotplug callbacks.

**Return:** `0` على النجاح، `errno` على الفشل.

**Key Details:**
- مفيش locking داخلي — الـ caller مسؤول عن الـ synchronization.

---

#### `rapl_package_remove_pmu()`

```c
void rapl_package_remove_pmu(struct rapl_package *rp);
```

بتلغي تسجيل الـ PMU وبتوقف الـ hrtimer. بتعمل cleanup لـ `pmu_data` وبتضبط `rp->has_pmu = false`.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `rp` | `struct rapl_package *` | الـ package المراد إزالة PMU منه |

**Return:** `void`

**Key Details:**
- بتستدعي `hrtimer_cancel()` عشان توقف الـ periodic timer.
- يجب استدعاؤها قبل `rapl_remove_package()` لو الـ PMU اتضافت.
- **Only available when** `CONFIG_PERF_EVENTS=y`.
- لو `CONFIG_PERF_EVENTS=n`، الـ function دي `static inline` فارغة — zero overhead.

---

#### `rapl_package_remove_pmu_locked()`

```c
void rapl_package_remove_pmu_locked(struct rapl_package *rp);
```

نفس `rapl_package_remove_pmu()` بس بتفترض إن الـ caller عنده الـ lock. Fast path للـ hotplug callbacks.

---

### Group 4: الـ Callbacks المطلوبة من الـ Backend Drivers

الـ `rapl_if_priv` بيحتوي على function pointers لازم الـ backend drivers (MSR، MMIO، TPMI) تملأها. دول الـ interface بين الـ core RAPL logic والـ hardware-specific register access.

---

#### `read_raw` callback

```c
int (*read_raw)(int id, struct reg_action *ra, bool pmu_ctx);
```

الـ backend driver بيوفر الـ function دي لقراءة raw register value. الـ core RAPL code بتستدعيها لكل عملية قراءة من الـ hardware.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `id` | `int` | CPU/die ID اللي المفروض تحصل القراءة عليه (للـ MSR، ده CPU affinity) |
| `ra` | `struct reg_action *` | يحتوي على `reg` (عنوان الـ register) و`mask` و`value` (output) و`err` (output) |
| `pmu_ctx` | `bool` | لو `true` → القراءة في سياق PMU sampling (ممكن تحتاج fast path) |

**Return:** `0` على النجاح، `errno` على الفشل. كمان `ra->err` بيتضبط.

**Key Details:**
- للـ MSR backend: بتستخدم `rdmsrl_on_cpu(id, ra->reg.msr, &ra->value)`.
- للـ MMIO backend: بتستخدم `readq(ra->reg.mmio)`.
- الـ `mask` بيتطبق على الـ value المقروء عشان يعزل الـ bits المطلوبة.

---

#### `write_raw` callback

```c
int (*write_raw)(int id, struct reg_action *ra);
```

الـ backend driver بيوفر الـ function دي لكتابة raw register value. الـ core RAPL code بتستدعيها عند تغيير power limits أو time windows.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `id` | `int` | CPU/die ID |
| `ra` | `struct reg_action *` | يحتوي على `reg` و`mask` و`value` (الـ bits اللي هتتكتب) |

**Return:** `0` على النجاح، `errno` على الفشل.

**Key Details:**
- بتعمل read-modify-write: بتقرأ الـ register الحالي، بتطبق `mask` و`value`، وبتكتب التاني.
- للـ MSR backend: `wrmsrl_on_cpu(id, ra->reg.msr, new_val)`.
- **لا يُستدعى في PMU context** — لذلك مفيش `pmu_ctx` parameter.

---

### Architecture Diagram

```
                    ┌─────────────────────────────────────┐
                    │         intel_rapl_common             │
                    │                                       │
                    │  rapl_add_package()                   │
                    │  rapl_find_package_domain()           │
                    │  rapl_remove_package()                │
                    └──────────┬───────────────────────────┘
                               │ powercap_register_zone()
                               ▼
                    ┌─────────────────────┐
                    │  powercap framework │
                    │  /sys/class/powercap│
                    └─────────────────────┘

   ┌─────────────────┐    rapl_if_priv     ┌──────────────────┐
   │  intel_rapl_msr │◄───────────────────►│ intel_rapl_mmio  │
   │  (MSR backend)  │    .read_raw        │  (MMIO backend)  │
   │                 │    .write_raw       │                  │
   └────────┬────────┘                    └──────────────────┘
            │ rdmsrl_on_cpu()                  readq() / writeq()
            ▼
   ┌─────────────────┐
   │  CPU MSR regs   │
   │  (per package)  │
   └─────────────────┘

   ┌──────────────────────────────────────────────────┐
   │                 rapl_package                     │
   │  ┌───────────┐  ┌───────────┐  ┌──────────────┐ │
   │  │  PACKAGE  │  │   PP0     │  │    DRAM      │ │
   │  │  domain   │  │  domain   │  │   domain     │ │
   │  │  PL1/PL2  │  │  PL1 only │  │  PL1 only    │ │
   │  └───────────┘  └───────────┘  └──────────────┘ │
   │                                                  │
   │  rapl_package_pmu_data (if CONFIG_PERF_EVENTS)   │
   │  ┌──────────────────────────────────────────┐    │
   │  │ hrtimer → prevents energy counter overflow│    │
   │  │ scale[] → Joule conversion per domain     │    │
   │  └──────────────────────────────────────────┘    │
   └──────────────────────────────────────────────────┘
```

---

### ملاحظات مهمة عن الـ Locking Model

الـ RAPL subsystem بيستخدم **CPU hotplug lock** (`cpus_read_lock`/`cpus_write_lock`) لحماية الـ package list لأن:

1. إضافة/إزالة packages بتحصل في CPU hotplug callbacks.
2. الـ `lead_cpu` (الـ CPU المستخدم لقراءة MSR) ممكن يروح offline، فلازم نختار غيره.
3. الـ `_cpuslocked` variants موجودة للـ hotplug callbacks نفسها (اللي بتشتغل وهي ممسكة الـ lock).
4. الـ non-locked variants للـ external callers اللي مش في hotplug context.

لو استدعيت `rapl_add_package_cpuslocked()` من context مش ماسك CPU hotplug lock → **deadlock أو data corruption** محتمل.
## Phase 5: دليل الـ Debugging الشامل

الـ **Intel RAPL** (Running Average Power Limit) subsystem بيتحكم في power limits للـ CPU packages, cores, uncore, DRAM, وـ Platform (PSys). الـ debugging هنا بيشمل مستويين: software (sysfs/debugfs/ftrace) وـ hardware (MSR registers/MMIO).

---

### Software Level

#### 1. Debugfs Entries

الـ RAPL مش بيستخدم debugfs بشكل مباشر كتير، بس الـ powercap framework وـ perf PMU بيظهروا بيانات مفيدة:

```bash
# Mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف الـ RAPL PMU events المتاحة
ls /sys/kernel/debug/tracing/events/power/

# شوف الـ powercap control type المسجل
ls /sys/kernel/debug/
```

الـ RAPL driver مش بيعمل debugfs entries خاصة بيه، كل البيانات بتيجي من sysfs powercap interface.

---

#### 2. Sysfs Entries

الـ **powercap sysfs** هو الواجهة الأساسية لـ RAPL. كل `rapl_domain` بيظهر كـ `powercap_zone`.

```
/sys/class/powercap/
└── intel-rapl/                          ← control_type (rapl_if_priv→control_type)
    ├── enabled
    ├── intel-rapl:0/                    ← rapl_package id=0 (Package domain)
    │   ├── name                         ← rapl_domain→name
    │   ├── energy_uj                    ← ENERGY_COUNTER primitive
    │   ├── max_energy_range_uj
    │   ├── power_uw                     ← AVERAGE_POWER primitive
    │   ├── enabled
    │   ├── constraint_0_power_limit_uw  ← POWER_LIMIT1 / rapl_power_limit
    │   ├── constraint_0_time_window_us  ← TIME_WINDOW1
    │   ├── constraint_0_name            ← "long_term"
    │   ├── constraint_1_power_limit_uw  ← POWER_LIMIT2
    │   ├── constraint_1_time_window_us  ← TIME_WINDOW2
    │   ├── constraint_1_name            ← "short_term"
    │   ├── intel-rapl:0:0/             ← PP0 (cores)
    │   └── intel-rapl:0:1/             ← PP1 (uncore/GPU) أو DRAM
    └── intel-rapl:1/                    ← package 1 (multi-socket)
```

```bash
# اقرأ الـ energy counter للـ package 0
cat /sys/class/powercap/intel-rapl/intel-rapl:0/energy_uj

# اقرأ الـ power limit الحالي (micro-watts)
cat /sys/class/powercap/intel-rapl/intel-rapl:0/constraint_0_power_limit_uw

# اقرأ الـ time window
cat /sys/class/powercap/intel-rapl/intel-rapl:0/constraint_0_time_window_us

# شوف اسم الـ domain
cat /sys/class/powercap/intel-rapl/intel-rapl:0/name

# شوف الـ constraint مـ locked ولا لأ (FW_LOCK / PL1_LOCK)
cat /sys/class/powercap/intel-rapl/intel-rapl:0/constraint_0_max_power_uw

# اطبع كل الـ powercap zones دفعة واحدة
find /sys/class/powercap/ -name "*.uj" -o -name "*power_limit*" | sort | xargs -I{} sh -c 'echo "{}:"; cat {}'
```

---

#### 3. Ftrace — Tracepoints والـ Events

```bash
# فعّل الـ tracing للـ power management events
echo 1 > /sys/kernel/debug/tracing/events/power/enable

# events مفيدة بشكل خاص للـ RAPL
echo 1 > /sys/kernel/debug/tracing/events/power/cpu_frequency/enable
echo 1 > /sys/kernel/debug/tracing/events/power/cpu_idle/enable

# trace الـ MSR reads/writes (لو MSR interface)
echo 1 > /sys/kernel/debug/tracing/events/msr/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 5
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep -i rapl

# function tracer على الـ RAPL functions
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'rapl_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 2
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

---

#### 4. Printk / Dynamic Debug

```bash
# فعّل dynamic debug للـ RAPL modules
echo 'module intel_rapl_common +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module intel_rapl_msr +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module intel_rapl_mmio +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل الـ debug messages في الـ RAPL source files
echo 'file intel_rapl_common.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file rapl.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug الـ active
cat /sys/kernel/debug/dynamic_debug/control | grep rapl

# شوف الـ kernel log فوراً
dmesg -w | grep -i rapl

# رفع مستوى الـ printk لـ KERN_DEBUG
echo 8 > /proc/sys/kernel/printk
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوصف |
|---|---|
| `CONFIG_INTEL_RAPL` | الـ RAPL core support |
| `CONFIG_INTEL_RAPL_MSR` | MSR interface driver |
| `CONFIG_INTEL_RAPL_MMIO` | MMIO interface driver |
| `CONFIG_POWERCAP` | الـ powercap framework الأساسي |
| `CONFIG_PERF_EVENTS` | يفعّل `rapl_package_pmu_data` وـ PMU support |
| `CONFIG_DEBUG_FS` | لازم لـ debugfs access |
| `CONFIG_DYNAMIC_DEBUG` | بيفعّل dynamic debug |
| `CONFIG_TRACING` | بيفعّل ftrace |
| `CONFIG_MSR` | بيسمح بقراءة MSR من userspace |
| `CONFIG_X86_MSR` | MSR device driver (`/dev/cpu/*/msr`) |
| `CONFIG_DEBUG_KERNEL` | general kernel debug |
| `CONFIG_LOCKDEP` | لو في مشكلة في الـ `raw_spinlock_t` في `rapl_package_pmu_data` |
| `CONFIG_PROVE_LOCKING` | يكتشف deadlocks في الـ RAPL locks |
| `CONFIG_KCSAN` | data race detection في الـ PMU timer code |

```bash
# تحقق من الـ config الحالي
grep -E 'RAPL|POWERCAP|PERF_EVENTS' /boot/config-$(uname -r)
```

---

#### 6. Subsystem-Specific Tools

```bash
# أداة turbostat: بتقرأ RAPL عبر MSR مباشرة وبتعرض power بشكل readable
turbostat --show PkgWatt,CorWatt,RAMWatt,GFXWatt --interval 1

# perf stat مع RAPL PMU events
perf stat -e power/energy-pkg/,power/energy-cores/,power/energy-ram/ -a sleep 5

# اطبع كل الـ RAPL perf events المتاحة
perf list | grep -i rapl
perf list | grep -i power

# rapl-read tool (لو متاح)
# أو استخدم powertop
powertop --time=5

# rdmsr لقراءة MSR registers مباشرة
# MSR_PKG_POWER_LIMIT = 0x610
modprobe msr
rdmsr -a 0x610   # Package Power Limit (POWER_LIMIT1 + POWER_LIMIT2)
rdmsr -a 0x611   # MSR_PKG_ENERGY_STATUS (ENERGY_COUNTER)
rdmsr -a 0x614   # MSR_PKG_POWER_INFO (THERMAL_SPEC_POWER, MAX_POWER, MIN_POWER)
rdmsr -a 0x606   # MSR_RAPL_POWER_UNIT (energy/power/time units)
rdmsr -a 0x638   # MSR_PP0_POWER_LIMIT (PP0/cores)
rdmsr -a 0x639   # MSR_PP0_ENERGY_STATUS
rdmsr -a 0x640   # MSR_PP1_POWER_LIMIT (PP1/GPU)
rdmsr -a 0x61C   # MSR_DRAM_POWER_LIMIT
rdmsr -a 0x65C   # MSR_PLATFORM_POWER_LIMIT (PSys)
```

---

#### 7. Common Error Messages → المعنى → الحل

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `RAPL: no valid RAPL interface found` | الـ CPU مش بيدعم RAPL أو الـ config غلط | تحقق من CPU generation وـ `CONFIG_INTEL_RAPL` |
| `RAPL: failed to read MSR 0x606` | فشل قراءة unit register | تحقق من `CONFIG_X86_MSR` وـ CPU privilege |
| `powercap: Domain X: power zone registration failed` | تعارض في الـ idr أو memory allocation | تحقق من `dmesg` للـ OOM errors |
| `RAPL: domain X power limit locked by BIOS` | الـ `FW_LOCK` أو `PL1_LOCK` bit مضروب | اتعامل مع BIOS settings أو اقبل الـ read-only |
| `intel_rapl_msr: CPU X package Y not found` | CPU hotplug race أو package mapping غلط | تحقق من الـ `lead_cpu` وـ `domain_map` |
| `RAPL: energy counter overflow` | الـ `ENERGY_COUNTER` wraparound | الـ driver المفروض يتعامل معاه، تحقق من `rapl_package_pmu_data→hrtimer` |
| `RAPL package: no active CPUs` | كل الـ CPUs في الـ package offline | تحقق من CPU hotplug logic |
| `failed to register RAPL PMU` | `CONFIG_PERF_EVENTS` مش enabled | فعّل الـ config أو ignore لو PMU مش محتاجه |
| `RAPL: invalid primitive value` | بيانات غلط من الـ MSR/MMIO | تحقق من register addresses في `rapl_if_priv→regs` |

---

#### 8. Strategic Points لـ dump_stack() / WARN_ON()

نقاط استراتيجية لو كنت بتكتب RAPL driver أو بتـ debug مشكلة محددة:

```c
/* في rapl_find_package_domain — لو رجع NULL فجأة */
struct rapl_package *rp = rapl_find_package_domain(id, priv, true);
WARN_ON(!rp);

/* في read_raw callback — لو reg مش initialized */
WARN_ON(!ra->reg.msr && !ra->reg.mmio);

/* في rapl_domain setup — لو domain_map فاضي */
WARN_ON(!rp->domain_map);

/* في PMU timer callback — لو scale == 0 */
WARN_ON(!rp->pmu_data.scale[domain_id]);

/* لو rapl_power_limit locked بشكل غير متوقع */
if (WARN_ON(rpl->locked))
    dump_stack();  /* يطبع الـ call chain كامل */

/* في reg_action error path */
if (ra->err) {
    WARN_ONCE(1, "RAPL: reg action failed, err=%d reg=0x%x\n",
              ra->err, ra->reg.msr);
    dump_stack();
}
```

---

### Hardware Level

#### 1. التحقق أن الـ Hardware State يطابق الـ Kernel State

```bash
# قارن الـ PL1 في kernel مع MSR مباشرة
# kernel value (micro-watts)
cat /sys/class/powercap/intel-rapl/intel-rapl:0/constraint_0_power_limit_uw

# MSR value — الـ bits [14:0] هي PL1 بـ units من MSR_RAPL_POWER_UNIT
rdmsr 0x610
# PL1 = bits[14:0], PL1_ENABLE = bit[15], PL2 = bits[46:32], PL2_ENABLE = bit[47]

# unit register — بيعرّف power/energy/time units
rdmsr 0x606
# Power unit = 2^-(bits[3:0]) watts
# Energy unit = 2^-(bits[12:8]) joules
# Time unit = 2^-(bits[19:16]) seconds

# Script يقارن kernel vs MSR
python3 << 'EOF'
import subprocess, struct

def rdmsr(reg):
    out = subprocess.check_output(['rdmsr', '-d', hex(reg)])
    return int(out.strip())

unit_msr = rdmsr(0x606)
power_unit = 2 ** -(unit_msr & 0xF)          # watts
energy_unit = 2 ** -((unit_msr >> 8) & 0x1F) # joules
time_unit = 2 ** -((unit_msr >> 16) & 0xF)   # seconds
print(f"Power unit:  {power_unit*1e6:.3f} µW")
print(f"Energy unit: {energy_unit*1e6:.3f} µJ")
print(f"Time unit:   {time_unit*1e6:.3f} µs")

pkg_limit = rdmsr(0x610)
pl1_raw = pkg_limit & 0x7FFF
pl1_watts = pl1_raw * power_unit
print(f"\nMSR PL1: {pl1_watts:.2f} W = {pl1_watts*1e6:.0f} µW")

with open('/sys/class/powercap/intel-rapl/intel-rapl:0/constraint_0_power_limit_uw') as f:
    kernel_uw = int(f.read())
print(f"Kernel PL1: {kernel_uw} µW = {kernel_uw/1e6:.2f} W")

diff = abs(pl1_watts*1e6 - kernel_uw)
print(f"Difference: {diff:.0f} µW {'✓ OK' if diff < 1000 else '✗ MISMATCH!'}")
EOF
```

---

#### 2. Register Dump Techniques

```bash
# الـ MSRs الأساسية لـ RAPL — اقرأهم كلهم دفعة
modprobe msr

echo "=== RAPL Unit Register ==="
rdmsr -p 0 0x606   # MSR_RAPL_POWER_UNIT — نفس على كل cores

echo "=== Package Domain ==="
rdmsr -p 0 0x610   # MSR_PKG_POWER_LIMIT    → RAPL_DOMAIN_REG_LIMIT
rdmsr -p 0 0x611   # MSR_PKG_ENERGY_STATUS  → RAPL_DOMAIN_REG_STATUS
rdmsr -p 0 0x613   # MSR_PKG_PERF_STATUS    → RAPL_DOMAIN_REG_PERF
rdmsr -p 0 0x614   # MSR_PKG_POWER_INFO     → RAPL_DOMAIN_REG_INFO

echo "=== PP0 (Cores) Domain ==="
rdmsr -p 0 0x638   # MSR_PP0_POWER_LIMIT
rdmsr -p 0 0x639   # MSR_PP0_ENERGY_STATUS
rdmsr -p 0 0x63A   # MSR_PP0_POLICY         → RAPL_DOMAIN_REG_POLICY
rdmsr -p 0 0x63B   # MSR_PP0_PERF_STATUS

echo "=== PP1 (Uncore/GPU) Domain ==="
rdmsr -p 0 0x640   # MSR_PP1_POWER_LIMIT
rdmsr -p 0 0x641   # MSR_PP1_ENERGY_STATUS
rdmsr -p 0 0x642   # MSR_PP1_POLICY

echo "=== DRAM Domain ==="
rdmsr -p 0 0x61C   # MSR_DRAM_POWER_LIMIT
rdmsr -p 0 0x61D   # MSR_DRAM_ENERGY_STATUS
rdmsr -p 0 0x61E   # MSR_DRAM_PERF_STATUS
rdmsr -p 0 0x61F   # MSR_DRAM_POWER_INFO

echo "=== PSys (Platform) Domain ==="
rdmsr -p 0 0x65C   # MSR_PLATFORM_POWER_LIMIT
rdmsr -p 0 0x64D   # MSR_PLATFORM_ENERGY_STATUS

# لو MMIO interface (RAPL_IF_MMIO) — استخدم devmem2
# devmem2 [physical_address] [type: b/h/w/l]
# العنوان بييجي من BIOS/ACPI tables
devmem2 0xFED159A0 w   # مثال على MMIO RAPL address — يختلف حسب platform
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

لما بيبقى في مشكلة hardware-level في الـ RAPL power delivery:

```
Measurement Points:
┌─────────────────────────────────────────────────────────┐
│  RAPL Hardware Debug Points                             │
│                                                         │
│  VR_HOT pin ────────► oscilloscope ch1                  │
│  (Voltage Regulator overheat = throttling)              │
│                                                         │
│  PROCHOT# pin ──────► oscilloscope ch2                  │
│  (active low = CPU requesting throttle)                 │
│                                                         │
│  CPU VCC rail ──────► oscilloscope ch3 (current probe)  │
│  (شوف الـ current spikes لما PL2 يتفعل)                 │
│                                                         │
│  DRAM VDD ──────────► oscilloscope ch4                  │
│  (لو DRAM RAPL domain مفعّل)                            │
└─────────────────────────────────────────────────────────┘
```

- **Sample Rate**: 1 MSa/s كافية لـ power limit events (time window ≥ 1ms)
- **Trigger**: على الـ PROCHOT# falling edge علشان تمسك بداية الـ throttle event
- **Logic Analyzer**: اربط على الـ I2C/PMBus بين CPU وـ VR علشان تشوف power commands

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log | التحقق |
|---|---|---|
| CPU يتجاوز الـ TDP باستمرار | `RAPL: PL2 exceeded repeatedly` + performance drop | راقب `turbostat` column `PkgWatt` vs TDP |
| BIOS lock على الـ power limits | الـ `constraint_0_power_limit_uw` read-only, `FW_LOCK` bit=1 | `rdmsr 0x610` bit[31] = 1 |
| Thermal throttling بسبب RAPL | `cpu: cpu X package Y throttled` في dmesg | `rdmsr 0x1B1` (IA32_THERM_STATUS) bit[4] |
| DRAM domain مش موجود | الـ DRAM zone مش موجود في sysfs | `rdmsr 0x61D` — لو صفر المحتوى دايماً، الـ CPU مش بيدعمه |
| PSys domain مش شغال | لا يوجد `intel-rapl:X` للـ platform | PSys محتاج platform firmware support |
| Energy counter مش بيزيد | `energy_uj` ثابت مع الوقت | تحقق من CPU power state، لو C10 دايماً clock gated |
| PMU overflow كتير | RAPL PMU events بيرجعوا values غريبة | `rapl_package_pmu_data→timer_interval` صغير جداً |

```bash
# تحقق من PROCHOT وـ throttle status
rdmsr 0x19C    # IA32_THERM_STATUS — bit[4] = throttle active
rdmsr 0x1B1    # IA32_PACKAGE_THERM_STATUS

# شوف الـ RAPL interrupt status
rdmsr 0x611    # MSR_PKG_ENERGY_STATUS — الـ counter wraparound
```

---

#### 5. Device Tree Debugging

الـ RAPL على الـ x86 مش بيستخدم Device Tree، بيستخدم ACPI بدلاً منه. بس الـ **TPMI interface** (`RAPL_IF_TPMI`) بيستخدم ACPI + platform device:

```bash
# شوف الـ ACPI tables المتعلقة بـ RAPL
acpidump | grep -i rapl
acpidump | grep -i psys

# الـ TPMI RAPL — بيظهر كـ platform device في sysfs
ls /sys/bus/platform/devices/ | grep tpmi
ls /sys/bus/platform/devices/ | grep rapl

# تحقق من الـ ACPI _PSS (Performance Supported States) وـ _PPC
acpidump -n SSDT | acpixtract -a && iasl -d SSDT.dat 2>/dev/null
grep -i "rapl\|power.limit" dsdt.dsl 2>/dev/null

# لو MMIO interface — تحقق من ACPI BERT/EINJ tables
# الـ physical addresses بتيجي من ACPI
dmesg | grep -i "rapl\|mmio\|tpmi"

# شوف الـ power management ACPI objects
cat /sys/bus/acpi/devices/*/status 2>/dev/null | head -20
ls /sys/firmware/acpi/tables/
```

---

### Practical Commands

#### Script شامل لـ RAPL Health Check

```bash
#!/bin/bash
# rapl_debug.sh — RAPL subsystem health check

echo "========================================"
echo "RAPL Subsystem Debug Report"
echo "========================================"

echo ""
echo "### 1. Kernel Modules ###"
lsmod | grep -E "rapl|powercap"

echo ""
echo "### 2. Powercap Zones ###"
find /sys/class/powercap -name "name" | sort | while read f; do
    zone=$(dirname $f)
    name=$(cat $f)
    energy=$(cat $zone/energy_uj 2>/dev/null || echo "N/A")
    echo "  Zone: $name | energy_uj: $energy"
done

echo ""
echo "### 3. Power Limits ###"
for zone in /sys/class/powercap/intel-rapl/intel-rapl:*; do
    name=$(cat $zone/name 2>/dev/null)
    for c in 0 1 2; do
        lim=$zone/constraint_${c}_power_limit_uw
        tw=$zone/constraint_${c}_time_window_us
        cname=$zone/constraint_${c}_name
        [ -f "$lim" ] || continue
        echo "  $name/constraint_$c: $(cat $cname 2>/dev/null) | limit=$(cat $lim)µW | window=$(cat $tw)µs"
    done
done

echo ""
echo "### 4. MSR Unit Register (CPU 0) ###"
modprobe msr 2>/dev/null
if command -v rdmsr &>/dev/null; then
    unit=$(rdmsr -d -p 0 0x606 2>/dev/null)
    if [ "$unit" ]; then
        pu=$((unit & 0xF))
        eu=$(( (unit >> 8) & 0x1F ))
        tu=$(( (unit >> 16) & 0xF ))
        echo "  Power unit:  2^-$pu W = $(python3 -c "print(f'{2**-$pu*1e6:.4f} µW')")"
        echo "  Energy unit: 2^-$eu J = $(python3 -c "print(f'{2**-$eu*1e6:.4f} µJ')")"
        echo "  Time unit:   2^-$tu s"
    fi
else
    echo "  rdmsr not available — install msr-tools"
fi

echo ""
echo "### 5. Package Power Limit MSR (CPU 0) ###"
if command -v rdmsr &>/dev/null; then
    pl=$(rdmsr -p 0 0x610 2>/dev/null)
    echo "  MSR_PKG_POWER_LIMIT (0x610) = $pl"
fi

echo ""
echo "### 6. Recent RAPL Kernel Messages ###"
dmesg | grep -iE "rapl|powercap|power.limit|throttl" | tail -20

echo ""
echo "### 7. RAPL PMU Events Available ###"
perf list 2>/dev/null | grep -i "power/" | head -10

echo ""
echo "### 8. turbostat (5s snapshot) ###"
if command -v turbostat &>/dev/null; then
    turbostat --show PkgWatt,CorWatt,RAMWatt --interval 1 --num_iterations 3 2>/dev/null
else
    echo "  turbostat not installed"
fi
```

```bash
chmod +x rapl_debug.sh
sudo ./rapl_debug.sh
```

---

#### قراءة Energy Counter وحساب الـ Average Power

```bash
#!/bin/bash
# احسب الـ average power للـ package خلال 10 ثواني

ZONE="/sys/class/powercap/intel-rapl/intel-rapl:0"
E1=$(cat $ZONE/energy_uj)
T1=$(date +%s%N)
sleep 10
E2=$(cat $ZONE/energy_uj)
T2=$(date +%s%N)

DELTA_E=$((E2 - E1))   # micro-joules
DELTA_T=$((T2 - T1))   # nano-seconds

# Power = Energy / Time (micro-joules / milli-seconds = milli-watts)
POWER_MW=$(python3 -c "print(f'{$DELTA_E / ($DELTA_T / 1e6):.2f} mW')")
echo "Average Package Power: $POWER_MW"
```

**تفسير الـ Output:**
```
Average Package Power: 12450.33 mW   ← ~12.5W، طبيعي للـ idle system
Average Package Power: 65000.00 mW   ← 65W، الـ CPU تحت load كامل
Average Package Power: 95000.00 mW   ← > TDP، الـ PL2 short-term burst
```

---

#### مراقبة الـ Power Limit Lock Status

```bash
# شوف لو الـ BIOS عامل lock على الـ power limits
# FW_LOCK = bit[31] في MSR_PKG_POWER_LIMIT
rdmsr -p 0 0x610 | python3 -c "
import sys
val = int(input(), 16)
pl1_locked = bool(val & (1 << 31))
pl2_locked = bool(val & (1 << 63))
pl1_enabled = bool(val & (1 << 15))
pl2_enabled = bool(val & (1 << 47))
pl1_raw = val & 0x7FFF
pl2_raw = (val >> 32) & 0x7FFF
print(f'PL1 Locked by BIOS: {pl1_locked}')   # rapl_power_limit→locked
print(f'PL2 Locked by BIOS: {pl2_locked}')
print(f'PL1 Enabled: {pl1_enabled}')          # PL1_ENABLE primitive
print(f'PL2 Enabled: {pl2_enabled}')          # PL2_ENABLE primitive
print(f'PL1 raw value: {pl1_raw}')
print(f'PL2 raw value: {pl2_raw}')
"
```

**تفسير الـ Output:**
```
PL1 Locked by BIOS: True   ← مش قادر تغير الـ power limit من kernel
PL2 Locked by BIOS: False  ← PL2 قابل للتغيير
PL1 Enabled: True          ← PL1_ENABLE bit set
PL2 Enabled: True          ← PL2_ENABLE bit set
PL1 raw value: 4096        ← القيمة × power_unit = الـ watt limit الفعلي
```

---

#### Perf RAPL Energy Measurement

```bash
# قياس energy consumption لأمر معين
perf stat -e power/energy-pkg/,power/energy-cores/,power/energy-ram/ \
    -a -- sleep 5

# مثال على الـ output وتفسيره:
# Performance counter stats for 'system wide':
#
#           45.67 Joules  power/energy-pkg/     ← Package total
#           23.10 Joules  power/energy-cores/   ← PP0 cores فقط
#            8.45 Joules  power/energy-ram/      ← DRAM
#
#        5.001234567 seconds time elapsed

# فرق بين pkg وـ cores = uncore + GPU power
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Package Power Limit مش شغال على Server بـ Intel Xeon Scalable

#### العنوان
**FW_LOCK** بيمنع تعديل `POWER_LIMIT1` على Xeon Scalable في Data Center

#### السياق
فريق power management في شركة cloud بتستخدم Intel Xeon Scalable Processors (Sapphire Rapids) على rack servers. بيحاولوا يضبطوا **PL1** (long-term power limit) للـ package domain عشان يتحكموا في power budget للـ rack، خصوصاً وقت peak workload.

#### المشكلة
بيعملوا write على:
```bash
echo 150000000 > /sys/class/powercap/intel-rapl/intel-rapl:0/constraint_0_power_limit_uw
```
بيرجع الأمر بدون error، بس القيمة مش بتتغير. الـ `POWER_LIMIT1` فاضل زي ما هو.

#### التحليل
الـ `rapl_power_limit` struct فيه field اسمه `locked`:

```c
struct rapl_power_limit {
    struct powercap_zone_constraint *constraint;
    struct rapl_domain *domain;
    const char *name;
    bool locked;       /* ← هنا المشكلة */
    u64 last_power_limit;
};
```

الـ `locked` flag بيتحدد من الـ primitive الاسمه `FW_LOCK` أو `PL1_LOCK` اللي موجودين في `rapl_primitives`:

```c
enum rapl_primitives {
    POWER_LIMIT1,
    POWER_LIMIT2,
    POWER_LIMIT4,
    ENERGY_COUNTER,
    FW_LOCK,       /* firmware lock — يغلق كل حدود الطاقة */
    FW_HIGH_LOCK,
    PL1_LOCK,      /* locks PL1 specifically */
    PL2_LOCK,
    PL4_LOCK,
    ...
};
```

الـ BIOS على الـ server ده فاتح الـ **FW_LOCK** bit في الـ `MSR_PKG_POWER_LIMIT` (MSR `0x610`)، يعني الـ firmware قفل الـ register. لما kernel يحاول يكتب عبر الـ `write_raw` callback الخاص بالـ `rapl_if_priv`، الـ hardware بيتجاهل الكتابة.

```c
struct rapl_if_priv {
    ...
    int (*read_raw)(int id, struct reg_action *ra, bool pmu_ctx);
    int (*write_raw)(int id, struct reg_action *ra);  /* ← بيتحط قيمة لكن HW مش بيقبلها */
    ...
};
```

الـ `reg_action` بيتبعت صح لكن الـ `ra->err` برضو بيرجع 0 لأن الـ MSR write مش بترجع error حتى لو الـ bit كان locked في بعض الـ implementations.

#### الحل
**أولاً — تأكيد المشكلة:**
```bash
# اقرأ MSR_PKG_POWER_LIMIT مباشرة
rdmsr -p 0 0x610
# لو bit 63 (lock bit) == 1 → FW_LOCK مفعّل

# أو افحص من sysfs
cat /sys/class/powercap/intel-rapl/intel-rapl:0/enabled
```

**ثانياً — الحل من BIOS:**
- ادخل BIOS Setup → Power Management → `Package Power Limit Lock` → **Disabled**
- أو استخدم الـ platform-specific tool زي `ipmitool` لتغيير BIOS settings remotely.

**ثالثاً — تأكيد بعد التعديل:**
```bash
rdmsr -p 0 0x610
# bit 63 لازم يكون 0 دلوقتي

echo 150000000 > /sys/class/powercap/intel-rapl/intel-rapl:0/constraint_0_power_limit_uw
cat /sys/class/powercap/intel-rapl/intel-rapl:0/constraint_0_power_limit_uw
# المفروض يرجع 150000000
```

#### الدرس المستفاد
الـ `FW_LOCK` و `PL1_LOCK` primitives في `rapl_primitives` enum مش مجرد tracking flags — دول بيعكسوا حالة hardware lock bits في الـ MSR registers. لما `rapl_power_limit.locked == true`، أي كتابة على الـ constraint بتتجاهلها الـ hardware صامتاً. دايماً افحص BIOS lock settings قبل ما تحاول تتحكم في power limits من الـ OS على production servers.

---

### السيناريو الثاني: DRAM Power Monitoring غلط على Laptop بـ Intel Core i7-12700K

#### العنوان
**`energy_unit`** مش صح في `rapl_domain` بيخلي DRAM energy readings غير دقيقة

#### السياق
مهندس embedded systems بيشتغل على laptop platform بـ **Intel Core i7-12700K** (Alder Lake). بيطوّر tool لمراقبة power consumption للـ DRAM خلال memory-intensive workloads. الـ tool بتقرأ energy counter من `/sys/class/powercap/intel-rapl/intel-rapl:0/intel-rapl:0:2/energy_uj` كل ثانية وبتحسب average power.

#### المشكلة
الـ readings بتظهر DRAM power أعلى من المتوقع بـ factor تقريباً 16x مقارنة بالـ hardware power analyzer. الـ package power readings صح، بس الـ DRAM domain وحده غلط.

#### التحليل
كل `rapl_domain` struct عنده units خاصة بيه:

```c
struct rapl_domain {
    char name[RAPL_DOMAIN_NAME_LENGTH];
    enum rapl_domain_type id;          /* RAPL_DOMAIN_DRAM هنا */
    union rapl_reg regs[RAPL_DOMAIN_REG_MAX];
    struct powercap_zone power_zone;
    struct rapl_domain_data rdd;
    struct rapl_power_limit rpl[NR_POWER_LIMITS];
    u64 attr_map;
    unsigned int state;
    unsigned int power_unit;
    unsigned int energy_unit;   /* ← ده الـ unit المستخدم في تحويل الـ raw counter */
    unsigned int time_unit;
    struct rapl_package *rp;
};
```

الـ `energy_unit` بيتقرأ من الـ `RAPL_DOMAIN_REG_UNIT` register (MSR `0x606` للـ package، أو MSR مختلف للـ DRAM على بعض platforms).

على **Alder Lake**، الـ DRAM domain عنده energy unit مختلف عن الـ package. الـ `rapl_if_priv` بيخزن الـ unit register:

```c
struct rapl_if_priv {
    ...
    union rapl_reg reg_unit;   /* ← register للـ units — واحد للـ package */
    union rapl_reg regs[RAPL_DOMAIN_MAX][RAPL_DOMAIN_REG_MAX]; /* per-domain regs */
    ...
};
```

المشكلة: على بعض الـ Alder Lake configurations، الـ DRAM energy unit في MSR `0x606` هو 15.3 µJ لكن الـ kernel بيستخدم قيمة الـ package unit (61 µJ) للـ DRAM domain بسبب bug في الـ detection logic، وده بيخلي النتيجة مضروبة في ~4 مش صح.

الـ `rapl_domain_data` بتخزن الـ raw values:
```c
struct rapl_domain_data {
    u64 primitives[NR_RAPL_PRIMITIVES];  /* ENERGY_COUNTER هنا */
    unsigned long timestamp;
};
```

لما بيتحسب الـ average power، الـ ENERGY_COUNTER بيتقسم على الـ `energy_unit` الغلط.

#### الحل
**تأكيد المشكلة:**
```bash
# اقرأ الـ raw DRAM energy unit من MSR
rdmsr -p 0 0x606  # package energy unit
# bits [12:8] = energy unit for package

# للـ DRAM — على بعض platforms الـ unit مختلف
# افحص kernel log
dmesg | grep -i "rapl"
dmesg | grep -i "dram"
```

**افحص الـ energy_unit الـ kernel بيستخدمه:**
```bash
# راقب الـ raw energy counter مع الوقت
watch -n1 "cat /sys/class/powercap/intel-rapl/intel-rapl:0/intel-rapl:0:2/energy_uj"
```

**الحل في الـ kernel:** الـ fix بيكون في الـ driver الخاص بالـ MSR interface، بيضاف detection logic للـ DRAM-specific energy unit عبر قراءة الـ `RAPL_DOMAIN_REG_INFO` register للـ DRAM domain بشكل منفصل وتخزين القيمة الصح في `rapl_domain.energy_unit`.

```bash
# verify بعد kernel patch
# قارن readings بالـ hardware power meter
turbostat --show PkgWatt,RAMWatt --interval 1
```

#### الدرس المستفاد
الـ `rapl_domain` struct بيعزل الـ units لكل domain (`power_unit`, `energy_unit`, `time_unit`) عشان بالظبط كل domain ممكن يكون له unit مختلف. على Alder Lake والـ generations اللي بعدها، الـ DRAM energy unit مش دايماً نفس الـ package. لازم تتحقق من الـ per-domain unit register وميس تفترضش إن الـ package unit ينفع لكل الـ domains.

---

### السيناريو الثالث: CPU Hotplug Crash عند إزالة Core في Server بـ Intel Xeon Scalable

#### العنوان
**`rapl_package`** بيتباشر إنه orphaned بعد CPU hotplug وبيسبب kernel panic

#### السياق
شركة cloud infrastructure بتستخدم **Intel Xeon Scalable** (Ice Lake SP) في servers بتدعم CPU hotplug لعزل faulty cores. فريق الـ SRE بيحاول يعمل offline لـ CPU core معيوب عبر:

```bash
echo 0 > /sys/devices/system/cpu/cpu15/online
```

#### المشكلة
بعد إزالة الـ CPU، الـ kernel بيعمل panic بـ:
```
BUG: kernel NULL pointer dereference
RIP: intel_rapl_msr_cpu_offline+0x...
```

#### التحليل
الـ `rapl_package` struct بيتتبع الـ active CPUs:

```c
struct rapl_package {
    unsigned int id;           /* logical die id */
    unsigned int nr_domains;
    unsigned long domain_map;
    struct rapl_domain *domains;
    struct powercap_zone *power_zone;
    unsigned long power_limit_irq;
    struct list_head plist;
    int lead_cpu;              /* ← الـ CPU المسؤول عن الـ MSR access */
    struct cpumask cpumask;    /* ← الـ CPUs الـ active في الـ package */
    char name[PACKAGE_DOMAIN_NAME_LENGTH];
    struct rapl_if_priv *priv;
#ifdef CONFIG_PERF_EVENTS
    bool has_pmu;
    struct rapl_package_pmu_data pmu_data;
#endif
};
```

الـ `lead_cpu` هو الـ CPU الوحيد اللي بيتم من خلاله الـ MSR read/write للـ package (عشان MSR access لازم يتعمل على نفس الـ package).

لما CPU 15 بيتعمل offline:
1. الـ `pcap_rapl_online` hotplug handler بيتنادى (المخزون في `rapl_if_priv.pcap_rapl_online`).
2. لو CPU 15 كان هو الـ `lead_cpu`، الـ code لازم يختار `lead_cpu` جديد من الـ `cpumask`.
3. الـ bug: race condition بين تحديث الـ `lead_cpu` وبين pending RAPL read operation بتحاول تشتغل على الـ old `lead_cpu` اللي بقى offline.

الـ `rapl_find_package_domain_cpuslocked` و `rapl_add_package_cpuslocked` هما الـ variants اللي المفروض تتنادى مع CPU hotplug lock ممسوك:

```c
struct rapl_package *rapl_find_package_domain_cpuslocked(int id,
                                    struct rapl_if_priv *priv,
                                    bool id_is_cpu);
struct rapl_package *rapl_add_package_cpuslocked(int id,
                                    struct rapl_if_priv *priv,
                                    bool id_is_cpu);
void rapl_remove_package_cpuslocked(struct rapl_package *rp);
```

المشكلة إن الـ driver كان بيستخدم الـ non-locked variants (`rapl_add_package` / `rapl_find_package_domain`) أثناء الـ hotplug path، وده بيخلي race condition.

#### الحل
**تشخيص المشكلة:**
```bash
# شغّل مع kernel crash dump
kdump + crash tool

# أو reproduce بشكل آمن في test environment
echo 0 > /sys/devices/system/cpu/cpu15/online
dmesg | tail -50
```

**الفهم من الكود:** الـ functions المنتهية بـ `cpuslocked` لازم تتنادى بس لما الـ `cpu_hotplug_lock` ممسوك. الـ regular variants بيمسكوا الـ lock داخلياً. في الـ hotplug callback، الـ lock بيكون ممسوك بالفعل، فالـ driver المفروض يستخدم الـ `cpuslocked` variants.

**الفيكس في الـ driver:**
```c
/* غلط — في hotplug callback */
rp = rapl_find_package_domain(cpu, priv, true);

/* صح — في hotplug callback */
rp = rapl_find_package_domain_cpuslocked(cpu, priv, true);
```

**اختبار الحل:**
```bash
# stress test hotplug
for i in $(seq 1 100); do
    echo 0 > /sys/devices/system/cpu/cpu15/online
    echo 1 > /sys/devices/system/cpu/cpu15/online
done
# لو مفيش crash → الـ fix صح
```

#### الدرس المستفاد
الـ API في `intel_rapl.h` بيفرّق صراحة بين الـ `cpuslocked` variants والـ regular ones عشان CPU hotplug synchronization صح. استخدام الـ variant الغلط بيعمل deadlock أو race condition. دايماً استخدم الـ `cpuslocked` variants جوه hotplug callbacks، والـ regular variants من الـ normal driver init/exit paths.

---

### السيناريو الرابع: PSys Power Limit مش بيظهر على Intel Core Ultra (Meteor Lake)

#### العنوان
الـ `RAPL_DOMAIN_PLATFORM` مش موجود في sysfs على **Intel Core Ultra** (Meteor Lake) Laptop

#### السياق
مهندس بيشتغل على thermal management لـ thin & light laptop بـ **Intel Core Ultra (Meteor Lake)**. بيحاول يستخدم PSys (Platform Power) domain عشان يتحكم في إجمالي power للـ platform (CPU + GPU + DRAM معاً). الـ Windows بيدعمه، لكن على Linux مش بيلاقي:

```
/sys/class/powercap/intel-rapl/intel-rapl:1/  ← مش موجود (PSys)
```

#### المشكلة
الـ RAPL_DOMAIN_PLATFORM مش بيظهر في sysfs، يعني الـ kernel مش بيدعم PSys على الـ platform ده.

#### التحليل
الـ `rapl_package` بيستخدم `domain_map` كـ bitmap عشان يتتبع الـ active domains:

```c
struct rapl_package {
    ...
    unsigned long domain_map;   /* bit map of active domains */
    struct rapl_domain *domains; /* array — بيتبنى runtime */
    ...
};
```

الـ `rapl_domain_type` enum بيعرّف الـ domains:

```c
enum rapl_domain_type {
    RAPL_DOMAIN_PACKAGE,    /* bit 0 */
    RAPL_DOMAIN_PP0,        /* bit 1 */
    RAPL_DOMAIN_PP1,        /* bit 2 */
    RAPL_DOMAIN_DRAM,       /* bit 3 */
    RAPL_DOMAIN_PLATFORM,   /* bit 4 — PSys */
    RAPL_DOMAIN_MAX,
};
```

الـ PSys domain بيتطلب:
1. الـ `RAPL_DOMAIN_PLATFORM` bit يكون set في `domain_map`.
2. الـ registers المناسبة تكون populated في `rapl_if_priv.regs[RAPL_DOMAIN_PLATFORM]`.
3. الـ `limits[RAPL_DOMAIN_PLATFORM]` يكون > 0.

على **Meteor Lake**، الـ interface هو **TPMI** (بدل MSR)، يعني `rapl_if_type == RAPL_IF_TPMI`:

```c
enum rapl_if_type {
    RAPL_IF_MSR,   /* traditional MSR-based */
    RAPL_IF_MMIO,  /* memory-mapped I/O */
    RAPL_IF_TPMI,  /* Topology Aware Register and PM Capsule Interface */
};
```

الـ `rapl_if_priv` للـ TPMI interface لازم يعمل map للـ platform domain registers بشكل مختلف. الـ PSys primitives:

```c
PSYS_POWER_LIMIT1,
PSYS_POWER_LIMIT2,
PSYS_PL1_ENABLE,
PSYS_PL2_ENABLE,
PSYS_TIME_WINDOW1,
PSYS_TIME_WINDOW2,
```

دول primitives منفصلة في الـ `rapl_primitives` enum خاصة بالـ PSys. لو الـ TPMI driver مش بيعمل map للـ `RAPL_DOMAIN_REG_LIMIT` الخاص بالـ PSys domain في الـ `regs[RAPL_DOMAIN_PLATFORM]` array، الـ domain بيتعمل skip.

#### الحل
**تأكيد الـ TPMI support:**
```bash
# افحص لو TPMI driver موجود
lsmod | grep tpmi
ls /sys/bus/auxiliary/devices/ | grep tpmi

# افحص الـ RAPL domains المتاحة
ls /sys/class/powercap/intel-rapl/

# افحص kernel messages
dmesg | grep -i "rapl\|tpmi\|psys\|platform"
```

**افحص الـ MSR fallback:**
```bash
# هل MSR_PLATFORM_POWER_LIMIT موجود؟
rdmsr 0x65C 2>/dev/null || echo "MSR not available — TPMI only"
```

**الفيكس:** في الـ TPMI RAPL driver، تأكد إن الـ PSys domain registers بتتقرأ من الـ TPMI capsule الصح وبتتحط في:

```c
priv->regs[RAPL_DOMAIN_PLATFORM][RAPL_DOMAIN_REG_LIMIT].val = psys_limit_addr;
priv->regs[RAPL_DOMAIN_PLATFORM][RAPL_DOMAIN_REG_STATUS].val = psys_status_addr;
priv->limits[RAPL_DOMAIN_PLATFORM] = 2; /* PL1 + PL2 */
```

وبعدين الـ domain_map bit بيتحدد تلقائياً لما الـ `rapl_add_package` تلاقي الـ registers موجودة.

**بعد الـ fix:**
```bash
ls /sys/class/powercap/intel-rapl/intel-rapl:1/
# المفروض يظهر PSys domain

cat /sys/class/powercap/intel-rapl/intel-rapl:1/name
# platform
```

#### الدرس المستفاد
الـ `domain_map` في `rapl_package` والـ `regs[RAPL_DOMAIN_MAX][RAPL_DOMAIN_REG_MAX]` في `rapl_if_priv` هما الـ gate للـ domain visibility. لو الـ per-interface driver (MSR, MMIO, أو TPMI) مش بيملأ الـ registers للـ `RAPL_DOMAIN_PLATFORM`، الـ domain بيتعمل skip. على الـ platforms الجديدة زي Meteor Lake اللي بتستخدم TPMI، كل domain لازم يتعمل enumerate بشكل صريح من الـ TPMI topology data.

---

### السيناريو الخامس: PMU Energy Counter بيعمل Overflow بدون Detection على Intel Core i9-13900K

#### العنوان
الـ `rapl_package_pmu_data.hrtimer` مش بيشتغل صح وبيخلي energy readings تعمل wrap-around على **Intel Core i9-13900K**

#### السياق
فريق performance engineering بيطور cloud workload energy monitoring tool على **Intel Core i9-13900K** (Raptor Lake) servers. بيستخدموا الـ Linux PMU interface عشان يقيسوا energy per-process بدقة عالية. الـ tool بتستخدم `perf_event_open` مع RAPL PMU.

#### المشكلة
بعد تشغيل heavy workload لمدة ساعات، الـ energy readings بتبقى سالبة أو بتعمل jump غريب. الـ total energy حُسبت غلط بـ عشرات الـ joules.

#### التحليل
الـ PMU support في `intel_rapl.h` محاط بـ `CONFIG_PERF_EVENTS`:

```c
#ifdef CONFIG_PERF_EVENTS
struct rapl_package_pmu_data {
    u64 scale[RAPL_DOMAIN_MAX];       /* scale per domain = 2^-32 Joules */
    raw_spinlock_t lock;               /* protects n_active + active_list */
    int n_active;                      /* عدد الـ active perf events */
    struct list_head active_list;      /* الـ active events */
    ktime_t timer_interval;            /* max interval قبل overflow */
    struct hrtimer hrtimer;            /* بيشتغل عشان يمنع overflow */
};
#endif
```

الـ energy counter في RAPL هو **32-bit** counter (على معظم الـ domains). على **Core i9-13900K** اللي package TDP 253W، الـ counter بيعمل overflow كل:

```
overflow_time = 2^32 × energy_unit
energy_unit (package) ≈ 61 µJ = 61 × 10^-6 J
overflow_time = 4294967296 × 61 × 10^-6 ≈ 262 seconds ≈ 4.4 دقائق
```

الـ `hrtimer` المفروض يشتغل قبل الـ overflow عشان يعمل software accumulation. الـ `timer_interval` بيتحسب من الـ `scale`:

```c
struct rapl_package_pmu_data {
    ...
    ktime_t timer_interval;  /* بيكون أصغر من الـ overflow time */
    struct hrtimer hrtimer;  /* callback بيقرأ الـ counter ويحفظه */
};
```

المشكلة: على الـ Core i9-13900K، لما الـ n_active == 0 (مفيش events شغالة)، الـ `hrtimer` بيتوقف. لو `perf stat` اتشغل، الـ event اتوقف مؤقتاً بسبب privilege issue، الـ `n_active` وصل 0، الـ timer اتوقف، والـ hardware counter عمل overflow قبل ما يتقرأ تاني.

الـ `lock` (raw_spinlock_t) في `rapl_package_pmu_data` بيحمي الـ `n_active` و `active_list`. Race condition: الـ hrtimer callback بياخد الـ lock، بيقرأ الـ counter، لكن لو الـ workload كان memory-intensive جداً وفيه interrupt latency عالية، الـ timer ممكن يتأخر وبعدين يلاقي إن الـ counter عمل overflow.

#### الحل
**تأكيد المشكلة:**
```bash
# افحص الـ RAPL PMU
ls /sys/devices/power/

# شوف الـ available events
perf list | grep power

# قيس مع verbose output
perf stat -e power/energy-pkg/ -I 1000 -p <pid>

# راقب الـ raw counter مباشرة
perf stat -e power/energy-pkg/ -- sleep 300
# لو الـ output فيه قيم سالبة أو anomalies → overflow problem
```

**Debug الـ timer interval:**
```bash
# افحص الـ scale من sysfs
cat /sys/devices/power/events/energy-pkg.scale
# المفروض يكون مثلاً: 6.103515625e-05 (= 2^-32 × 2^32 × energy_unit)

# احسب overflow time يدوياً
python3 -c "
scale = 6.103515625e-05  # Joules per event unit
max_count = 2**32
max_energy = scale * max_count  # total Joules before overflow
max_power_w = 300  # worst case for i9-13900K
overflow_secs = max_energy / max_power_w
print(f'Overflow after ~{overflow_secs:.1f} seconds at {max_power_w}W')
"
```

**الفيكس في الـ kernel:** تأكد إن `timer_interval` بيتحسب بناءً على أعلى `THERMAL_SPEC_POWER` ممكن للـ package، مش على average power:

```c
/* في rapl_package_pmu_data initialization */
/* timer_interval لازم يكون < overflow_time عند max TDP */
/* لـ i9-13900K max TDP = 253W */
pmu_data->timer_interval = ms_to_ktime(
    (u64)max_energy_range_uj / max_tdp_uw * 1000 / 2  /* نص الـ overflow time */
);
```

**اختبار الحل:**
```bash
# شغّل high-power workload لفترة طويلة
stress-ng --cpu 24 --timeout 3600 &
perf stat -e power/energy-pkg/ -I 10000 -- sleep 3600
# لازم مفيش negative values أو jumps
```

**تأكد من الـ PMU registration:**
```bash
# تأكد الـ rapl_package_add_pmu اشتغل
dmesg | grep -i "rapl.*pmu"
cat /sys/devices/power/cpumask
```

#### الدرس المستفاد
الـ `rapl_package_pmu_data.hrtimer` هو الـ mechanism الوحيد اللي بيمنع الـ 32-bit energy counter من الـ overflow. الـ `timer_interval` لازم يتحسب بناءً على **أقصى** power consumption ممكن للـ package (الـ `MAX_POWER` primitive)، مش الـ average. على الـ high-TDP processors زي **Core i9-13900K** (253W)، الـ overflow ممكن يحصل في أقل من 5 دقائق. الـ PMU energy monitoring في environments حساسة للـ accuracy لازم تأخذ في الاعتبار الـ `scale` و الـ timer interval بشكل صريح.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي Intel RAPL و powercap framework بالتفصيل:

| المقال | الوصف |
|--------|-------|
| [Power Capping Framework and RAPL Driver](https://lwn.net/Articles/567620/) | الـ overview الرئيسي للـ unified power capping interface والـ RAPL driver |
| [Power Capping Framework and RAPL Driver (نسخة تانية)](https://lwn.net/Articles/569535/) | تغطية إضافية للـ framework مع تفاصيل sysfs interface |
| [RAPL (Running Average Power Limit) driver](https://lwn.net/Articles/545745/) | مقال تقني عن الـ RAPL driver قبل merge في kernel |
| [introduce intel_rapl driver](https://lwn.net/Articles/444887/) | أول patch series قدّمت الـ intel_rapl driver للـ kernel |
| [Power Capping Framework](https://lwn.net/Articles/562015/) | تفاصيل الـ powercap framework نفسه اللي بيشتغل عليه RAPL |
| [powercap/intel_rapl: Introduce RAPL TPMI support](https://lwn.net/Articles/929547/) | إضافة الـ TPMI interface (Emerald Rapids وما بعده) |
| [perf/x86: add Intel RAPL PMU support](https://lwn.net/Articles/573602/) | دعم قراءة RAPL counters من خلال الـ perf subsystem |
| [Documentation/power/powercap/powercap.txt (3.13)](https://lwn.net/Articles/574224/) | التوثيق الرسمي للـ powercap اللي اتشاف مع kernel 3.13 |
| [Introduce powercap userspace frontend](https://lwn.net/Articles/915834/) | واجهة userspace للتحكم في الـ powercap zones |
| [TPMI control and debugfs support](https://lwn.net/Articles/937978/) | دعم debugfs للـ TPMI interface |

---

### التوثيق الرسمي في الـ Kernel

#### الـ Documentation paths في الـ kernel source tree

```
Documentation/power/powercap/powercap.rst     # الوثيقة الرئيسية للـ powercap framework
Documentation/ABI/testing/sysfs-class-powercap  # واجهة sysfs للـ powercap
```

#### على docs.kernel.org

- [Power Capping Framework — The Linux Kernel documentation](https://docs.kernel.org/power/powercap/powercap.html)
  - بيشرح شجرة الـ power zones وهيكل الـ constraints
  - بيوضح الـ sysfs interface تحت `/sys/class/powercap/intel-rapl/`
  - بيشرح الفرق بين `energy_uj` و `constraint_*_power_limit_uw`

#### الـ Kconfig options المرتبطة

| Option | الملف | الوصف |
|--------|-------|-------|
| `CONFIG_INTEL_RAPL` | `drivers/powercap/Kconfig` | الـ core RAPL library |
| `CONFIG_INTEL_RAPL_MSR` | `drivers/powercap/Kconfig` | RAPL via MSR registers |
| `CONFIG_INTEL_RAPL_MMIO` | `drivers/powercap/Kconfig` | RAPL via MMIO registers |
| `CONFIG_INTEL_RAPL_TPMI` | `drivers/powercap/Kconfig` | RAPL via TPMI (Emerald Rapids+) |

---

### Commits مهمة في الـ Kernel History

#### أول merge للـ RAPL driver (kernel 3.13)

الـ patch series الأصلية قدّمها **Srinivas Pandruvada** و **Jacob Pan** من Intel:

- [Patchwork: Introduce Intel RAPL power capping driver](https://patchwork.kernel.org/project/linux-pm/patch/1379606435-15871-7-git-send-email-srinivas.pandruvada@linux.intel.com/)

#### إضافة RAPL TPMI support (kernel 6.5)

```
[PATCH 0/15] powercap/intel_rapl: Introduce RAPL TPMI support
Author: Zhang Rui <rui.zhang@intel.com>
Date: 2023-03-16
```

- [LKML thread للـ TPMI RAPL patches](https://lore.kernel.org/lkml/20230316153841.3666-1-rui.zhang@intel.com/T/)
- [PATCH v2 series على lore.kernel.org](https://lore.kernel.org/all/ZPbJBanVmoMuOhMR@intel.com/T/)

#### إضافة Power Limit4 لـ Alder Lake-N و Raptor Lake-P

- [Patchwork: Add Power Limit4 support](https://patchwork.kernel.org/project/linux-pm/patch/20220726130229.24634-1-sumeet.r.pawnikar@intel.com/)

---

### مناقشات الـ Mailing List

#### الـ RAPL AMD support patches

لما AMD أضافت RAPL counters لـ Fam17h/Fam19h، اتناقش على LKML:

- [LKML: PATCH v3 0/4 — Enable RAPL for AMD Fam17h and Fam19h](https://lkml.kernel.org/lkml/9ea15f21febf47d5d6f62911fe0141a2ae5d5e2b.camel@linux.intel.com/t/)
- [Mail Archive: Re: PATCH v3 0/4](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2375243.html)

#### ثغرة PLATYPUS وتأمين الـ RAPL interface

**PLATYPUS** (CVE-2020-8694, CVE-2020-8695) — ثغرة side-channel خطيرة بتستغل RAPL لسرقة AES keys من Intel SGX. الـ fix كان تقييد الوصول لـ powercap من unprivileged users:

- [PLATYPUS Official Website](https://platypusattack.com/)
- [PLATYPUS Research Paper (PDF)](https://platypusattack.com/platypus.pdf)
- [RAPL accessible to a container — Security Advisory](https://github.com/containerd/containerd/security/advisories/GHSA-7ww5-4wqc-m92c)

---

### Kernel Newbies — تاريخ RAPL عبر الـ Kernel Versions

الـ RAPL support اتطور على مدار سنين في kernelnewbies.org:

| Kernel Version | الإضافة | الرابط |
|---------------|---------|--------|
| Linux 3.13 | أول دعم لـ powercap framework والـ intel_rapl driver | [kernelnewbies.org/Linux_3.13](https://kernelnewbies.org/Linux_3.13) |
| Linux 4.3 | دعم Broadwell-H و Skylake H/S | [kernelnewbies.org/Linux_4.3](https://kernelnewbies.org/Linux_4.3) |
| Linux 4.7 | إضافة PSys domain (Platform domain) | [kernelnewbies.org/Linux_4.7](https://kernelnewbies.org/Linux_4.7) |
| Linux 4.9 | دعم Apollo Lake | [kernelnewbies.org/Linux_4.9](https://kernelnewbies.org/Linux_4.9) |
| Linux 4.17 | دعم Cannon Lake | [kernelnewbies.org/Linux_4.17](https://kernelnewbies.org/Linux_4.17) |
| Linux 5.8 | إضافة amd_energy driver لـ AMD Fam17h+ | [kernelnewbies.org/Linux_5.8](https://kernelnewbies.org/Linux_5.8) |
| Linux 6.14 | آخر updates للـ RAPL | [kernelnewbies.org/Linux_6.14](https://kernelnewbies.org/Linux_6.14) |

---

### elinux.org

الـ elinux.org مش عنده صفحة مخصصة لـ RAPL، لأن RAPL بيخص mainly الـ Intel x86 processors وهو بعيد عن الـ embedded Linux focus الأساسي للـ elinux. للمعلومات عن power management في embedded Linux بشكل عام:

- [elinux.org — Main Page](https://elinux.org/Main_Page)

---

### مصادر تقنية خارجية

#### قراءة RAPL energy counters من Linux

ده مرجع ممتاز من جامعة Maine يشرح كل طرق القراءة:

- [Reading RAPL energy measurements from Linux — EECE Maine](https://web.eece.maine.edu/~vweaver/projects/rapl/)
- [Linux support for Power Measurement Interfaces](https://web.eece.maine.edu/~vweaver/projects/rapl/rapl_support.html)

#### الـ powercap C bindings

- [GitHub: powercap/powercap — C bindings to the Linux Power Capping Framework](https://github.com/powercap/powercap)
- [GitHub: powercap/raplcap — RAPL power capping C interface](https://github.com/powercap/raplcap)

#### Phoronix — Intel TPMI Driver

- [Intel TPMI Linux Driver Published To Enhance Power Management Handling](https://www.phoronix.com/news/Intel-TPMI-Linux-Driver)

---

### الكتب الموصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)

- **الفصل المهم**: Chapter 14 — The Linux Device Model
  - بيشرح الـ `kobject`, `sysfs`, والـ device hierarchy اللي بيبني عليه الـ powercap framework
- الكتاب متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)

- **الفصل المهم**: Chapter 11 — Timers and Time Management
  - لأن RAPL بيستخدم timestamps وحساب average power
- **الفصل المهم**: Chapter 17 — Devices and Modules
  - لفهم الـ driver model اللي بيشتغل عليه intel_rapl

#### Understanding the Linux Kernel — Bovet & Cesati

- **الفصل المهم**: Chapter 6 — Interrupts and Exceptions
  - مهم لفهم الـ CPU hotplug callbacks الموجودة في `rapl_if_priv`

#### Intel Software Developer's Manual (SDM)

ده أهم مرجع تقني للـ MSR registers نفسها:

- **Volume 3B**: Chapter 14 — Power and Thermal Management
  - بيشرح MSR_PKG_POWER_LIMIT, MSR_PKG_ENERGY_STATUS, MSR_RAPL_POWER_UNIT
- متاح مجاناً من Intel:
  - `https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html`

---

### Search Terms للبحث عن معلومات أكتر

```
# للبحث في LKML و lore.kernel.org
intel_rapl driver kernel
powercap framework linux
RAPL MSR power limit
intel_rapl_msr.c
intel_rapl_common.c
TPMI RAPL interface
rapl_package rapl_domain
CONFIG_INTEL_RAPL_MSR

# للبحث في Intel documentation
RAPL "Running Average Power Limit" MSR
MSR_PKG_POWER_LIMIT 0x610
MSR_RAPL_POWER_UNIT 0x606
Intel SDM chapter 14 power management

# للبحث عن security aspects
PLATYPUS RAPL CVE-2020-8694
powercap energy_uj permission
RAPL side channel attack mitigation

# للبحث عن tools
perf stat energy RAPL
turbostat RAPL power
powertop RAPL
```

---

### الـ Source Files المرتبطة في الـ Kernel

الملفات دي هي القلب الحقيقي للـ implementation، بتكمّل صورة `intel_rapl.h`:

```
include/linux/intel_rapl.h          # الـ header الرئيسي (الملف اللي بندرسه)
include/linux/powercap.h            # الـ powercap framework API

drivers/powercap/intel_rapl_common.c  # الـ shared RAPL logic
drivers/powercap/intel_rapl_msr.c     # MSR-based interface driver
drivers/powercap/intel_rapl_mmio.c    # MMIO-based interface driver
drivers/powercap/intel_rapl_tpmi.c    # TPMI-based interface driver (6.5+)
drivers/powercap/powercap_sys.c       # الـ powercap core framework

arch/x86/events/intel/rapl.c         # RAPL via perf_event PMU
```
## Phase 8: Writing simple module

### الفكرة

**`rapl_add_package`** هي الدالة المُصدَّرة الأنسب للـ hook — بتُسجِّل **RAPL package** جديد (socket كامل) في الـ powercap subsystem. ده حدث نادر ومفيد: بيحصل لما kernel يكتشف CPU package جديد أو لما driver زي `intel_rapl_msr` يتحمَّل. هنستخدم **kprobe** عشان نعترض الاستدعاء ونطبع معلومات عن الـ package اللي اتضاف.

---

### الـ kprobe على `rapl_add_package`

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * rapl_pkg_probe.c
 *
 * Intercept rapl_add_package() calls using a kprobe and log
 * information about every new RAPL package being registered.
 */

#include <linux/module.h>       /* MODULE_* macros, module_init/exit        */
#include <linux/kernel.h>       /* pr_info, pr_err                          */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, etc.     */
#include <linux/intel_rapl.h>   /* struct rapl_if_priv, rapl_if_type names  */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe on rapl_add_package — log new RAPL packages");

/* ------------------------------------------------------------------ *
 * helper: turn rapl_if_type enum into a human-readable string         *
 * ------------------------------------------------------------------ */
static const char *rapl_if_name(enum rapl_if_type t)
{
    switch (t) {
    case RAPL_IF_MSR:  return "MSR";
    case RAPL_IF_MMIO: return "MMIO";
    case RAPL_IF_TPMI: return "TPMI";
    default:           return "UNKNOWN";
    }
}

/* ------------------------------------------------------------------ *
 * pre-handler: runs just BEFORE rapl_add_package() executes           *
 *                                                                     *
 * Prototype we are hooking:                                           *
 *   struct rapl_package *rapl_add_package(int id,                     *
 *                                         struct rapl_if_priv *priv,  *
 *                                         bool id_is_cpu);            *
 *                                                                     *
 * On x86-64 the arguments arrive in registers:                        *
 *   regs->di  → id        (int)                                       *
 *   regs->si  → priv      (struct rapl_if_priv *)                     *
 *   regs->dx  → id_is_cpu (bool)                                      *
 * ------------------------------------------------------------------ */
static int rapl_pkg_pre_handler(struct kprobe *kp, struct pt_regs *regs)
{
    /* Extract the three arguments from CPU registers (x86-64 ABI) */
    int                  id        = (int)regs->di;
    struct rapl_if_priv *priv      = (struct rapl_if_priv *)regs->si;
    bool                 id_is_cpu = (bool)regs->dx;

    if (!priv) {
        pr_info("rapl_probe: rapl_add_package called with NULL priv!\n");
        return 0;
    }

    /* Log everything useful about the package being registered */
    pr_info("rapl_probe: rapl_add_package() called\n");
    pr_info("rapl_probe:   id=%d  id_is_cpu=%s  interface=%s\n",
            id,
            id_is_cpu ? "true (cpu#)" : "false (die#)",
            rapl_if_name(priv->type));

    return 0; /* returning 0 means: let the real function run normally */
}

/* ------------------------------------------------------------------ *
 * kprobe descriptor                                                   *
 * symbol_name tells the kernel which function to instrument           *
 * ------------------------------------------------------------------ */
static struct kprobe rapl_kp = {
    .symbol_name = "rapl_add_package",
    .pre_handler = rapl_pkg_pre_handler,
};

/* ------------------------------------------------------------------ *
 * module_init: register the kprobe so the hook becomes active         *
 * ------------------------------------------------------------------ */
static int __init rapl_probe_init(void)
{
    int ret;

    ret = register_kprobe(&rapl_kp);
    if (ret < 0) {
        pr_err("rapl_probe: register_kprobe failed (%d)\n", ret);
        pr_err("rapl_probe: is CONFIG_KPROBES=y and symbol exported?\n");
        return ret;
    }

    pr_info("rapl_probe: kprobe planted on %s at %p\n",
            rapl_kp.symbol_name, rapl_kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ *
 * module_exit: MUST unregister to stop the hook before code is freed  *
 * ------------------------------------------------------------------ */
static void __exit rapl_probe_exit(void)
{
    unregister_kprobe(&rapl_kp);
    pr_info("rapl_probe: kprobe removed from rapl_add_package\n");
}

module_init(rapl_probe_init);
module_exit(rapl_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ `MODULE_*` macros و`module_init`/`module_exit` — أساس أي kernel module |
| `linux/kernel.h` | الـ `pr_info` / `pr_err` للطباعة على kernel log |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل API الـ kprobe |
| `linux/intel_rapl.h` | تعريف `struct rapl_if_priv` و`enum rapl_if_type` عشان نقرأ بيانات الـ argument بشكل type-safe |

**الـ** `linux/intel_rapl.h` ضروري عشان نـ cast الـ `regs->si` لـ `struct rapl_if_priv *` بأمان ونوصل لـ `priv->type`.

---

#### دالة `rapl_if_name`

دالة مساعدة بسيطة بتحوِّل الـ `enum rapl_if_type` لـ string مقروء (MSR/MMIO/TPMI). بتخلي الـ log مفهوم بدون ما تحتاج تفسر أرقام.

---

#### الـ pre-handler

الـ pre-handler بيتشغَّل **قبل** ما `rapl_add_package` تبدأ تنفيذها فعلاً. الـ `struct pt_regs *regs` بيحتوي على حالة registers وقت الاستدعاء — على x86-64 الـ ABI بيحط أول 3 arguments في `rdi`، `rsi`، `rdx` اللي في kernel struct بيبقوا `regs->di`، `regs->si`، `regs->dx`.

بنعمل null-check على `priv` الأول عشان ميحصلش panic لو جه NULL بالغلط، وبعدين نطبع:
- الـ `id` (رقم الـ CPU أو الـ die)
- هل الـ id ده لـ CPU رقم ولا die رقم
- نوع الـ RAPL interface (MSR هو الأشهر على AMD/Intel القديم)

الـ `return 0` ضروري يفضل صفر — لو رجعنا قيمة غير صفر الـ kprobe هيـ skip الدالة الأصلية وده خطر.

---

#### `struct kprobe rapl_kp`

الـ `.symbol_name` هو اللي بيقول للـ kernel "اعمل hook على الدالة دي بالاسم". الـ kernel بيبحث في الـ kallsyms ويحسب العنوان تلقائياً. الـ `.pre_handler` هو الـ callback اللي هيتشغَّل قبل الدالة.

---

#### `rapl_probe_init`

`register_kprobe` بتحط breakpoint instruction (int3 على x86) في أول الدالة المستهدفة. لو فشلت (مثلاً الدالة مش موجودة أو `CONFIG_KPROBES` مش مفعَّل) بنرجع error وما يتحمَّلش الـ module.

---

#### `rapl_probe_exit`

`unregister_kprobe` **إجباري** في الـ exit — لو module اتشال من الذاكرة والـ kprobe لسه مسجَّل، أي استدعاء لـ `rapl_add_package` هيحاول يجري callback code اتمسح وده kernel panic فوري. الـ unregister بيشيل الـ breakpoint ويستردِّ الكود الأصلي.

---

### Makefile

```makefile
obj-m += rapl_pkg_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ module

```bash
# Build
make

# Load — RAPL domains usually registered at boot, so trigger via modprobe
sudo insmod rapl_pkg_probe.ko

# Force a re-registration by reloading the RAPL MSR driver
sudo modprobe -r intel_rapl_msr
sudo modprobe intel_rapl_msr

# Check the output
sudo dmesg | grep rapl_probe

# Expected output (example on a 2-socket server):
# rapl_probe: kprobe planted on rapl_add_package at ffffffffc0xxxxxx
# rapl_probe: rapl_add_package() called
# rapl_probe:   id=0  id_is_cpu=false (die#)  interface=MSR
# rapl_probe: rapl_add_package() called
# rapl_probe:   id=1  id_is_cpu=false (die#)  interface=MSR

# Unload cleanly
sudo rmmod rapl_pkg_probe
sudo dmesg | grep rapl_probe
# rapl_probe: kprobe removed from rapl_add_package
```

لو الجهاز AMD أو مش بيدعم RAPL، `intel_rapl_msr` مش موجود والـ output هيكون `register_kprobe failed (-22)` — ده طبيعي ومتوقَّع.
