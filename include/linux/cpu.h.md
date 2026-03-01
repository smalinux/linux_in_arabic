## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem اللي ينتمي ليه الملف ده؟

**الـ** `include/linux/cpu.h` ينتمي لـ subsystem اسمه **CPU HOTPLUG** — وده مُعرَّف صراحةً في ملف `MAINTAINERS` تحت نفس الاسم، والـ maintainers بتوعه هما Thomas Gleixner و Peter Zijlstra.

---

### الصورة الكبيرة — تخيل الموضوع كده

تخيل عندك سيرفر ضخم شغال 24/7 — مستشفى، بنك، أي حاجة مش تقدر توقفها. جوا السيرفر ده فيه 64 CPU. في يوم من الأيام، واحدة من الـ CPUs دي بدأت تعمل مشاكل. إيه اللي بتعمله؟

في الأيام القديمة: توقف الجهاز كله، تشيل الـ CPU، تشتغل تاني. مش مقبول.

الـ Linux kernel جاب حل اسمه **CPU Hotplug** — أنت تقدر تشيل CPU وانت شغال، أو تضيف CPU جديدة وانت شغال، من غير ما توقف أي خدمة.

**الـ** `include/linux/cpu.h` هو **البوابة الرئيسية** لكل الـ API بتاع الـ CPU في الـ kernel. بيعرّف:
- الـ `struct cpu` — التمثيل الأساسي لأي CPU كـ device في الـ kernel
- الـ interface بتاع الـ CPU Hotplug (تشغيل/إيقاف CPUs)
- الـ CPU Idle management (إيه اللي بيحصل لما الـ CPU مش بتعمل حاجة)
- كشف الثغرات الأمنية المتعلقة بالـ CPU (Spectre, Meltdown, وغيرهم)
- الـ SMT (Simultaneous Multi-Threading) control — يعني التحكم في الـ HyperThreading

---

### ليه الملف ده موجود؟ ما هو هدفه؟

#### 1. تمثيل الـ CPU كـ Device

```c
struct cpu {
    int node_id;       /* رقم الـ NUMA node اللي فيه الـ CPU */
    int hotpluggable;  /* هل الـ CPU دي ممكن تتشال وانت شغال؟ */
    struct device dev; /* الـ CPU نفسها كـ device عادية في kernel */
};
```

الـ kernel بيشوف الـ CPU كـ "device" عادية زي الـ USB أو الـ hard disk. ده بيخليه يعاملها بنفس الـ framework الموحد بتاع الـ devices. النتيجة: الـ CPU بتظهر في `/sys/devices/system/cpu/cpuN/` وتقدر تتحكم فيها من هناك.

#### 2. CPU Hotplug — إيقاف وتشغيل CPUs وانت شغال

ده قلب الموضوع. الـ kernel بيدير كل CPU خلال **state machine** معقدة — من `OFFLINE` لـ `ONLINE` ورجوع. كل subsystem في الـ kernel (scheduler, IRQs, timers...) لازم يعرف إيه اللي يعمله لما CPU جديدة تيجي أو CPU قديمة تمشي.

```
CPU_UP_PREPARE (0x0003)  →  CPU_ONLINE (0x0002)
CPU_ONLINE     (0x0002)  →  CPU_DEAD   (0x0007)
CPU_DEAD       (0x0007)  →  CPU_POST_DEAD (0x0009)
CPU_BROKEN     (0x000B)  ← حالة خاصة: الـ CPU اتكسرت ومش قادرة تموت بشكل سليم
```

#### 3. CPU Idle — اللي بيحصل لما الـ CPU تفضى

لما الـ CPU مفيش عندها شغل، الـ kernel مش بيسيبها تاكل كهربا وهي فاضية — بيحطها في "idle state". الـ header بيعرّف:

```c
void __noreturn cpu_startup_entry(enum cpuhp_state state);
void cpu_idle_poll_ctrl(bool enable);
bool cpu_in_idle(unsigned long pc);
void arch_cpu_idle(void);         /* كل architecture بتعمل idle بطريقتها */
void __noreturn arch_cpu_idle_dead(void); /* الـ CPU ماتت فعلاً */
```

#### 4. Security Vulnerabilities — كشف الثغرات

جزء كبير من الملف ده مخصص لكشف ثغرات الـ CPU الأمنية وإظهارها للـ user space عبر sysfs. ده جه بعد كارثة Spectre/Meltdown في 2018.

```c
extern ssize_t cpu_show_meltdown(...);
extern ssize_t cpu_show_spectre_v1(...);
extern ssize_t cpu_show_spectre_v2(...);
extern ssize_t cpu_show_spec_store_bypass(...);
extern ssize_t cpu_show_l1tf(...);
extern ssize_t cpu_show_mds(...);
extern ssize_t cpu_show_retbleed(...);
extern ssize_t cpu_show_ghostwrite(...);
// ... والقائمة بتطول مع كل ثغرة جديدة بتتكشف
```

كل function دي بتعمل حاجة واحدة: لما تعمل `cat /sys/devices/system/cpu/vulnerabilities/meltdown` تشوف "Mitigation: PTI" أو "Vulnerable". ده هو الـ interface اللي بيوصل المعلومة دي للـ user.

#### 5. CPU Mitigations Control

```c
enum cpu_attack_vectors {
    CPU_MITIGATE_USER_KERNEL,  /* هجوم من user process على kernel data */
    CPU_MITIGATE_USER_USER,    /* هجوم من process على process تانية */
    CPU_MITIGATE_GUEST_HOST,   /* هجوم من VM على الـ host */
    CPU_MITIGATE_GUEST_GUEST,  /* هجوم من VM على VM تانية */
};
```

الـ kernel بيفرق بين أنواع الهجمات المختلفة عشان يعرف يطبق الـ mitigation الصح من غير ما يخسر performance في الأماكن اللي مش محتاج فيها حماية.

#### 6. SMP و Suspend/Resume

لما الجهاز بيروح في **sleep/suspend**، الـ kernel لازم يوقف كل الـ CPUs التانية (secondary CPUs) ويصحّيهم بعد كده:

```c
// وقت الـ suspend:
suspend_disable_secondary_cpus();  /* بيوقف كل الـ CPUs غير الـ primary */

// وقت الـ resume:
suspend_enable_secondary_cpus();   /* بيصحّي الـ CPUs التانية */
```

---

### القصة الكاملة — إيه اللي بيحصل لما CPU تتضاف Hotplug؟

**السيناريو:** سيرفر شغال، حد ضاف CPU جديدة أو ضغط على enable في sysfs.

```
1. الـ user كتب: echo 1 > /sys/devices/system/cpu/cpu3/online

2. drivers/base/cpu.c  →  استقبل الطلب من sysfs
3. kernel/cpu.c        →  بدأ state machine بتاع الـ hotplug
4. BIOS/Firmware       →  صحّى الـ CPU الجديدة
5. arch code           →  عمل initialize للـ CPU (registers, caches, ...)
6. notify_cpu_starting()  →  أخبر كل الـ subsystems إن فيه CPU جديدة
7. scheduler           →  أضاف الـ CPU للـ load balancing
8. IRQ subsystem       →  ممكن يعمل migrate لبعض الـ interrupts
9. CPU وصلت لـ CPUHP_ONLINE  →  جاهزة تشتغل بشكل كامل
```

كل ده منظّم عبر الـ `cpuhp_state` enum الموجود في `cpuhotplug.h` اللي الملف بتاعنا بيـ include ه.

---

### الملفات المرتبطة اللي المطوّر لازم يعرفها

| الملف | الدور |
|-------|-------|
| `include/linux/cpu.h` | **الملف الحالي** — الـ API الرئيسية |
| `include/linux/cpuhotplug.h` | الـ state machine بتاع الـ hotplug وكل الـ callbacks |
| `include/linux/cpu_smt.h` | التحكم في الـ SMT/HyperThreading |
| `include/linux/cpuhplock.h` | الـ locking primitives بتاع الـ hotplug |
| `kernel/cpu.c` | الـ implementation الفعلية لكل الـ hotplug logic |
| `drivers/base/cpu.c` | تسجيل الـ CPUs كـ devices + الـ sysfs interface |
| `kernel/smpboot.c` | bringup الـ secondary CPUs في الـ SMP |
| `kernel/cpu_pm.c` | الـ CPU power management notifications |
| `arch/x86/kernel/cpu/` | الـ x86-specific CPU initialization |
| `Documentation/core-api/cpu_hotplug.rst` | الـ documentation الرسمية |

---

### الملفات اللي بتكوّن الـ Subsystem ده

**Core:**
- `kernel/cpu.c` — قلب الـ hotplug state machine
- `kernel/smpboot.c` — بدء تشغيل الـ CPUs في الـ SMP

**Headers:**
- `include/linux/cpu.h` — الـ main API header (**ملفنا**)
- `include/linux/cpuhotplug.h` — الـ state machine enum والـ callbacks API
- `include/linux/cpu_smt.h` — الـ SMT control
- `include/linux/cpuhplock.h` — الـ locking

**Device Layer:**
- `drivers/base/cpu.c` — الـ sysfs integration وتسجيل الـ CPUs كـ devices

**Architecture-specific:**
- `arch/x86/kernel/smpboot.c` — الـ x86 bringup
- `arch/arm64/kernel/smp.c` — الـ ARM64 bringup

**Security:**
- `arch/x86/kernel/cpu/bugs.c` — الـ implementation الفعلية لكل `cpu_show_*` functions على الـ x86
## Phase 2: شرح الـ CPU Subsystem Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في أي نظام SMP (Symmetric Multi-Processing) أو حتى single-core embedded system، الـ kernel محتاج يجاوب على أسئلة زي:

- إمتى الـ CPU ده شغّال وإمتى مش شغّال؟
- لو CPU اتضاف أو اتشال وانا بشتغل (hotplug)، مين اللي يتعلم بالكلام ده ومين يعمل إيه؟
- إزاي أعرض معلومات الـ CPU للـ userspace (sysfs)؟
- إزاي أتحكم في الـ SMT threads (Hyper-Threading) وأعطّل بعضها لو في vulnerability؟
- إزاي أحمي النظام من الـ CPU vulnerabilities زي Meltdown وSpectre؟

من غير framework موحد، كل driver أو subsystem هيعمل الكلام ده بطريقته الخاصة، وهيبقى في تشابك وduplicated code وrace conditions في كل حاجة بتتعلق بـ CPU state.

---

### الحل — الـ kernel اتعامل إزاي؟

الـ kernel قسّم المشكلة لـ 3 محاور:

1. **التمثيل (Representation):** كل CPU بيتعامل كـ `device` عادي في الـ device model، يعني ليه entries في sysfs، وممكن يتعمله add/remove زي أي hardware device.

2. **الـ Hotplug State Machine:** بدل ما تقول "الـ CPU اتشغّل" و بس، الـ kernel بيمشي بـ state machine مرتبة من `CPUHP_OFFLINE` لـ `CPUHP_ONLINE`، وكل subsystem بيسجّل نفسه على state معين عشان يعمل الـ setup/teardown بالترتيب الصح.

3. **الـ Security Mitigations:** لأن الـ CPU hardware نفسه بيكون فيه vulnerabilities، الـ subsystem بيوفر واجهة موحدة للـ mitigation ويعرضها في sysfs عشان الـ admin والـ userspace يشوفوها.

---

### الـ Analogy — المبنى والمصعد والأمن

تخيّل مجمع شركات كبير (الـ kernel). كل طابق فيه شركات (subsystems). المصاعد هي الـ CPUs.

- **الـ `struct cpu`** = بطاقة تعريف المصعد (رقمه، الطابق اللي عليه، هل ممكن يتشال وانت بتشتغل؟)
- **الـ `cpuhp_state` enum** = قائمة خطوات تشغيل/إيقاف المصعد بالترتيب: فتح الكهرباء، تشغيل الأبواب، تشغيل الطوارئ... إلخ
- **الـ `cpuhp_setup_state()`** = شركة بتسجّل نفسها عند إدارة المبنى وتقول "لما المصعد رقم X بييجي، ابعتلنا notification عشان نجهّز الـ server room"
- **الـ sysfs entries** = لوحة الحالة في الاستقبال اللي بتعرض حالة كل مصعد وأي مشاكل أمنية فيه
- **الـ SMT control** = قرار إدارة المبنى إنها تعطّل نص المصاعد في ساعات معينة لأسباب أمنية (زي Spectre/MDS)
- **الـ mitigations** = تقارير الأمن اللي بتتعلق على الحيطة بتقول "المصعد ده متأثر بـ vulnerability X، واحنا عاملين حاجة Y"
- **الـ `arch_cpu_is_hotpluggable()`** = فريق هندسة المبنى (الـ arch layer) هو اللي يقرر الـ CPU ده ممكن يتشال فعلاً من الـ hardware ولا لأ

---

### البنية العامة — Big Picture Architecture

```
+-----------------------------------------------------------+
|                     User Space                             |
|  /sys/devices/system/cpu/cpu0/online                      |
|  /sys/devices/system/cpu/vulnerabilities/meltdown         |
+---------------------------|-------------------------------+
                            | sysfs read/write
+---------------------------|-------------------------------+
|               Linux Kernel                                 |
|                                                           |
|  +------------------+    +-----------------------------+  |
|  |   cpu.h API      |    |  cpuhotplug.h State Machine  |  |
|  |                  |    |                             |  |
|  | register_cpu()   |    |  CPUHP_OFFLINE              |  |
|  | get_cpu_device() |    |     |                       |  |
|  | cpu_add_dev_attr |    |     v PREPARE callbacks     |  |
|  | cpu_show_*()     |    |  CPUHP_BRINGUP_CPU          |  |
|  +--------+---------+    |     |                       |  |
|           |              |     v STARTING callbacks    |  |
|           |              |  CPUHP_AP_ONLINE            |  |
|  +--------v---------+    |     |                       |  |
|  |  Device Model    |    |     v ONLINE callbacks      |  |
|  |  (struct device) |    |  CPUHP_ONLINE               |  |
|  |  drivers/base/   |    +-----------------------------+  |
|  |  cpu.c           |                                     |
|  +--------+---------+    +-----------------------------+  |
|           |              |   CPU Mitigations           |  |
|           |              |   cpu_mitigations_off()     |  |
|  +--------v---------+    |   cpu_attack_vector_*()     |  |
|  |  struct cpu      |    |   smt_mitigations           |  |
|  |  .node_id        |    +-----------------------------+  |
|  |  .hotpluggable   |                                     |
|  |  .dev            |    +-----------------------------+  |
|  +------------------+    |   SMT Control (cpu_smt.h)   |  |
|                          |   cpu_smt_control           |  |
|  Consumers:              |   cpu_smt_disable()         |  |
|  - scheduler             |   cpuhp_smt_enable()        |  |
|  - perf subsystem        +-----------------------------+  |
|  - IRQ subsystem                                          |
|  - RCU                                                    |
|  - workqueues                                             |
|                                                           |
|  Arch Layer (arm64, x86, etc.):                           |
|  arch_register_cpu(), arch_cpu_is_hotpluggable()          |
|  arch_cpu_idle(), arch_cpuhp_kick_ap_alive()              |
+-----------------------------------------------------------+
|                     Hardware                               |
|   CPU0   CPU1   CPU2   CPU3  (physical cores/threads)     |
+-----------------------------------------------------------+
```

---

### الـ Structs وعلاقتها ببعض

```
struct cpu
+--------------------+
| node_id            |  <-- الـ NUMA node اللي الـ CPU ده عليه
| hotpluggable       |  <-- هل ممكن يتشال وانت بتشتغل؟
| dev                |  <-- embedded struct device (Device Model)
+--------------------+
         |
         | embedded في
         v
struct device  (من include/linux/device.h)
+--------------------+
| kobj               |  <-- kobject = entry في sysfs
| bus_type *bus      |  --> cpu_subsys (الـ bus كلها للـ CPUs)
| parent             |  --> /sys/devices/system/cpu/
| driver_data        |
| ...                |
+--------------------+
```

**الـ device model** هو subsystem تاني مهم: هو اللي بيدير الـ sysfs، وبيربط الـ hardware objects بالـ kernel objects. لازم تفهمه قبل ما تكمل، في اختصار: كل hardware entity في الـ kernel بتتمثل كـ `struct device`، وده بيعطيها sysfs presence وlifecycle management تلقائي.

---

### الـ Hotplug State Machine بالتفصيل

ده أهم concept في الـ file. الـ `enum cpuhp_state` بيعرّف ordered states من 0 (`CPUHP_OFFLINE`) لـ `CPUHP_ONLINE`.

الـ state space اتقسم لـ 3 sections:

```
CPUHP_OFFLINE (0)
    |
    |  [PREPARE section]
    |  Callbacks run on the CONTROL CPU
    |  e.g., allocate per-cpu data structures
    |
CPUHP_BRINGUP_CPU
    |
    |  [STARTING section]
    |  Callbacks run on the HOTPLUGGED CPU itself
    |  Interrupts DISABLED — low-level init only
    |  e.g., init GIC, init timer, init cache coherency
    |
CPUHP_AP_ONLINE
    |
    |  [ONLINE section]
    |  Callbacks run on the HOTPLUGGED CPU
    |  Interrupts and preemption ENABLED
    |  e.g., register with scheduler, start workqueues
    |
CPUHP_ONLINE
```

**الـ teardown = reverse**: لما CPU بيتشال، نفس الـ callbacks بتتنفذ بالعكس من `CPUHP_ONLINE - 1` لـ `CPUHP_OFFLINE`.

#### إزاي driver أو subsystem يسجّل نفسه؟

```c
/* مثال: IRQ subsystem بيسجّل نفسه على state STARTING */
cpuhp_setup_state(
    CPUHP_AP_IRQ_GIC_STARTING,   /* متى يتنفذ في الـ state machine */
    "irq/gic:starting",           /* اسم للـ debugging */
    gic_starting_cpu,             /* startup callback */
    gic_dying_cpu                 /* teardown callback */
);
```

لما CPU جديد يشتغل، الـ kernel بيمشي الـ states بالترتيب وبيكال كل callback مسجّل. لما CPU يتشال، بيكال الـ teardown callbacks بالعكس.

#### الـ Dynamic States

مش كل subsystem محتاج state ثابت في الـ enum. في حالات كتير بيستخدموا:
- `CPUHP_BP_PREPARE_DYN` — dynamic state في الـ PREPARE section
- `CPUHP_AP_ONLINE_DYN` — dynamic state في الـ ONLINE section

ده بيخلي الـ enum محدود الحجم ومش بيكبر مع كل driver جديد.

---

### الـ SMT (Simultaneous Multi-Threading) Control

**الـ SMT** = Hyper-Threading في Intel، أو الـ threads الـ logical per physical core. كل physical core ممكن يكون فيه 2+ logical CPUs.

```c
enum cpuhp_smt_control {
    CPU_SMT_ENABLED,          /* كل الـ SMT threads شغّالين */
    CPU_SMT_DISABLED,         /* بس thread واحد per core */
    CPU_SMT_FORCE_DISABLED,   /* معطّل بالقوة — مش ممكن يتفعّل تاني */
    CPU_SMT_NOT_SUPPORTED,    /* الـ hardware مش بيدعم SMT */
    CPU_SMT_NOT_IMPLEMENTED,  /* الـ kernel compile من غير CONFIG_HOTPLUG_SMT */
};
```

ليه مهم؟ لأن **MDS، Spectre-v2، L1TF** كلها vulnerabilities ممكن تتأثر لو SMT threads بيشتغلوا جنب بعض. بعض الـ mitigations بتشترط إنك تعطّل الـ SMT.

```c
/* في حالة vulnerability، الـ kernel ممكن يعمل: */
cpu_smt_disable(true);  /* force = true يعني مش بيتفتح تاني */

/* أو يتحكم في عدد الـ threads اللي شغّالة */
cpu_smt_set_num_threads(num_threads, max_threads);
```

---

### الـ CPU Vulnerabilities Interface

الـ file بيعرّف مجموعة `cpu_show_*()` functions، كل واحدة بتكشف vulnerability معينة في sysfs:

```
/sys/devices/system/cpu/vulnerabilities/
├── meltdown         → cpu_show_meltdown()
├── spectre_v1       → cpu_show_spectre_v1()
├── spectre_v2       → cpu_show_spectre_v2()
├── spec_store_bypass → cpu_show_spec_store_bypass()
├── l1tf             → cpu_show_l1tf()
├── mds              → cpu_show_mds()
├── tsx_async_abort  → cpu_show_tsx_async_abort()
├── itlb_multihit    → cpu_show_itlb_multihit()
├── srbds            → cpu_show_srbds()
├── mmio_stale_data  → cpu_show_mmio_stale_data()
├── retbleed         → cpu_show_retbleed()
├── spec_rstack_overflow → cpu_show_spec_rstack_overflow()
├── gds              → cpu_show_gds()
├── reg_file_data_sampling → cpu_show_reg_file_data_sampling()
├── ghostwrite       → cpu_show_ghostwrite()
├── old_microcode    → cpu_show_old_microcode()
├── indirect_target_selection → cpu_show_indirect_target_selection()
├── tsa              → cpu_show_tsa()
└── vmscape          → cpu_show_vmscape()
```

الـ output بيكون نص زي: `"Mitigation: PTI"` أو `"Vulnerable"` أو `"Not affected"`.

---

### الـ Attack Vectors Enum

```c
enum cpu_attack_vectors {
    CPU_MITIGATE_USER_KERNEL,   /* user process يهاجم kernel */
    CPU_MITIGATE_USER_USER,     /* user process يهاجم user process تاني */
    CPU_MITIGATE_GUEST_HOST,    /* VM guest يهاجم الـ host */
    CPU_MITIGATE_GUEST_GUEST,   /* VM يهاجم VM تاني */
    NR_CPU_ATTACK_VECTORS,
};
```

ده تجريد مهم: بدل ما تقول "عطّل الـ mitigation X"، الـ kernel بيفكر في الـ attack vectors. الـ function `cpu_attack_vector_mitigated()` بتجاوب: هل النظام ده محمي من النوع ده من الهجمات؟

---

### الـ CPU Idle Interface

```c
void arch_cpu_idle(void);          /* الـ arch بتحط الـ CPU في low-power state */
void arch_cpu_idle_prepare(void);  /* تحضير قبل الـ idle */
void arch_cpu_idle_enter(void);    /* دخل الـ idle loop */
void arch_cpu_idle_exit(void);     /* اصحى من الـ idle */
void arch_tick_broadcast_enter(void); /* قبل ما الـ local timer يتوقف */
void arch_tick_broadcast_exit(void);  /* بعد ما الـ CPU يصحى */
void arch_cpu_idle_dead(void);     /* CPU اتشال خلاص، مش هيرجع */

void play_idle_precise(u64 duration_ns, u64 latency_ns);
/* idle لمدة محددة بالـ nanoseconds — للـ real-time وpower management */
```

**الـ `cpu_startup_entry()`** هي الـ function اللي كل AP (Application Processor) بيدخلها بعد ما يكمّل الـ hotplug sequence. هي بتدخل الـ idle loop الأساسية وبتفضل فيها لحد ما الـ scheduler يديها task.

---

### الـ Suspend/Resume Integration

```c
/* لما النظام بيعمل suspend */
suspend_disable_secondary_cpus()
    --> freeze_secondary_cpus(primary_cpu)
        /* بيوقف كل الـ CPUs غير الـ primary عن طريق hotplug */

/* لما النظام بيرجع */
suspend_enable_secondary_cpus()
    --> thaw_secondary_cpus()
        /* بيرجّع الـ CPUs تاني عن طريق hotplug */
```

**الـ `CONFIG_PM_SLEEP_SMP_NONZERO_CPU`**: بيخلي الـ kernel يعمل suspend على CPU غير الـ CPU0، مفيد في الأنظمة اللي الـ CPU0 ممكن يكون disabled أو محجوز.

---

### اللي الـ Subsystem بيمتلكه vs اللي بيفوّضه للـ Arch/Drivers

| المسؤولية | الـ CPU Subsystem يعملها | الـ Arch Layer يعملها |
|-----------|--------------------------|----------------------|
| تمثيل الـ CPU كـ device | `register_cpu()`, `struct cpu` | — |
| sysfs attributes | `cpu_add_dev_attr()` | — |
| vulnerability reporting interface | `cpu_show_*()` declarations | الـ implementation الفعلي |
| hotplug state machine | ترتيب الـ states والـ dispatch | الـ `arch_cpuhp_kick_ap_alive()` |
| SMT control logic | `cpuhp_smt_enable/disable()` | `arch_cpu_is_hotpluggable()` |
| idle loop entry point | `cpu_startup_entry()` | `arch_cpu_idle()` |
| suspend/resume CPU freeze | `freeze_secondary_cpus()` | تفاصيل الـ arch stop |
| physical CPU probe/release | — | `arch_cpu_probe()`, `arch_cpu_release()` |
| mitigation policy decision | `cpu_mitigations_off()` flag | الـ microcode والـ hardware features |

---

### الـ NUMA Integration

**الـ NUMA (Non-Uniform Memory Access)** هو subsystem بيوصف الـ topology: أي CPU قريب من أي memory. الـ `struct cpu` فيها `node_id` اللي بيقول الـ CPU ده على أنهي NUMA node.

```
NUMA Node 0          NUMA Node 1
+-----------+        +-----------+
| CPU0 CPU1 |        | CPU2 CPU3 |
| RAM 0-16G |        | RAM 16-32G|
+-----------+        +-----------+
      \                    /
       \                  /
        +--- Interconnect +
```

الـ scheduler والـ memory allocator بيستخدموا الـ `node_id` عشان يقرروا: الـ task ده يتشغّل على أنهي CPU، والـ memory تتوزّع إزاي، عشان يقللوا الـ latency من الـ cross-node access.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — الـ Macros والـ Enums والـ Config Options

#### CPU State Macros (arch-level, مش hotplug core)

| Macro | القيمة | المعنى |
|---|---|---|
| `CPU_ONLINE` | `0x0002` | الـ CPU شغال |
| `CPU_UP_PREPARE` | `0x0003` | الـ CPU بيتشغّل دلوقتي |
| `CPU_DEAD` | `0x0007` | الـ CPU مات |
| `CPU_DEAD_FROZEN` | `0x0008` | الـ CPU أتعطّل وقت الـ unplug |
| `CPU_POST_DEAD` | `0x0009` | الـ CPU اتشال بنجاح |
| `CPU_BROKEN` | `0x000B` | الـ CPU مماتش بشكل صح |

#### `enum cpuhp_state` — أقسام الـ State Machine

| القسم | النطاق | بيتنفّذ على مين |
|---|---|---|
| **PREPARE** | `CPUHP_OFFLINE` → `CPUHP_BRINGUP_CPU` | الـ control CPU (BP) |
| **STARTING** | `CPUHP_AP_IDLE_DEAD` → `CPUHP_TEARDOWN_CPU` | الـ hotplugged CPU (IRQs مقفولة) |
| **ONLINE** | `CPUHP_AP_ONLINE_IDLE` → `CPUHP_ONLINE` | الـ hotplugged CPU (من hotplug thread) |

أهم states في كل قسم:

| State | القسم | الوظيفة |
|---|---|---|
| `CPUHP_OFFLINE` | PREPARE | نقطة البداية |
| `CPUHP_BP_PREPARE_DYN` | PREPARE | dynamic states للـ BP |
| `CPUHP_BRINGUP_CPU` | PREPARE | لحظة إطلاق الـ AP |
| `CPUHP_AP_OFFLINE` | STARTING | الـ AP لسه مش online |
| `CPUHP_AP_SCHED_STARTING` | STARTING | بدء الـ scheduler على الـ AP |
| `CPUHP_AP_ONLINE` | STARTING | الـ AP وصل online |
| `CPUHP_TEARDOWN_CPU` | STARTING | نقطة الـ teardown |
| `CPUHP_AP_ONLINE_IDLE` | ONLINE | الـ AP idle في الـ hotplug thread |
| `CPUHP_AP_ONLINE_DYN` | ONLINE | dynamic states للـ AP |
| `CPUHP_AP_ACTIVE` | ONLINE | الـ AP شغّال فعلاً |
| `CPUHP_ONLINE` | ONLINE | نهاية الـ state machine |

#### `enum cpuhp_smt_control` — التحكم في الـ SMT

| القيمة | المعنى |
|---|---|
| `CPU_SMT_ENABLED` | الـ SMT مفعّل |
| `CPU_SMT_DISABLED` | الـ SMT متعطّل |
| `CPU_SMT_FORCE_DISABLED` | الـ SMT متعطّل بالقوة (أمان) |
| `CPU_SMT_NOT_SUPPORTED` | الـ hardware مش بيدعم SMT |
| `CPU_SMT_NOT_IMPLEMENTED` | مفيش SMT support في الـ kernel |

#### `enum cpu_attack_vectors` — متجهات الهجوم للـ Mitigations

| القيمة | المعنى |
|---|---|
| `CPU_MITIGATE_USER_KERNEL` | هجوم من user-space على kernel |
| `CPU_MITIGATE_USER_USER` | هجوم بين عمليتين user-space |
| `CPU_MITIGATE_GUEST_HOST` | هجوم من VM على الـ host |
| `CPU_MITIGATE_GUEST_GUEST` | هجوم بين VMs |
| `NR_CPU_ATTACK_VECTORS` | العدد الكلي للمتجهات |

#### `enum smt_mitigations` — مستويات حماية الـ SMT

| القيمة | المعنى |
|---|---|
| `SMT_MITIGATIONS_OFF` | لا حماية |
| `SMT_MITIGATIONS_AUTO` | الـ kernel يقرر |
| `SMT_MITIGATIONS_ON` | حماية كاملة |

#### Config Options الأساسية

| Config | الأثر لو مش معرّف |
|---|---|
| `CONFIG_SMP` | `cpu_maps_update_begin/done` و`add_cpu` تبقى no-ops |
| `CONFIG_HOTPLUG_CPU` | `unregister_cpu` و locking APIs تبقى stubs |
| `CONFIG_HOTPLUG_SMT` | كل SMT control functions تبقى stubs |
| `CONFIG_PM_SLEEP_SMP` | `freeze/thaw_secondary_cpus` مش موجودين |
| `CONFIG_CPU_MITIGATIONS` | `cpu_mitigations_off()` ترجع `true` دايماً |
| `CONFIG_GENERIC_CPU_DEVICES` | per-CPU array `cpu_devices` مش متعرّف |

---

### 1. الـ Structs المهمة

#### `struct cpu`

**الغرض:** التمثيل الأساسي لأي CPU في الـ kernel — بيوصف الـ CPU كـ device في الـ sysfs تحت `/sys/devices/system/cpu/cpuN/`.

```c
struct cpu {
    int node_id;      /* رقم الـ NUMA node اللي فيه الـ CPU */
    int hotpluggable; /* لو 1: بيتعمل له control file في sysfs */
    struct device dev;/* الـ CPU كـ device في driver model */
};
```

| الحقل | النوع | الوظيفة |
|---|---|---|
| `node_id` | `int` | بيربط الـ CPU بالـ NUMA node بتاعه |
| `hotpluggable` | `int` | لو != 0 → الـ kernel بيعمل `/sys/devices/system/cpu/cpuN/online` |
| `dev` | `struct device` | الـ CPU embedded كـ generic device — بيتعمله register في device model |

**الربط بـ structs تانية:**
- `struct device` (مضمّن): بيربط الـ CPU بـ `cpu_subsys` (الـ bus) وبيدي الـ CPU اسم وـ sysfs attributes.
- **الـ `struct device` بيحتوي** على pointer لـ `struct device_node` (device tree node) في الأنظمة اللي بتستخدم OF.

---

### 2. رسم علاقات الـ Structs

```
struct cpu
┌─────────────────────────────┐
│  node_id: int               │──────────────────► NUMA node
│  hotpluggable: int          │
│  dev: struct device ────────┼──────┐
└─────────────────────────────┘      │
                                     ▼
                           struct device
                    ┌──────────────────────────┐
                    │  kobj: struct kobject    │──► sysfs entry
                    │  bus: &cpu_subsys ───────┼──► struct bus_type
                    │  of_node ────────────────┼──► struct device_node
                    │  driver_data             │    (device tree)
                    └──────────────────────────┘
                                     │
                                     ▼
                           struct bus_type (cpu_subsys)
                    ┌──────────────────────────┐
                    │  name: "cpu"             │
                    │  dev_attrs               │──► sysfs attributes
                    └──────────────────────────┘


per-CPU array (CONFIG_GENERIC_CPU_DEVICES):
DECLARE_PER_CPU(struct cpu, cpu_devices)
  cpu_devices[0] ──► struct cpu { node_id=0, dev=... }
  cpu_devices[1] ──► struct cpu { node_id=0, dev=... }
  cpu_devices[N] ──► struct cpu { node_id=M, dev=... }
```

---

### 3. Lifecycle Diagrams

#### دورة حياة الـ CPU كـ Device

```
boot_cpu_init()
      │
      ▼
  [mark boot CPU online in cpumask]
      │
      ▼
register_cpu(struct cpu *cpu, int num)
      │
      ├─► cpu->node_id = cpu_to_node(num)
      ├─► cpu->hotpluggable = hotpluggable? → يعمل "online" sysfs file
      └─► device_register(&cpu->dev)
                │
                ▼
          [CPU ظهر في sysfs]
          /sys/devices/system/cpu/cpuN/
                │
                │  (لو hotpluggable)
                ▼
          /sys/devices/system/cpu/cpuN/online


           [طول عمر الـ system]
                │
                ▼
         cpu_add_dev_attr() / cpu_add_dev_attr_group()
         [إضافة attributes زي vulnerability info]


      (CONFIG_HOTPLUG_CPU فقط)
                │
                ▼
         unregister_cpu(struct cpu *cpu)
                │
                └─► device_unregister(&cpu->dev)
                          │
                          ▼
                    [اتشال من sysfs]
```

#### دورة حياة الـ CPU Hotplug State Machine

```
CPU-UP Path:
═════════════
OFFLINE
  │
  ▼ [BP: PREPARE callbacks — control CPU بينفّذهم]
CPUHP_BP_KICK_AP
  │
  ▼ [BP: بيطلق الـ AP]
CPUHP_BRINGUP_CPU ──────────────────────────► AP يبدأ يشتغل
                                                   │
                              ◄────────────── AP_OFFLINE
                              [STARTING callbacks — IRQs off]
                                                   │
                                             AP_ONLINE
                                                   │
                              [ONLINE callbacks — hotplug thread]
                                                   │
                                            AP_ONLINE_IDLE
                                                   │
                                              AP_ACTIVE
                                                   │
                                             CPUHP_ONLINE ✓

CPU-DOWN Path:
══════════════
CPUHP_ONLINE
  │
  ▼ [ONLINE teardown callbacks — reverse order]
AP_ACTIVE → AP_ONLINE_IDLE
  │
  ▼ [stop_machine]
TEARDOWN_CPU ◄──────────────── AP_ONLINE_IDLE
  │
  ▼ [STARTING teardown callbacks على الـ AP]
AP_OFFLINE
  │
  ▼ [AP بيعمل play_dead() / idle]
AP_IDLE_DEAD
  │
  ▼ [BP: PREPARE teardown callbacks]
OFFLINE ✓
```

#### دورة حياة الـ Suspension (PM Sleep)

```
suspend_disable_secondary_cpus()
         │
         ▼
freeze_secondary_cpus(primary_cpu)
         │
         ├─► [كل CPU غير الـ primary يتشال]
         ├─► cpuhp_tasks_frozen = true
         └─► [CPUs يفضلوا frozen]

         [الـ system بيعمل suspend]

suspend_enable_secondary_cpus()
         │
         ▼
thaw_secondary_cpus()
         │
         └─► [كل CPU متجمّد يرجع online]
```

---

### 4. Call Flow Diagrams

#### تسجيل CPU جديد

```
arch code calls register_cpu(cpu, num)
  │
  ▼
drivers/base/cpu.c: register_cpu()
  ├─► cpu->node_id = cpu_to_node(num)
  ├─► cpu->dev.bus = &cpu_subsys
  ├─► cpu->dev.type = &cpu_type
  └─► device_register(&cpu->dev)
        │
        ▼
      kobject_add()   ──────────────────► /sys/devices/system/cpu/cpuN/
        │
        ▼
      bus_probe_device()
        │
        ▼
      [hotpluggable?]
        │── YES ──► device_create_file("online")
        └── NO  ──► skip
```

#### إضافة Vulnerability Attribute

```
driver/subsystem calls cpu_add_dev_attr(attr)
  │
  ▼
[loop على كل online CPU]
  │
  ▼
device_create_file(get_cpu_device(i), attr)
  │
  ▼
/sys/devices/system/cpu/cpuN/<attr_name>
  │
  ▼ (on read)
cpu_show_meltdown() / cpu_show_spectre_v1() / ...
  │
  └─► arch reads MSRs or static keys
        │
        └─► returns "Mitigation: ..." or "Vulnerable"
```

#### CPU Hotplug — add_cpu() Flow

```
userspace writes 1 to /sys/devices/system/cpu/cpuN/online
  │
  ▼
cpu_device_up(dev)
  │
  ▼
add_cpu(cpu_num)
  │
  ▼
cpus_write_lock()          ← يمسك الـ hotplug lock
  │
  ▼
cpu_up(cpu, CPUHP_ONLINE)
  │
  ├─► [PREPARE callbacks على الـ BP — sequential]
  │      notify_cpu_starting() داخل الـ STARTING section
  │
  ├─► arch_cpuhp_kick_ap_alive(cpu, idle_task)
  │      │
  │      └─► platform-specific: SIPI/wake signal
  │
  ├─► [AP بيشتغل ويعمل STARTING callbacks]
  │
  └─► [ONLINE callbacks في hotplug thread]
        │
        ▼
cpus_write_unlock()
```

#### cpu_startup_entry() — Loop الـ CPU الجديد

```
arch bringup code
  │
  ▼
cpu_startup_entry(CPUHP_AP_ONLINE_IDLE)
  │
  ▼ [noreturn — مبيرجعش]
do_idle()
  │
  ├─► cpu_idle_poll_ctrl() — polling mode?
  │       │── YES ──► cpu_relax() loop
  │       └── NO  ──► arch_cpu_idle()
  │                     │
  │                     └─► platform WFI/HLT
  │
  └─► [لما يجي interrupt] ──► schedule() ──► back to idle loop
```

#### Mitigation Check Flow

```
security code / sysfs read
  │
  ▼
cpu_attack_vector_mitigated(CPU_MITIGATE_GUEST_HOST)
  │
  ▼ (CONFIG_CPU_MITIGATIONS=y)
[check smt_mitigations global]
  │
  ├─► SMT_MITIGATIONS_OFF  ──► return false
  ├─► SMT_MITIGATIONS_ON   ──► return true
  └─► SMT_MITIGATIONS_AUTO ──► depends on cpu_smt_control
                                  │
                                  └─► CPU_SMT_ENABLED? ──► vulnerable
                                      CPU_SMT_DISABLED? ──► mitigated
```

---

### 5. Locking Strategy

#### الـ Locks الموجودة

الـ `cpu.h` نفسه بيعرّف لocking API كاملة جوه `cpuhplock.h`. في المشهد ده اتنين مستويات:

| الـ Lock | النوع | بيحمي إيه |
|---|---|---|
| `cpus_read_lock` / `cpus_read_unlock` | rwsem (reader) | بيمنع تغيير CPU topology — أي code بيتعامل مع online CPUs لازم يمسكه |
| `cpus_write_lock` / `cpus_write_unlock` | rwsem (writer) | حصري لـ hotplug operations — add/remove CPU |

#### Lock Ordering

```
cpus_write_lock()        ← الأعلى في الـ hierarchy
    │
    ▼
cpu_maps_update_begin()  ← بيحمي cpu_online_mask / cpu_present_mask
    │
    ▼
[per-CPU data modification]
    │
    ▼
cpu_maps_update_done()
    │
    ▼
cpus_write_unlock()
```

**القاعدة الذهبية:**
- أي code بيقرأ `cpu_online_mask` أو بيتعامل مع per-CPU state → `cpus_read_lock()` أولاً.
- الـ hotplug path وحده اللي يعمل `cpus_write_lock()`.
- **ممنوع** تمسك `cpus_write_lock` وأنت شايل أي lock تاني فوقه — هيعمل deadlock.

#### تفاصيل الـ Locking في الـ Hotplug Callbacks

```
PREPARE callbacks:
  بيتنفّذوا على الـ control CPU
  interrupts: enabled
  preemption: enabled
  lock required: cpus_write_lock() ممسوكة بالفعل

STARTING callbacks:
  بيتنفّذوا على الـ hotplugged CPU
  interrupts: DISABLED ← مهم جداً
  preemption: DISABLED
  القيد: ممنوع أي lock تتطلب interrupt أو sleepable context

ONLINE callbacks:
  بيتنفّذوا على الـ hotplugged CPU من hotplug thread
  interrupts: enabled
  preemption: enabled
  عادي تمسك أي lock هنا
```

#### الـ `_cpuslocked` Variants

الـ API بيوفّر نسختين لكل function:

```c
/* لو أنت بتعمل cpus_read_lock() من برّا */
cpuhp_setup_state_cpuslocked(state, name, startup, teardown);

/* لو أنت مش شايل الـ lock */
cpuhp_setup_state(state, name, startup, teardown);
```

ده بيمنع double-locking لو الـ caller عنده `cpus_read_lock` بالفعل.

#### الـ SMT Locking

```
smt_mitigations   ← global variable
cpu_smt_control   ← global enum

التعديل عليهم:
  cpu_smt_disable(force)           ← من boot params أو sysfs
  cpu_smt_set_num_threads(n, max)  ← من BIOS/arch code

القراءة:
  cpu_attack_vector_mitigated()    ← read-only, no lock needed
  (الـ values بتتغيّر نادراً وبس وقت boot أو hotplug)
```

#### الـ `cpu_maps_update_begin/done` للـ UP Systems

لو `CONFIG_SMP` مش معرّف، الاتنين بيبقوا no-ops. في الـ SMP، بيعملوا serialize للتعديل على:
- `cpu_online_mask`
- `cpu_present_mask`
- `cpu_possible_mask`

```
cpu_maps_update_begin()
  │
  ├─► [lock cpumap_mutex internally]
  │
  [تعديل cpu bitmasks]
  │
  └─► cpu_maps_update_done()
        │
        └─► [unlock cpumap_mutex]
              + [notify waiters]
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Registration & Device Management

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `boot_cpu_init` | `void boot_cpu_init(void)` | تهيئة الـ boot CPU في الـ cpu_possible/present/online masks |
| `boot_cpu_hotplug_init` | `void boot_cpu_hotplug_init(void)` | تهيئة الـ hotplug state machine للـ boot CPU |
| `cpu_init` | `void cpu_init(void)` | تهيئة الـ per-CPU state على المستوى المعماري |
| `register_cpu` | `int register_cpu(struct cpu*, int)` | تسجيل الـ CPU كـ device في الـ sysfs |
| `unregister_cpu` | `void unregister_cpu(struct cpu*)` | إزالة الـ CPU device من الـ sysfs |
| `get_cpu_device` | `struct device* get_cpu_device(unsigned)` | جلب الـ device pointer لـ CPU محدد |
| `cpu_device_create` | `struct device* cpu_device_create(...)` | إنشاء child device مرتبط بالـ CPU |
| `arch_register_cpu` | `int arch_register_cpu(int)` | تسجيل arch-specific CPU device |
| `arch_unregister_cpu` | `void arch_unregister_cpu(int)` | إلغاء تسجيل arch-specific CPU device |
| `cpu_is_hotpluggable` | `bool cpu_is_hotpluggable(unsigned)` | هل يدعم الـ CPU الـ hotplug؟ |
| `arch_cpu_is_hotpluggable` | `bool arch_cpu_is_hotpluggable(int)` | نسخة معمارية من نفس الفحص |

#### Sysfs Attribute Management

| Function | الغرض |
|---|---|
| `cpu_add_dev_attr` | إضافة `device_attribute` واحدة لكل CPU devices |
| `cpu_remove_dev_attr` | إزالة `device_attribute` من كل CPU devices |
| `cpu_add_dev_attr_group` | إضافة `attribute_group` كاملة لكل CPU devices |
| `cpu_remove_dev_attr_group` | إزالة `attribute_group` من كل CPU devices |

#### Vulnerability Mitigation Show Functions

| Function | الـ Vulnerability |
|---|---|
| `cpu_show_meltdown` | Meltdown (CVE-2017-5754) |
| `cpu_show_spectre_v1` | Spectre v1 (CVE-2017-5753) |
| `cpu_show_spectre_v2` | Spectre v2 (CVE-2017-5715) |
| `cpu_show_spec_store_bypass` | SSB (CVE-2018-3639) |
| `cpu_show_l1tf` | L1TF (CVE-2018-3620) |
| `cpu_show_mds` | MDS (CVE-2018-12130) |
| `cpu_show_tsx_async_abort` | TAA (CVE-2019-11135) |
| `cpu_show_itlb_multihit` | ITLB Multihit (CVE-2018-12207) |
| `cpu_show_srbds` | SRBDS (CVE-2020-0543) |
| `cpu_show_mmio_stale_data` | MMIO Stale Data (CVE-2022-21123) |
| `cpu_show_retbleed` | Retbleed (CVE-2022-29900) |
| `cpu_show_spec_rstack_overflow` | RAS Overflow (CVE-2023-20569) |
| `cpu_show_gds` | GDS / Downfall (CVE-2022-40982) |
| `cpu_show_reg_file_data_sampling` | RFDS |
| `cpu_show_ghostwrite` | GhostWrite |
| `cpu_show_old_microcode` | حالة الـ microcode القديم |
| `cpu_show_indirect_target_selection` | ITS |
| `cpu_show_tsa` | TSA |
| `cpu_show_vmscape` | VMscape |

#### SMP / Hotplug Runtime

| Function | الغرض |
|---|---|
| `add_cpu` | إضافة CPU جديدة وإخطار الـ hotplug state machine |
| `cpu_device_up` | تشغيل CPU عبر الـ device interface |
| `notify_cpu_starting` | إرسال إشعار STARTING لكل الـ notifiers |
| `cpu_maps_update_begin` | الحصول على الـ cpu_add_remove_lock |
| `cpu_maps_update_done` | تحرير الـ cpu_add_remove_lock |
| `bringup_hibernate_cpu` | إعادة تشغيل الـ CPU بعد hibernate |
| `bringup_nonboot_cpus` | تشغيل الـ secondary CPUs عند الـ boot |
| `arch_cpu_rescan_dead_smt_siblings` | إعادة فحص الـ SMT siblings الميتة |

#### Suspend / PM Sleep

| Function | الغرض |
|---|---|
| `freeze_secondary_cpus` | إيقاف الـ secondary CPUs قبل الـ suspend |
| `thaw_secondary_cpus` | إعادة تشغيل الـ secondary CPUs بعد الـ resume |
| `suspend_disable_secondary_cpus` | wrapper لـ `freeze_secondary_cpus` |
| `suspend_enable_secondary_cpus` | wrapper لـ `thaw_secondary_cpus` |

#### Idle & Lifecycle

| Function | الغرض |
|---|---|
| `cpu_startup_entry` | نقطة دخول الـ idle loop — لا ترجع أبدًا |
| `cpu_idle_poll_ctrl` | تمكين/تعطيل الـ idle polling |
| `cpu_in_idle` | هل الـ PC ضمن نطاق الـ idle code؟ |
| `play_idle_precise` | تشغيل الـ CPU في حالة idle لمدة محددة بالـ ns |
| `cpuhp_report_idle_dead` | إخطار الـ hotplug core أن الـ CPU مات في الـ idle |

#### Arch Idle Hooks

| Function | الغرض |
|---|---|
| `arch_cpu_idle` | الـ arch-specific idle implementation |
| `arch_cpu_idle_prepare` | تحضير قبل الدخول للـ idle |
| `arch_cpu_idle_enter` | دخول الـ idle (disable interrupts etc.) |
| `arch_cpu_idle_exit` | الخروج من الـ idle |
| `arch_cpu_idle_dead` | خطاف الموت للـ CPU عند الـ hotplug-out |
| `arch_tick_broadcast_enter` | إعداد الـ tick broadcast قبل deep idle |
| `arch_tick_broadcast_exit` | إلغاء الـ tick broadcast بعد الخروج |
| `arch_cpu_finalize_init` | إنهاء تهيئة الـ CPU بعد الـ boot |

#### CPU Mitigations

| Function | الغرض |
|---|---|
| `cpu_mitigations_off` | هل mitigations=off في الـ cmdline؟ |
| `cpu_mitigations_auto_nosmt` | هل mitigations=auto,nosmt؟ |
| `cpu_attack_vector_mitigated` | هل vector محدد مُدار بـ mitigation؟ |

#### Indirect Branch / LP Status (arch)

| Function | الغرض |
|---|---|
| `arch_get_indir_br_lp_status` | جلب حالة الـ indirect branch loop prevention لـ task |
| `arch_set_indir_br_lp_status` | ضبط الحالة |
| `arch_lock_indir_br_lp_status` | قفل الحالة (لا يمكن تغييرها بعدها) |

#### SMT Control (من cpu_smt.h)

| Function | الغرض |
|---|---|
| `cpu_smt_disable` | تعطيل الـ SMT (اختياريًا بالقوة) |
| `cpu_smt_set_num_threads` | تحديد عدد الـ SMT threads المسموح بها |
| `cpu_smt_possible` | هل الـ SMT متاح على هذا الـ hardware؟ |
| `cpuhp_smt_enable` | تمكين الـ SMT عبر الـ hotplug core |
| `cpuhp_smt_disable` | تعطيل الـ SMT عبر الـ hotplug core |

---

### Group 1: Initialization Functions

هذه الـ functions تُستدعى في المراحل الأولى من boot لتهيئة الـ CPU subsystem قبل أن يبدأ الـ scheduler أو الـ SMP bringup.

---

#### `boot_cpu_init`

```c
void boot_cpu_init(void);
```

الـ function دي بتعمل تهيئة أولية للـ boot CPU (CPU 0 في الغالب) عن طريق ضبط الـ bits المقابلة له في الأربع masks: `cpu_possible_mask`, `cpu_present_mask`, `cpu_online_mask`, `cpu_active_mask`. بتتأكد إن الـ boot CPU ظاهر للـ kernel قبل ما أي كود تاني يتشغل.

**Parameters:** لا يأخذ parameters.

**Return value:** `void`.

**Key details:**
- بتتستدعى من `start_kernel()` قبل أي SMP initialization.
- بتستخدم `smp_processor_id()` لتحديد الـ boot CPU.
- لا يوجد لocking لأنه single-threaded في هذه المرحلة.

**Who calls it:** `start_kernel()` في `init/main.c`.

---

#### `boot_cpu_hotplug_init`

```c
void boot_cpu_hotplug_init(void);
```

بتهيئ الـ per-CPU hotplug state الخاص بالـ boot CPU وتضبط الـ `cpuhp_state` إلى `CPUHP_ONLINE`. الـ hotplug state machine يحتاج إن الـ boot CPU يبدأ بـ state معروف قبل ما يبدأ يتعامل مع الـ secondary CPUs.

**Parameters:** لا يأخذ parameters.

**Return value:** `void`.

**Key details:**
- بتتستدعى بعد `boot_cpu_init()` مباشرة.
- بتضبط الـ `this_cpu_write(cpuhp_state.state, CPUHP_ONLINE)`.
- لا تحتاج locking — single CPU context.

**Who calls it:** `start_kernel()` بعد `boot_cpu_init()`.

---

#### `cpu_init`

```c
void cpu_init(void);
```

بتُنفذ التهيئة المعمارية الخاصة بكل CPU (arch-specific per-CPU init). على x86 مثلًا، بتضبط الـ TSS, GDT segments, NMI stack. كل architecture عندها implementation خاصة.

**Parameters:** لا يأخذ parameters.

**Return value:** `void`.

**Key details:**
- مُعرَّفة في كل architecture بشكل مستقل.
- بتتستدعى لكل CPU عند الـ bringup.
- على x86: بتضبط الـ `cpu_tss_rw`, `load_TR_desc()`, `load_mm_ldt()`.

**Who calls it:** كود الـ SMP bringup في `arch/x86/kernel/cpu/common.c`.

---

### Group 2: Registration & Device Management

هذه الـ functions تربط الـ CPU بالـ Linux device model عبر الـ sysfs تحت `/sys/devices/system/cpu/cpuN/`.

---

#### `register_cpu`

```c
int register_cpu(struct cpu *cpu, int num);
```

بتسجل الـ CPU كـ `struct device` في الـ device model وتعرضها في الـ sysfs. الـ `struct cpu` يحتوي على `struct device` مدمج فيه، والـ `num` هو رقم الـ CPU. لو الـ CPU `hotpluggable`، بتنشئ ملف `online` في الـ sysfs يسمح بـ hotplug من user space.

```c
struct cpu {
    int node_id;       /* NUMA node */
    int hotpluggable;  /* creates /sys/.../online if set */
    struct device dev; /* embedded device */
};
```

**Parameters:**
- `cpu`: pointer للـ `struct cpu` — يجب أن يكون مهيأ مسبقًا.
- `num`: رقم الـ CPU (logical CPU number).

**Return value:** `0` نجاح، أو error code سالب.

**Key details:**
- بتضبط `cpu->dev.id = num` وتستدعي `device_register()`.
- لو `hotpluggable = 1`، بتضيف attribute `online` لـ sysfs.
- Implementation في `drivers/base/cpu.c`.

**Who calls it:** الـ arch-specific code عند init أو عبر `arch_register_cpu()`.

---

#### `get_cpu_device`

```c
struct device *get_cpu_device(unsigned cpu);
```

بترجع الـ `struct device*` المرتبط بالـ CPU رقم `cpu`. بتستخدم الـ per-CPU array المبني أثناء الـ registration.

**Parameters:**
- `cpu`: رقم الـ CPU (logical).

**Return value:** pointer للـ device أو `NULL` لو الـ CPU غير مسجل.

**Key details:**
- لا تحتاج locking — read-only access لـ static per-CPU data.
- مستخدمة كثيرًا في الـ driver subsystems للوصول للـ CPU device.

**Who calls it:** أي driver أو subsystem يحتاج CPU device pointer.

---

#### `cpu_device_create`

```c
struct device *cpu_device_create(struct device *parent, void *drvdata,
                                 const struct attribute_group **groups,
                                 const char *fmt, ...);
```

بتنشئ child device مرتبط بـ CPU device معين (مثل cache device أو PMU device). بتستخدم `__printf(4, 5)` لأن الاسم يُبنى بـ format string.

**Parameters:**
- `parent`: الـ CPU device الأب.
- `drvdata`: driver private data مرتبط بالـ device الجديد.
- `groups`: مجموعات الـ sysfs attributes.
- `fmt, ...`: format string لاسم الـ device.

**Return value:** pointer للـ device الجديد، أو `ERR_PTR(-ERRNO)` عند الفشل.

**Key details:**
- بتستدعى من subsystems زي الـ cache topology و PMU drivers.
- الـ device بيتحذف عند `device_unregister()`.

**Who calls it:** `drivers/base/cpu.c` و cache/PMU drivers.

---

#### `cpu_is_hotpluggable`

```c
bool cpu_is_hotpluggable(unsigned cpu);
```

بتفحص إذا كان الـ CPU المحدد يدعم الـ hotplug. بتراجع الـ `hotpluggable` field في الـ `struct cpu` والقيود المعمارية.

**Parameters:**
- `cpu`: رقم الـ CPU.

**Return value:** `true` لو يدعم الـ hotplug، `false` لو لا.

**Key details:**
- البعض من الـ CPUs ممكن تكون non-hotpluggable حتى لو CONFIG_HOTPLUG_CPU=y.
- بتستشير `arch_cpu_is_hotpluggable()` داخليًا.

---

#### `arch_cpu_probe` و `arch_cpu_release`

```c
ssize_t arch_cpu_probe(const char *buf, size_t count);   /* CONFIG_HOTPLUG_CPU */
ssize_t arch_cpu_release(const char *buf, size_t count); /* CONFIG_HOTPLUG_CPU */
```

هذان الـ arch hooks يسمحان لـ user space بـ probe أو release CPU عبر sysfs write. الـ `buf` يحتوي على المعلومات المُدخلة من user space. مستخدمة على المعماريات زي PowerPC.

---

### Group 3: Sysfs Attribute Management

هذه الـ functions بتضيف/تزيل custom sysfs attributes على **كل** CPU devices في نفس الوقت. مفيدة جدًا للـ drivers اللي محتاجة تعرض معلومات على مستوى الـ CPU.

---

#### `cpu_add_dev_attr`

```c
int cpu_add_dev_attr(struct device_attribute *attr);
```

بتضيف `device_attribute` واحدة لكل CPU device موجودة ومُسجَّلة. يعني لو عندك 8 CPUs، الـ attribute بتظهر في `/sys/devices/system/cpu/cpu{0..7}/`.

**Parameters:**
- `attr`: pointer لـ `struct device_attribute` — يجب أن يكون static أو persistent في الذاكرة.

**Return value:** `0` نجاح، أو error code سالب لو فشل على أي CPU.

**Key details:**
- بتستدعي `device_create_file()` لكل CPU.
- لو فشل على CPU معين، بتكمل ولا بترجع عند أول خطأ — الـ behavior هنا يختلف حسب الـ implementation.
- المُستدعي مسؤول عن الـ cleanup عبر `cpu_remove_dev_attr()`.

**Who calls it:** CPU frequency drivers، power management subsystems.

---

#### `cpu_remove_dev_attr`

```c
void cpu_remove_dev_attr(struct device_attribute *attr);
```

عكس `cpu_add_dev_attr()`. بتشيل الـ attribute من كل CPU devices.

---

#### `cpu_add_dev_attr_group` / `cpu_remove_dev_attr_group`

```c
int cpu_add_dev_attr_group(struct attribute_group *attrs);
void cpu_remove_dev_attr_group(struct attribute_group *attrs);
```

نفس فكرة `cpu_add_dev_attr` لكن بتتعامل مع `attribute_group` كاملة. أفضل لو عندك مجموعة attributes مترابطة (مثلًا كل attributes الـ cache).

---

### Group 4: Vulnerability Mitigation Show Functions

كل الـ functions دي signature واحدة:

```c
ssize_t cpu_show_XXXX(struct device *dev,
                      struct device_attribute *attr,
                      char *buf);
```

**الـ pattern العام:**
- بتقرأ حالة الـ mitigation من variable أو enum خاص بكل vulnerability.
- بتكتب نص قابل للقراءة البشرية في `buf` زي `"Mitigation: Retpoline"` أو `"Vulnerable"` أو `"Not affected"`.
- بترجع عدد البايتات المكتوبة (ssize_t).

**Parameters:**
- `dev`: الـ CPU device — غير مستخدم عادةً لكن مطلوب من interface الـ sysfs.
- `attr`: الـ device attribute — غير مستخدم عادةً.
- `buf`: page-sized buffer للكتابة فيه (PAGE_SIZE بايت).

**Return value:** `ssize_t` — طول النص المكتوب.

**Key details:**
- بتظهر في `/sys/devices/system/cpu/vulnerabilities/XXX`.
- كل function مرتبطة بـ `DEVICE_ATTR` macro في `drivers/base/cpu.c`.
- الـ arch-specific mitigation state محفوظة في variables زي `spectre_v2_enabled` أو enums.

**مثال للـ Spectre v2 على x86:**

```
/sys/devices/system/cpu/vulnerabilities/spectre_v2
→ "Mitigation: Retpoline; IBPB: conditional; IBRS_FW; STIBP: always-on"
```

**Who calls it:** الـ sysfs infrastructure عند read من المستخدم أو الـ tools زي `spectre-meltdown-checker`.

---

### Group 5: SMP Runtime Functions

هذه الـ functions تدير دورة حياة الـ CPU في بيئة SMP — من الـ bringup للـ teardown.

---

#### `add_cpu`

```c
int add_cpu(unsigned int cpu);
```

بتبدأ عملية إضافة CPU جديدة إلى الـ system عبر الـ hotplug state machine. بتحرك الـ CPU من state `CPUHP_OFFLINE` إلى `CPUHP_ONLINE` بشكل تسلسلي.

**Parameters:**
- `cpu`: رقم الـ CPU المراد تشغيله.

**Return value:** `0` نجاح، أو error code سالب.

**Key details:**
- بتستدعي `cpu_up()` داخليًا.
- بتأخذ الـ `cpu_add_remove_lock` ضمنيًا.
- في non-SMP config: `static inline` ترجع `0`.

**Pseudocode:**
```
add_cpu(cpu):
    lock(cpu_add_remove_lock)
    cpu_up(cpu, CPUHP_ONLINE)
    unlock(cpu_add_remove_lock)
```

**Who calls it:** user space عبر `/sys/.../cpu/cpuN/online`, أو كود الـ hotplug.

---

#### `cpu_device_up`

```c
int cpu_device_up(struct device *dev);
```

الـ sysfs write handler لملف `online`. لما يكتب المستخدم `1` في `/sys/devices/system/cpu/cpuN/online`، الـ sysfs بتستدعي هذه الـ function.

**Parameters:**
- `dev`: الـ CPU device المراد تشغيله.

**Return value:** `0` نجاح، أو error code.

**Key details:**
- بتستخرج الـ CPU number من الـ device ثم تستدعي `add_cpu()`.

---

#### `notify_cpu_starting`

```c
void notify_cpu_starting(unsigned int cpu);
```

بتُطلق الـ STARTING callbacks في الـ hotplug state machine للـ CPU الجديد. بتتستدعى من الـ AP (Application Processor) نفسه بعد أن يكمل التهيئة المعمارية الأساسية وقبل أن يدخل الـ scheduler.

**Parameters:**
- `cpu`: رقم الـ CPU.

**Key details:**
- بتتستدعى بـ interrupts disabled.
- بتمر بكل الـ STARTING states بالترتيب.
- لو فشل أي callback، الـ CPU يُعاد إغلاقه.

**Who calls it:** `arch/x86/kernel/smpboot.c` عبر `start_secondary()`.

---

#### `cpu_maps_update_begin` / `cpu_maps_update_done`

```c
void cpu_maps_update_begin(void);
void cpu_maps_update_done(void);
```

الـ pair دي بتحمي التعديلات على الـ CPU maps (possible/present/online/active). بتستخدمان الـ `cpu_add_remove_lock` mutex.

**Key details:**
- يجب استخدامهما دائمًا معًا.
- في non-SMP: `static inline` فارغتان.
- كل كود يعدل الـ CPU topology يجب أن يمسك هذا الـ lock.

---

#### `bringup_hibernate_cpu`

```c
int bringup_hibernate_cpu(unsigned int sleep_cpu);
```

بتُعيد تشغيل الـ CPU اللي كان يشغل الـ hibernation code. مطلوبة لأن هذا الـ CPU كان offline أثناء الـ resume من الـ disk image.

**Parameters:**
- `sleep_cpu`: رقم الـ CPU اللي كان نائمًا.

**Return value:** `0` نجاح، أو error code.

**Key details:**
- بتتستدعى من مسار الـ resume في `kernel/power/hibernate.c`.
- بتحتاج تشغيل الـ CPU قبل أن تكمل الـ resume sequence.

---

#### `bringup_nonboot_cpus`

```c
void bringup_nonboot_cpus(unsigned int max_cpus);
```

بتشغل كل الـ secondary CPUs أثناء الـ boot حتى حد `max_cpus`. بتتكرر على كل الـ possible CPUs وتحاول تشغيل كل واحدة.

**Parameters:**
- `max_cpus`: الحد الأقصى للـ CPUs المراد تشغيلها (من الـ cmdline `maxcpus=`).

**Key details:**
- مستدعاة من `smp_init()` في `kernel/smp.c`.
- لو `max_cpus = 0`، لا يتم تشغيل أي CPU.

---

### Group 6: Suspend / Power Management Sleep

هذه الـ functions تدير إيقاف وإعادة تشغيل الـ CPUs أثناء دورة الـ suspend/resume.

---

#### `freeze_secondary_cpus`

```c
int freeze_secondary_cpus(int primary);
```

بتوقف كل الـ CPUs ما عدا الـ `primary` CPU قبل الـ suspend. بتمر بكل الـ online CPUs وتعملها hotplug-out بشكل منظم.

**Parameters:**
- `primary`: رقم الـ CPU اللي يبقى يشتغل (يدير الـ suspend).

**Return value:** `0` نجاح، أو error code لو فشل إيقاف أي CPU.

**Key details:**
- `CONFIG_PM_SLEEP_SMP` مطلوب.
- بتتستدعى من `suspend_disable_secondary_cpus()` wrapper.
- لو `CONFIG_PM_SLEEP_SMP_NONZERO_CPU=y`، الـ primary بيبقى CPU رقم `-1` (أول CPU available).

---

#### `thaw_secondary_cpus`

```c
void thaw_secondary_cpus(void);
```

بتُعيد تشغيل كل الـ CPUs اللي اتوقفت بواسطة `freeze_secondary_cpus()` بعد اكتمال الـ resume.

**Key details:**
- بتتستدعى من `suspend_enable_secondary_cpus()`.
- لا تقبل error — الـ resume يجب أن يكتمل.

---

#### `suspend_disable_secondary_cpus`

```c
static inline int suspend_disable_secondary_cpus(void);
```

**Wrapper** حول `freeze_secondary_cpus()`. بتحدد الـ primary CPU اعتمادًا على `CONFIG_PM_SLEEP_SMP_NONZERO_CPU`:
- لو disabled: primary = CPU 0.
- لو enabled: primary = -1 (أي CPU).

```c
static inline int suspend_disable_secondary_cpus(void)
{
    int cpu = 0;
    if (IS_ENABLED(CONFIG_PM_SLEEP_SMP_NONZERO_CPU))
        cpu = -1;
    return freeze_secondary_cpus(cpu);
}
```

---

### Group 7: Idle Subsystem

هذه الـ functions تدير دورة الـ idle للـ CPU — من الدخول للـ idle حتى الموت عند الـ hotplug-out.

---

#### `cpu_startup_entry`

```c
void __noreturn cpu_startup_entry(enum cpuhp_state state);
```

الـ function دي هي نقطة دخول الـ idle loop — **لا ترجع أبدًا**. بتبدأ تشغيل الـ `do_idle()` loop اللي بيتحكم في الانتقال بين الـ idle states وتنفيذ الـ softirqs والتحقق من الـ need_resched.

**Parameters:**
- `state`: الـ hotplug state الابتدائي (عادةً `CPUHP_AP_ONLINE_IDLE` أو `CPUHP_ONLINE`).

**Return value:** لا يرجع (`__noreturn`).

**Key details:**
- كل CPU (boot و secondary) تمر بهذه الـ function.
- الـ `do_idle()` بداخلها تستدعي `cpuidle_idle_call()` أو `arch_cpu_idle()`.
- الـ hotplug teardown بيخرج منها عبر `play_dead()`.

**Pseudocode:**
```
cpu_startup_entry(state):
    cpuhp_online_idle(state)     // mark ONLINE_IDLE if needed
    while (true):
        do_idle()                // idle + serve softirqs
        // never reaches here
```

**Who calls it:** بعد `start_secondary()` لكل AP، وبعد `rest_init()` للـ boot CPU.

---

#### `cpu_idle_poll_ctrl`

```c
void cpu_idle_poll_ctrl(bool enable);
```

بتتحكم في الـ idle polling mode. لما تكون مُفعَّلة، الـ CPU بيـ spin في loop بدل ما يدخل في حالة power-saving حقيقية. مفيدة لتقليل الـ latency في الـ real-time scenarios.

**Parameters:**
- `enable`: `true` لتفعيل الـ polling، `false` لإلغائه.

**Key details:**
- بتعدل الـ per-CPU `cpu_idle_force_poll` counter.
- المُستدعي مسؤول عن الـ balance (كل enable يحتاج disable مقابل له).

---

#### `cpu_in_idle`

```c
bool cpu_in_idle(unsigned long pc);
```

بتتحقق إذا كان الـ program counter المعطى يقع داخل كود الـ idle. مستخدمة من الـ NMI handlers والـ watchdog لتحديد إذا كان الـ CPU في idle أثناء stall.

**Parameters:**
- `pc`: قيمة الـ program counter.

**Return value:** `true` لو الـ PC في نطاق الـ idle code.

---

#### `play_idle_precise`

```c
void play_idle_precise(u64 duration_ns, u64 latency_ns);
```

بتضع الـ CPU في حالة idle لمدة محددة بالـ nanoseconds مع مراعاة متطلبات الـ latency. مستخدمة من الـ real-time و hypervisor subsystems لإدارة الـ CPU time.

**Parameters:**
- `duration_ns`: مدة الـ idle بالـ nanoseconds.
- `latency_ns`: الحد الأقصى المسموح به لـ exit latency.

**Key details:**
- بتحترم الـ cpuidle governors وتختار state مناسب.
- مستخدمة من KVM لـ idle injection.

---

#### `cpuhp_report_idle_dead`

```c
void cpuhp_report_idle_dead(void);  /* CONFIG_HOTPLUG_CPU */
```

بتُطلَق من الـ AP عندما يموت في الـ idle loop أثناء الـ hotplug-out. بتُخطر الـ hotplug core (البـ BP) أن الـ AP أتم عملية الموت بنجاح.

**Key details:**
- بتُستدعى من `play_dead()` على الـ AP.
- الـ BP كان ينتظر هذا الإخطار عبر polling أو WFI.
- في non-HOTPLUG: `static inline` فارغة.

---

### Group 8: Arch Idle Hooks

الـ Linux idle subsystem بيوفر hooks للـ architecture لتنفيذ الـ idle بطريقتها الخاصة.

---

#### `arch_cpu_idle`

```c
void arch_cpu_idle(void);
```

الـ arch-specific idle implementation. على x86: بتستدعي `halt()` (HLT instruction). على ARM: `wfi`. على RISC-V: `wfi` كذلك.

**Key details:**
- بتُستدعى بـ interrupts disabled (من `do_idle()`).
- يجب أن تُمكّن الـ interrupts قبل ما تنتهي لأن interrupt هو ما يوقظ الـ CPU.

---

#### `arch_cpu_idle_prepare`

```c
void arch_cpu_idle_prepare(void);
```

بتُستدعى مرة واحدة قبل أن يدخل الـ CPU في أول idle. للـ setup المطلوب قبل الـ idle الأول.

---

#### `arch_cpu_idle_enter` / `arch_cpu_idle_exit`

```c
void arch_cpu_idle_enter(void);
void arch_cpu_idle_exit(void);
```

بتُستدعيان في كل دورة idle. الـ enter: لضبط flags أو registers قبل الـ idle. الـ exit: لاستعادة الحالة بعده.

**Key details:**
- الـ tick و timer state ممكن يحتاجوا تعديل هنا.
- على بعض الـ architectures: تستخدم لـ local timer management.

---

#### `arch_tick_broadcast_enter` / `arch_tick_broadcast_exit`

```c
void arch_tick_broadcast_enter(void);
void arch_tick_broadcast_exit(void);
```

لما الـ CPU بيدخل في deep idle state بتوقف فيها الـ local timer، الـ kernel محتاج يستخدم broadcast timer يصحي الـ CPU. الـ enter بتُفعّل هذا الـ broadcast mode. الـ exit بتُلغيه.

**Key details:**
- مرتبطة بالـ `tick_broadcast_*` framework في `kernel/time/tick-broadcast.c`.
- ضرورية لـ states زي C3 على x86 أو retention mode على ARM.

---

#### `arch_cpu_idle_dead`

```c
void __noreturn arch_cpu_idle_dead(void);
```

بتُستدعى عند الـ hotplug-out لتنفيذ arch-specific CPU death sequence. على x86: تستدعي `native_play_dead()` اللي بيدخل الـ CPU في HLT loop. **لا ترجع أبدًا**.

---

#### `arch_cpu_finalize_init`

```c
void arch_cpu_finalize_init(void);  /* CONFIG_ARCH_HAS_CPU_FINALIZE_INIT */
```

بتُستدعى بعد اكتمال كل الـ CPU initialization ولكن قبل أن يبدأ الـ kernel في تشغيل الـ user space. مستخدمة على x86 لإنهاء تهيئة الـ microcode والـ mitigations.

---

### Group 9: CPU Mitigations

هذه الـ functions تتعلق بالتحكم في الـ speculative execution mitigations على مستوى الـ kernel.

---

#### `cpu_mitigations_off`

```c
bool cpu_mitigations_off(void);  /* CONFIG_CPU_MITIGATIONS */
```

بتفحص إذا كان المستخدم مرر `mitigations=off` في الـ kernel command line. لو `true`، يعني كل الـ mitigations مُعطَّلة لصالح الـ performance.

**Return value:** `true` لو mitigations off، `false` لو مفعلة.

**Key details:**
- في non-MITIGATIONS config: دائمًا ترجع `true` (يعني off).
- بتُستخدم من كل الـ arch-specific mitigation code.

---

#### `cpu_mitigations_auto_nosmt`

```c
bool cpu_mitigations_auto_nosmt(void);  /* CONFIG_CPU_MITIGATIONS */
```

بتفحص إذا كان الـ kernel يعمل في وضع `mitigations=auto,nosmt`. في هذا الوضع، الـ SMT يُعطَّل تلقائيًا لو الـ vulnerability تتطلب ذلك.

**Return value:** `true` لو في وضع auto+nosmt.

**Key details:**
- في non-MITIGATIONS config: دائمًا `false`.
- مرتبطة بقرار تعطيل الـ SMT في الـ `cpu_smt_disable()` path.

---

#### `cpu_attack_vector_mitigated`

```c
bool cpu_attack_vector_mitigated(enum cpu_attack_vectors v);
```

بتفحص إذا كان attack vector محدد مُدار بـ mitigation فعّالة. الـ vectors المعرَّفة:

```c
enum cpu_attack_vectors {
    CPU_MITIGATE_USER_KERNEL,   /* user → kernel attacks */
    CPU_MITIGATE_USER_USER,     /* user → user attacks */
    CPU_MITIGATE_GUEST_HOST,    /* VM guest → host attacks */
    CPU_MITIGATE_GUEST_GUEST,   /* VM guest → guest attacks */
    NR_CPU_ATTACK_VECTORS,
};
```

**Parameters:**
- `v`: نوع الـ attack vector.

**Return value:** `true` لو المُهاجَم مُحمي.

**Key details:**
- الـ `smt_mitigations` variable (من `cpu_smt.h`) بتؤثر في الـ GUEST_GUEST و USER_USER vectors.
- في non-MITIGATIONS config: دائمًا `false`.

---

### Group 10: SMT Control Functions

**الـ SMT (Simultaneous Multi-Threading)** — زي الـ Hyper-Threading على Intel — هو مصدر لعدة vulnerabilities لأن الـ threads في نفس الـ physical core بيشاركوا الـ hardware buffers.

---

#### `cpu_smt_disable`

```c
void cpu_smt_disable(bool force);
```

بتُعطّل الـ SMT. لو `force = true`، بتضبط الـ `cpu_smt_control = CPU_SMT_FORCE_DISABLED` وتمنع إعادة التفعيل. بتمر على كل الـ SMT sibling threads وتعملهم hotplug-out.

**Parameters:**
- `force`: `true` لتعطيل دائم لا يمكن عكسه.

**Key details:**
- `CONFIG_SMP && CONFIG_HOTPLUG_SMT` مطلوب.
- مرتبطة بالـ `smt_mitigations` global variable.

---

#### `cpu_smt_set_num_threads`

```c
void cpu_smt_set_num_threads(unsigned int num_threads,
                              unsigned int max_threads);
```

بتحدد عدد الـ SMT threads المسموح بتشغيلها في كل physical core. مفيدة لتقليل الـ attack surface دون تعطيل الـ SMT كليًا.

**Parameters:**
- `num_threads`: عدد الـ threads المطلوب.
- `max_threads`: الحد الأقصى المدعوم من الـ hardware.

---

#### `cpuhp_smt_enable` / `cpuhp_smt_disable`

```c
int cpuhp_smt_enable(void);
int cpuhp_smt_disable(enum cpuhp_smt_control ctrlval);
```

بتتعامل مع الـ SMT enable/disable عبر الـ hotplug state machine بشكل منظم — لأن تشغيل/إيقاف الـ SMT threads هو في الأساس hotplug operation.

**Parameters لـ `cpuhp_smt_disable`:**
- `ctrlval`: قيمة الـ control (`CPU_SMT_DISABLED` أو `CPU_SMT_FORCE_DISABLED`).

**Return value:** `0` نجاح، أو error code.

---

### Group 11: Indirect Branch / Loop Prevention (arch Security Hooks)

هذه الـ functions جديدة نسبيًا وتتعلق بمكافحة هجمات الـ speculative execution عبر التحكم في الـ indirect branch behavior على مستوى الـ task.

---

#### `arch_get_indir_br_lp_status`

```c
int arch_get_indir_br_lp_status(struct task_struct *t,
                                 unsigned long __user *status);
```

بتجلب حالة الـ indirect branch loop prevention (LP) للـ task المحدد وتضعها في الـ user space pointer. مستخدمة من syscall مثلًا لإتاحة قراءة هذا الـ security attribute.

**Parameters:**
- `t`: الـ task المستهدف.
- `status`: user space pointer للكتابة فيه.

**Return value:** `0` نجاح، أو `-EFAULT` / `-EINVAL`.

---

#### `arch_set_indir_br_lp_status`

```c
int arch_set_indir_br_lp_status(struct task_struct *t,
                                 unsigned long status);
```

بتضبط حالة الـ indirect branch LP للـ task. تسمح لـ user space بتفعيل أو تعطيل هذا الـ mitigation للـ task نفسه.

---

#### `arch_lock_indir_br_lp_status`

```c
int arch_lock_indir_br_lp_status(struct task_struct *t,
                                  unsigned long status);
```

بتقفل الـ indirect branch LP status — بعد الاستدعاء لا يمكن تغيير الحالة مرة أخرى. مفيدة للـ high-security contexts.

**Key details:**
- الثلاث functions معمارية بالكامل (arch-specific).
- مرتبطة بالـ Indirect Target Selection (ITS) mitigation على Intel.
- بتتستدعى من syscall interface (prctl أو مشابه).

---

### ملاحظات عامة على الـ Locking والـ Context

```
Function                    | Lock Required              | IRQ Context OK?
----------------------------|---------------------------|----------------
register_cpu()              | none (init time)          | No
cpu_maps_update_begin()     | acquires lock             | No
add_cpu()                   | cpu_add_remove_lock       | No
notify_cpu_starting()       | none                      | IRQs disabled
cpu_startup_entry()         | none (owns CPU)           | No (preemptible after)
arch_cpu_idle()             | none                      | IRQs disabled
freeze_secondary_cpus()     | cpu_add_remove_lock       | No
cpu_show_*()                | none (read-only)          | No
cpu_mitigations_off()       | none (atomic read)        | Yes
```

### ASCII: CPU Lifecycle Overview

```
                    boot_cpu_init()
                         │
                    boot_cpu_hotplug_init()
                         │
                    bringup_nonboot_cpus()
                         │
              ┌──────────▼──────────┐
              │    register_cpu()   │  ←── sysfs /sys/devices/system/cpu/cpuN
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │    add_cpu()        │  ←── hotplug state machine
              │  notify_cpu_starting│
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │ cpu_startup_entry() │  ←── idle loop (never returns)
              │     do_idle()       │
              │   arch_cpu_idle()   │
              └──────────┬──────────┘
                         │ (hotplug-out)
              ┌──────────▼──────────┐
              │ arch_cpu_idle_dead()│  ←── CPU dies here
              │cpuhp_report_idle_dead│
              └─────────────────────┘
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs المتعلقة بالـ CPU Hotplug

الـ kernel بيعرض معلومات الـ hotplug states عبر debugfs. بعد mount الـ debugfs:

```bash
mount -t debugfs none /sys/kernel/debug
```

| المسار | المحتوى |
|---|---|
| `/sys/kernel/debug/cpu/hotplug/states` | قائمة بكل الـ hotplug states المسجلة واسمها |
| `/sys/kernel/debug/cpu/hotplug/fail` | لو ضبطت state رقم هنا، الـ hotplug بيتعمل fail عندها |

```bash
# اقرأ كل الـ states المسجلة
cat /sys/kernel/debug/cpu/hotplug/states

# اجبر فشل الـ hotplug عند state معينة (للتستينج)
echo 168 > /sys/kernel/debug/cpu/hotplug/fail

# ارجع للحالة الطبيعية
echo -1 > /sys/kernel/debug/cpu/hotplug/fail
```

**الـ `states` file** بيطبع كل entry بالشكل:
```
  1: name: "cpuhp/create:threads"
  2: name: "perf/x86/prepare"
...
```
الأرقام دي هي قيم الـ `enum cpuhp_state` مباشرةً.

---

#### 2. مدخلات الـ sysfs المتعلقة بالـ CPU

الـ CPUs بتظهر في `/sys/devices/system/cpu/` — ده المسار الأساسي اللي `drivers/base/cpu.c` بيعمله register فيه.

```bash
# اعرض كل الـ CPUs الموجودة
ls /sys/devices/system/cpu/

# حالة كل CPU (online/offline)
cat /sys/devices/system/cpu/cpu0/online

# هل الـ CPU hotpluggable؟ (بيتحكم فيه hotpluggable field في struct cpu)
cat /sys/devices/system/cpu/cpu1/hotpluggable

# الـ topology (الـ node_id بيتعكس هنا)
cat /sys/devices/system/cpu/cpu0/topology/physical_package_id
cat /sys/devices/system/cpu/cpu0/topology/core_id

# SMT control (cpu_smt_control / cpu_smt_num_threads)
cat /sys/devices/system/cpu/smt/control
cat /sys/devices/system/cpu/smt/active

# mitigations (cpu_show_meltdown, cpu_show_spectre_v1, ...إلخ)
cat /sys/devices/system/cpu/vulnerabilities/meltdown
cat /sys/devices/system/cpu/vulnerabilities/spectre_v1
cat /sys/devices/system/cpu/vulnerabilities/spectre_v2
cat /sys/devices/system/cpu/vulnerabilities/spec_store_bypass
cat /sys/devices/system/cpu/vulnerabilities/l1tf
cat /sys/devices/system/cpu/vulnerabilities/mds
cat /sys/devices/system/cpu/vulnerabilities/tsx_async_abort
cat /sys/devices/system/cpu/vulnerabilities/itlb_multihit
cat /sys/devices/system/cpu/vulnerabilities/srbds
cat /sys/devices/system/cpu/vulnerabilities/mmio_stale_data
cat /sys/devices/system/cpu/vulnerabilities/retbleed
cat /sys/devices/system/cpu/vulnerabilities/spec_rstack_overflow
cat /sys/devices/system/cpu/vulnerabilities/gds
cat /sys/devices/system/cpu/vulnerabilities/reg_file_data_sampling
cat /sys/devices/system/cpu/vulnerabilities/ghostwrite
cat /sys/devices/system/cpu/vulnerabilities/indirect_target_selection
```

**اعمل online/offline لـ CPU بـ sysfs:**

```bash
# offline
echo 0 > /sys/devices/system/cpu/cpu3/online

# online
echo 1 > /sys/devices/system/cpu/cpu3/online
```

---

#### 3. استخدام الـ ftrace مع CPU Hotplug

الـ tracepoints الأساسية:

```bash
# اعرض الـ events المتاحة للـ cpuhp
ls /sys/kernel/debug/tracing/events/cpuhp/

# فعّل كل events الـ cpuhp
echo 1 > /sys/kernel/debug/tracing/events/cpuhp/enable

# أو فعّل event معين فقط
echo 1 > /sys/kernel/debug/tracing/events/cpuhp/cpuhp_enter/enable
echo 1 > /sys/kernel/debug/tracing/events/cpuhp/cpuhp_exit/enable
echo 1 > /sys/kernel/debug/tracing/events/cpuhp/cpuhp_multi_enter/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل hotplug وبعدين اقرأ النتيجة
echo 0 > /sys/devices/system/cpu/cpu1/online
cat /sys/kernel/debug/tracing/trace
```

**مثال على output:**

```
systemd-1234  [000] .... cpuhp_enter: cpu: 0001 target: 0 step: CPUHP_AP_ONLINE (168)
systemd-1234  [000] .... cpuhp_exit:  cpu: 0001 state: 0 step: CPUHP_AP_ONLINE ret: 0
```

لو `ret` مش 0 → في مشكلة في الـ callback بتاع الـ state دي.

**لتتبع دالة `cpu_startup_entry`:**

```bash
echo "cpu_startup_entry" > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. تفعيل الـ printk / Dynamic Debug للـ subsystem ده

**الـ dynamic debug** بيخليك تفعّل `pr_debug()` و `dev_dbg()` بدون إعادة compile:

```bash
# فعّل كل debug messages في ملفات الـ cpu hotplug
echo 'file drivers/base/cpu.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/cpu.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/smp.c +p' > /sys/kernel/debug/dynamic_debug/control

# اعرض الـ debug messages الفعّالة حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep cpu

# رفع مستوى الـ loglevel لتشوف كل الـ printk
dmesg -n 8
# أو
echo 8 > /proc/sys/kernel/printk
```

**تابع الـ kernel log أثناء hotplug:**

```bash
dmesg -w | grep -E "(CPU|hotplug|cpuhp|online|offline)"
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| Option | الوظيفة |
|---|---|
| `CONFIG_HOTPLUG_CPU` | يفعّل دعم الـ CPU hotplug كلياً |
| `CONFIG_DEBUG_HOTPLUG_CPU0` | يسمح بـ hotplug لـ CPU0 (نادراً — للتستينج فقط) |
| `CONFIG_CPU_HOTPLUG_STATE_CONTROL` | يضيف debugfs interface للتحكم في states |
| `CONFIG_HOTPLUG_SMT` | يفعّل SMT hotplug وبالتالي `cpu_smt_control` |
| `CONFIG_GENERIC_CPU_DEVICES` | يعرف `DECLARE_PER_CPU(struct cpu, cpu_devices)` |
| `CONFIG_CPU_MITIGATIONS` | يفعّل الـ mitigation functions (meltdown, spectre...إلخ) |
| `CONFIG_DEBUG_PREEMPT` | يكتشف preemption issues أثناء hotplug callbacks |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في `cpu_maps_update_begin/done` |
| `CONFIG_PROVE_LOCKING` | يتحقق من صحة الـ locking order |
| `CONFIG_KASAN` | يكتشف memory corruption في per-cpu structures |
| `CONFIG_KCSAN` | يكتشف data races في الـ hotplug path |
| `CONFIG_PM_SLEEP_SMP` | يفعّل `freeze_secondary_cpus` أثناء suspend |
| `CONFIG_HARDLOCKUP_DETECTOR` | يكتشف لو CPU وقف أثناء bringup |
| `CONFIG_SOFTLOCKUP_DETECTOR` | يكتشف لو الـ hotplug thread اتعلق |

```bash
# اتحقق من القيم الحالية
grep -E "HOTPLUG|CPU_MITIGATIONS|SMT|GENERIC_CPU" /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الـ `cpupower` tool** — بيتحكم في CPU frequency والـ idle states:

```bash
# اعرض حالة كل CPUs
cpupower -c all frequency-info
cpupower -c all idle-info

# اعرض الـ governor الحالي
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

**الـ `numactl`** — مهم لأن `struct cpu` بيحتوي على `node_id`:

```bash
# اعرض الـ NUMA topology
numactl --hardware
numactl --show

# اعرض أي CPUs على أي NUMA node
cat /sys/devices/system/cpu/cpu*/topology/physical_package_id
```

**أداة `lscpu`:**

```bash
lscpu -e   # عرض تفصيلي per-CPU
lscpu      # عرض الـ topology الكاملة
```

**SMT/Hyperthreading control:**

```bash
# اقرأ حالة SMT (cpu_smt_control)
cat /sys/devices/system/cpu/smt/control
# القيم: on | off | forceoff | notsupported | notimplemented

# عدد الـ SMT threads (cpu_smt_num_threads)
cat /sys/devices/system/cpu/smt/active

# عطّل SMT (بيستخدم cpuhp_smt_disable)
echo off > /sys/devices/system/cpu/smt/control
```

---

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `CPU X is not hotpluggable` | الـ CPU مش hotpluggable — الـ `hotpluggable` field في `struct cpu` = 0 | تحقق من `arch_cpu_is_hotpluggable()` — قد يكون architecture limitation |
| `CPU X is already offline` | محاولة offline لـ CPU هو أصلاً offline | تحقق من `cpu_online(cpu)` قبل الأمر |
| `Error taking CPU X down` | فشل في إحدى teardown callbacks | شوف الـ ftrace cpuhp_exit event للـ `ret` code |
| `CPU X is the last CPU, can't be offlined` | لا يمكن عمل offline لآخر CPU online | طبيعي — system يحتاج CPU واحد على الأقل |
| `cpuhp_invoke_callback: ... returned ... on CPU X` | callback رجع error code | شوف اسم الـ state في `/sys/kernel/debug/cpu/hotplug/states` |
| `CPU X hard LOCKUP` | CPU وقف تماماً أثناء bringup/teardown | hardware issue أو interrupt storm أثناء hotplug |
| `smpboot: do_boot_cpu X timed out` | الـ AP CPU فشل يرد في الوقت المحدد | BIOS/firmware issue أو باص مشكلة — تحقق من ACPI tables |
| `CPU X is frozen` | CPU في حالة `CPU_DEAD_FROZEN` — تجاوز timeout عند unplug | hardware لا يستجيب لـ SIPI — تحقق من الـ power management |
| `CPU_BROKEN: CPU X failed to come up` | الـ CPU وصل state `CPU_BROKEN` | hardware fault — استبدل الـ CPU أو راجع الـ firmware |
| `Detected Spectre v2 ... mitigation: ...` | تقرير تلقائي عند boot | معلوماتي — قارن مع `/sys/devices/system/cpu/vulnerabilities/spectre_v2` |
| `MDS: Vulnerable` | `cpu_show_mds` بترجع vulnerable | حدّث الـ microcode: `iucode-tool` أو `intel-microcode` |
| `SRBDS: Vulnerable: No microcode` | `cpu_show_srbds` — microcode قديم | راجع `cpu_show_old_microcode` في sysfs |

---

#### 8. نقاط إضافة `dump_stack()` و `WARN_ON()`

**في `register_cpu()`** — لو `register_cpu` فشل ورجع error:

```c
int ret = register_cpu(cpu, num);
if (WARN_ON(ret < 0)) {
    /* log the topology state at failure time */
    dump_stack();
}
```

**في الـ hotplug callbacks** — عند state transitions غير متوقعة:

```c
/* في startup callback */
WARN_ON(!cpu_online(smp_processor_id()));

/* لو cpu_maps_update_begin/done اتخطت */
WARN_ON_ONCE(cpu_hotplug_disabled);
```

**نقاط استراتيجية:**

| المكان | الهدف |
|---|---|
| بعد `cpu_maps_update_begin()` | تحقق من consistency الـ cpu_online_mask |
| في `boot_cpu_init()` | تحقق من إن الـ boot CPU initialized صح |
| في `notify_cpu_starting()` | تحقق من الـ CPU state قبل notify |
| في `freeze_secondary_cpus()` | تحقق من إن كل CPUs اتجمدت فعلاً |
| في `add_cpu()` return path | log الـ cpuhp_state عند فشل الـ bringup |

---

### Hardware Level

#### 1. التحقق من حالة الـ Hardware مقابل حالة الـ Kernel

الـ `struct cpu` بيحتوي على `node_id` (الـ NUMA node) و`hotpluggable`. لازم يتطابقوا مع الـ hardware الفعلي:

```bash
# قارن الـ node_id في kernel مع الـ hardware
for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
    echo -n "$(basename $cpu): node="
    cat $cpu/topology/physical_package_id 2>/dev/null || echo "N/A"
done

# تحقق من الـ ACPI CPU topology
acpidump -n MADT | acpixtract -a && iasl -d MADT.dat

# قارن مع kernel view
cat /sys/bus/cpu/devices/*/uevent
```

**الـ APIC IDs** — `arch_match_cpu_phys_id()` بتطابق الـ physical ID:

```bash
# اعرض الـ APIC ID لكل CPU
cat /sys/devices/system/cpu/cpu*/acpi_id 2>/dev/null
# أو من dmesg
dmesg | grep "APIC id"
```

---

#### 2. تقنيات الـ Register Dump

للوصول لـ CPU-specific MSRs (مفيد لفحص microcode version وحالة الـ mitigations):

```bash
# قراءة MSR (يحتاج modprobe msr)
modprobe msr

# اقرأ microcode version (MSR 0x8B) من CPU0
rdmsr -p 0 0x8b

# اقرأ IA32_ARCH_CAPABILITIES (MSR 0x10A) - مهم لـ Meltdown/Spectre
rdmsr -p 0 0x10a

# اقرأ SPEC_CTRL MSR (0x48) - Spectre v2 mitigation
rdmsr -p 0 0x48

# اقرأ FLUSH_CMD MSR (0x10B) - MDS mitigation
rdmsr -p 0 0x10b
```

**لو `rdmsr` مش موجود:**

```bash
# استخدم /dev/cpu/X/msr مباشرة
dd if=/dev/cpu/0/msr bs=8 count=1 skip=$((0x8B)) 2>/dev/null | xxd
```

**لقراءة الـ memory-mapped registers (نادر لـ CPU subsystem):**

```bash
# استخدم devmem2 لعناوين physical memory
devmem2 0xFEE00020 w   # قراءة APIC ID register (local APIC)
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

أثناء CPU hotplug:

- **راقب خط RESET#** على الـ CPU socket — عند offline الـ CPU بيتبعت له INIT IPI ثم SIPI
- **راقب خط PWR_GOOD** — لو بيتأخر بعد power-on → سبب شائع لـ `do_boot_cpu timed out`
- **خطوط JTAG/SWD** — استخدم الـ JTAG debugger (OpenOCD أو Intel ITP) لفحص CPU registers مباشرة أثناء الـ bringup

```
CPU Bringup Sequence على الـ Bus:
  BP                     AP
  |--[INIT IPI]--------->|
  |--[SIPI:vector]------->|
  |                       |-- starts executing at vector
  |<--[alive signal]------|  (cpuhp_ap_sync_alive)
  |                       |-- goes through cpuhp states
```

**للـ Oscilloscope:** قس الـ timing بين إرسال الـ SIPI وبين ظهور أول NMI/interrupt من الـ AP. اللي بيتجاوز 100ms يعتبر timeout.

---

#### 4. المشكلات الشائعة في الـ Hardware → أنماط في الـ Kernel Log

| المشكلة الـ Hardware | النمط في الـ dmesg |
|---|---|
| CPU لا يستجيب لـ SIPI | `smpboot: do_boot_cpu X timed out` |
| مشكلة في الـ BIOS MADT table | `ACPI: MADT: X CPUs ... ignoring` |
| الـ microcode لم يُحمّل | `microcode: ... not updated early` + `SRBDS: Vulnerable: No microcode` |
| مشكلة NUMA node mapping | `NUMA: Node X ... has no CPUs` |
| Thermal throttling شديد | `CPU X: Core temperature above threshold` ثم auto-offline |
| فشل في power delivery للـ AP | `CPU X is now offline` بعد ثوانٍ من `online` |
| SMT غير مدعوم في الـ hardware | `cpu_smt_control = CPU_SMT_NOT_SUPPORTED` |

---

#### 5. الـ Device Tree Debugging (للأنظمة المعتمدة على DT)

الـ `arch_find_n_match_cpu_physical_id()` بتبحث في الـ DT عن CPU nodes:

```bash
# اعرض CPU nodes في الـ DT المستخدم حالياً
cat /proc/device-tree/cpus/cpu*/reg | xxd

# أو باستخدام dtc
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "cpu@"

# تحقق من الـ physical IDs في الـ DT مقابل kernel
dmesg | grep "CPU.*phys"
```

**تحقق من تطابق `reg` property في DT مع الـ MPIDR على ARM:**

```bash
# اقرأ MPIDR لكل CPU
for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
    echo -n "$(basename $cpu): "
    cat $cpu/regs/identification/mpidr 2>/dev/null || echo "N/A"
done
```

**لو الـ DT فيه `status = "disabled"` لـ CPU:**

```bash
# شوف لو CPU موجود في DT لكن disabled
dtc -I fs /proc/device-tree 2>/dev/null | grep -B2 "disabled" | grep cpu
# ده بيمنع register_cpu() من تسجيله
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للـ CPU Hotplug Debugging

**1. طبع كل المعلومات الأساسية دفعة واحدة:**

```bash
#!/bin/bash
echo "=== CPU Online Mask ==="
cat /sys/devices/system/cpu/online

echo "=== CPU Present Mask ==="
cat /sys/devices/system/cpu/present

echo "=== CPU Possible Mask ==="
cat /sys/devices/system/cpu/possible

echo "=== SMT Control ==="
cat /sys/devices/system/cpu/smt/control 2>/dev/null || echo "SMT not supported"

echo "=== Hotpluggable CPUs ==="
for f in /sys/devices/system/cpu/cpu*/hotpluggable; do
    cpu=$(echo $f | grep -o 'cpu[0-9]*')
    val=$(cat $f 2>/dev/null)
    echo "  $cpu: $val"
done

echo "=== Vulnerabilities ==="
for v in /sys/devices/system/cpu/vulnerabilities/*; do
    echo "  $(basename $v): $(cat $v)"
done
```

**2. اعمل offline/online لـ CPU مع مراقبة:**

```bash
#!/bin/bash
CPU=3
echo "--- Taking CPU$CPU offline ---"
dmesg -c > /dev/null  # clear dmesg
echo 0 > /sys/devices/system/cpu/cpu${CPU}/online
echo "--- dmesg after offline ---"
dmesg

echo "--- Bringing CPU$CPU online ---"
dmesg -c > /dev/null
echo 1 > /sys/devices/system/cpu/cpu${CPU}/online
echo "--- dmesg after online ---"
dmesg
```

**3. تفعيل كامل الـ tracing لـ hotplug:**

```bash
#!/bin/bash
# Setup tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo 1 > /sys/kernel/debug/tracing/events/cpuhp/enable
echo 4096 > /sys/kernel/debug/tracing/buffer_size_kb
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Trigger hotplug
echo 0 > /sys/devices/system/cpu/cpu2/online
sleep 0.5
echo 1 > /sys/devices/system/cpu/cpu2/online
sleep 0.5

# Collect results
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace > /tmp/hotplug_trace.txt
echo "Saved to /tmp/hotplug_trace.txt"
```

**4. اختبار الـ hotplug failure injection:**

```bash
# اعمل fail عند state 168 (CPUHP_AP_ONLINE مثلاً)
STATE=168
echo $STATE > /sys/kernel/debug/cpu/hotplug/fail

# حاول online CPU
echo 1 > /sys/devices/system/cpu/cpu1/online

# شوف الـ error
dmesg | tail -20

# ارجع للطبيعي
echo -1 > /sys/kernel/debug/cpu/hotplug/fail
```

**مثال على output ناجح لـ hotplug trace:**

```
kworker/0:1-12  [000] d... cpuhp_enter: cpu: 0001 target: 189 step: CPUHP_AP_ONLINE_IDLE (196)
kworker/0:1-12  [000] d... cpuhp_exit:  cpu: 0001 state: 196 step: CPUHP_AP_ONLINE_IDLE ret: 0
kworker/0:1-12  [000] d... cpuhp_enter: cpu: 0001 target: 189 step: CPUHP_AP_SMPBOOT_THREADS (200)
kworker/0:1-12  [000] d... cpuhp_exit:  cpu: 0001 state: 200 step: CPUHP_AP_SMPBOOT_THREADS ret: 0
```

**كل `ret: 0`** = نجاح. أي قيمة تانية = error code من الـ callback.

**5. فحص الـ microcode version وحالة الـ mitigations:**

```bash
# microcode version
grep microcode /proc/cpuinfo | head -1

# أو عبر MSR
modprobe msr
rdmsr -a 0x8b  # يطبع لكل CPU

# مقارنة شاملة للـ mitigations
for v in /sys/devices/system/cpu/vulnerabilities/*; do
    printf "%-40s: %s\n" "$(basename $v)" "$(cat $v)"
done
```

**مثال output:**

```
ghostwrite                              : Not affected
indirect_target_selection               : Mitigation: Aligned Branch/Return Thunks
itlb_multihit                           : Not affected
l1tf                                    : Not affected
mds                                     : Not affected
meltdown                                : Not affected
mmio_stale_data                         : Not affected
reg_file_data_sampling                  : Not affected
retbleed                                : Not affected
spec_rstack_overflow                    : Not affected
spec_store_bypass                       : Mitigation: Speculative Store Bypass disabled via prctl
spectre_v1                              : Mitigation: usercopy/swapgs barriers and __user pointer sanitization
spectre_v2                              : Mitigation: Enhanced / Automatic IBRS; IBPB: conditional; RSB filling; PBRSB-eIBRS: SW sequence; BHI: BHI_DIS_S
srbds                                   : Not affected
tsx_async_abort                         : Not affected
```

**6. تشخيص مشكلة SMT:**

```bash
# حالة SMT الحالية
cat /sys/devices/system/cpu/smt/control
# on / off / forceoff / notsupported / notimplemented

# عدد الـ threads
cat /sys/devices/system/cpu/smt/active

# عطّل SMT وتحقق
echo off > /sys/devices/system/cpu/smt/control
cat /sys/devices/system/cpu/online  # المفروض يقل لنص العدد

# أعد التفعيل
echo on > /sys/devices/system/cpu/smt/control
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — CPU Hotplug بيفشل عند Suspend/Resume

#### العنوان
**الـ Secondary CPU مش بيرجع بعد الـ Suspend في Gateway صناعي**

#### السياق
شركة بتبني industrial IoT gateway بالـ RK3562 (quad-core Cortex-A53) شغّالة على kernel مخصص. المنتج محتاج يدخل deep sleep كل 30 ثانية لتوفير الطاقة، ويصحى لما تيجي packet على UART أو I2C من الـ sensors.

#### المشكلة
بعد أول suspend/resume ناجح، الـ CPU2 و CPU3 بيظهروا offline في `/sys/devices/system/cpu/` ومش بيجوا online تاني. الـ UART driver بيشتكي من latency عالي لأنه بيشتغل على CPU0 بس.

#### التحليل
الـ `cpu.h` بيعرّف:

```c
#ifdef CONFIG_PM_SLEEP_SMP
extern int freeze_secondary_cpus(int primary);
extern void thaw_secondary_cpus(void);

static inline int suspend_disable_secondary_cpus(void)
{
    int cpu = 0;
    if (IS_ENABLED(CONFIG_PM_SLEEP_SMP_NONZERO_CPU))
        cpu = -1;
    return freeze_secondary_cpus(cpu); /* بيفريز كل CPU ما عدا primary */
}

static inline void suspend_enable_secondary_cpus(void)
{
    thaw_secondary_cpus(); /* المفروض يرجعهم */
}
```

الـ `thaw_secondary_cpus()` بتمر على كل CPU وبتعمل `add_cpu()` — اللي هو:

```c
int add_cpu(unsigned int cpu); /* معرّف في cpu.h تحت CONFIG_SMP */
```

الـ `add_cpu()` بتستدعي state machine الـ hotplug وبتمشي من `CPUHP_OFFLINE` لـ `CPUHP_ONLINE`. المشكلة: الـ RK3562 BSP فيه bug في `arch_cpu_idle_dead()` — اللي بيتوقع الـ CPU يكون في state معيّن قبل ما `bringup_hibernate_cpu()` تشتغل.

```c
int bringup_hibernate_cpu(unsigned int sleep_cpu);
/* بتفترض الـ CPU كان hibernate مش offline كامل */
```

لما الـ CPU بيوصل لـ `CPU_BROKEN`:
```c
#define CPU_BROKEN  0x000B /* CPU did not die properly */
```

الـ kernel بيعتبره broken وبيرفض يعمله online تاني.

#### الحل

```bash
# تأكد من الحالة
cat /sys/devices/system/cpu/cpu2/online
# output: 0

dmesg | grep -i "cpu2\|hotplug\|broken"
# هتلاقي: "CPU2 is now offline" أو "CPU2: failed to come online"

# اختبار manual bringup
echo 1 > /sys/devices/system/cpu/cpu2/online
cat /sys/devices/system/cpu/cpu2/online
```

الحل في الـ BSP: تأكد إن `arch_cpu_idle_dead()` بتعمل `cpuhp_report_idle_dead()` صح:

```c
/* في arch/arm64/kernel/smp.c للـ RK3562 BSP */
void arch_cpu_idle_dead(void)
{
    /* لازم تتأكد إن الـ CPU يسجل نفسه قبل ما يموت */
    cpuhp_report_idle_dead(); /* معرّف في cpu.h */
    cpu_park_loop();
    /* هنا CPU بيتوقف */
}
```

```c
/* cpu.h */
#ifdef CONFIG_HOTPLUG_CPU
void cpuhp_report_idle_dead(void); /* critical: لازم يتستدعى قبل park */
#endif
```

#### الدرس المستفاد
لما `CPU_BROKEN` يظهر، المشكلة دايما في الـ teardown path مش الـ bringup. `cpuhp_report_idle_dead()` في `cpu.h` لازم تتستدعى من `arch_cpu_idle_dead()` قبل ما الـ CPU يدخل park loop — غيرها الـ kernel مش هيعرف إن الـ CPU مات بشكل نظيف.

---

### السيناريو 2: Android TV Box على Allwinner H616 — SMT Mitigation بيأثر على الـ Performance

#### العنوان
**الـ Video Decode بيتقطع بسبب SMT Mitigations غلط Configuration**

#### السياق
منتج Android TV box بالـ Allwinner H616 (quad-core Cortex-A53). الـ SoC ده مفيش فيه SMT أصلاً (كل core بيكون thread واحد). المطوّر شغّل kernel config قديم من machine تانية وبيلاقي إن الـ VP9 decode بيتقطع والـ CPU utilization عالي بشكل غير طبيعي.

#### المشكلة
الـ kernel boot arguments فيها `mitigations=auto` وده بيخلي بعض الـ mitigations تشتغل على سيستم ما معهوش ثغرات SMT أصلاً.

#### التحليل
الـ `cpu.h` بيعرّف:

```c
enum smt_mitigations {
    SMT_MITIGATIONS_OFF,   /* disable كل SMT mitigations */
    SMT_MITIGATIONS_AUTO,  /* kernel يقرر حسب الـ CPU */
    SMT_MITIGATIONS_ON,    /* force enable */
};

#ifdef CONFIG_CPU_MITIGATIONS
extern bool cpu_mitigations_off(void);
extern bool cpu_mitigations_auto_nosmt(void); /* auto لكن بيعطل SMT */
extern bool cpu_attack_vector_mitigated(enum cpu_attack_vectors v);
extern enum smt_mitigations smt_mitigations;
```

وفي نفس الوقت:

```c
enum cpu_attack_vectors {
    CPU_MITIGATE_USER_KERNEL,  /* spectre v1, v2 */
    CPU_MITIGATE_USER_USER,    /* mds, taa */
    CPU_MITIGATE_GUEST_HOST,   /* L1TF */
    CPU_MITIGATE_GUEST_GUEST,
    NR_CPU_ATTACK_VECTORS,
};
```

الـ H616 مفيش فيه SMT — بس الـ `cpu_mitigations_auto_nosmt()` بترجع `true` في بعض الـ config لأن الـ generic path بتشوف إن `cpu_smt_control` مش `CPU_SMT_NOT_IMPLEMENTED`:

```c
/* من cpu_smt.h */
enum cpuhp_smt_control {
    CPU_SMT_ENABLED,
    CPU_SMT_DISABLED,
    CPU_SMT_FORCE_DISABLED,
    CPU_SMT_NOT_SUPPORTED,    /* السيستم ما بيدعمش SMT */
    CPU_SMT_NOT_IMPLEMENTED,  /* الـ Kconfig مش enabled */
};
```

لو الـ BSP ما عمّلش `cpu_smt_control = CPU_SMT_NOT_SUPPORTED` صح، الـ kernel ممكن يطبق mitigations زيادة عن اللازم وده بيأثر على throughput.

#### الحل

```bash
# تحقق من الحالة الحالية
cat /sys/devices/system/cpu/smt/control
# المفروض: notimplemented

# شوف الـ vulnerabilities
cat /sys/devices/system/cpu/vulnerabilities/*

# لو شايف "Mitigation: ..." على CPU مفيش فيه SMT:
grep "smt" /proc/cpuinfo
```

في الـ kernel cmdline:
```bash
# في /boot/extlinux/extlinux.conf أو DT bootargs
APPEND ... mitigations=off
# أو بشكل أدق:
APPEND ... nosmt=force spectre_v2=off l1tf=off
```

أو على مستوى الـ BSP:
```c
/* في arch/arm64/kernel/cpufeature.c للـ H616 */
static void __init cpu_smt_init(void)
{
    /* H616 ما فيهوش hyperthreading */
    cpu_smt_disable(false);
    /* ده بيخلي cpu_smt_control = CPU_SMT_DISABLED */
}
```

#### الدرس المستفاد
على SoCs زي Allwinner H616 اللي ما فيهاش SMT، لازم `cpu_smt_control` يتضبط على `CPU_SMT_NOT_SUPPORTED` أو `CPU_SMT_NOT_IMPLEMENTED` — ده بيخلي `cpu_mitigations_auto_nosmt()` ترجع `false` ويوفر CPU cycles ثمينة للـ media pipeline.

---

### السيناريو 3: Custom Board Bring-up على STM32MP1 — `register_cpu()` بيفشل بـ ENODEV

#### العنوان
**الـ CPU ما بيظهرش في sysfs أثناء Board Bring-up**

#### السياق
مهندس بيشتغل على custom board بالـ STM32MP1 (dual Cortex-A7 + Cortex-M4). الـ bring-up جديد ومفيش BSP vendor. الـ kernel بيبوت بس `/sys/devices/system/cpu/cpu1/` مش موجود، وده بيخلي الـ userspace tools زي `taskset` تفشل.

#### المشكلة
الـ `cpu1` مش متسجّل في sysfs رغم إنه شغّال فعلاً (الـ SMP بيشتغل والـ cpu1 بياخد interrupts).

#### التحليل
الـ `cpu.h` بيعرّف:

```c
struct cpu {
    int node_id;       /* رقم الـ NUMA node */
    int hotpluggable;  /* لو 1، بيعمل sysfs control file */
    struct device dev; /* الـ device embedded في الـ struct */
};

extern int register_cpu(struct cpu *cpu, int num);
/* بتسجل الـ CPU في sysfs تحت /sys/devices/system/cpu/cpuN */
```

و:
```c
#ifdef CONFIG_GENERIC_CPU_DEVICES
DECLARE_PER_CPU(struct cpu, cpu_devices);
/* per-CPU variable — كل CPU عنده struct cpu خاص بيه */
#endif
```

الـ flow المفروض يكون:
```
arch_register_cpu(1)
  → register_cpu(&cpu_devices[1], 1)
    → device_register(&cpu->dev)
      → يظهر في /sys/devices/system/cpu/cpu1/
```

المشكلة: الـ custom board ما عندهاش `arch_register_cpu()` implementation للـ STM32MP1 port، والـ generic fallback:

```c
extern int arch_register_cpu(int cpu);
/* لازم الـ arch يعرّفها — لو مش موجودة الـ linker بيشتكي */
```

بس في حالتنا، الـ BSP عرّف stub فارغة بترجع `-ENODEV` بدل ما تستدعي `register_cpu()`.

#### الحل

```bash
# تأكد من المشكلة
ls /sys/devices/system/cpu/
# cpu0  cpufreq  cpuidle  isolated  kernel_max  modalias  ...
# cpu1 غير موجود!

# تأكد إن الـ cpu1 شغّال فعلاً
cat /proc/cpuinfo | grep processor
# processor: 0
# processor: 1  ← موجود في cpuinfo بس مش في sysfs
```

الحل في الـ BSP:
```c
/* في arch/arm/mach-stm32/platsmp.c */
int __init arch_register_cpu(int num)
{
    struct cpu *c = &per_cpu(cpu_devices, num);

    /*
     * STM32MP1: cpu1 مش hotpluggable في hardware level
     * بس لازم نسجله في sysfs
     */
    c->hotpluggable = 0; /* مش hotpluggable */
    c->node_id = 0;      /* single NUMA node */

    return register_cpu(c, num); /* من cpu.h */
}
```

بعد الـ fix:
```bash
ls /sys/devices/system/cpu/cpu1/
# cache  cpufreq  online  topology  uevent
```

#### الدرس المستفاد
`register_cpu()` و `arch_register_cpu()` في `cpu.h` هما entry point لتسجيل الـ CPU في sysfs. أثناء الـ bring-up، حتى لو الـ CPU شغّال فعلاً، لازم `arch_register_cpu()` تستدعي `register_cpu()` صح — وإلا الـ userspace كله أعمى للـ CPU ده.

---

### السيناريو 4: Automotive ECU على i.MX8 — `cpu_show_spectre_v2` بيكشف معلومات حساسة في الـ Production Image

#### العنوان
**الـ Security Audit بيلاقي إن Vulnerability Info متاحة لأي User في الـ ECU**

#### السياق
شركة automotive بتبني ECU للسيارات بالـ i.MX8QM (quad Cortex-A72). الـ ECU بيشتغل Linux مع multi-user environment — بعض الـ processes بتشتغل بـ user غير privileged. الـ security audit بيلاقي إن أي user ممكن يقرأ detailed vulnerability info عن الـ CPU.

#### المشكلة
الـ sysfs files تحت `/sys/devices/system/cpu/vulnerabilities/` متاحة للـ read لأي user — وده بيكشف معلومات عن CPU microarchitecture ممكن تُساعد في هجمات.

#### التحليل
الـ `cpu.h` بيعرّف مجموعة من الـ `cpu_show_*` functions:

```c
/* كل function دي بتعمل sysfs attribute */
extern ssize_t cpu_show_spectre_v2(struct device *dev,
                                   struct device_attribute *attr, char *buf);
extern ssize_t cpu_show_meltdown(struct device *dev,
                                 struct device_attribute *attr, char *buf);
extern ssize_t cpu_show_retbleed(struct device *dev,
                                 struct device_attribute *attr, char *buf);
extern ssize_t cpu_show_spec_rstack_overflow(struct device *dev,
                                             struct device_attribute *attr,
                                             char *buf);
/* ... إلخ */
```

الـ i.MX8QM بيستخدم Cortex-A72 واللي عنده:
- Spectre v2: "Mitigation: CSV2, BHB"
- Meltdown: "Not affected"

الـ output من `/sys/devices/system/cpu/vulnerabilities/spectre_v2` بيدي information دقيقة عن الـ microarchitecture version والـ mitigations المطبّقة — وده بالنسبة للـ attacker information قيّمة.

في `drivers/base/cpu.c`، الـ attributes بتتسجل بـ permissions `0444` (world-readable) بشكل افتراضي.

#### الحل

**خيار 1: Restrict permissions عبر udev rules**
```bash
# في /etc/udev/rules.d/99-cpu-vuln.rules
SUBSYSTEM=="cpu", KERNEL=="cpu[0-9]*", \
    RUN+="/bin/chmod 400 /sys/devices/system/cpu/vulnerabilities/*"
```

**خيار 2: Custom kernel config**
```kconfig
# في .config
CONFIG_CPU_MITIGATIONS=y
# ما تبنيش الـ vulnerability sysfs في production
CONFIG_SYSFS_DEPRECATED=n
```

**خيار 3: LSM policy (SELinux/AppArmor)**
```
# SELinux policy
allow automotive_app sysfs_cpu:file { read };
neverallow untrusted_app sysfs_cpu:file { read };
```

**خيار 4: Kernel patch لتغيير الـ mode**
```c
/* في drivers/base/cpu.c */
static struct attribute *cpu_root_attrs[] = {
    /* بدل 0444 نستخدم 0400 للـ vulnerabilities */
    &dev_attr_vulnerabilities.attr, /* mode = 0400 */
    NULL
};
```

#### الدرس المستفاد
الـ `cpu_show_*` functions في `cpu.h` بتعمل sysfs interface مفتوح بشكل افتراضي. في بيئات embedded production خصوصاً automotive، لازم تراجع الـ sysfs permissions كجزء من الـ security hardening — الـ vulnerability details ممكن تكون information leakage.

---

### السيناريو 5: IoT Sensor Hub على AM62x — `cpu_idle_poll_ctrl` بيعمل Battery Drain

#### العنوان
**الـ Sensor Hub بتسرّب الطاقة بسبب Idle Poll مفعّل بالغلط**

#### السياق
IoT sensor hub بالـ TI AM62x (quad Cortex-A53) — المنتج المفروض يقعد في low-power idle معظم الوقت وبيصحى كل 100ms لو في I2C data من الـ sensors. الـ battery المفروض تدوم 6 شهور، بس في الواقع بتخلص في 3 أسابيع.

#### المشكلة
الـ CPU ما بيدخلش deep idle رغم إنه مش شايل load. الـ power measurement بيوري إن الـ SoC شايل ~200mW بدل الـ 5mW المتوقعة في idle.

#### التحليل
الـ `cpu.h` بيعرّف:

```c
void cpu_idle_poll_ctrl(bool enable);
/* بتفعّل أو بتعطّل busy-wait polling بدل الـ WFI/WFE */

bool cpu_in_idle(unsigned long pc);
/* بتتحقق لو الـ CPU في idle loop */

void arch_cpu_idle(void);          /* WFI/WFE فعلي */
void arch_cpu_idle_prepare(void);  /* استعداد قبل الـ idle */
void arch_cpu_idle_enter(void);    /* دخول الـ idle */
void arch_cpu_idle_exit(void);     /* خروج من الـ idle */
```

وكمان:
```c
void play_idle_precise(u64 duration_ns, u64 latency_ns);
/* بتعمل idle لمدة محددة بدقة — للـ real-time scenarios */
```

بعد التحقيق، الـ driver المسؤول عن قراءة الـ I2C sensors كان بيستخدم:
```c
/* في sensor_hub_driver.c — كود غلط */
static int sensor_hub_probe(struct platform_device *pdev)
{
    /* المطوّر فكر إن ده بيسرّع الـ response time */
    cpu_idle_poll_ctrl(true); /* WRONG: بيعمل busy-wait poll */
    ...
}
```

`cpu_idle_poll_ctrl(true)` بتعمل الـ kernel يعمل **polling** في idle loop بدل ما ينام بـ `WFI` (Wait For Interrupt). النتيجة: الـ CPU بيفضل يشتغل على 100% حتى وهو "idle".

#### الحل

```bash
# تأكد من المشكلة
cat /sys/kernel/debug/cpuidle/cpu0/state0/usage
# usage عالي جداً في poll state

powertop --time=10
# هتلاقي: C0 (poll) 99% بدل C2/C3

# شوف الـ idle states
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name
# poll
# WFI
# retention  ← المفروض يكون هنا معظم الوقت
```

الحل في الـ driver:
```c
/* في sensor_hub_driver.c — الكود الصح */
static int sensor_hub_probe(struct platform_device *pdev)
{
    /*
     * ما نستخدمش cpu_idle_poll_ctrl(true) هنا.
     * الـ kernel بيختار الـ idle state المناسب أوتوماتيك.
     * لو محتاجين latency منخفض نستخدم PM QoS بدل poll.
     */
    pm_qos_add_request(&hub->pm_qos_req,
                       PM_QOS_CPU_LATENCY,
                       100); /* 100µs max latency */
    ...
}

static void sensor_hub_remove(struct platform_device *pdev)
{
    pm_qos_remove_request(&hub->pm_qos_req);
}
```

بعد الـ fix:
```bash
powertop --time=10
# C2 (WFI)      45%
# C3 (retention) 50%
# C0 (active)    5%
# Power: ~8mW ✓
```

#### الدرس المستفاد
`cpu_idle_poll_ctrl(true)` في `cpu.h` مصمّمة لـ use cases محدودة جداً زي الـ debugging أو الـ real-time latency measurements. في production embedded systems، استخدامها بيمنع الـ CPU من دخول deep idle states تماماً. الحل الصح هو PM QoS لتحديد الـ latency requirements وترك الـ kernel يختار الـ idle state المناسب عبر `arch_cpu_idle()`.
## Phase 7: مصادر ومراجع

### توثيق Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| [`Documentation/core-api/cpu_hotplug.rst`](https://docs.kernel.org/core-api/cpu_hotplug.html) | الـ reference الرئيسي لـ CPU hotplug في kernel الحديث — بيشرح الـ state machine وطريقة استخدام `cpuhp_setup_state()` |
| [`include/linux/cpu.h`](https://github.com/torvalds/linux/blob/master/include/linux/cpu.h) | الـ header الرئيسي اللي اتكلمنا عنه — بيعرف `struct cpu` والـ APIs الأساسية |
| [`include/linux/cpuhotplug.h`](https://github.com/torvalds/linux/blob/master/include/linux/cpuhotplug.h) | بيحدد الـ `enum cpuhp_state` وكل states الـ hotplug |
| [`drivers/base/cpu.c`](https://github.com/torvalds/linux/blob/master/drivers/base/cpu.c) | الـ implementation الأساسي لـ `register_cpu()` وتسجيل الـ sysfs entries |
| [`kernel/cpu.c`](https://github.com/torvalds/linux/blob/master/kernel/cpu.c) | قلب الـ hotplug logic — `add_cpu()`, `remove_cpu()`, الـ state machine |
| [`Documentation/arch/x86/x86_64/cpu-hotplug-spec.rst`](https://docs.kernel.org/6.12/arch/x86/x86_64/cpu-hotplug-spec.html) | مواصفات ACPI و firmware support للـ CPU hotplug على x86-64 |

---

### مقالات LWN.net

دي أهم المقالات على LWN.net اللي بتغطي تطور CPU hotplug في الـ kernel:

#### الـ Core Articles

**[Rationalizing CPU hotplugging — LWN.net](https://lwn.net/Articles/537562/)**
الـ article ده بيتكلم عن المشاكل التاريخية في الـ notifier-based API القديم وليه كان محتاج rework. بيتكلم عن kernel 2.6.16 اللي اتضاف فيه الـ Documentation الأصلي سنة 2006.

**[Documentation/cpu-hotplug.txt — LWN.net](https://lwn.net/Articles/537570/)**
النسخة الأصلية من الـ documentation اللي اتضافت مع أول دعم للـ hotplug — مفيدة لفهم الـ history.

**[cpu/hotplug: Core infrastructure for cpu hotplug rework — LWN.net](https://lwn.net/Articles/677686/)**
الـ patch series بتاع Thomas Gleixner سنة 2016 اللي عمل الـ rework الكامل — حذف ~4000 سطر كود وعمل الـ state machine الحديث. ده المقال الأهم لفهم الـ design الحالي.

**[CPU hotplug rework - episode I — LWN.net](https://lwn.net/Articles/535764/)**
أول episode في سلسلة الـ rework — بيشرح الـ motivation والـ plan.

**[Optimizing CPU hotplug locking — LWN.net](https://lwn.net/Articles/569686/)**
بيتكلم عن تحسينات الـ locking في الـ hotplug path وإزاي الـ semaphore-based approach اكتشف ~25 deadlock كانوا مخبيين في الكود القديم.

**[crash: Kernel handling of CPU and memory hot un/plug — LWN.net](https://lwn.net/Articles/930858/)**
مقال حديث بيتكلم عن إزاي الـ kernel بيتعامل مع failure scenarios في الـ CPU hotplug.

**[Documentation for CPU hotplug support — LWN.net](https://lwn.net/Articles/159561/)**
الـ documentation الأصلي اللي اتضاف للـ kernel من زمان — مرجع تاريخي مهم.

#### الـ Official Kernel Docs على LWN Mirror

**[CPU hotplug in the Kernel — Linux Kernel Documentation](https://static.lwn.net/kerneldoc/core-api/cpu_hotplug.html)**
الـ mirror بتاع LWN للـ official kernel docs — بيشرح الـ three sections: PREPARE, STARTING, ONLINE.

---

### مناقشات Mailing List المهمة

| الرابط | الموضوع |
|--------|---------|
| [lkml.org/lkml/2016/2/26/806](https://lkml.org/lkml/2016/2/26/806) | Thomas Gleixner's patch 00/20 — الـ core hotplug rework الأصلي |
| [lore.kernel.org — parallel CPU bringup](https://lore.kernel.org/lkml/20230414225551.858160935@linutronix.de/) | سلسلة 37 patch للـ parallel CPU bringup من Thomas Gleixner 2023 |
| [lore.kernel.org — remove obsolete functions](https://lore.kernel.org/lkml/20161221192112.005642358@linutronix.de/) | إزالة الـ obsolete hotplug register/unregister functions |
| [lkml.iu.edu — cpuhp_setup_state return value](https://lkml.iu.edu/hypermail/linux/kernel/1612.1/04848.html) | توضيح الـ return value بتاع `__cpuhp_setup_state()` |

---

### Kernelnewbies.org

**[Linux_2_6_13 — CPU hotplug على i386](https://kernelnewbies.org/Linux_2_6_13)**
أول دعم لـ CPU hotplug على معمارية i386 — kernel 2.6.13.

**[Logical vs Physical CPU hotplug](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2011-August/002951.html)**
شرح الفرق بين:
- `CONFIG_HOTPLUG_CPU` → logical online/offline (software)
- `CONFIG_ACPI_HOTPLUG_CPU` → physical add/remove (hardware)

**[Linux_4.6 — PowerPC hotplug](https://kernelnewbies.org/Linux_4.6)**
إضافة hotplug support لـ E6500 PowerPC.

**[LinuxChanges](https://kernelnewbies.org/LinuxChanges)**
صفحة التغييرات الشاملة — ابحث فيها عن "hotplug" لتتبع التطور عبر الـ releases.

---

### Kernel Commits المهمة

```bash
# الـ commit الأساسي للـ hotplug rework (Thomas Gleixner, 2016)
git log --oneline --all | grep -i "hotplug.*rework\|rework.*hotplug"

# الـ commit اللي أضاف cpuhp_setup_state
git log --oneline --all --grep="cpuhp_setup_state"

# تاريخ cpu.h
git log --oneline include/linux/cpu.h

# الـ commit اللي أضاف parallel CPU bringup
git log --oneline --all --grep="parallel.*bringup\|bringup.*parallel"
```

| الـ Commit | الوصف |
|-----------|-------|
| `512f09801b` | توضيح الـ return value بتاع `__cpuhp_setup_state()` — Boris Ostrovsky |
| `2016 rework series` | Thomas Gleixner's 20-patch series لإعادة كتابة الـ hotplug core |
| `2023 parallel bringup` | سلسلة 37 patch لـ parallel CPU startup — تسريع boot time |

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)

- **الفصل 14**: Linux Device Model — بيشرح `struct device`, `bus_type`, وإزاي CPU بيتسجل كـ device
- **الفصل 3**: Char Drivers — أساسيات الـ sysfs attributes اللي بتظهر في `/sys/devices/system/cpu/`
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)

- **الفصل 7**: Interrupts and Interrupt Handlers — مهم لفهم الـ `notify_cpu_starting()`
- **الفصل 9**: An Introduction to Kernel Synchronization — الـ locking في hotplug paths
- **الفصل 17**: Devices and Modules — الـ device model اللي `struct cpu` بتبني عليه

#### Professional Linux Kernel Architecture — Wolfgang Mauerer

- **الفصل 4**: Task Management — فهم الـ CPU affinity وعلاقته بالـ hotplug
- **الفصل 7**: SMP — الـ symmetric multiprocessing وإزاي الـ hotplug بيتكامل معاه

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- **الفصل 15**: Debugging Embedded Linux — بيتكلم عن الـ CPU power states في الـ embedded context
- مفيد لفهم `freeze_secondary_cpus()` و `thaw_secondary_cpus()` في الـ suspend/resume flow

---

### مسارات البحث للمزيد

#### على Google/DuckDuckGo

```
"cpuhp_setup_state" site:lore.kernel.org
"cpu_hotplug" site:docs.kernel.org
linux kernel "struct cpu" hotpluggable sysfs
"boot_cpu_init" linux kernel explanation
"cpu_maps_update_begin" linux kernel locking
"freeze_secondary_cpus" suspend SMP linux
"cpuhp_report_idle_dead" linux kernel
linux kernel CPU mitigations sysfs attributes
"cpu_show_spectre" linux kernel sysfs
```

#### على lore.kernel.org

```
https://lore.kernel.org/linux-kernel/?q=cpu+hotplug
https://lore.kernel.org/linux-kernel/?q=cpuhp_setup_state
```

#### على Elixir Cross-Referencer

```
https://elixir.bootlin.com/linux/latest/source/include/linux/cpu.h
https://elixir.bootlin.com/linux/latest/A/ident/register_cpu
https://elixir.bootlin.com/linux/latest/A/ident/cpuhp_setup_state
```

---

### ملخص المصادر حسب الأولوية

| الأولوية | المصدر | ليه مهم |
|---------|--------|---------|
| ⭐⭐⭐ | [docs.kernel.org/core-api/cpu_hotplug.html](https://docs.kernel.org/core-api/cpu_hotplug.html) | الـ official reference الأكثر اكتمالاً |
| ⭐⭐⭐ | [LWN: hotplug rework](https://lwn.net/Articles/677686/) | فهم الـ design الحديث |
| ⭐⭐⭐ | [LWN: Rationalizing hotplugging](https://lwn.net/Articles/537562/) | فهم الـ history والـ motivation |
| ⭐⭐ | [github: include/linux/cpuhotplug.h](https://github.com/torvalds/linux/blob/master/include/linux/cpuhotplug.h) | الـ state machine كامل |
| ⭐⭐ | [LWN: Optimizing hotplug locking](https://lwn.net/Articles/569686/) | فهم الـ locking model |
| ⭐⭐ | [kernel x86-64 hotplug spec](https://docs.kernel.org/6.12/arch/x86/x86_64/cpu-hotplug-spec.html) | firmware/ACPI integration |
| ⭐ | [kernelnewbies: logical vs physical](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2011-August/002951.html) | فهم الفرق بين logical وphysical hotplug |
| ⭐ | [lkml: parallel bringup 2023](https://lore.kernel.org/lkml/20230414225551.858160935@linutronix.de/) | أحدث تطوير في الـ bringup path |
## Phase 8: Writing simple module

### الفكرة

**الـ** `cpuhp_setup_state()` هي من أفضل نقاط الـ hook في `cpu.h` — بتخليك تسجّل callback اتنين: واحد لما CPU بييجي online، وواحد لما بيتشال. ده بيحصل في كل hotplug event حقيقي (زي `echo 0 > /sys/devices/system/cpu/cpu3/online`).

بدل kprobe، هنستخدم الـ API نفسه اللي الـ kernel بيوفره: `cpuhp_setup_state_nocalls` — بنسجّل state ديناميكي (`CPUHP_AP_ONLINE_DYN`) من غير ما نستدعي الـ startup على الـ CPUs الشغالة حاليًا، وفي الـ exit بنعمل `cpuhp_remove_state`.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * cpu_hotplug_monitor.c
 *
 * Registers a dynamic CPU hotplug state to observe CPU online/offline events.
 * Uses cpuhp_setup_state_nocalls() from include/linux/cpuhotplug.h.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit          */
#include <linux/kernel.h>      /* pr_info()                                  */
#include <linux/cpu.h>         /* cpu_is_hotpluggable(), struct cpu           */
#include <linux/cpuhotplug.h>  /* cpuhp_setup_state_nocalls, cpuhp_remove_state */
#include <linux/smp.h>         /* num_online_cpus()                           */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDoc");
MODULE_DESCRIPTION("Monitor CPU hotplug online/offline events via cpuhp state");

/* Holds the allocated dynamic hotplug state ID returned by setup */
static enum cpuhp_state hp_state;

/*
 * cpu_coming_online - called on the target CPU just before it goes fully online
 * @cpu: the CPU number being brought up
 *
 * Runs in the context of the hotplug thread on the control CPU (ONLINE section).
 * Interrupts and preemption are enabled here.
 */
static int cpu_coming_online(unsigned int cpu)
{
    /* cpu_is_hotpluggable: check if this CPU has a sysfs remove file */
    pr_info("cpu_monitor: CPU%u coming ONLINE | hotpluggable=%d | online_count_before=%u\n",
            cpu,
            cpu_is_hotpluggable(cpu),
            num_online_cpus());  /* still does not include this CPU yet */
    return 0;  /* non-zero would abort the bringup */
}

/*
 * cpu_going_offline - called on the target CPU as it is being taken offline
 * @cpu: the CPU number being removed
 *
 * Also runs from the hotplug thread on the control CPU (reverse teardown order).
 */
static int cpu_going_offline(unsigned int cpu)
{
    pr_info("cpu_monitor: CPU%u going OFFLINE | hotpluggable=%d | online_count_before=%u\n",
            cpu,
            cpu_is_hotpluggable(cpu),
            num_online_cpus());  /* still includes this CPU */
    return 0;
}

static int __init cpu_monitor_init(void)
{
    int ret;

    /*
     * cpuhp_setup_state_nocalls: register startup+teardown callbacks for a
     * dynamic state in the ONLINE section (CPUHP_AP_ONLINE_DYN).
     * "nocalls" means do NOT invoke startup on already-online CPUs now —
     * we only want to observe future events.
     */
    ret = cpuhp_setup_state_nocalls(CPUHP_AP_ONLINE_DYN,
                                    "cpu_monitor:online",   /* name in debugfs */
                                    cpu_coming_online,      /* startup  */
                                    cpu_going_offline);     /* teardown */
    if (ret < 0) {
        pr_err("cpu_monitor: failed to register hotplug state: %d\n", ret);
        return ret;
    }

    /* On success, ret holds the allocated state enum value */
    hp_state = ret;
    pr_info("cpu_monitor: loaded, watching hotplug events (state=%d)\n", hp_state);
    return 0;
}

static void __exit cpu_monitor_exit(void)
{
    /*
     * cpuhp_remove_state: unregister the state AND invoke teardown callbacks
     * on all currently online CPUs so nothing is left dangling.
     * Using the saved hp_state ID we got from setup.
     */
    cpuhp_remove_state(hp_state);
    pr_info("cpu_monitor: unloaded\n");
}

module_init(cpu_monitor_init);
module_exit(cpu_monitor_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | ماكروهات الـ module الأساسية: `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info()` و `pr_err()` للطباعة في kernel log |
| `linux/cpu.h` | `cpu_is_hotpluggable()` عشان نجيب معلومة مفيدة عن الـ CPU |
| `linux/cpuhotplug.h` | تعريفات `cpuhp_state`, `cpuhp_setup_state_nocalls`, `cpuhp_remove_state` |
| `linux/smp.h` | `num_online_cpus()` عشان نطبع عدد الـ CPUs الشغالة وقت الحدث |

---

#### الـ `hp_state` المتغير

**الـ** `cpuhp_setup_state_nocalls()` بترجع قيمة موجبة هي الـ state ID الديناميكي اللي اتخصص للـ module ده. لازم نحتفظ بيها عشان نستخدمها في الـ `cpuhp_remove_state()` وقت الـ unload — من غيرها مش هيعرف يشيل الـ callbacks الصح.

---

#### الـ `cpu_coming_online` callback

الـ callback ده بيتنادى من الـ hotplug thread على الـ control CPU في الـ **ONLINE section** — يعني interrupts و preemption شغالين وممكن تعمل أي حاجة عادية. الـ argument الوحيد هو رقم الـ CPU اللي بييجي online. لو رجعنا قيمة سالبة، الـ kernel هيوقف الـ bringup ويعمل rollback.

---

#### الـ `cpu_going_offline` callback

نفس الـ signature — بس بيتنادى في الـ **reverse teardown order** لما CPU بيتشال. في اللحظة دي الـ CPU لسه محسوب في `num_online_cpus()`، فالقيمة المطبوعة هتكون شاملاه.

---

#### الـ `module_init`

استخدمنا `CPUHP_AP_ONLINE_DYN` كـ state لأنه slot ديناميكي متاح للـ modules — مش محتاجين نضيف entry ثابتة في `enum cpuhp_state`. الـ kernel بيختار أول slot فاضي ويرجع قيمته. الـ `_nocalls` variant مهمة هنا عشان متحتاجش تتنادى على الـ CPUs الشغالة عند التسجيل.

---

#### الـ `module_exit`

`cpuhp_remove_state` (مش `_nocalls`) بيستدعي الـ teardown callback على كل الـ CPUs الشغالة قبل ما يشيل الـ registration — ده يضمن إن مفيش state متعلق. لو استخدمنا `cpuhp_remove_state_nocalls` هيشيل بس التسجيل من غير ما ينادي teardown.

---

### تشغيل وتجربة

```bash
# Build
make -C /lib/modules/$(uname -r)/build M=$PWD modules

# Load
insmod cpu_hotplug_monitor.ko

# Trigger a hotplug event (requires CONFIG_HOTPLUG_CPU=y)
echo 0 > /sys/devices/system/cpu/cpu3/online   # offline
echo 1 > /sys/devices/system/cpu/cpu3/online   # online

# Watch the output
dmesg | grep cpu_monitor
```

**مثال للـ output المتوقع:**

```
cpu_monitor: loaded, watching hotplug events (state=135)
cpu_monitor: CPU3 going OFFLINE | hotpluggable=1 | online_count_before=8
cpu_monitor: CPU3 coming ONLINE | hotpluggable=1 | online_count_before=7
cpu_monitor: unloaded
```

---

### Makefile

```makefile
obj-m += cpu_hotplug_monitor.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
