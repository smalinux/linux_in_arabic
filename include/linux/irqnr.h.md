## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ `irqnr.h` جزء من **IRQ Subsystem** — النظام الجوهري اللي بيدير كل الـ interrupts في الـ Linux kernel. المسؤول عنه: **Thomas Gleixner**، والكود موجود في `kernel/irq/` والـ entry في `MAINTAINERS` اسمه: **IRQ SUBSYSTEM**.

---

### القصة من الأول: إيه هو الـ Interrupt؟

تخيل إنك شغال مطعم وعندك 50 طاولة. مش ممكن تدور على كل طاولة كل ثانية تسأل "محتاج حاجة؟" — ده **polling** وبياكل وقت. البديل إن كل طاولة عندها **زرار** لما يضغطوا عليه، أنت بتيجي على طول. ده هو الـ **interrupt**.

الـ CPU بالظبط كده — بدل ما يسأل كل device "فيه بيانات؟" كل لحظة، الـ device بيبعت **interrupt signal** للـ CPU لما يكون جاهز. الـ CPU بيوقف اللي بيعمله، بيتعامل مع الـ interrupt، وبيرجع.

---

### المشكلة: لو عندك مئات الـ Interrupts؟

السيستم الحديث عنده مئات الـ devices: keyboard، NIC، disk، USB، timer، GPU، إلخ. كل واحد بيولد interrupts بـ **رقم مختلف** — ده الـ **IRQ number** (زي رقم الطاولة في المطعم). الـ kernel محتاج:

1. يعرف **كام interrupt في السيستم؟**
2. يقدر يـ**iterate** على كل الـ interrupts الموجودة.
3. يوصل لـ **descriptor** كل interrupt (المعلومات الكاملة عنه).

---

### دور الـ `irqnr.h`: فهرس الـ Interrupts

الـ `irqnr.h` هو الـ **header** اللي بيوفر:

- **العدد الكلي للـ IRQs** في السيستم (`irq_get_nr_irqs` / `irq_set_nr_irqs`).
- **Lookup function** توصل من رقم IRQ لـ `irq_desc` (الـ descriptor الكامل): `irq_to_desc()`.
- **Iteration macros** تمشي على كل الـ IRQs أو بس الـ active ones.

ببساطة هو بيجاوب: *"عندنا كام interrupt؟ وإزاي أمشي عليهم واحد واحد؟"*

---

### الـ API اللي بيقدمها الملف

```c
// كام interrupt عند السيستم دلوقتي؟
unsigned int irq_get_nr_irqs(void);

// غير العدد (بيتعمل أثناء الـ boot لما الـ HW يُكتشف)
unsigned int irq_set_nr_irqs(unsigned int nr);

// من رقم IRQ → الـ descriptor الكامل (struct irq_desc)
extern struct irq_desc *irq_to_desc(unsigned int irq);

// الـ IRQ التالي اللي allocated بعد offset
unsigned int irq_get_next_irq(unsigned int offset);
```

---

### الـ Iteration Macros: ليه هي مهمة؟

الكرنل مش بيحجز IRQ numbers بشكل متتالي دايمًا — ممكن يكون عنده IRQ 0, 1, 5, 47, 200 وبس. عشان كده الـ macros دي ضرورية:

**`for_each_irq_desc(irq, desc)`** — يمشي على كل الـ IRQs ويجيب الـ `irq_desc` لكل واحد:
```c
// مثال: طباعة كل IRQ موجود
struct irq_desc *desc;
unsigned int irq;

for_each_irq_desc(irq, desc) {
    pr_info("IRQ %u: %s\n", irq, desc->name);
}
```

**`for_each_irq_desc_reverse(irq, desc)`** — نفسه لكن من الآخر (لـ teardown scenarios).

**`for_each_active_irq(irq)`** — بس الـ IRQs اللي فعلاً allocated (يتخطى الفراغات):
```c
// أسرع لأنه بيستخدم irq_get_next_irq() يتجاوز الفراغات
for_each_active_irq(irq) {
    // بس IRQs حقيقية، مش gaps
}
```

**`for_each_irq_nr(irq)`** — بيمشي من 0 للـ nr_irqs بشكل خطي:
```c
for_each_irq_nr(irq) {
    // كل رقم من 0 لـ nr_irqs-1، حتى لو مفيش descriptor
}
```

---

### الـ Implementation: فين الحقيقي؟

الـ `irqnr.h` مجرد declarations — الـ implementation في:

| الملف | الدور |
|---|---|
| `kernel/irq/irqdesc.c` | تعريف `irq_get_nr_irqs`, `irq_set_nr_irqs`, `irq_get_next_irq` |
| `kernel/irq/irqdesc.c` | تعريف `irq_to_desc()` — بيبحث في **maple tree** اسمها `sparse_irqs` |

الـ `sparse_irqs` هي data structure من نوع **maple tree** (شجرة بيانات سريعة) بتخزن كل الـ `irq_desc` objects وبتسمح بـ lookup سريع من رقم IRQ لـ descriptor.

---

### الصورة الكاملة للـ IRQ Subsystem

```
Hardware Device
      |
      | (interrupt signal)
      v
  Interrupt Controller (e.g., GIC على ARM)
      |
      | (IRQ number)
      v
  irq_to_desc(irq) ← irqnr.h يوفر ده
      |
      v
  struct irq_desc  ← irqdesc.h يعرفها
      |
      | (handle_irq)
      v
  irq_handler_t action ← كود الـ driver
```

---

### الملفات المرتبطة اللي المفروض تعرفها

| الملف | الدور |
|---|---|
| `include/linux/irqnr.h` | **الملف ده** — العدد والـ iteration |
| `include/linux/irqdesc.h` | تعريف `struct irq_desc` — كل معلومات الـ IRQ |
| `include/linux/irq.h` | الـ API الكامل للـ IRQ subsystem |
| `include/linux/irqhandler.h` | تعريف `irq_handler_t` (نوع الـ handler) |
| `include/linux/irqreturn.h` | تعريف `IRQ_HANDLED` / `IRQ_NONE` |
| `include/linux/irqdomain.h` | الـ mapping بين HW IRQ numbers وـ Linux IRQ numbers |
| `kernel/irq/irqdesc.c` | implementation كل functions الـ `irqnr.h` |
| `kernel/irq/manage.c` | `request_irq()`, `free_irq()` |
| `kernel/irq/handle.c` | تنفيذ الـ IRQ handling flow |
| `kernel/irq/chip.c` | واجهة الـ IRQ chip (الـ HW controller) |
| `kernel/irq/proc.c` | `/proc/interrupts` — اللي بتشوفه بـ `cat /proc/interrupts` |
| `drivers/irqchip/` | drivers الـ interrupt controllers الفعلية (GIC, APIC, إلخ) |
## Phase 2: شرح الـ IRQ Numbering & Descriptor Framework

### المشكلة اللي بتحلها

الـ hardware بتوّلد interrupts — الـ UART قال "في بيانات"، الـ GPIO قال "ضغطوا الزرار"، الـ DMA قال "خلصت النقل". الـ CPU عنده خط واحد أو عدة خطوط فيزيائية (IRQ lines) — بس الـ devices المئات.

المشكلة المحورية: **ازاي الـ kernel يعرف لما جه interrupt، يروح لأنهي handler بالظبط؟**

من غير framework منظم، كل architecture كانت بتعمل حلها الخاص: array ثابت، linked list، أو worst case — `if/else` كبيرة. النتيجة: كود غير portable، صعب التوسع، ومش بيعرف يتعامل مع الـ hotplug أو dynamic IRQ allocation.

---

### الحل اللي اختاره الـ kernel

الـ kernel عمل **Generic IRQ Subsystem** — abstraction layer كامل يفصل بين:

- **IRQ number** (رقم منطقي، يستخدمه الـ software)
- **HW IRQ number** (رقم فيزيائي في الـ interrupt controller)
- **`struct irq_desc`** (الـ descriptor — قلب كل شيء)

الملف `irqnr.h` بالذات بيوفر:
1. API عشان تعرف كام IRQ موجود (`irq_get_nr_irqs`)
2. طريقة تحوّل من رقم لـ descriptor (`irq_to_desc`)
3. Iteration macros تمشي على كل الـ IRQs بطريقة آمنة

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Hardware Layer                        │
│  GPIO Controller   UART    SPI    Timer    GIC/APIC      │
│      HW IRQ 5    HW IRQ 2  ...            (HW side)      │
└───────────────────────┬─────────────────────────────────┘
                        │  Physical IRQ signal
                        ▼
┌─────────────────────────────────────────────────────────┐
│              IRQ Domain Layer  (irq_domain)              │
│   Maps: HW IRQ number  ──────►  Linux IRQ number        │
│   (per interrupt controller)                             │
└───────────────────────┬─────────────────────────────────┘
                        │  Linux virq (e.g., IRQ 42)
                        ▼
┌─────────────────────────────────────────────────────────┐
│         IRQ Descriptor Table  (irq_desc[])               │
│                                                          │
│  irq 0: struct irq_desc { handle_irq, action, chip... } │
│  irq 1: struct irq_desc { ... }                          │
│  irq N: struct irq_desc { ... }                          │
│                                                          │
│  irq_to_desc(irq) ──► pointer to irq_desc               │
│  irq_get_nr_irqs() ──► total count                       │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│              High-level Flow Handler                     │
│  handle_level_irq / handle_edge_irq / handle_fasteoi_irq│
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│              IRQ Action Chain (struct irqaction)         │
│   driver 1 handler ──► driver 2 handler ──► NULL        │
│   (request_irq registered these)                        │
└─────────────────────────────────────────────────────────┘
```

---

### تشريح الـ struct irq_desc

ده الـ central object اللي `irq_to_desc()` بترجعه:

```c
struct irq_desc {
    struct irq_common_data  irq_common_data;  /* shared data بين chip وdriver */
    struct irq_data         irq_data;         /* chip-specific data + HW irq number */
    struct irqstat __percpu *kstat_irqs;      /* per-CPU interrupt counter */
    irq_flow_handler_t      handle_irq;       /* high-level flow handler */
    struct irqaction        *action;          /* linked list of registered handlers */
    unsigned int            depth;            /* كام مرة اتعمل disable (nested) */
    raw_spinlock_t          lock;             /* SMP protection */
    /* ... SMP affinity, threading, PM fields ... */
} ____cacheline_internodealigned_in_smp;
```

العلاقة بين الـ structs:

```
struct irq_desc
├── irq_common_data ──► handler_data (driver private)
│                  └──► affinity (cpumask)
├── irq_data ──────► irq  (Linux IRQ number)
│              ├──► hwirq (HW IRQ number)
│              ├──► chip  ──► struct irq_chip (mask/unmask/ack...)
│              └──► domain ──► struct irq_domain (HW→SW mapping)
├── handle_irq ────► handle_level_irq() أو handle_edge_irq() إلخ
└── action ────────► struct irqaction
                         ├── handler ──► driver's ISR
                         ├── name
                         ├── dev_id
                         └── next ──► struct irqaction (shared IRQ)
```

---

### SPARSE_IRQ vs Static Array

الـ kernel بيدعم طريقتين لتخزين الـ descriptors:

| الوضع | الآلية | متى يُستخدم |
|-------|--------|-------------|
| `!CONFIG_SPARSE_IRQ` | `irq_desc[NR_IRQS]` — array ثابت في الـ BSS | embedded بموارد محدودة، عدد IRQs معروف |
| `CONFIG_SPARSE_IRQ` | `radix tree` أو `xarray` — dynamic allocation | servers، hotplug، عدد IRQs كبير/متغير |

الـ `irq_to_desc(irq)` بتتعامل مع الحالتين — الـ caller مش محتاج يعرف الفرق.

---

### الـ Iteration Macros في `irqnr.h`

أربع macros، كل واحدة لها غرض مختلف:

#### 1. `for_each_irq_desc(irq, desc)`
بتمشي على **كل الـ IRQs** وبترجع الـ `irq_desc` لكل واحد. بتتخطى الـ `NULL` entries (IRQs مش allocated).

```c
/* مثال: طباعة كل الـ IRQs المسجّلة */
unsigned int irq;
struct irq_desc *desc;

for_each_irq_desc(irq, desc) {
    if (irq_desc_has_action(desc))
        pr_info("IRQ %u: %s\n", irq, desc->name ?: "unnamed");
}
```

**التقنية الداخلية:** بتستخدم double `for` loop — الأولى بتجيب `nr_irqs` مرة واحدة وبتخزنها، التانية بتعمل iteration. ده بيحمي من race condition لو `nr_irqs` اتغير أثناء الـ loop.

#### 2. `for_each_irq_desc_reverse(irq, desc)`
نفس الأولى لكن من آخر IRQ لأوله. مفيد في teardown أو priority scanning.

#### 3. `for_each_active_irq(irq)`
بتستخدم `irq_get_next_irq(offset)` — بتتخطى فراغات الـ sparse table مباشرةً بدون ما تمر على كل عنصر. أسرع لما الـ IRQs مش consecutive.

```c
unsigned int irq;
for_each_active_irq(irq) {
    /* irq هنا دايماً active، مفيش NULL check محتاج */
    do_something_with(irq);
}
```

#### 4. `for_each_irq_nr(irq)`
الأبسط — بس يلوب من 0 لـ `nr_irqs`. مش بيجيب الـ `irq_desc`، بس الرقم. مفيد لو أنت هتعمل `irq_to_desc` بنفسك بعدين أو مش محتاجه أصلاً.

---

### تشبيه من الواقع — مبنى إداري ضخم

تخيل شركة كبيرة فيها مبنى متعدد الأدوار:

| مفهوم kernel | التشبيه |
|-------------|---------|
| **HW IRQ number** | رقم الغرفة الفيزيائي على الباب (204B) |
| **Linux IRQ number (virq)** | رقم الموظف في نظام الشركة الداخلي (emp#1042) |
| **`irq_domain`** | جدول تحويل: "غرفة 204B = موظف 1042" |
| **`struct irq_desc`** | ملف الموظف الكامل: مهامه، تاريخه، إيميله، مديره |
| **`irq_to_desc()`** | قسم HR — قوله رقم الموظف، هيجيبلك ملفه |
| **`irq_get_nr_irqs()`** | إجمالي عدد الموظفين في الشركة |
| **`for_each_irq_desc`** | جولة على كل الملفات في HR |
| **`for_each_active_irq`** | جولة بس على الموظفين اللي شغّالين فعلاً (مش الإجازات) |
| **`struct irqaction`** | قائمة مهام الموظف — ممكن يعمل أكتر من وظيفة (shared IRQ) |
| **`handle_irq` (flow handler)** | مدير الموظف — بيقرر إزاي يوزع الشغل |
| **`irq_chip`** | الـ hardware فيزيكيًا — التليفون، الكمبيوتر (الأدوات) |

الإضافة المهمة على التشبيه: لو مبنى فيه 10 موظفين بس مش متتاليين (gaps في الأرقام) — `for_each_active_irq` بتقفز من 1042 لـ 1089 مباشرةً بدون ما تراجع كل رقم في النص. أما `for_each_irq_nr` فبتمشي 1042, 1043, 1044... واحدة واحدة.

---

### الـ Core Abstraction

الفكرة المحورية هي **الفصل التام بين ثلاثة مستويات:**

```
[Driver Level]    request_irq(virq, handler)  ← يتعامل مع Linux IRQ number فقط
      ↕
[Generic IRQ]     irq_desc: flow control, threading, stats, affinity
      ↕
[Chip Level]      irq_chip: mask/unmask/ack/eoi — hardware specific
```

الـ `irqnr.h` بيعيش في **Generic IRQ Level** — بيوفر الـ numbering وال iteration بدون ما يعرف أي حاجة عن الـ hardware.

---

### الـ Subsystem بيمتلك إيه؟ وبيفوّض إيه؟

#### الـ Generic IRQ Subsystem بيمتلك:
- إدارة الـ `irq_desc` table (allocation، lookup، iteration)
- الـ flow handlers (level، edge، fasteoi، oneshot)
- الـ threading model للـ IRQ handlers
- الـ SMP affinity logic
- الـ spurious interrupt detection
- الـ stats والـ `/proc/interrupts` reporting
- الـ `for_each_*` macros وكل الـ iteration API

#### بيفوّض للـ drivers والـ chip code:
- الـ `irq_chip` operations (mask، unmask، ack — hardware specific)
- الـ `irq_domain` mapping (HW IRQ → Linux IRQ، implemented per-controller)
- الـ actual ISR logic (الـ `irqaction->handler` اللي بيكتبه driver developer)
- الـ `irq_set_type()` (edge/level configuration — hardware specific)

---

### مفاهيم مرتبطة تحتاج تعرفها

- **`irq_domain`** — الـ subsystem المسؤول عن الـ mapping من HW IRQ لـ Linux IRQ. لما بتشتغل مع interrupt controller جديد في ARM، ده أول حاجة بتعملها.
- **`irq_chip`** — الـ abstraction للـ interrupt controller hardware operations (mask/unmask/ack). كل controller بيعمل implementation خاصة.
- **`irq_work`** — subsystem بيسمح بتأجيل عمل من داخل interrupt context، بيُستخدم في الـ `irq_redirect` اللي موجود في `irq_desc`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Macros والـ API — Cheatsheet

| الاسم | النوع | الغرض |
|---|---|---|
| `for_each_irq_desc(irq, desc)` | macro | يلف على كل `irq_desc` مسجّل بالترتيب التصاعدي |
| `for_each_irq_desc_reverse(irq, desc)` | macro | نفسه لكن بترتيب تنازلي |
| `for_each_active_irq(irq)` | macro | يلف فقط على الـ IRQs النشطة (بيتخطى الفراغات) |
| `for_each_irq_nr(irq)` | macro | يلف على كل أرقام الـ IRQ من 0 لـ `nr_irqs` |
| `irq_get_nr_irqs()` | function | يرجع عدد الـ IRQs المسجّلة حالياً |
| `irq_set_nr_irqs(nr)` | function | يضبط الحد الأقصى لعدد الـ IRQs |
| `irq_to_desc(irq)` | function | يحوّل رقم IRQ لـ pointer على `irq_desc` |
| `irq_get_next_irq(offset)` | function | يرجع أول IRQ نشط من `offset` فصاعداً |

#### الفرق بين الـ Macros

| Macro | يزور الـ NULL descs؟ | يحتاج `desc`؟ | الاستخدام |
|---|---|---|---|
| `for_each_irq_desc` | لا (يتخطاها) | نعم | تحتاج الـ desc نفسها |
| `for_each_irq_desc_reverse` | لا | نعم | تحتاج ترتيب عكسي |
| `for_each_active_irq` | — | لا | تحتاج رقم IRQ فقط |
| `for_each_irq_nr` | يزور الكل | لا | عدّ أو فحص بسيط |

---

### 1. الـ Structs المهمة

#### `struct irq_desc` — قلب نظام الـ interrupts

الـ **`irq_desc`** هو الـ descriptor المركزي لكل interrupt في الـ kernel. كل رقم IRQ له `irq_desc` واحد بالضبط.

| الـ field | النوع | الغرض |
|---|---|---|
| `irq_common_data` | `struct irq_common_data` | بيانات مشتركة بين الـ IRQ والـ chip |
| `irq_data` | `struct irq_data` | بيانات الـ chip الخاصة بهذا IRQ |
| `kstat_irqs` | `struct irqstat __percpu *` | إحصائيات per-CPU |
| `handle_irq` | `irq_flow_handler_t` | الـ high-level flow handler |
| `action` | `struct irqaction *` | قائمة الـ handlers المسجّلة بـ `request_irq()` |
| `depth` | `unsigned int` | عمق الـ nesting لـ `irq_disable()` |
| `wake_depth` | `unsigned int` | عمق الـ wake enable |
| `lock` | `raw_spinlock_t` | يحمي الـ descriptor في SMP |
| `request_mutex` | `struct mutex` | يحمي عملية الـ request/free |
| `threads_active` | `atomic_t` | عدد الـ threaded handlers الشغّالة |
| `wait_for_threads` | `wait_queue_head_t` | لـ `synchronize_irq()` |
| `redirect` | `struct irq_redirect` | إعادة توجيه الـ IRQ لـ CPU تاني |
| `affinity_hint` | `const struct cpumask *` | تلميح لـ userspace للـ affinity |
| `pending_mask` | `cpumask_var_t` | ماسك الـ IRQs في انتظار إعادة التوزيع (SMP) |
| `parent_irq` | `int` | رقم الـ parent IRQ في الـ hierarchy |
| `name` | `const char *` | اسم الـ flow handler في `/proc/interrupts` |

#### `struct irqstat` — إحصائيات الـ interrupt

| الـ field | الغرض |
|---|---|
| `cnt` | عداد الـ interrupts الفعلي real-time |
| `ref` | snapshot للعداد (فقط مع `CONFIG_GENERIC_IRQ_STAT_SNAPSHOT`) |

#### `struct irq_redirect` — إعادة توجيه الـ IRQ

| الـ field | الغرض |
|---|---|
| `work` | `irq_work` item لتشغيل الـ handler على CPU مختلف |
| `target_cpu` | الـ CPU المستهدف لو الـ CPU الحالي مش في الـ affinity mask |

---

### 2. علاقات الـ Structs — ASCII Diagram

```
irq number (unsigned int)
        │
        ▼
  irq_to_desc(irq)
        │
        ▼
┌─────────────────────────────────────────────┐
│              struct irq_desc                │
│                                             │
│  ┌──────────────────────┐                  │
│  │  irq_common_data     │◄─── shared by    │
│  │  (handler_data, etc) │     chip & IRQ   │
│  └──────────────────────┘                  │
│                                             │
│  ┌──────────────────────┐                  │
│  │  irq_data            │──► struct irq_chip│
│  │  (.irq, .hwirq,      │    (mask/unmask/ │
│  │   .chip, .domain)    │     ack/eoi)     │
│  └──────────────────────┘                  │
│                                             │
│  handle_irq ──────────────► irq_flow_handler│
│                             (handle_level,  │
│                              handle_edge,   │
│                              handle_fasteoi)│
│                                             │
│  action ──────────────────► struct irqaction│
│                              │  .handler   │
│                              │  .dev_id    │
│                              └► next ──► ...│
│                                             │
│  kstat_irqs (per-CPU) ──────► struct irqstat│
│                               .cnt          │
│                                             │
│  lock (raw_spinlock_t)                      │
│  request_mutex                              │
│                                             │
│  redirect ──────────────────► struct irq_redirect
│                               .work         │
│                               .target_cpu   │
└─────────────────────────────────────────────┘
        │
        │ (CONFIG_SPARSE_IRQ)
        ▼
  radix_tree / xarray
  (بدل static array)
```

#### في حالة `CONFIG_SPARSE_IRQ = n`

```
irq_desc[NR_IRQS]   ← static global array
    [0] → irq_desc
    [1] → irq_desc
    ...
    [N] → irq_desc
```

#### في حالة `CONFIG_SPARSE_IRQ = y`

```
irq number
    │
    ▼
radix_tree_lookup(&irq_desc_tree, irq)
    │
    ▼
irq_desc  (dynamically allocated, protected by RCU)
```

---

### 3. الـ Lifecycle Diagram

```
Bootstrap / alloc_irq_desc()
         │
         ▼
  [irq_desc allocated]
  (static array أو dynamic xarray)
         │
         ▼
  irq_domain_alloc_irqs()
  ─ يربط hwirq ← → virq
  ─ يضبط irq_data.chip
  ─ يضبط handle_irq = handle_bad_irq
         │
         ▼
  request_irq() / request_threaded_irq()
  ─ يضيف struct irqaction للـ action list
  ─ يضبط handle_irq الصح (level/edge/fasteoi)
  ─ يشغّل الـ chip->irq_startup()
         │
         ▼
  [IRQ نشط — يستقبل interrupts]
         │
         ▼
  free_irq()
  ─ يشيل الـ irqaction من القائمة
  ─ لو القائمة فاضية → يضبط handle_irq = handle_bad_irq
  ─ يشغّل chip->irq_shutdown()
         │
         ▼
  irq_free_descs()  (sparse IRQ فقط)
  ─ يمسح الـ descriptor من الـ xarray
  ─ يحرر الذاكرة بعد RCU grace period
```

---

### 4. Call Flow Diagrams

#### الـ for_each_irq_desc macro — كيف يشتغل داخلياً

```
for_each_irq_desc(irq, desc)
    │
    ├─ irq_get_nr_irqs()          ← يقرأ nr_irqs global
    │         │
    │         ▼
    │   __nr_irqs__ = N
    │
    └─ loop: irq = 0 → N-1
          │
          ├─ irq_to_desc(irq)
          │       │
          │       ├─ [SPARSE_IRQ]  → xa_load(&irq_desc_tree, irq)
          │       └─ [!SPARSE_IRQ] → &irq_desc[irq]
          │
          ├─ if (!desc) → skip   ← يتخطى الـ holes
          │
          └─ else → ينفذ body الـ loop بالـ desc
```

#### الـ for_each_active_irq macro — الفرق

```
for_each_active_irq(irq)
    │
    └─ irq_get_next_irq(0)
          │
          ▼
    [يبحث في bitmap أو xarray عن أول IRQ نشط]
          │
          ▼
    loop body
          │
          ▼
    irq_get_next_irq(irq + 1)   ← يقفز للـ active التالي مباشرةً
```

#### معالجة الـ interrupt الفعلية (generic_handle_irq_desc)

```
hardware interrupt fires
        │
        ▼
arch_do_IRQ() / handle_arch_irq()
        │
        ▼
irq_to_desc(virq)
        │
        ▼
generic_handle_irq_desc(desc)
        │
        ▼
desc->handle_irq(desc)          ← الـ flow handler
  [handle_level_irq /
   handle_edge_irq /
   handle_fasteoi_irq]
        │
        ├─ desc->lock  (raw_spin_lock)
        │
        ├─ chip->irq_mask() أو irq_ack()
        │
        ├─ action->handler(irq, dev_id)
        │     │
        │     └─ (threaded) → wake_up_process(thread)
        │
        └─ chip->irq_unmask() أو irq_eoi()
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks في `irq_desc`

| الـ Lock | النوع | يحمي إيه |
|---|---|---|
| `desc->lock` | `raw_spinlock_t` | كل حاجة في الـ descriptor: الـ action list، الـ depth، الـ status، الـ chip calls |
| `desc->request_mutex` | `struct mutex` | عملية الـ `request_irq()` / `free_irq()` — يمنع race بين thread بيطلب وتاني بيحرر |
| `irq_lock_sparse()` | global mutex (SPARSE_IRQ) | يحمي الـ xarray لما بنضيف أو بنمسح descriptors |

#### ترتيب الـ Locking (Lock Ordering)

```
irq_lock_sparse()          ← أعلى مستوى (global)
    │
    └─ desc->request_mutex  ← مستوى per-descriptor
            │
            └─ desc->lock (raw_spinlock) ← أدنى مستوى، irq-safe
```

**قاعدة**: لازم تاخد `request_mutex` قبل `desc->lock` عشان تتجنب deadlock بين الـ `request_irq()` والـ interrupt handler نفسه.

#### الـ RCU في `CONFIG_SPARSE_IRQ`

- الـ `irq_to_desc()` بيشتغل تحت **RCU read lock** — بيسمح بقراءة الـ descriptor من الـ interrupt context من غير spinlock.
- لما بنمسح descriptor → بيتحرر بعد **RCU grace period** باستخدام `desc->rcu` (`struct rcu_head`).
- الـ `for_each_irq_desc` بيتعمل فيه explicit `irq_lock_sparse()` لو المتصل محتاج stability كاملة أثناء الـ iteration.

#### الـ Per-CPU Stats بدون lock

الـ `kstat_irqs` هو `__percpu` pointer — كل CPU بيكتب في نسخته الخاصة من `irqstat.cnt` من غير ما يحتاج lock على الإطلاق. ده تصميم مقصود لتفادي الـ contention في hot path الـ interrupt handling.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

| Function / Macro | النوع | الغرض |
|---|---|---|
| `irq_get_nr_irqs()` | Function | بترجع عدد الـ IRQs المتاحة في النظام |
| `irq_set_nr_irqs(nr)` | Function | بتضبط الحد الأقصى لعدد الـ IRQs |
| `irq_to_desc(irq)` | Function | بتترجم رقم IRQ لـ `struct irq_desc*` |
| `irq_get_next_irq(offset)` | Function | بتجيب أول IRQ نشط ابتداءً من `offset` |
| `for_each_irq_desc(irq, desc)` | Macro | بيمشي على كل الـ IRQ descriptors بالترتيب |
| `for_each_irq_desc_reverse(irq, desc)` | Macro | بيمشي عليهم بالعكس |
| `for_each_active_irq(irq)` | Macro | بيمشي على الـ IRQs النشطة بس |
| `for_each_irq_nr(irq)` | Macro | بيمشي على أرقام الـ IRQs فقط (بدون desc) |

---

### Category 1: IRQ Count Management

الـ functions دي مسؤولة عن إدارة الحد الأقصى لعدد الـ interrupt lines المتاحة في النظام. الـ kernel بيحتفظ بـ `nr_irqs` عالمية تحكم كل loops وكل allocations في الـ IRQ subsystem.

---

#### `irq_get_nr_irqs`

```c
unsigned int irq_get_nr_irqs(void) __pure;
```

بترجع القيمة الحالية لـ `nr_irqs` — عدد الـ interrupt numbers المتاحة في النظام. الـ `__pure` attribute بتقول للـ compiler إن الـ function مفيهاش side effects وبترجع نفس النتيجة لنفس الـ state، فيقدر يعمل لها CSE (Common Subexpression Elimination). بتستخدمها الـ macros الأربعة في الـ header ده لتحديد حدود الـ loops.

**Parameters:** لا يوجد.

**Return:** الـ `unsigned int` بيمثل عدد الـ IRQs — قيمة `nr_irqs`.

**Key details:**
- الـ `__pure` معناها إن الـ compiler يقدر يكاش الـ result، لكن في الـ macros بيتم حفظها في `__nr_irqs__` يدوياً عشان نضمن قراءة واحدة consistent.
- الـ implementation موجودة في `kernel/irq/irqdesc.c`.

**Caller context:** بيتنادى من الـ macros التانية في نفس الـ header، ومن أي code محتاج يعرف حجم الـ IRQ space.

---

#### `irq_set_nr_irqs`

```c
unsigned int irq_set_nr_irqs(unsigned int nr);
```

بتضبط الحد الأقصى لعدد الـ IRQs. بتُستخدم من الـ board-level init أو الـ architecture setup لتوسيع أو تقليص الـ IRQ space. الـ function دي ممكن ترفض التغيير لو الـ `nr` أصغر من العدد الحالي أو بيتعارض مع الـ allocated descriptors.

**Parameters:**
- `nr` — العدد الجديد المطلوب للـ IRQs.

**Return:** الـ `unsigned int` — القيمة اللي اتضبطت فعلاً (ممكن تختلف لو رفضت التعديل).

**Key details:**
- بيتنادى مبكراً في الـ boot من `arch_probe_nr_irqs()` أو الـ machine-specific init.
- مفيش locking صريح في الـ signature، لكن الـ implementation بتأمن الـ write على `nr_irqs`.
- لو حاولت تزود عن `MAX_IRQ_SPARSE_NR` أو الـ compile-time limit بيرفض.

**Caller context:** Early boot فقط، قبل ما الـ IRQ subsystem يبدأ يخدم requests.

---

### Category 2: IRQ Descriptor Lookup

الـ descriptor lookup هو القلب في الـ IRQ subsystem — كل operation على IRQ بتبدأ بـ `irq_to_desc`.

---

#### `irq_to_desc`

```c
extern struct irq_desc *irq_to_desc(unsigned int irq);
```

بتترجم رقم الـ interrupt (الـ Linux virtual IRQ number) لـ pointer على الـ `struct irq_desc` الخاصة بيه. الـ `struct irq_desc` هي الـ central data structure في الـ IRQ subsystem — بتحتوي على الـ handler, الـ action chain, الـ affinity mask, الـ stats, وكل metadata للـ interrupt.

**Parameters:**
- `irq` — الـ Linux virtual IRQ number (مش الـ hardware IRQ).

**Return:** `struct irq_desc*` أو `NULL` لو الـ IRQ مش موجود أو مش allocated.

**Key details:**
- في `CONFIG_SPARSE_IRQ`: الـ implementation بتعمل radix tree lookup في `irq_desc_tree`.
- في non-sparse: بتعمل direct array index على `irq_desc[]` الـ global array.
- الـ callers **لازم** يتحققوا من `NULL` قبل الاستخدام — الـ macros في الـ header دي بتعمل ده automatically.
- مفيش locking — الـ RCU بيحمي الـ sparse tree، الـ caller مسؤول عن الـ RCU read lock لو محتاج.

**Caller context:** أي context — لكن في atomic context لازم تكون واخد بالك من الـ RCU.

```
irq_to_desc(irq):
    if SPARSE_IRQ:
        return radix_tree_lookup(&irq_desc_tree, irq)
    else:
        if irq >= nr_irqs: return NULL
        return &irq_desc[irq]
```

---

#### `irq_get_next_irq`

```c
unsigned int irq_get_next_irq(unsigned int offset);
```

بتجيب رقم أول IRQ **نشط** (allocated/active) ابتداءً من `offset` شامله. بتُستخدم في `for_each_active_irq` لتخطي الأرقام الـ unallocated في الـ sparse IRQ space.

**Parameters:**
- `offset` — رقم الـ IRQ اللي هيبدأ منه الـ search (inclusive).

**Return:** رقم أول IRQ نشط >= `offset`، أو قيمة >= `nr_irqs` لو مفيش.

**Key details:**
- في `CONFIG_SPARSE_IRQ`: بتعمل radix tree scan — أسرع من loop على كل الأرقام.
- في non-sparse: بتمشي على الـ array وتدور على أول descriptor مش `NULL`.
- الـ macro `for_each_active_irq` بتستخدمها بـ `irq + 1` عشان تتقدم للـ next.

**Caller context:** Process أو softirq context، مش في hardirq context عادةً.

---

### Category 3: Iteration Macros

الـ macros دي بتوفر patterns موحدة للـ traversal على كل الـ IRQ space. كلها بتحفظ `nr_irqs` في variable محلي `__nr_irqs__` في بداية الـ loop عشان تضمن consistent snapshot.

---

#### `for_each_irq_desc`

```c
#define for_each_irq_desc(irq, desc) \
    for (unsigned int __nr_irqs__ = irq_get_nr_irqs(); __nr_irqs__; \
         __nr_irqs__ = 0) \
        for (irq = 0, desc = irq_to_desc(irq); irq < __nr_irqs__; \
             irq++, desc = irq_to_desc(irq)) \
            if (!desc) \
                ; \
            else
```

بيمشي على كل الـ IRQ numbers من 0 لـ `nr_irqs - 1`، وبيجيب الـ `irq_desc*` لكل واحد. بيتخطى الـ IRQs اللي `desc == NULL` تلقائياً (الـ `if (!desc) ;` بتعمل continue ضمني).

**Parameters (loop variables):**
- `irq` — متغير من نوع `unsigned int` بيتحدث في كل iteration.
- `desc` — متغير من نوع `struct irq_desc*` بيتحدث في كل iteration.

**Key details:**
- الـ outer loop `for(...; __nr_irqs__; __nr_irqs__ = 0)` هي trick بتضمن إن `irq_get_nr_irqs()` بيتنادى مرة واحدة بس — بيحفظ القيمة في `__nr_irqs__` وبعدين يصفرها في الـ increment عشان الـ outer loop يوقف في الدورة التانية.
- الـ `if (!desc) ; else` بيعني: لو `desc == NULL` نعدي (الـ semicolon = empty body)، غير كذلك ننفذ الـ loop body.
- الـ body بييجي بعد الـ `else` مباشرة.

**Caller context:** Process context فقط — بياخد وقت في الـ sparse systems.

**مثال:**
```c
struct irq_desc *desc;
unsigned int irq;

for_each_irq_desc(irq, desc) {
    /* desc is never NULL here */
    pr_info("IRQ %u: name=%s\n", irq, desc->name ?: "none");
}
```

---

#### `for_each_irq_desc_reverse`

```c
# define for_each_irq_desc_reverse(irq, desc) \
    for (irq = irq_get_nr_irqs() - 1, desc = irq_to_desc(irq); \
         irq >= 0; irq--, desc = irq_to_desc(irq)) \
        if (!desc) \
            ; \
        else
```

نفس `for_each_irq_desc` بالظبط، لكن بيمشي من `nr_irqs - 1` لـ 0 عكسياً. بيُستخدم في سيناريوهات الـ teardown أو لو الـ priority order مهم.

**Parameters:** نفس الـ `for_each_irq_desc`.

**Key details:**
- لاحظ إن مفيش `__nr_irqs__` هنا — الـ `nr_irqs` بيتنادى مرة واحدة في الـ init expression.
- الـ termination condition هي `irq >= 0` وده شغال صح لأن `irq` هنا `int` مش `unsigned int` — تأكد من نوع المتغير في الـ caller!
- الـ `int` مطلوب هنا عشان الـ `>= 0` يشتغل صح مع الـ decrement لـ `-1`.

**مثال:**
```c
int irq; /* must be int, not unsigned int */
struct irq_desc *desc;

for_each_irq_desc_reverse(irq, desc) {
    free_irq_desc(desc); /* teardown in reverse */
}
```

---

#### `for_each_active_irq`

```c
#define for_each_active_irq(irq) \
    for (unsigned int __nr_irqs__ = irq_get_nr_irqs(); __nr_irqs__; \
         __nr_irqs__ = 0) \
        for (irq = irq_get_next_irq(0); irq < __nr_irqs__; \
             irq = irq_get_next_irq(irq + 1))
```

بيمشي على الـ active/allocated IRQs **فقط**، بيتخطى الـ gaps في الـ sparse IRQ space بكفاءة باستخدام `irq_get_next_irq`. مفيهوش `desc` variable — بس بيدي الـ `irq` number.

**Parameters (loop variable):**
- `irq` — متغير من نوع `unsigned int`.

**Key details:**
- أكفأ من `for_each_irq_desc` في الـ sparse systems لأن `irq_get_next_irq` بتقفز فوق الـ gaps.
- في dense (non-sparse) systems الفرق بسيط.
- لو محتاج الـ `desc` جوا الـ loop، تقدر تعمل `irq_to_desc(irq)` يدوياً.

**مثال:**
```c
unsigned int irq;

for_each_active_irq(irq) {
    struct irq_desc *desc = irq_to_desc(irq);
    if (desc && irqd_is_level_type(&desc->irq_data))
        pr_info("Level IRQ: %u\n", irq);
}
```

---

#### `for_each_irq_nr`

```c
#define for_each_irq_nr(irq) \
    for (unsigned int __nr_irqs__ = irq_get_nr_irqs(); __nr_irqs__; \
         __nr_irqs__ = 0) \
        for (irq = 0; irq < __nr_irqs__; irq++)
```

أبسط الـ macros — بيمشي على كل الأرقام من 0 لـ `nr_irqs - 1` بدون ما يجيب الـ `desc`. مناسب لما تكون محتاج الرقم بس، أو لما الـ lookup بييجي داخلياً في الـ loop body.

**Parameters (loop variable):**
- `irq` — متغير من نوع `unsigned int`.

**Key details:**
- مفيش فلترة للـ NULL descriptors — لو حتعمل `irq_to_desc` جوا يدوياً لازم تتحقق من `NULL`.
- أبسط loop structure، مناسب للـ statistics collection أو sequential operations.

**مثال:**
```c
unsigned int irq;
unsigned long total_count = 0;

for_each_irq_nr(irq) {
    struct irq_desc *desc = irq_to_desc(irq);
    if (desc)
        total_count += desc->tot_count;
}
```

---

### ملاحظات مهمة عن الـ Macros

#### الـ `__nr_irqs__` Pattern

الـ double-loop pattern في `for_each_irq_desc`, `for_each_active_irq`, و `for_each_irq_nr` مش عشمل — ده بيضمن:
1. `irq_get_nr_irqs()` بيتنادى **مرة واحدة بس** في بداية الـ iteration.
2. لو اتغيرت `nr_irqs` أثناء الـ loop (نادر جداً)، الـ loop مش هتتأثر.
3. الـ outer loop بتتحكم بالـ snapshot دون إنها تضيف overhead حقيقي.

```
for (outer: __nr_irqs__ = get_nr(); __nr_irqs__; __nr_irqs__ = 0)
    ^-- runs once, stores snapshot     ^-- 2nd outer iter: stops
    for (inner: irq = 0; irq < __nr_irqs__; irq++)
        ^-- actual work
```

#### الـ NULL Check Pattern

```c
if (!desc)
    ;       /* skip — continue to next iteration */
else
    /* loop body comes here */
```

ده الطريقة الوحيدة تعمل فيها conditional skip في macro بدون `continue` (اللي مش بتشتغل صح في nested macros).

#### الـ Sparse vs Dense IRQ

| | Sparse (`CONFIG_SPARSE_IRQ`) | Dense |
|---|---|---|
| `irq_to_desc` | Radix tree lookup | Array index |
| `irq_get_next_irq` | Tree scan (fast) | Linear scan |
| `for_each_active_irq` | يتخطى الـ gaps بكفاءة | مفيش فرق |
| Memory | Allocates only used descs | Static array for all |
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem اللي بنتكلم عنه هو **IRQ numbering والـ iteration** — الكود في `irqnr.h` بيوفر الـ macros اللي بتمشي على كل الـ IRQs في النظام (`for_each_irq_desc`, `for_each_active_irq`, إلخ) وبتربط بين الـ IRQ number والـ `irq_desc` struct. الـ debugging هنا بيركز على: كام IRQ عندنا؟ إيه اللي registered؟ إيه اللي بيـ fire؟ إيه اللي مش شغال؟

---

### Software Level

#### 1. debugfs entries

لازم يكون `CONFIG_GENERIC_IRQ_DEBUGFS=y` عشان تشتغل.

```
/sys/kernel/debug/irq/
├── domains/          ← كل irq_domain مسجل في النظام
│   └── <domain-name>/
│       ├── name
│       ├── size
│       └── mapped   ← عدد الـ IRQs المـ mapped فعلاً
└── irqs/
    └── <irq-number>/
        ├── actions       ← اسم الـ handler المسجل
        ├── chip_name
        ├── hwirq         ← الـ hardware IRQ number
        ├── type          ← edge/level
        ├── status        ← flags: enabled/disabled/pending...
        ├── depth         ← عدد مرات irq_disable() بدون irq_enable()
        └── affinity      ← CPU mask
```

```bash
# اقرأ كل اللي عندك عن IRQ رقم 27
cat /sys/kernel/debug/irq/irqs/27/*

# شوف كل الـ domains المسجلة
ls /sys/kernel/debug/irq/domains/

# dump كامل لكل IRQ (بيأخذ وقت)
for d in /sys/kernel/debug/irq/irqs/*/; do
    echo "=== $d ==="; cat "$d"/* 2>/dev/null; echo
done
```

#### 2. sysfs entries

```
/sys/kernel/irq/
└── <irq-number>/
    ├── actions       ← اسم الـ driver اللي عمل request_irq()
    ├── chip_name     ← اسم الـ irq_chip
    ├── hwirq
    ├── name          ← flow handler name
    ├── per_cpu       ← 0 أو 1
    ├── type          ← none/edge/level
    ├── wakeup        ← 0 أو 1
    └── affinity_hint
```

```bash
# شوف كل الـ IRQs اللي عندها action (يعني فيها driver)
grep -r . /sys/kernel/irq/*/actions 2>/dev/null

# شوف نوع كل IRQ
for irq in /sys/kernel/irq/*/; do
    n=$(basename $irq)
    act=$(cat $irq/actions 2>/dev/null)
    typ=$(cat $irq/type 2>/dev/null)
    [ -n "$act" ] && echo "IRQ $n: $typ [$act]"
done
```

#### 3. /proc entries

```bash
# أهم ملف: إحصائيات كل IRQ لكل CPU
cat /proc/interrupts

# مثال على output:
#            CPU0       CPU1
#   0:         46          0   IO-APIC    2-edge      timer
#  27:       1024        890   PCI-MSI 524288-edge   xhci_hcd
# ERR:          0
# MIS:          0   ← missed interrupts (spurious)

# إحصائيات سريعة
watch -n1 cat /proc/interrupts

# مقارنة ما اتغير في الـ 5 ثواني دول
A=$(cat /proc/interrupts); sleep 5; B=$(cat /proc/interrupts)
diff <(echo "$A") <(echo "$B")
```

#### 4. ftrace — tracepoints للـ IRQ subsystem

```bash
# شوف كل الـ events المتاحة للـ IRQs
ls /sys/kernel/debug/tracing/events/irq/

# الـ events المهمة:
# irq_handler_entry  ← قبل ما الـ handler يتشغل
# irq_handler_exit   ← بعد ما الـ handler ينتهي
# softirq_entry/exit
# irq_matrix_*       ← CPU affinity matrix events

# تفعيل tracing للـ IRQ handlers
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo 1 > events/irq/irq_handler_entry/enable
echo 1 > events/irq/irq_handler_exit/enable
echo 1 > tracing_on
sleep 2
echo 0 > tracing_on
cat trace | head -50

# فلترة على IRQ معين (مثلاً IRQ 27)
echo 'irq == 27' > events/irq/irq_handler_entry/filter
echo 1 > events/irq/irq_handler_entry/enable

# function tracer للدوال المرتبطة بالـ IRQ iteration
echo function > current_tracer
echo 'irq_to_desc irq_get_nr_irqs irq_get_next_irq' > set_ftrace_filter
echo 1 > tracing_on
```

#### 5. printk / dynamic debug

```bash
# تفعيل dynamic debug للـ IRQ core
echo 'file kernel/irq/*.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/irq/irqdesc.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages اللي اتفعلت
cat /sys/kernel/debug/dynamic_debug/control | grep 'kernel/irq'

# في kernel cmdline (للـ boot-time debugging)
# dyndbg="file kernel/irq/irqdesc.c +p"

# لو عايز تشوف الـ sparse IRQ allocation
echo 'module irq +p' > /sys/kernel/debug/dynamic_debug/control

# تتبع في الـ dmesg
dmesg -w | grep -E 'irq|IRQ|interrupt'
```

#### 6. Kernel config options للـ debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_GENERIC_IRQ_DEBUGFS` | يفعّل `/sys/kernel/debug/irq/` كله |
| `CONFIG_SPARSE_IRQ` | dynamic allocation للـ `irq_desc` بدل array ثابت |
| `CONFIG_GENERIC_IRQ_STAT_SNAPSHOT` | يضيف `ref` counter في `irqstat` للمقارنة |
| `CONFIG_IRQ_DOMAIN_DEBUG` | debug entries للـ irq domains في debugfs |
| `CONFIG_HARDIRQ_ACCOUNTING` | إحصائيات دقيقة عن وقت الـ hardirq |
| `CONFIG_DEBUG_SHIRQ` | يـ simulate إن الـ shared IRQ اتـ fired عند الـ free |
| `CONFIG_PROVE_LOCKING` | lockdep على الـ irq_desc lock |
| `CONFIG_IRQ_PIPELINE` | للـ real-time, يفعّل IRQ pipelining |
| `CONFIG_GENERIC_PENDING_IRQ` | دعم الـ IRQ balancing مع pending mask |
| `CONFIG_HARDIRQS_SW_RESEND` | يسمح بـ software resend للـ IRQs |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(GENERIC_IRQ|SPARSE_IRQ|DEBUG_SHIRQ|IRQ_DOMAIN)'
# أو
grep -E 'CONFIG_(GENERIC_IRQ|SPARSE_IRQ|DEBUG_SHIRQ)' /boot/config-$(uname -r)
```

#### 7. جدول رسائل الـ error الشائعة

| رسالة في dmesg | المعنى | الحل |
|---|---|---|
| `irq N: nobody cared` | IRQ بيـ fire بس مفيش handler بيـ handle it (spurious) | افحص الـ hardware affinity، تأكد إن الـ driver صح registered |
| `Disabling IRQ #N` | الكيرنل disable الـ IRQ تلقائياً بسبب كتير spurious | افحص الـ hardware، ممكن مشكلة في الـ trigger type (edge vs level) |
| `do_IRQ: N.N No irq handler for vector` | IRQ vector وصل للـ CPU بس مفيش `irq_desc` مناسب | مشكلة في الـ APIC routing أو `irq_domain` mapping |
| `irq: type mismatch, failed to map hwirq` | الـ irq_domain ما قدرش يعمل map للـ hardware IRQ | راجع الـ Device Tree أو ACPI tables |
| `kernel BUG at kernel/irq/irqdesc.c` | الـ `irq_to_desc()` رجع NULL في مكان ما كانش متوقع | مشكلة في الـ sparse IRQ allocation أو race condition |
| `request_irq failed with error -EBUSY` | IRQ متخد من driver تاني | تأكد من الـ IRQF_SHARED flag، أو افحص مين صاحب الـ IRQ |
| `irq_create_mapping: no irq_chip?` | الـ irq_domain مش عنده chip مضبوط | راجع الـ irq_domain_ops ودالة `map` |
| `WARNING: possible circular locking` | lockdep كشف إن في حد عامل lock على `irq_desc->lock` بترتيب غلط | راجع الـ locking order في الـ driver |
| `IRQ handler type mismatch for IRQ N` | اتنين driver حاولوا يـ request نفس الـ IRQ بـ type مختلف | استخدم `IRQF_SHARED` وتأكد إن كل الـ devices بيشاركوا نفس الـ IRQ config |

#### 8. أماكن استراتيجية لـ `dump_stack()` و `WARN_ON()`

```c
/* في irq_to_desc() لو رجع NULL في مكان غير متوقع */
struct irq_desc *irq_to_desc(unsigned int irq)
{
    struct irq_desc *desc = /* ... */;
    WARN_ON_ONCE(!desc && irq < irq_get_nr_irqs());
    return desc;
}

/* في for_each_irq_desc — لو desc رجع NULL لـ irq معروف */
for_each_irq_desc(irq, desc) {
    /* desc هنا مضمون مش NULL بسبب الـ if(!desc) check في الـ macro */
    WARN_ON(irq_desc_get_irq(desc) != irq); /* consistency check */
}

/* لو irq_get_nr_irqs() رجع قيمة غريبة */
void some_init(void)
{
    unsigned int nr = irq_get_nr_irqs();
    WARN_ON(nr == 0 || nr > 1048576); /* sanity check */
    dump_stack(); /* لو عايز تعرف مين استدعى التهيئة */
}

/* تحقق إن الـ iteration بتشتغل صح */
unsigned int count = 0;
for_each_active_irq(irq) {
    count++;
    WARN_ON(count > irq_get_nr_irqs()); /* protection من infinite loop */
}
```

---

### Hardware Level

#### 1. التحقق إن الـ hardware state بيطابق الـ kernel state

```bash
# قارن الـ hardware IRQs (من APIC/GIC) مع الـ kernel view
# على x86: شوف الـ IOAPIC routing
cat /proc/iomem | grep -i apic
# لو عندك acpidump
acpidump | grep -A5 'APIC\|MADT'

# على ARM: شوف الـ GIC registers (لو عارف base address)
# GIC Distributor base عادةً في Device Tree
cat /sys/firmware/devicetree/base/interrupt-controller*/reg 2>/dev/null | xxd

# قارن الـ hwirq مع الـ Linux IRQ number
for d in /sys/kernel/debug/irq/irqs/*/; do
    irq=$(basename $d)
    hwirq=$(cat $d/hwirq 2>/dev/null)
    chip=$(cat $d/chip_name 2>/dev/null)
    act=$(cat $d/actions 2>/dev/null)
    [ -n "$act" ] && printf "Linux IRQ %-4s -> HW IRQ %-4s [%s] %s\n" "$irq" "$hwirq" "$chip" "$act"
done
```

#### 2. Register dump techniques

```bash
# devmem2 لقراءة registers مباشرة (محتاج root + CONFIG_STRICT_DEVMEM=n أو معطل)
# مثال: قراءة GIC Distributor ISENABLER على ARM
# أول حاجة: اعرف الـ base address من Device Tree أو datasheet
GIC_DIST_BASE=0x08000000
devmem2 $((GIC_DIST_BASE + 0x100)) w  # GICD_ISENABLER0
devmem2 $((GIC_DIST_BASE + 0x200)) w  # GICD_ICPENDR0 (pending)
devmem2 $((GIC_DIST_BASE + 0x300)) w  # GICD_ISACTIVER0 (active)

# على x86: قراءة IOAPIC
# عنوان IOAPIC من ACPI أو /proc/iomem
IOAPIC_BASE=0xFEC00000
devmem2 $IOAPIC_BASE w          # IOAPIC index register
devmem2 $((IOAPIC_BASE + 0x10)) w  # IOAPIC data register

# بديل أمان أكتر: /dev/mem مع lspci لـ PCI MSI
lspci -vvv | grep -A10 'MSI\|MSI-X'

# على ARM64: مع /dev/mem
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 4096, offset=0x08000000)
    val = struct.unpack('<I', mm[0x100:0x104])[0]
    print(f'GICD_ISENABLER0 = 0x{val:08x}')
    mm.close()
"
```

#### 3. Logic Analyzer / Oscilloscope

لما بتحاول تـ debug مشكلة hardware IRQ:

```
الـ scenario الأول: IRQ مش بيـ fire
─────────────────────────────────────
1. شيل الـ probe على الـ interrupt line للـ device
2. اعمل الـ operation اللي المفروض تـ trigger الـ IRQ
3. لو الـ line مش بتتغير → مشكلة في الـ hardware/firmware
4. لو الـ line بتتغير بس الـ kernel مش شايفها → مشكلة في الـ routing

الـ scenario التاني: spurious IRQs
──────────────────────────────────
1. شيل الـ probe على الـ interrupt line
2. دور على glitches أو noise
3. قيس الـ rise/fall time (المفروض < 10ns للـ digital signals)
4. افحص الـ pull-up/pull-down resistors

الـ scenario التالت: edge vs level confusion
────────────────────────────────────────────
- Edge-triggered: الـ kernel المفروض يشوف pulse واحدة
- Level-triggered: الـ line المفروض تفضل asserted لحد ما الـ device يـ clear it
- لو الـ device بيـ clear الـ interrupt قبل ما الـ handler يشتغل → lost interrupt
  الحل: استخدم IRQF_SHARED أو افحص الـ trigger type في DT
```

```bash
# تحقق من الـ trigger type اللي الكيرنل مسجله
cat /sys/kernel/irq/27/type
# المتوقع: "edge-rising" أو "level-high" إلخ

# قارن مع اللي في Device Tree
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 'interrupt'
```

#### 4. Hardware issues الشائعة → kernel log patterns

| المشكلة | الـ kernel log | الـ diagnosis |
|---|---|---|
| Floating interrupt line | `irq N: nobody cared` متكرر | افحص الـ pull-up resistors، تأكد من الـ grounding |
| Level IRQ مش بيتـ clear | `irq N: nobody cared` + الـ handler بيـ return `IRQ_NONE` | الـ driver مش بيـ clear الـ interrupt flag في الـ hardware register |
| Edge IRQ بيتـ miss | Counter في `/proc/interrupts` مش بيزيد صح | الـ pulse قصيرة جداً، ضبط الـ debounce أو غيّر لـ level-triggered |
| APIC routing غلط (x86) | `do_IRQ: N.N No irq handler for vector` | راجع الـ BIOS/UEFI settings، أو الـ ACPI tables |
| GIC misconfiguration (ARM) | `irq: type mismatch, failed to map hwirq` | راجع الـ Device Tree: `interrupts` property و `interrupt-cells` |
| Shared IRQ conflict | `request_irq failed with error -EBUSY` | تأكد من الـ `IRQF_SHARED` في كل الـ drivers اللي بتشارك الـ IRQ |
| CPU affinity غلط | الـ interrupts مش متوزعة على الـ CPUs | راجع الـ `irqbalance` daemon أو اضبط manually |

#### 5. Device Tree debugging

```bash
# dump الـ Device Tree كـ DTS
dtc -I fs -O dts /sys/firmware/devicetree/base -o /tmp/current.dts 2>/dev/null
less /tmp/current.dts

# ابحث عن كل الـ interrupt declarations
grep -n 'interrupts\|interrupt-parent\|interrupt-controller' /tmp/current.dts

# تحقق من الـ interrupt-cells لكل controller
grep -B2 'interrupt-cells' /tmp/current.dts

# مثال على DTS صح لـ GIC على ARM:
# interrupt-controller@8000000 {
#     compatible = "arm,gic-v3";
#     #interrupt-cells = <3>;   ← 3 cells: type, number, flags
#     interrupt-controller;
# };
# my-device {
#     interrupts = <GIC_SPI 27 IRQ_TYPE_LEVEL_HIGH>;
#     interrupt-parent = <&gic>;
# };

# تحقق إن الـ mapping صح
cat /sys/kernel/debug/irq/domains/*/name   # اعرف اسم الـ domain
# ثم
ls /sys/kernel/debug/irq/domains/<domain-name>/

# تحقق من الـ hwirq → virq mapping
grep -r . /sys/kernel/debug/irq/irqs/*/hwirq 2>/dev/null | sort -t/ -k7 -n
```

---

### Practical Commands — ملخص جاهز للنسخ

#### فحص عام للـ IRQ system

```bash
#!/bin/bash
# irq_debug_overview.sh — فحص شامل لحالة الـ IRQs

echo "=== Total IRQ count ==="
# عدد الـ IRQs المسجلة في الكيرنل
ls /sys/kernel/irq/ | wc -l

echo "=== Active IRQs (have handlers) ==="
grep -r . /sys/kernel/irq/*/actions 2>/dev/null | \
    grep -v '^$' | \
    sed 's|/sys/kernel/irq/||;s|/actions:| -> |'

echo "=== IRQ counts per CPU ==="
cat /proc/interrupts

echo "=== Spurious/Error counters ==="
tail -3 /proc/interrupts
```

#### فحص IRQ معين بالتفصيل

```bash
#!/bin/bash
# irq_inspect.sh <irq_number>
IRQ=${1:-27}

echo "=== IRQ $IRQ Details ==="
echo "Actions:   $(cat /sys/kernel/irq/$IRQ/actions 2>/dev/null)"
echo "Chip:      $(cat /sys/kernel/irq/$IRQ/chip_name 2>/dev/null)"
echo "HW IRQ:    $(cat /sys/kernel/irq/$IRQ/hwirq 2>/dev/null)"
echo "Type:      $(cat /sys/kernel/irq/$IRQ/type 2>/dev/null)"
echo "Per-CPU:   $(cat /sys/kernel/irq/$IRQ/per_cpu 2>/dev/null)"
echo "Wakeup:    $(cat /sys/kernel/irq/$IRQ/wakeup 2>/dev/null)"

if [ -d /sys/kernel/debug/irq/irqs/$IRQ ]; then
    echo ""
    echo "=== debugfs details ==="
    for f in /sys/kernel/debug/irq/irqs/$IRQ/*; do
        echo "$(basename $f): $(cat $f 2>/dev/null)"
    done
fi

echo ""
echo "=== Hit counts per CPU ==="
grep "^ *$IRQ:" /proc/interrupts
```

#### تتبع الـ IRQ handlers بـ ftrace

```bash
#!/bin/bash
# irq_trace.sh <irq_number> <duration_seconds>
IRQ=${1:-27}
DUR=${2:-3}
TDIR=/sys/kernel/debug/tracing

echo 0 > $TDIR/tracing_on
echo > $TDIR/trace
echo nop > $TDIR/current_tracer

# فعّل الـ events
echo 1 > $TDIR/events/irq/irq_handler_entry/enable
echo 1 > $TDIR/events/irq/irq_handler_exit/enable

# filter على الـ IRQ المطلوب
echo "irq == $IRQ" > $TDIR/events/irq/irq_handler_entry/filter
echo "irq == $IRQ" > $TDIR/events/irq/irq_handler_exit/filter

echo 1 > $TDIR/tracing_on
echo "Tracing IRQ $IRQ for $DUR seconds..."
sleep $DUR
echo 0 > $TDIR/tracing_on

echo "=== Results ==="
cat $TDIR/trace | grep -v '^#' | head -100

# تفسير الـ output:
# kworker/0:1-123   [000] .... 1234.5: irq_handler_entry: irq=27 name=xhci_hcd
# kworker/0:1-123   [000] .... 1234.5: irq_handler_exit:  irq=27 ret=handled
# "ret=unhandled" يعني الـ handler مش صاحب الـ IRQ دي (shared IRQ)
```

#### مراقبة الـ IRQ rate في real-time

```bash
#!/bin/bash
# irq_rate.sh — حساب معدل الـ IRQs في الثانية
prev=$(awk '/^ *[0-9]/{sum=0; for(i=2;i<=NF-2;i++) sum+=$i; print $1, sum}' /proc/interrupts)
sleep 1
curr=$(awk '/^ *[0-9]/{sum=0; for(i=2;i<=NF-2;i++) sum+=$i; print $1, sum}' /proc/interrupts)

# حساب الفرق
diff <(echo "$prev") <(echo "$curr") | grep '^>' | while read _ irq count; do
    prev_count=$(echo "$prev" | awk -v i="$irq" '$1==i{print $2}')
    rate=$((count - prev_count))
    [ $rate -gt 0 ] && printf "IRQ %-4s: %d/sec\n" "$irq" "$rate"
done | sort -k2 -rn | head -20
```

#### تحقق من الـ for_each_irq_desc iteration

```c
/* كود kernel module للـ debug — يطبع كل الـ IRQ descriptors */
#include <linux/module.h>
#include <linux/irqnr.h>
#include <linux/irqdesc.h>

static int __init irq_debug_init(void)
{
    struct irq_desc *desc;
    unsigned int irq;
    unsigned int total = 0, active = 0;

    pr_info("Total nr_irqs = %u\n", irq_get_nr_irqs());

    /* استخدام الـ macro المعرف في irqnr.h */
    for_each_irq_desc(irq, desc) {
        total++;
        if (irq_desc_has_action(desc)) {
            active++;
            pr_info("IRQ %u: handler=%ps depth=%u count=%u\n",
                    irq,
                    desc->handle_irq,
                    desc->depth,
                    desc->irq_count);
        }
    }

    pr_info("Total: %u, Active: %u\n", total, active);
    return 0;
}

static void __exit irq_debug_exit(void) {}
module_init(irq_debug_init);
module_exit(irq_debug_exit);
MODULE_LICENSE("GPL");

/* تشغيل:
 * make -C /lib/modules/$(uname -r)/build M=$PWD modules
 * insmod irq_debug.ko
 * dmesg | grep 'IRQ\|nr_irqs'
 * rmmod irq_debug
 */
```

#### تفسير نتائج `/proc/interrupts`

```
           CPU0       CPU1       CPU2       CPU3
  0:          9          0          0          0   IO-APIC    2-edge      timer
 27:       4521       3890       4102       3955   PCI-MSI 524288-edge   xhci_hcd
 28:          0          0          0          0   PCI-MSI 360448-edge   aerdrv
NMI:          5          5          5          5   Non-maskable interrupts
LOC:     123456     123789     122345     124001   Local timer interrupts
ERR:          0                               ← لو > 0 فيه مشكلة في الـ APIC
MIS:          0                               ← missed interrupts (spurious)

التفسير:
- الـ IRQ 27 متوزع بشكل متوازن على الـ 4 CPUs → الـ irqbalance شغال صح
- الـ IRQ 28 = 0 → الـ device مش بيتكلم (أو مش initialized)
- ERR > 0 → مشكلة hardware في الـ APIC/PIC
- MIS > 0 → spurious interrupts → افحص الـ trigger type
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — interrupt storm بيعطّل الـ system

#### العنوان
**Interrupt Storm** من UART غير مضبوط بيعطّل الـ gateway بالكامل

#### السياق
شركة بتعمل **industrial gateway** بيجمع بيانات من sensors عبر UART على **RK3562**. المنتج شغّال في مصنع. فجأة الـ gateway بدأ يتجمّد بعد بضع ساعات من التشغيل.

#### المشكلة
الـ UART line مش مربوط صح كهربائيًا، بيولّد noise يتحوّل لـ spurious interrupts بشكل متواصل. الـ system بيبقى overwhelmed.

#### التحليل
المهندس كتب debugging loop باستخدام `for_each_irq_desc`:

```c
#include <linux/irqnr.h>

// dump all IRQ descriptors to find the noisy one
static void dump_irq_stats(void)
{
    struct irq_desc *desc;
    unsigned int irq;

    for_each_irq_desc(irq, desc) {
        // irq_to_desc() called internally — NULL check done by the macro
        if (desc->irqs_unhandled > 1000)
            pr_warn("IRQ %u: unhandled=%u — possible storm\n",
                    irq, desc->irqs_unhandled);
    }
}
```

الـ macro `for_each_irq_desc` بيعمل الآتي:
1. بيقرأ `irq_get_nr_irqs()` مرة واحدة بس في أول الـ outer loop — بيخزّنها في `__nr_irqs__` عشان تكون consistent طول الـ iteration.
2. لكل `irq` من 0 للـ `__nr_irqs__`، بيجيب `irq_to_desc(irq)`.
3. لو `desc == NULL` (IRQ مش allocated)، بيعمل `;` فاضي ويعدّي — **مش بيتجاهل، بيتخطى بشكل آمن**.
4. لما `desc != NULL`، بينفّذ الـ body.

لقوا IRQ 45 (UART2) عنده `irqs_unhandled` وصلت لـ 50,000+.

#### الحل
```bash
# تأكيد من /proc
cat /proc/interrupts | grep "45:"

# تعطيل الـ IRQ مؤقتًا للتشخيص
echo 0 > /proc/irq/45/smp_affinity_list

# في الـ DT: إضافة pull-up على الـ RX line
# في rk3562-gateway.dts:
&uart2 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart2_xfer>;
    status = "okay";
};

&pinctrl {
    uart2 {
        uart2_xfer: uart2-xfer {
            rockchip,pins =
                <1 RK_PB2 1 &pcfg_pull_up>;  /* RX with pull-up */
        };
    };
};
```

#### الدرس المستفاد
الـ `for_each_irq_desc` بيوفّر طريقة آمنة وشاملة لـ iterate على كل الـ IRQs — الـ NULL check المدمج فيه بيحمي من الـ sparse IRQ spaces. لازم تستخدمه في أي debugging tool بدل ما تعمل loop يدوي.

---

### السيناريو 2: Android TV Box على Allwinner H616 — IRQ numbering بعد إضافة peripheral

#### العنوان
**IRQ Renumbering** بعد إضافة USB hub بيكسر الـ HDMI audio driver

#### السياق
منتج **Android TV Box** على **Allwinner H616**. المهندس أضاف USB hub جديد للـ BOM. بعد الـ kernel rebuild، الـ HDMI audio بطّل يشتغل.

#### المشكلة
الـ driver القديم كان بيستخدم hardcoded IRQ number (32) بدل ما يستخدم `irq_to_desc()` + name-based lookup. لما USB hub اتضاف، الـ IRQ numbers اتغيّرت.

#### التحليل
الـ `irq_get_nr_irqs()` بترجع العدد الكلي للـ IRQs allocated في النظام. لما USB hub اتضاف، الـ `irq_set_nr_irqs()` اتاستدعت بقيمة أكبر، وإعادة الترتيب غيّرت الـ mapping.

المهندس استخدم `for_each_active_irq` عشان يعمل scan:

```c
#include <linux/irqnr.h>
#include <linux/irqdesc.h>

// find IRQ by its registered name
static int find_irq_by_name(const char *name)
{
    unsigned int irq;

    for_each_active_irq(irq) {
        struct irq_desc *desc = irq_to_desc(irq);
        if (desc && desc->action && desc->action->name)
            if (strcmp(desc->action->name, name) == 0)
                return irq;
    }
    return -ENOENT;
}
```

`for_each_active_irq` بيستخدم `irq_get_next_irq(offset)` — بيتخطى الـ IRQs الفاضية ويرجع بس اللي عندها `irq_desc` فعلي. ده أسرع من `for_each_irq_nr` في الأنظمة اللي فيها IRQ sparse space.

#### الحل
```c
/* بدل الـ hardcoded number */
static int hdmi_audio_irq;

static int hdmi_audio_probe(struct platform_device *pdev)
{
    /* استخدام platform_get_irq بيعتمد على DT — مش على hardcoded number */
    hdmi_audio_irq = platform_get_irq_byname(pdev, "hdmi-audio");
    if (hdmi_audio_irq < 0)
        return hdmi_audio_irq;

    return request_irq(hdmi_audio_irq, hdmi_audio_handler,
                       IRQF_SHARED, "hdmi-audio", pdev);
}
```

#### الدرس المستفاد
الـ `irq_get_nr_irqs()` / `irq_set_nr_irqs()` بيوضّحوا إن الـ IRQ space ديناميكي. hardcoding IRQ numbers جريمة — استخدم دايمًا `platform_get_irq()` أو اللي بيبني على `irq_to_desc()`.

---

### السيناريو 3: IoT Sensor على STM32MP1 — suspend/resume بيكسر I2C

#### العنوان
بعض الـ IRQs بتصحى الـ system من الـ sleep — **spurious wakeup** على STM32MP1

#### السياق
جهاز **IoT sensor** صغير يشتغل على بطارية. بيستخدم **STM32MP1** مع I2C sensor. المشكلة إن الجهاز بيصحى من الـ deep sleep كل دقيقتين بدون سبب واضح، بيصرف البطارية بسرعة.

#### المشكلة
أحد الـ I2C IRQs مش مضبوط كـ `IRQF_NO_SUSPEND` ولا كـ wake source حقيقي، بس بيولّد edge خلال الـ suspend transition.

#### التحليل
المهندس استخدم `for_each_irq_desc_reverse` عشان يبدأ من أعلى IRQ number — الـ peripherals المتأخرة في الـ probe غالبًا بتاخد أرقام أعلى:

```c
#include <linux/irqnr.h>
#include <linux/irqdesc.h>
#include <linux/irq.h>

// scan IRQs in reverse to find wake-enabled ones
static void audit_wake_irqs(void)
{
    struct irq_desc *desc;
    int irq; /* must be signed — for_each_irq_desc_reverse uses irq >= 0 */

    for_each_irq_desc_reverse(irq, desc) {
        /*
         * for_each_irq_desc_reverse starts at irq_get_nr_irqs()-1
         * and counts DOWN to 0. irq must be signed int because
         * the loop condition is (irq >= 0).
         */
        if (desc->wake_depth > 0)
            pr_info("IRQ %d [%s]: wake_depth=%u\n",
                    irq,
                    desc->action ? desc->action->name : "unknown",
                    desc->wake_depth);
    }
}
```

لاحظ إن `for_each_irq_desc_reverse` بيشترط إن `irq` يكون `int` مش `unsigned int` — لأن الـ loop condition هي `irq >= 0`، ولو كانت unsigned هيبقى الشرط دايمًا true وهيحصل infinite loop.

الـ scan كشف إن I2C4 IRQ عنده `wake_depth = 1` بالغلط.

#### الحل
```c
/* في الـ i2c sensor driver */
static int sensor_probe(struct platform_device *pdev)
{
    int irq = platform_get_irq(pdev, 0);

    /* تأكيد إن الـ IRQ مش مفعّل كـ wake source */
    irq_set_irq_wake(irq, 0);  /* disable wake explicitly */

    return request_threaded_irq(irq, NULL, sensor_thread_fn,
                                IRQF_ONESHOT | IRQF_NO_SUSPEND,
                                "stm32-sensor", pdev);
}
```

```dts
/* في الـ DTS */
&i2c4 {
    sensor@48 {
        compatible = "vendor,temp-sensor";
        reg = <0x48>;
        interrupt-parent = <&gpioe>;
        interrupts = <3 IRQ_TYPE_EDGE_FALLING>;
        /* wakeup-source; */  /* removed — was causing spurious wakeup */
    };
};
```

#### الدرس المستفاد
الـ `for_each_irq_desc_reverse` مفيد لما تعرف إن المشكلة في الـ peripherals المضافة حديثًا (أرقامها أعلى). لازم تنتبه لـ signed/unsigned في المتغير `irq` لأن الـ macro صمّمه حسب الـ loop direction.

---

### السيناريو 4: Automotive ECU على i.MX8 — تحقق من IRQ budget قبل الـ runtime

#### العنوان
**IRQ Budget Validation** في بداية التشغيل على **i.MX8** لـ AUTOSAR-like system

#### السياق
شركة بتبني **automotive ECU** على **i.MX8**. المنتج بيشغّل RTOS on one core وLinux on another (AMP setup). الـ safety requirement إن عدد الـ IRQs الفعلية المستخدمة في Linux لازم ميعدّيش حد معين.

#### المشكلة
بعد كل kernel update، الـ validation team لازم يتحقق يدويًا من عدد الـ IRQs. محتاجين طريقة آلية.

#### التحليل
الـ `irq_get_nr_irqs()` بترجع الـ maximum allocated IRQ number (مش عدد الـ IRQs الفعلية). عشان تعرف الـ active ones بالضبط، محتاج تستخدم `for_each_active_irq` أو `for_each_irq_desc`:

```c
#include <linux/irqnr.h>
#include <linux/irqdesc.h>

#define ECU_MAX_ALLOWED_IRQS 64  /* automotive safety budget */

/* called at boot from initcall */
static int __init ecu_irq_budget_check(void)
{
    struct irq_desc *desc;
    unsigned int irq;
    unsigned int active_count = 0;

    /* for_each_irq_desc skips NULL descs automatically */
    for_each_irq_desc(irq, desc) {
        if (desc->action)  /* has at least one registered handler */
            active_count++;
    }

    pr_info("ECU IRQ budget: %u active / %u max / limit %u\n",
            active_count, irq_get_nr_irqs(), ECU_MAX_ALLOWED_IRQS);

    if (active_count > ECU_MAX_ALLOWED_IRQS) {
        pr_crit("IRQ budget exceeded! System halted for safety.\n");
        panic("IRQ budget violation");
    }

    return 0;
}
late_initcall(ecu_irq_budget_check);
```

الـ `for_each_irq_desc` هنا بيعمل snapshot من `irq_get_nr_irqs()` في الأول — لو أي driver حاول يضيف IRQ جديد أثناء الـ iteration، الـ `__nr_irqs__` مش هيتغير في نفس الـ loop pass، وده بيضمن consistency.

#### الحل
الكود نفسه هو الحل. بالإضافة لـ:

```bash
# تحقق يدوي سريع
cat /proc/interrupts | grep -v "^CPU" | wc -l

# أو
wc -l /proc/interrupts
```

#### الدرس المستفاد
الـ `irq_get_nr_irqs()` معلّمة بـ `__pure` — يعني الـ compiler ممكن يـ cache نتيجتها لو استدعيتها أكتر من مرة بدون side effects. لكن الـ `for_each_irq_desc` هو اللي بيضمن الـ snapshot الصحيح لأنه بيحفظ القيمة في `__nr_irqs__` مرة واحدة.

---

### السيناريو 5: Custom Board Bring-up على AM62x — IRQ لما بيظهر في /proc بس handler مش بيتنفذ

#### العنوان
**SPI IRQ** موجود في `/proc/interrupts` بس callbacks مش بتتنفّذ على **AM62x**

#### السياق
فريق bring-up شغّال على **custom board** مبنية على **TI AM62x** (automotive-grade). ضافوا SPI flash controller جديد. الـ driver اتكتب وشغّال على eval kit، بس على الـ custom board الـ interrupt callback مش بيتنفّذ أبدًا.

#### المشكلة
الـ IRQ line متوصّل غلط في الـ PCB — الـ SPI controller IRQ متوصّل بـ GPIO line غير مضبوطة كـ input. النتيجة: الـ IRQ مسجّل وظاهر في `/proc/interrupts`، بس count = 0 دايمًا.

#### التحليل
المهندس كتب diagnostic باستخدام `for_each_irq_nr` — أبسط الـ macros، بيمشي على كل الأرقام حتى لو مفيش `irq_desc`:

```c
#include <linux/irqnr.h>
#include <linux/irqdesc.h>

/*
 * for_each_irq_nr: iterates irq from 0 to irq_get_nr_irqs()-1
 * does NOT call irq_to_desc — just gives the number.
 * Use when you need the number itself, not the descriptor.
 */
static void dump_zero_count_irqs(void)
{
    unsigned int irq;

    for_each_irq_nr(irq) {
        struct irq_desc *desc = irq_to_desc(irq);  /* manual lookup */

        if (!desc || !desc->action)
            continue;

        /* check if IRQ is registered but never fired */
        if (desc->tot_count == 0 && desc->action->name)
            pr_warn("IRQ %u [%s]: registered but count=0 — check HW\n",
                    irq, desc->action->name);
    }
}
```

الفرق بين `for_each_irq_nr` و `for_each_irq_desc`:
| الـ macro | بيعمل irq_to_desc تلقائي؟ | NULL check تلقائي؟ | متى تستخدمه؟ |
|---|---|---|---|
| `for_each_irq_nr` | لأ | لأ | لما تحتاج الرقم بس |
| `for_each_irq_desc` | أيوه | أيوه | لما تحتاج الـ descriptor |
| `for_each_active_irq` | لأ (يدوي) | — | لتجاهل الـ sparse gaps بسرعة |
| `for_each_irq_desc_reverse` | أيوه | أيوه | من الأعلى للأسفل |

الـ scan كشف إن `spi2-irq` عنده count = 0 من بداية التشغيل.

#### الحل
```bash
# تحقق من GPIO direction
gpioinfo | grep -A2 "spi2"

# في الـ DTS — كان فيه خطأ في الـ interrupt-parent
# كان:
&spi2 {
    interrupt-parent = <&gpio0>;
    interrupts = <15 IRQ_TYPE_LEVEL_HIGH>;
};

# المفروض يكون:
&spi2 {
    interrupt-parent = <&main_gpio1>;  /* correct GPIO bank */
    interrupts = <23 IRQ_TYPE_LEVEL_HIGH>;
};
```

بعد تصحيح الـ DTS وإعادة الـ flash، الـ `/proc/interrupts` بدأ يعدّ normally.

#### الدرس المستفاد
الـ `for_each_irq_nr` هو الـ raw iterator — مش بيعمل أي checks. مفيد لما تعمل correlation بين IRQ numbers وأشياء خارجية (GPIO banks, DT nodes). دايمًا تحقق من `tot_count` في الـ `irq_desc` كأول خطوة لتشخيص الـ "registered but never fires" problem.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| [`Documentation/core-api/genericirq.rst`](https://docs.kernel.org/core-api/genericirq.html) | الـ reference الأساسي لـ generic IRQ subsystem — يشرح `irq_desc`، الـ flow handlers، وعلاقة الـ Linux IRQ number بالـ hardware |
| [`Documentation/core-api/irq/irq-domain.rst`](https://docs.kernel.org/core-api/irq/irq-domain.html) | مكتبة الـ `irq_domain` — كيف يُعمل الـ mapping بين الـ hwirq والـ Linux IRQ number اللي تتعامل معاه `irqnr.h` |
| [`Documentation/core-api/irq/concepts.rst`](https://docs.kernel.org/core-api/irq/concepts.html) | تعريف الـ IRQ من منظور الـ kernel — نقطة بداية ممتازة |
| `include/linux/irqnr.h` | الملف المصدري نفسه — يحتوي `for_each_irq_desc`، `for_each_active_irq`، `irq_get_nr_irqs` |
| `include/uapi/linux/irqnr.h` | الجزء الـ userspace-visible من نفس الـ header |
| `kernel/irq/irqdomain.c` | تنفيذ الـ `irq_domain` mapping library |
| `kernel/irq/irqdesc.c` | تنفيذ `irq_to_desc()`، `irq_get_nr_irqs()`، `irq_get_next_irq()` |

---

### مقالات LWN.net

دي أهم المقالات على [lwn.net](https://lwn.net) اللي تتعلق بالـ IRQ numbering وبنية الـ subsystem:

| المقال | الأهمية |
|--------|---------|
| [A new generic IRQ layer](https://lwn.net/Articles/184750/) | يشرح إعادة كتابة الـ IRQ layer لتعميمها — الـ foundation اللي بُني عليه `irqnr.h` الحالي |
| [IRQ-domain.txt — v3.7](https://lwn.net/Articles/487684/) | snapshot من توثيق الـ `irq_domain` حين تقديمه — يوضح ليه الـ Linux IRQ numbers انفصلت عن الـ hardware numbers |
| [Documentation/IRQ-domain.txt (v3.19)](https://lwn.net/Articles/625547/) | نسخة أحدث من نفس التوثيق بعد نضوج الـ API |
| [Clean up IRQ mapping code](https://lwn.net/Articles/72844/) | مناقشة مبكرة لتنظيف الـ IRQ mapping — تاريخ الـ `for_each_irq` macros |
| [generic irq subsystem, x64 port](https://lwn.net/Articles/104847/) | الـ port للـ x86-64 — يُظهر كيف استُخدم `nr_irqs` وعدد الـ descriptors |
| [new irq tracer](https://lwn.net/Articles/320336/) | إضافة الـ tracing للـ IRQ subsystem — مرتبط بالـ iteration على الـ IRQs |
| [Linux generic IRQ handling — kernel docs على lwn](https://static.lwn.net/kerneldoc/core-api/genericirq.html) | mirror للتوثيق الرسمي على static.lwn.net |

---

### Kernel Commits ذات الصلة

| الـ Commit / الملف | الوصف |
|--------------------|-------|
| [`include/linux/irqnr.h` — git history](https://github.com/torvalds/linux/commits/master/include/linux/irqnr.h) | تاريخ كامل لتعديلات الملف على GitHub |
| [`kernel/irq/irqdesc.c` — git history](https://github.com/torvalds/linux/commits/master/kernel/irq/irqdesc.c) | تاريخ تنفيذ `irq_to_desc()` و`irq_get_nr_irqs()` |
| [torvalds/linux — `irqnr.h` الحالي](https://github.com/torvalds/linux/blob/master/include/linux/irqnr.h) | الملف في أحدث إصدار على master |
| [git.kernel.org — log لـ irqnr.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/include/linux/irqnr.h) | الـ official kernel git — يعرض كل commit غيّر الملف ده |

---

### مناقشات Mailing List

| الرابط | الموضوع |
|--------|---------|
| [LKML.org](https://lkml.org/) | الأرشيف الرئيسي للـ Linux Kernel Mailing List |
| [lore.kernel.org/linux-kernel](https://lore.kernel.org/linux-kernel/) | الأرشيف الرسمي الحديث — ابحث عن `irqnr` أو `nr_irqs` أو `for_each_irq_desc` |
| [Kernel-managed IRQ affinity discussion](https://lists.gt.net/linux/kernel/3554349) | مناقشة real-time IRQ affinity — مرتبطة بالـ iteration على الـ IRQs |
| [ERROR: Disabling IRQ — mailing list](https://lists.gt.net/linux/kernel/487089) | مثال على مشاكل الـ IRQ numbering في الـ production |

---

### KernelNewbies.org

| الصفحة | المحتوى |
|--------|---------|
| [Internals of Interrupt Handling](https://kernelnewbies.org/KernelHacking-HOWTO/Overview_of_the_Kernel_Source_Code/Internals_of_Interrupt_Handling) | نظرة داخلية على آليات الـ interrupt handling — يشمل `irq_desc` والـ IRQ flow |
| [Details of `do_IRQ()` function](https://kernelnewbies.org/KernelHacking-HOWTO/Overview_of_the_Kernel_Source_Code/Internals_of_Interrupt_Handling/Details_of_do_IRQ()_function) | تفاصيل `do_IRQ()` — نقطة الدخول التي تستخدم الـ IRQ number من `irqnr.h` |
| [New HOWTO: Exceptions and Interrupts](https://kernelnewbies.org/New_Kernel_Hacking_HOWTO/Subsystems/Exceptions_and_Interrupts_Handling) | النسخة الحديثة من نفس التوثيق |
| [do_IRQ() — New HOWTO](https://kernelnewbies.org/New_Kernel_Hacking_HOWTO/Subsystems/Exceptions_and_Interrupts_Handling/Details_of_do_IRQ()_function) | شرح محدّث لـ `do_IRQ()` مع الـ generic IRQ layer |

---

### eLinux.org

| الصفحة | المحتوى |
|--------|---------|
| [Soft IRQ Threads](https://elinux.org/Soft_IRQ_Threads) | تحويل الـ softirq لـ threads — يوضح العلاقة بين الـ IRQ numbers والـ threading |
| [Fix IRQ Domain DT support issues and gpio IRQ](https://elinux.org/Fix_IRQ_Domain_DT_support_issues_and_gpio_IRQ) | إصلاح مشاكل الـ `irq_domain` مع الـ Device Tree — عملي جداً لفهم الـ IRQ numbering في الـ embedded |
| [Real Time Working Group](https://elinux.org/Real_Time_Working_Group) | جهود الـ PREEMPT_RT وتأثيرها على الـ IRQ threading والـ numbering |

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 10**: *Interrupt Handling* — يشرح `request_irq()`، الـ IRQ numbers، و`/proc/interrupts`
- متاح مجاناً: [lwn.net/Kernel/LDD2/ch09.lwn](https://lwn.net/Kernel/LDD2/ch09.lwn) (النسخة الثانية) و[free-electrons.com/doc/books/ldd3.pdf](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 7**: *Interrupts and Interrupt Handlers* — يغطي `irq_desc`، الـ IRQ numbers، و`for_each_irq_desc` بشكل عملي
- مرجع لا غنى عنه لفهم السياق الكامل لـ `irqnr.h`

#### Understanding the Linux Kernel — Bovet & Cesati (3rd Edition)
- **الفصل 4**: *Interrupts and Exceptions* — تفاصيل معمّقة عن الـ IRQ descriptor tables والـ numbering
- [O'Reilly preview](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/ch04s06.html)

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 14**: *Kernel Debugging Techniques* — يشمل قراءة `/proc/interrupts` وتتبع الـ IRQ numbers في الـ embedded systems
- مهم لفهم كيف تختلف الـ IRQ numbers بين الـ architectures

---

### توثيق إضافي على الويب

| المصدر | الرابط |
|--------|--------|
| الـ Kernel docs الرسمية — IRQ concepts | [docs.kernel.org/core-api/irq/concepts.html](https://docs.kernel.org/core-api/irq/concepts.html) |
| الـ Kernel docs — generic IRQ (v4.18) | [kernel.org/doc/html/v4.18/core-api/genericirq.html](https://www.kernel.org/doc/html/v4.18/core-api/genericirq.html) |
| Linux Kernel Labs — Interrupts | [linux-kernel-labs.github.io — interrupts](https://linux-kernel-labs.github.io/refs/heads/master/lectures/interrupts.html) |
| Linux Inside — External Hardware Interrupts | [0xax.gitbooks.io/linux-insides](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-7.html) |
| Opensource.com — How Linux handles interrupts | [opensource.com/article/20/10/linux-kernel-interrupts](https://opensource.com/article/20/10/linux-kernel-interrupts) |
| IRQ-domain.txt — kernel.org | [kernel.org/doc/Documentation/IRQ-domain.txt](https://www.kernel.org/doc/Documentation/IRQ-domain.txt) |

---

### Search Terms للبحث عن معلومات إضافية

```
linux kernel irq_desc for_each_irq_desc
linux kernel irq_get_nr_irqs implementation
linux irq_domain hwirq to virq mapping
linux kernel NR_IRQS sparse irq
linux irq_to_desc radix tree
kernel irq_alloc_descs CONFIG_SPARSE_IRQ
linux for_each_active_irq irq_get_next_irq
linux kernel interrupt descriptor table irqnr
LKML "irq_get_nr_irqs" OR "for_each_irq_desc"
linux kernel generic irq subsystem history
```
## Phase 8: Writing simple module

### الفكرة

**`irq_get_next_irq()`** هي function مُصدَّرة بتاخد `offset` وترجع رقم الـ IRQ الـ active التالي — ده بالضبط اللي بيستخدمه الـ macro `for_each_active_irq`. هنعمل kprobe على الـ `irq_get_next_irq` عشان نشوف كل مرة بيتسأل فيها عن الـ IRQ التالي ونطبع الـ `offset` اللي جاي وقيمة الـ return.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_irqnr.c
 * Hook irq_get_next_irq() to trace IRQ iteration requests.
 */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit        */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe          */
#include <linux/irqnr.h>        /* irq_get_next_irq prototype              */
#include <linux/printk.h>       /* pr_info                                  */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDoc-AR");
MODULE_DESCRIPTION("kprobe on irq_get_next_irq to trace IRQ iteration");

/* ------------------------------------------------------------------ */
/* pre-handler: يتنفذ قبل ما الـ function الأصلية تشتغل               */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * الـ first argument بيكون في rdi على x86-64.
     * regs->di يحمل قيمة الـ offset اللي اتبعت لـ irq_get_next_irq().
     */
    unsigned int offset = (unsigned int)regs->di;

    pr_info("kprobe_irqnr: irq_get_next_irq called with offset=%u\n", offset);
    return 0; /* 0 = continue normal execution */
}

/* ------------------------------------------------------------------ */
/* post-handler: يتنفذ بعد ما الـ function تخلص بنجاح                 */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * الـ return value على x86-64 بيكون في rax.
     * نطبعه عشان نشوف أنهي IRQ active اتعمله discover.
     */
    unsigned int next_irq = (unsigned int)regs->ax;

    pr_info("kprobe_irqnr: irq_get_next_irq returned next_irq=%u\n", next_irq);
}

/* ------------------------------------------------------------------ */
/* struct kprobe: بتحدد الـ symbol المستهدف والـ handlers              */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "irq_get_next_irq",  /* الاسم المُصدَّر في kallsyms  */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* module_init: بيسجّل الـ kprobe في الـ kernel                       */
/* ------------------------------------------------------------------ */
static int __init kprobe_irqnr_init(void)
{
    int ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_irqnr: register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("kprobe_irqnr: planted on %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit: لازم نشيل الـ kprobe قبل ما الـ module يتـunload      */
/* ------------------------------------------------------------------ */
static void __exit kprobe_irqnr_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe_irqnr: removed from %s\n", kp.symbol_name);
}

module_init(kprobe_irqnr_init);
module_exit(kprobe_irqnr_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `<linux/module.h>` | ماكروهات الـ module الأساسية: `MODULE_LICENSE`، `module_init`، `module_exit` |
| `<linux/kprobes.h>` | الـ `struct kprobe` وفنكشنات `register_kprobe` / `unregister_kprobe` |
| `<linux/irqnr.h>` | prototype بتاع `irq_get_next_irq` نفسها عشان الكود يكون clean |
| `<linux/printk.h>` | الـ `pr_info` و `pr_err` للـ logging |

---

#### الـ `handler_pre`

الـ pre-handler بيشتغل **قبل** ما الـ function الأصلية تتنفذ، فممكن نشوف الـ arguments كما دخلت. على x86-64 أول argument بيتحط في `regs->di`، فبنقرأ منه الـ `offset` اللي بيطلب الكود اللي فوق `for_each_active_irq` إيه الـ IRQ التالي بعده.

---

#### الـ `handler_post`

الـ post-handler بيشتغل **بعد** ما الـ function ترجع بنجاح. الـ return value على x86-64 بيكون في `regs->ax`، فبنطبع رقم الـ IRQ الـ active اللي اتوجد، ده بيخلينا نعمل correlation بين كل request وإجابته.

---

#### الـ `struct kprobe`

الـ `.symbol_name` بيخلي الـ kernel يدور على العنوان بنفسه من الـ kallsyms بدل ما نحدد عنوان manually — أكتر portable وأأمن.

---

#### الـ `module_init` / `module_exit`

`register_kprobe` بتزرع breakpoint في الـ kernel text عند عنوان الـ function. لو فشلت (مثلاً الـ symbol مش موجود أو الـ kprobes disabled) بنرجع error ومنكملش. في الـ exit **لازم** نعمل `unregister_kprobe` قبل الـ unload عشان لو فضل الـ breakpoint موجود من غير handler هيحصل kernel crash فوراً.

---

### طريقة البناء والتشغيل

```bash
# Makefile بسيط جنب الـ .c
obj-m += kprobe_irqnr.o

# build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# insert
sudo insmod kprobe_irqnr.ko

# شوف الـ output (أي حاجة بتعمل for_each_active_irq هتطبع)
sudo dmesg -w | grep kprobe_irqnr

# remove
sudo rmmod kprobe_irqnr
```

> **ملاحظة:** الـ kprobes لازم تكون مفعّلة في الـ kernel config — `CONFIG_KPROBES=y`. لو الـ kernel compiled بـ `CONFIG_KPROBES_ON_FTRACE` فالأداء أحسن بكتير.
