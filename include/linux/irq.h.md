## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem ده؟

الـ `include/linux/irq.h` هو القلب النابض لـ **IRQ Subsystem** في الـ Linux kernel — المسؤول عنه Thomas Gleixner وموجود في MAINTAINERS تحت اسم **"IRQ SUBSYSTEM"**.

---

### الفكرة من البداية — تخيل معايا

تخيل إن عندك مطعم كبير (الـ CPU)، وفيه كلبير (kernel) بيدير الشغل. الزبائن (hardware devices) زي الكيبورد والنت كارد والديسك — كل واحد فيهم لما يحتاج حاجة بيـ"دق جرس" بدل ما يفضل يصيح طول الوقت.

الـ **Interrupt** ده هو الجرس ده. لما الكيبورد يُدخل حرف، بيبعت إشارة كهربية للـ CPU تقوله "عندك شغل". الـ CPU يوقف اللي بيعمله، يشوف مين دق، ويخدمه، وبعدين يرجع.

المشكلة؟ الأجهزة مختلفة جداً:
- فيه أجهزة بتبعت **Edge** (نبضة واحدة زي زر الجرس)
- فيه أجهزة بتبعت **Level** (بتفضل تصيح لحد ما تتخدم، زي تليفون بيرن)
- فيه آلاف الـ interrupts من أنواع مختلفة
- في SMP عندك أكتر من CPU، فمين يرد على الجرس؟

هنا بييجي دور **Generic IRQ Subsystem** — طبقة تجريد موحدة بين الـ hardware المختلفة والـ kernel code.

---

### الـ `irq.h` — بيعمل إيه بالظبط؟

الملف ده هو **العقد الرسمي (contract)** بين ٣ أطراف:

```
  [Hardware Controller (e.g., GIC, APIC)]
           ↕  (irq_chip callbacks)
  [Generic IRQ Core — kernel/irq/]
           ↕  (irq_desc + irq_action)
  [Device Drivers — request_irq()]
```

الملف بيعرّف:

1. **أنواع الـ trigger** — Edge/Level وكل اتجاهاتهم
2. **`struct irq_chip`** — الـ vtable (جدول الدوال) اللي أي interrupt controller لازم يملأه عشان يكلم الـ kernel
3. **`struct irq_data`** — البيانات اللي بتتنقل من الـ kernel للـ chip عند كل عملية
4. **`struct irq_common_data`** — بيانات مشتركة بين كل الـ irqchips لنفس الـ IRQ
5. **Flow handlers** — الدوال اللي بتحدد "ازاي تتعامل مع النوع ده من الـ interrupt" (handle_level_irq, handle_edge_irq...)
6. **Generic Chip framework** — طريقة أسرع لكتابة drivers الـ interrupt controllers البسيطة
7. **IRQ Matrix** — لإدارة تخصيص الـ interrupt vectors على الـ CPUs (مهم في x86/MSI)
8. **IPI** — الـ Inter-Processor Interrupts، لما CPU يكلم CPU تاني

---

### القصة الكاملة — رحلة Interrupt واحدة

```
Keyboard يضغط مفتاح
        ↓
GPIO Controller يرفع الـ IRQ line
        ↓
Interrupt Controller (e.g., GIC في ARM)
 يشوف الـ hwirq number
        ↓
يترجمه لـ Linux IRQ number (عن طريق irq_domain)
        ↓
يلاقي الـ irq_desc الخاص بيه
        ↓
handle_irq() → handle_edge_irq() أو handle_level_irq()
        ↓
chip->irq_ack() أو chip->irq_mask() حسب النوع
        ↓
ينفذ الـ irqaction chain (كود الـ driver)
        ↓
chip->irq_eoi() (End of Interrupt)
        ↓
الـ CPU يرجع لشغله
```

---

### أهم الـ Structures في الملف

| Structure | دورها |
|---|---|
| `struct irq_chip` | الـ vtable الخاص بـ interrupt controller — أي chip لازم يملأها |
| `struct irq_data` | بيانات الـ interrupt اللي بتتنقل للـ chip functions |
| `struct irq_common_data` | بيانات مشتركة (affinity, msi_desc, node) |
| `struct irq_desc` | (معرفة في irqdesc.h) الـ descriptor الكامل لكل IRQ |
| `struct irq_chip_generic` | chip مجمعة للـ controllers البسيطة |
| `struct irq_chip_type` | نوع flow (edge/level) لكل generic chip |

---

### الـ irq_chip — قلب الموضوع

```c
struct irq_chip {
    const char  *name;
    void        (*irq_mask)(struct irq_data *data);    // اكتم الـ interrupt
    void        (*irq_unmask)(struct irq_data *data);  // فكه
    void        (*irq_ack)(struct irq_data *data);     // أكد الاستلام
    void        (*irq_eoi)(struct irq_data *data);     // End of Interrupt
    int         (*irq_set_affinity)(...);              // حدد مين يرد (SMP)
    int         (*irq_set_type)(...);                  // edge ولا level؟
    // ... والمزيد
};
```

كل interrupt controller في الدنيا (APIC في x86، GIC في ARM، متحكمات GPIO) بيملأ الـ struct ده، والـ kernel بيتكلم معاهم بنفس الطريقة.

---

### الـ irq_data — بيانات تنزل للـ chip

```c
struct irq_data {
    unsigned int     irq;       // رقم الـ IRQ في Linux
    irq_hw_number_t  hwirq;     // رقمه في الـ hardware
    struct irq_chip  *chip;     // مين المسؤول عنه
    struct irq_domain *domain;  // الترجمة بين hwirq و irq
    void             *chip_data;// بيانات خاصة بالـ chip
    struct irq_data  *parent_data; // لو فيه hierarchy (مثلاً GPIO فوق GIC)
};
```

---

### الـ Flow Handlers — منطق التعامل مع الـ Interrupt

| Handler | متى يُستخدم |
|---|---|
| `handle_level_irq` | Level-triggered — يعمل mask أول، ينفذ، بعدين unmask |
| `handle_edge_irq` | Edge-triggered — يعمل ack أول، ممكن يتمسك interrupt تاني |
| `handle_fasteoi_irq` | الأسرع — ARM GIC وغيره، EOI في الآخر بس |
| `handle_simple_irq` | أبسط حالة، بدون mask/unmask |
| `handle_percpu_irq` | كل CPU بيتعامل مع نسخته الخاصة |
| `handle_nested_irq` | للـ interrupts المتشعبة (GPIO فوق I2C مثلاً) |

---

### Generic Chip Framework — للـ Controllers البسيطة

بدل ما كل driver يكتب نفس الكود من أول وجديد، الـ kernel قدّم `irq_chip_generic`:

```
irq_chip_generic
├── reg_base        ← base address في الـ MMIO
├── irq_base        ← أول IRQ number
├── irq_cnt         ← عدد الـ IRQs في الـ chip
└── chip_types[]    ← مصفوفة من irq_chip_type (edge + level مثلاً)
    ├── chip        ← الـ irq_chip نفسه
    ├── regs        ← offsets للـ registers (enable/disable/mask/ack)
    └── handler     ← flow handler المناسب
```

بكده driver الـ interrupt controller كله ممكن يبقى ١٠ أسطر بس بدل مئات الأسطر.

---

### الـ IRQ Matrix — لـ x86 وعالم MSI

في x86، الـ hardware بيتعرف على أرقام من 0 لـ 255 فقط (interrupt vectors). في SMP وعندك كذا CPU وكذا device — مين يأخد إيه؟ الـ `irq_matrix` بيحل المشكلة ده: بيخصص vectors للـ CPUs وبيتتبع المتاح منهم.

---

### الـ IPI — لما الـ CPUs تتكلم مع بعض

**Inter-Processor Interrupt** — CPU 0 محتاج يقول لـ CPU 1 "وقف التنفيذ ودوّر الـ TLB". ده بيتم عن طريق:
- `ipi_send_single()` — لـ CPU واحد بعينه
- `ipi_send_mask()` — لمجموعة CPUs

---

### الملفات المكوّنة للـ Subsystem

**Core Headers:**
- `/include/linux/irq.h` — الملف ده، العقد الرئيسي
- `/include/linux/irqdesc.h` — تعريف `struct irq_desc`
- `/include/linux/irqdomain.h` — الترجمة بين hwirq وlinux irq
- `/include/linux/irqhandler.h` — typedef الـ `irq_flow_handler_t`
- `/include/linux/irqreturn.h` — تعريف `IRQ_HANDLED / IRQ_NONE`
- `/include/linux/irqnr.h` — ماكرو الـ NR_IRQS
- `/include/linux/interrupt.h` — الـ API للـ device drivers (request_irq وغيره)

**Core Implementation:**
- `/kernel/irq/chip.c` — تنفيذ دوال الـ irq_chip
- `/kernel/irq/handle.c` — الـ flow handlers (handle_level_irq etc.)
- `/kernel/irq/manage.c` — request_irq, free_irq, enable/disable
- `/kernel/irq/irqdesc.c` — تخصيص وإدارة الـ irq_desc
- `/kernel/irq/irqdomain.c` — الترجمة بين hwirq وlinux irq
- `/kernel/irq/generic-chip.c` — الـ Generic Chip framework
- `/kernel/irq/affinity.c` — توزيع الـ interrupts على الـ CPUs
- `/kernel/irq/spurious.c` — كشف الـ spurious interrupts
- `/kernel/irq/matrix.c` — إدارة الـ IRQ vectors (x86/MSI)
- `/kernel/irq/ipi.c` — الـ IPI infrastructure
- `/kernel/irq/pm.c` — suspend/resume للـ interrupts
- `/kernel/irq/msi.c` — دعم MSI/MSI-X

**Hardware Drivers (examples):**
- `/drivers/irqchip/irq-gic.c` — ARM GIC (Generic Interrupt Controller)
- `/drivers/irqchip/irq-gic-v3.c` — ARM GICv3
- `/drivers/irqchip/irq-bcm2835.c` — Raspberry Pi
- `/arch/x86/kernel/apic/` — x86 APIC

---

### ليه الملف ده مهم؟

بدونه، كل driver كان هيتكلم مباشرة مع الـ hardware controller بطريقته الخاصة — فوضى كاملة. الـ `irq.h` هو اللي خلى الـ kernel يشتغل على ARM وx86 وRISC-V وكل الـ architectures بنفس الكود، وخلى كتابة interrupt controller driver تبقى موحدة وسهلة.
## Phase 2: شرح الـ Generic IRQ Framework

---

### المشكلة: ليه الـ Framework ده موجود أصلاً؟

في الأيام الأولى للـ kernel، كل architecture كانت بتعمل interrupt handling بطريقتها الخاصة — كود متكرر، منفصل، وصعب الصيانة. كل platform (x86, ARM, MIPS, PowerPC) كانت بتكتب نفس الـ logic من الصفر.

المشكلة أعمق من كده:

1. **Hardware Diversity**: كل SoC وجهاز جديد بيجيب interrupt controller مختلف — GIC على ARM، APIC على x86، وعشرات الـ custom controllers في عالم embedded.
2. **Translation Problem**: الـ hardware بيتكلم بـ "hwirq" (رقم في الـ controller نفسه)، لكن الـ kernel بيتكلم بـ "Linux IRQ number" (رقم virtual). محتاج حد يترجم بينهم.
3. **Flow Diversity**: نفس الـ chip ممكن يعمل level-triggered لبعض الـ lines وedge-triggered لغيرها، وكل flow ليه logic مختلفة.
4. **Chaining**: في بعض الـ SoCs فيه interrupt controllers متداخلة — controller جوا controller. مين يدير ده؟

الـ Generic IRQ Framework ظهر عشان يحل كل ده بطبقة تجريد موحدة.

---

### الحل: ازاي الـ Kernel اتعامل مع المشكلة؟

الـ kernel اعتمد مبدأ الفصل الكامل بين ثلاث طبقات:

| الطبقة | المسؤولية |
|--------|-----------|
| **Hardware Layer** (`irq_chip`) | يتكلم مع registers الـ controller مباشرة |
| **Flow Layer** (`irq_flow_handler_t`) | يقرر إيه اللي يحصل لما interrupt يجي (mask? ack? thread?) |
| **Action Layer** (`irqaction`) | كود الـ driver اللي بيعالج الـ interrupt فعلاً |

الـ kernel بيربطهم كلهم في struct واحد اسمه `irq_desc` — الـ **interrupt descriptor** — اللي بيمثل كل interrupt في الـ system.

---

### التشبيه الواقعي: نظام الاستقبال في المستشفى

تخيل مستشفى كبير فيها أجهزة تنبيه (monitors) لكل مريض:

- **الـ `irq_chip`** = جهاز الـ monitor نفسه — بيعرف يـ beep، يـ silence نفسه، يـ reset alarm. ده hardware محدد.
- **الـ `irq_flow_handler_t`** = بروتوكول قسم الاستقبال — لما alarm يدق، إيه الخطوات؟ (acknowledge الجهاز، اطلب الممرض، wait for response). ده السياسة.
- **الـ `irqaction`** = الممرض/الدكتور المحدد اللي بيتعامل مع المريض — ده الـ driver handler.
- **الـ `irq_desc`** = ملف المريض الكامل — فيه بيانات الجهاز، البروتوكول المتبع، والطاقم المسؤول.
- **الـ `irq_domain`** = سنترال المستشفى — بيحول رقم الغرفة (hwirq) لرقم داخلي (Linux IRQ number) وبيعرف مين المسؤول.

**التعميق في التشبيه:**

| مفهوم في التشبيه | المقابل في الـ Kernel |
|-----------------|----------------------|
| رقم غرفة المريض | `hwirq` — رقم في نطاق الـ controller |
| الرقم الداخلي للاستقبال | Linux `irq` number — virtual |
| سنترال المستشفى | `irq_domain` — يعمل mapping |
| ملف المريض | `struct irq_desc` |
| جهاز المانيتور | `struct irq_chip` |
| بروتوكول التعامل | `irq_flow_handler_t` (handle_level_irq, handle_edge_irq...) |
| الطاقم الطبي | `struct irqaction` chain |
| حالة المريض (نايم/مستيقظ) | `IRQD_*` flags في `irq_common_data.state_use_accessors` |

---

### البنية الكبيرة: أين يقع الـ Framework في الـ Kernel؟

```
┌─────────────────────────────────────────────────────────────┐
│                    User Space / Drivers                      │
│              request_irq() / free_irq()                     │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│              Generic IRQ Core (kernel/irq/)                  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               struct irq_desc[]                     │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │   │
│  │  │  irq_desc[0] │  │  irq_desc[1] │  │   ...    │  │   │
│  │  │ ┌──────────┐ │  │ ┌──────────┐ │  └──────────┘  │   │
│  │  │ │irq_data  │ │  │ │irq_data  │ │               │   │
│  │  │ │irq_chip* │ │  │ │irq_chip* │ │               │   │
│  │  │ │domain*   │ │  │ │domain*   │ │               │   │
│  │  │ └──────────┘ │  │ └──────────┘ │               │   │
│  │  │ handle_irq() │  │ handle_irq() │               │   │
│  │  │ action*      │  │ action*      │               │   │
│  │  └──────────────┘  └──────────────┘               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   irq_domain                        │   │
│  │    hwirq ←──────────────────────→ Linux IRQ#        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│            irq_chip implementations (drivers/irqchip/)       │
│                                                             │
│   ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│   │  ARM GIC     │  │  GPIO expndr │  │  APIC (x86)     │  │
│   │  irq_chip    │  │  irq_chip    │  │  irq_chip       │  │
│   └──────┬───────┘  └──────┬───────┘  └────────┬────────┘  │
└──────────┼─────────────────┼───────────────────┼────────────┘
           │                 │                   │
┌──────────▼─────────────────▼───────────────────▼────────────┐
│                    Physical Hardware                          │
│            GIC-400      PCF8574       Local APIC             │
└─────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: التجريد الأساسي

الـ framework بيقوم على فكرة واحدة جوهرية:

> **كل interrupt في الـ system ليه `irq_desc` واحد بيجمع كل المعلومات اللازمة لمعالجته.**

الـ `irq_desc` هو الـ central object. خلينا نرسم ازاي الـ structs بتتربط مع بعض:

```
struct irq_desc
├── struct irq_common_data  irq_common_data   ← shared across chip hierarchy
│   ├── state_use_accessors  (IRQD_* flags)
│   ├── cpumask affinity      (SMP: أي CPU يستقبل الـ IRQ)
│   ├── cpumask effective_affinity
│   └── msi_desc*             (لو MSI interrupt)
│
├── struct irq_data          irq_data         ← per-chip data
│   ├── irq          (Linux virtual IRQ number)
│   ├── hwirq        (hardware IRQ number داخل الـ domain)
│   ├── irq_chip*    chip ←──────────────────────────────┐
│   ├── irq_domain*  domain                               │
│   ├── irq_data*    parent_data  (hierarchy)             │
│   └── chip_data*   (private data للـ chip)              │
│                                                         │
├── irq_flow_handler_t  handle_irq   ← flow handler       │
│   (handle_level_irq / handle_edge_irq / ...)            │
│                                                         │
├── irqaction*          action       ← driver handlers     │
│   ├── handler()                                         │
│   ├── thread_fn()                                       │
│   └── next*           (chained handlers)                │
│                                                         │
└── irqstat __percpu*   kstat_irqs   ← per-CPU statistics  │
                                                          │
struct irq_chip  ◄────────────────────────────────────────┘
├── name
├── irq_startup()
├── irq_shutdown()
├── irq_enable()
├── irq_disable()
├── irq_ack()        ← إشعار الـ controller ان الـ IRQ اتستلم
├── irq_mask()       ← منع الـ IRQ من الوصول
├── irq_unmask()     ← السماح للـ IRQ بالوصول
├── irq_eoi()        ← End Of Interrupt
├── irq_set_type()   ← edge/level configuration
├── irq_set_affinity() ← CPU routing
└── flags            (IRQCHIP_* behavioral flags)
```

---

### الـ Flow Handlers: الطبقة الوسطى الحاسمة

الـ flow handler هو المنطق اللي بيحكم ازاي الـ interrupt بيتمنج **بعد** ما الـ hardware يطلقه وقبل ما الـ driver handler ينفذ.

الـ kernel بيوفر built-in handlers لأشيع الـ flow types:

| الـ Handler | متى يُستخدم؟ |
|-------------|-------------|
| `handle_level_irq()` | Level-triggered: لازم يـ mask الـ IRQ أثناء المعالجة عشان ما يطلعش تاني |
| `handle_edge_irq()` | Edge-triggered: الـ hardware بيـ latch الـ interrupt، ممكن نـ ack ونكمل |
| `handle_fasteoi_irq()` | Modern GIC-style: نرسل EOI بس في الآخر، والـ controller يدير الباقي |
| `handle_simple_irq()` | للـ chained/embedded cases، من غير أي masking |
| `handle_percpu_irq()` | IPIs وTimer interrupts — كل CPU بيعالجها لنفسه |
| `handle_nested_irq()` | الـ interrupt بيتعالج في thread context (مش hardirq context) |
| `handle_bad_irq()` | Spurious interrupts — log and return |

**مثال عملي: handle_level_irq**

```
                    IRQ يجي
                       │
                       ▼
              irq_chip->irq_mask()   ← نمنع الـ IRQ من التكرار
                       │
                       ▼
              irq_chip->irq_ack()    ← نخبر الـ controller "استلمنا"
                       │
                       ▼
           handle_irq_event()        ← ننادي الـ action handlers
                       │
                       ▼
              irq_chip->irq_unmask() ← نفتح الـ IRQ تاني
```

**مثال عملي: handle_edge_irq**

```
                    IRQ يجي
                       │
                       ▼
              irq_chip->irq_ack()    ← نـ ack فوراً (الـ hardware latched)
                       │
                       ▼
           handle_irq_event()        ← ننادي الـ action handlers
                       │
                       ▼
                    (no unmask needed)
```

---

### الـ IRQ State Machine: IRQD_* Flags

الـ `state_use_accessors` field في `irq_common_data` بيخزن حالة الـ interrupt كـ bitmask. ده مش مجرد flags عشوائية — ده state machine كامل:

```
                   ┌─────────────────┐
                   │   ALLOCATED     │
                   └────────┬────────┘
                            │ request_irq()
                            ▼
                   ┌─────────────────┐
                   │   STARTED       │  ← IRQD_IRQ_STARTED
                   └────────┬────────┘
                            │
              ┌─────────────┼──────────────┐
              │             │              │
              ▼             ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────────┐
        │ DISABLED │  │  MASKED  │  │  INPROGRESS  │
        │IRQD_IRQ_ │  │IRQD_IRQ_ │  │ IRQD_IRQ_    │
        │DISABLED  │  │ MASKED   │  │ INPROGRESS   │
        └──────────┘  └──────────┘  └──────────────┘
```

الـ accessor functions الـ `static inline` في `irq.h` (زي `irqd_irq_disabled()`, `irqd_irq_masked()`) بتقرأ وتكتب الـ flags دي بطريقة آمنة — بتحمي الـ `__private` field من الوصول المباشر.

---

### الـ irq_chip Flags: سلوك الـ Chip

الـ `flags` field في `irq_chip` بيوصف capabilities وrequirements خاصة بالـ hardware:

| الـ Flag | معناه |
|----------|-------|
| `IRQCHIP_SET_TYPE_MASKED` | لازم تـ mask الـ IRQ قبل تغيير الـ type |
| `IRQCHIP_EOI_IF_HANDLED` | ابعت EOI بس لو الـ handler قال `IRQ_HANDLED` |
| `IRQCHIP_MASK_ON_SUSPEND` | mask الـ IRQ أثناء الـ suspend |
| `IRQCHIP_SUPPORTS_NMI` | الـ chip قادر يبعت NMIs |
| `IRQCHIP_ONESHOT_SAFE` | الـ oneshot mode ما محتاجش mask/unmask صريح |
| `IRQCHIP_IMMUTABLE` | ما تغيرش أي حاجة في الـ chip (للـ chained chips) |
| `IRQCHIP_MOVE_DEFERRED` | الـ affinity migration تتم في interrupt context |

---

### الـ Generic IRQ Chip: التجريد الإضافي

الـ `irq_chip_generic` هو طبقة تجريد **فوق** الـ `irq_chip` العادية — مفيدة للـ controllers اللي بتدير مجموعة interrupts بـ registers موحدة:

```
struct irq_chip_generic
├── reg_base        ← base address للـ controller في الـ memory
├── mask_cache      ← cached value للـ mask register (نتجنب قراية الـ HW كل مرة)
├── irq_base        ← أول Linux IRQ number في المجموعة
├── irq_cnt         ← عدد الـ IRQs اللي الـ chip ده بيديرها
├── lock            ← حماية الـ cache والـ register access
│
└── chip_types[]    ← array من struct irq_chip_type
    ├── chip_types[0]   ← للـ edge-triggered lines
    │   ├── chip    (irq_chip ops)
    │   ├── regs    (enable/disable/mask/ack offsets)
    │   └── handler (handle_edge_irq)
    │
    └── chip_types[1]   ← للـ level-triggered lines
        ├── chip    (irq_chip ops مختلفة)
        ├── regs    (offsets مختلفة)
        └── handler (handle_level_irq)
```

**مثال واقعي**: GPIO controller على ARM SoC زي `stm32-gpio`. عنده 16 pin كـ interrupts، كلهم بيشاركوا نفس الـ registers. بدل ما يكتب 16 `irq_chip` منفصلة، بيستخدم `irq_chip_generic` واحدة.

```c
gc = irq_alloc_generic_chip("stm32-gpio", 1, irq_base,
                             reg_base, handle_edge_irq);
gc->chip_types[0].regs.ack   = STM32_EDGE_PENDING_REG;
gc->chip_types[0].regs.mask  = STM32_IRQ_MASK_REG;
gc->chip_types[0].chip.irq_mask     = irq_gc_mask_set_bit;
gc->chip_types[0].chip.irq_unmask   = irq_gc_mask_clr_bit;
gc->chip_types[0].chip.irq_ack      = irq_gc_ack_set_bit;
irq_setup_generic_chip(gc, IRQ_MSK(16), IRQ_GC_INIT_MASK_CACHE, 0, 0);
```

---

### الـ Hierarchy Support: الـ Chained Controllers

على ARM SoC الحديث، ممكن يكون عندك:

```
GIC (ARM Generic Interrupt Controller)
  └── GPIO Controller (connected to one GIC IRQ)
        ├── GPIO pin 0  → Linux IRQ #100
        ├── GPIO pin 1  → Linux IRQ #101
        └── GPIO pin 2  → Linux IRQ #102
```

هنا الـ GPIO interrupts بتمر بـ "hierarchy":
- `irq_data` للـ GPIO pin بيشاور على `parent_data` اللي هو `irq_data` الـ GIC
- لما driver يعمل `enable_irq(100)`, الـ framework بيـ walk الـ hierarchy ويـ enable على كل level

```c
/* irq_data hierarchy */
irq_data (GPIO pin 0)
  ├── hwirq = 0
  ├── chip  = gpio_irq_chip
  └── parent_data ──► irq_data (GIC)
                         ├── hwirq = 32   (GPIO IRQ في GIC)
                         └── chip  = gic_irq_chip
```

الـ `irq_chip_*_parent()` functions (زي `irq_chip_mask_parent()`, `irq_chip_ack_parent()`) موجودة عشان الـ chip يقدر يفوض العملية للـ parent controller بسهولة.

---

### ايه اللي الـ Framework بيمتلكه vs. ايه اللي يفوضه للـ Drivers

| الـ Generic IRQ Core يمتلك | يفوض للـ Driver / irq_chip |
|--------------------------|--------------------------|
| تخصيص وإدارة `irq_desc` | التعامل مع registers الـ controller |
| الـ flow logic (level/edge/percpu) | ترجمة hwirq ↔ Linux IRQ (irq_domain) |
| Spurious interrupt detection | تحديد trigger type لكل line |
| IRQ threading وscheduling | Power management hooks (suspend/resume) |
| SMP affinity management framework | chip-specific affinity routing |
| `/proc/interrupts` statistics | MSI message composition |
| Wakeup source management framework | الـ actual hardware programming |
| Lockdep integration للـ IRQ locks | Bus locking (I2C GPIO expanders) |

---

### ربط الكل ببعض: رحلة الـ Interrupt

```
1. Hardware يطلق interrupt
        │
        ▼
2. CPU يوصل لـ arch-specific entry point (asm)
        │
        ▼
3. generic_handle_domain_irq(domain, hwirq)
   └─► irq_domain يترجم hwirq → Linux IRQ#
        │
        ▼
4. handle_irq_desc(desc)
   └─► desc->handle_irq(desc)  [flow handler]
        │
        ├─► irq_chip->irq_ack() أو irq_mask()
        │
        ▼
5. handle_irq_event(desc)
   └─► ينادي كل irqaction في الـ chain
        │
        ├─► action->handler()  [driver ISR]
        │
        └─► لو threaded: يصحى الـ thread ويشغل action->thread_fn()
        │
        ▼
6. irq_chip->irq_unmask() أو irq_eoi()
        │
        ▼
7. عودة للـ interrupted context
```

---

### ملاحظة: الـ Subsystems المرتبطة

- **الـ `irq_domain` subsystem**: مسؤول عن الـ mapping بين hwirq و Linux IRQ numbers — ده subsystem منفصل بس متكامل مع الـ Generic IRQ Framework تماماً.
- **الـ `irqchip` drivers** (`drivers/irqchip/`): الـ implementations الفعلية للـ `irq_chip` interface لكل controller (GIC, APIC, etc.).
- **الـ `genirq` core** (`kernel/irq/`): الكود الرئيسي للـ framework نفسه.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Flags — Cheatsheet

#### IRQ_TYPE_* — نوع الـ trigger

| Flag | Value | المعنى |
|------|-------|---------|
| `IRQ_TYPE_NONE` | `0x00` | مفيش type محدد |
| `IRQ_TYPE_EDGE_RISING` | `0x01` | trigger على الحافة الصاعدة |
| `IRQ_TYPE_EDGE_FALLING` | `0x02` | trigger على الحافة النازلة |
| `IRQ_TYPE_EDGE_BOTH` | `0x03` | الحافتين |
| `IRQ_TYPE_LEVEL_HIGH` | `0x04` | level عالي |
| `IRQ_TYPE_LEVEL_LOW` | `0x08` | level واطي |
| `IRQ_TYPE_SENSE_MASK` | `0x0F` | mask لكل bits الـ sense |
| `IRQ_TYPE_DEFAULT` | `0x0F` | يخلي الـ HW يحدد الـ default |
| `IRQ_TYPE_PROBE` | `0x10` | جاري الـ probing |

#### IRQ_* — status flags (bits 8+)

| Flag | Bit | المعنى |
|------|-----|---------|
| `IRQ_LEVEL` | 8 | الـ interrupt level-triggered |
| `IRQ_PER_CPU` | 9 | خاص بكل CPU، مش بالـ affinity |
| `IRQ_NOPROBE` | 10 | لا يتعمله autoprobe |
| `IRQ_NOREQUEST` | 11 | مش متاح لـ `request_irq()` |
| `IRQ_NOAUTOEN` | 12 | ما يتفعلش أوتوماتيك |
| `IRQ_NO_BALANCING` | 13 | ممنوع الـ affinity balancing |
| `IRQ_NESTED_THREAD` | 15 | thread متداخل في thread تاني |
| `IRQ_NOTHREAD` | 16 | ممنوع الـ threading |
| `IRQ_PER_CPU_DEVID` | 17 | الـ dev_id هو per-cpu variable |
| `IRQ_IS_POLLED` | 18 | يتعمله poll من interrupt تاني |
| `IRQ_DISABLE_UNLAZY` | 19 | disable فوري مش lazy |
| `IRQ_HIDDEN` | 20 | مش هيبان في `/proc/interrupts` |
| `IRQ_NO_DEBUG` | 21 | مستثنى من `note_interrupt()` |

#### IRQD_* — state bits في `irq_common_data.state_use_accessors`

| Flag | Bit | المعنى |
|------|-----|---------|
| `IRQD_TRIGGER_MASK` | 0–3 | نوع الـ trigger الحالي |
| `IRQD_SETAFFINITY_PENDING` | 8 | في affinity change معلق |
| `IRQD_ACTIVATED` | 9 | اتعمله activate |
| `IRQD_NO_BALANCING` | 10 | balancing معطل |
| `IRQD_PER_CPU` | 11 | per-CPU interrupt |
| `IRQD_AFFINITY_SET` | 12 | الـ affinity اتضبط |
| `IRQD_LEVEL` | 13 | level-triggered |
| `IRQD_WAKEUP_STATE` | 14 | مضبوط كـ wakeup source |
| `IRQD_IRQ_DISABLED` | 16 | معطل حاليًا |
| `IRQD_IRQ_MASKED` | 17 | متعمله mask حاليًا |
| `IRQD_IRQ_INPROGRESS` | 18 | بيتنفذ دلوقتي |
| `IRQD_WAKEUP_ARMED` | 19 | الـ wakeup جاهز |
| `IRQD_FORWARDED_TO_VCPU` | 20 | بيتبعت لـ vCPU |
| `IRQD_AFFINITY_MANAGED` | 21 | الـ kernel بيدير الـ affinity |
| `IRQD_IRQ_STARTED` | 22 | في حالة startup |
| `IRQD_MANAGED_SHUTDOWN` | 23 | اتعمله shutdown بسبب affinity mask فاضية |
| `IRQD_SINGLE_TARGET` | 24 | يروح لـ CPU واحد بس |
| `IRQD_DEFAULT_TRIGGER_SET` | 25 | الـ trigger الافتراضي اتضبط |
| `IRQD_CAN_RESERVE` | 26 | يقدر يدخل reservation mode |
| `IRQD_HANDLE_ENFORCE_IRQCTX` | 27 | لازم يتشغل من interrupt context حقيقي |
| `IRQD_AFFINITY_ON_ACTIVATE` | 28 | ضبط الـ affinity وقت الـ activation |
| `IRQD_IRQ_ENABLED_ON_SUSPEND` | 29 | مفعل وقت الـ suspend |
| `IRQD_RESEND_WHEN_IN_PROGRESS` | 30 | يتعاد إرساله لو كان شغال |

#### IRQCHIP_* — flags بتوع الـ irq_chip

| Flag | Bit | المعنى |
|------|-----|---------|
| `IRQCHIP_SET_TYPE_MASKED` | 0 | يعمل mask قبل `irq_set_type()` |
| `IRQCHIP_EOI_IF_HANDLED` | 1 | `irq_eoi()` بس لو اتعمل handle |
| `IRQCHIP_MASK_ON_SUSPEND` | 2 | يعمل mask للـ non-wake irqs في الـ suspend |
| `IRQCHIP_ONOFFLINE_ENABLED` | 3 | يستدعي on/offline callbacks بس لو enabled |
| `IRQCHIP_SKIP_SET_WAKE` | 4 | يتجاهل `irq_set_wake()` |
| `IRQCHIP_ONESHOT_SAFE` | 5 | الـ oneshot مش محتاج mask/unmask |
| `IRQCHIP_EOI_THREADED` | 6 | محتاج `eoi()` عند الـ unmask في threaded mode |
| `IRQCHIP_SUPPORTS_LEVEL_MSI` | 7 | يدعم Level MSI |
| `IRQCHIP_SUPPORTS_NMI` | 8 | يقدر يبعت NMI |
| `IRQCHIP_ENABLE_WAKEUP_ON_SUSPEND` | 9 | يفعّل/يعطل الـ wake irqs في الـ suspend |
| `IRQCHIP_AFFINITY_PRE_STARTUP` | 10 | يضبط الـ affinity قبل الـ startup |
| `IRQCHIP_IMMUTABLE` | 11 | محدش يغير فيه |
| `IRQCHIP_MOVE_DEFERRED` | 12 | ينقل الـ interrupt في الـ interrupt context الفعلي |

#### enum irq_gc_flags — flags الـ Generic Chip

| Flag | Bit | المعنى |
|------|-----|---------|
| `IRQ_GC_INIT_MASK_CACHE` | 0 | يقرأ الـ mask register لتهيئة الـ cache |
| `IRQ_GC_INIT_NESTED_LOCK` | 1 | lock class nested (للـ GPIO chips) |
| `IRQ_GC_MASK_CACHE_PER_TYPE` | 2 | كل chip type عنده cache منفصل |
| `IRQ_GC_NO_MASK` | 3 | متحسبش `irq_data->mask` |
| `IRQ_GC_BE_IO` | 4 | I/O بـ big-endian |

#### IRQ_SET_MASK_OK_* — قيم return بتوع `irq_set_affinity()`

| Value | المعنى |
|-------|---------|
| `IRQ_SET_MASK_OK` | تمام، الـ core يحدث الـ affinity |
| `IRQ_SET_MASK_OK_NOCOPY` | تمام، الـ chip حدّث الـ affinity بنفسه |
| `IRQ_SET_MASK_OK_DONE` | تمام + وقف تنفيذ الـ child chips |

---

### الـ Structs المهمة

#### 1. `struct irq_common_data`

**الغرض:** بيانات مشتركة بين كل الـ irqchips اللي بتخدم نفس الـ IRQ — بتتشارك بين كل levels في الـ hierarchy.

| Field | النوع | الوصف |
|-------|-------|--------|
| `state_use_accessors` | `unsigned int` (private) | بتوع الـ `IRQD_*` — لازم تستخدم الـ accessor functions |
| `node` | `unsigned int` | NUMA node للـ balancing |
| `handler_data` | `void *` | data خاص بالـ irq_chip methods |
| `msi_desc` | `struct msi_desc *` | الـ MSI descriptor لو الـ IRQ هو MSI |
| `affinity` | `cpumask_var_t` | الـ CPUs المسموحة بتشغيل الـ interrupt عليها |
| `effective_affinity` | `cpumask_var_t` | الـ affinity الفعلية (subset من `affinity`) |
| `ipi_offset` | `unsigned int` | offset أول CPU في الـ IPI affinity mask |

**ارتباطاتها:** متضمنة مباشرة في `struct irq_desc` وبتشير ليها `struct irq_data` عن طريق pointer.

---

#### 2. `struct irq_data`

**الغرض:** البيانات اللي بتتبعت لـ functions الـ irq_chip — بتمثل IRQ واحد على مستوى chip معين.

| Field | النوع | الوصف |
|-------|-------|--------|
| `mask` | `u32` | bitmask محسوب مسبقًا للـ chip registers |
| `irq` | `unsigned int` | رقم الـ IRQ في Linux (virtual) |
| `hwirq` | `irq_hw_number_t` | رقم الـ IRQ في الـ hardware (محلي لكل domain) |
| `common` | `struct irq_common_data *` | pointer للبيانات المشتركة |
| `chip` | `struct irq_chip *` | الـ chip اللي بيتحكم في الـ IRQ ده |
| `domain` | `struct irq_domain *` | الـ domain المسؤول عن الترجمة hwirq ↔ virq |
| `parent_data` | `struct irq_data *` | في الـ hierarchy: pointer للـ level الأعلى |
| `chip_data` | `void *` | data خاص بالـ chip (private per-chip) |

**ارتباطاتها:** متضمنة في `struct irq_desc`، وبتشير لـ `irq_chip` و`irq_domain` و`irq_common_data`.

---

#### 3. `struct irq_chip`

**الغرض:** الـ vtable بتاعة الـ interrupt controller — كل function pointer هو hook للـ kernel يقدر يستدعيه.

| Field / Callback | الوصف |
|------------------|--------|
| `name` | اسم الـ chip في `/proc/interrupts` |
| `irq_startup` | تشغيل الـ IRQ لأول مرة (default: `irq_enable`) |
| `irq_shutdown` | إيقاف الـ IRQ (default: `irq_disable`) |
| `irq_enable` | تفعيل (default: `irq_unmask`) |
| `irq_disable` | تعطيل |
| `irq_ack` | إشعار بداية interrupt جديد |
| `irq_mask` | حجب الـ IRQ |
| `irq_mask_ack` | حجب + acknowledge في خطوة واحدة |
| `irq_unmask` | رفع الـ mask |
| `irq_eoi` | End Of Interrupt |
| `irq_set_affinity` | تحديد CPUs المسموحة (SMP) |
| `irq_retrigger` | إعادة إرسال الـ IRQ للـ CPU |
| `irq_set_type` | ضبط نوع الـ trigger (edge/level) |
| `irq_set_wake` | تفعيل/تعطيل wakeup من suspend |
| `irq_bus_lock` / `irq_bus_sync_unlock` | للـ chips على bus بطيء (مثل I2C) |
| `irq_suspend` / `irq_resume` / `irq_pm_shutdown` | Power management hooks |
| `irq_compose_msi_msg` / `irq_write_msi_msg` | تكوين وكتابة MSI message |
| `irq_get_irqchip_state` / `irq_set_irqchip_state` | قراءة/كتابة الحالة الداخلية للـ chip |
| `irq_set_vcpu_affinity` | توجيه الـ IRQ لـ vCPU في الـ virtualization |
| `ipi_send_single` / `ipi_send_mask` | إرسال IPI |
| `irq_nmi_setup` / `irq_nmi_teardown` | إعداد/تنظيف الـ NMI |
| `irq_force_complete_move` | إجبار اكتمال حركة الـ IRQ المعلقة |
| `flags` | `IRQCHIP_*` flags |

---

#### 4. `struct irq_desc`

**الغرض:** الـ descriptor الرئيسي لكل IRQ في النظام — القلب اللي بيربط كل حاجة ببعض.

| Field | الوصف |
|-------|--------|
| `irq_common_data` | البيانات المشتركة (متضمنة مش pointer) |
| `irq_data` | بيانات الـ chip (متضمنة مش pointer) |
| `kstat_irqs` | إحصائيات per-CPU |
| `handle_irq` | الـ high-level flow handler (edge/level/...) |
| `action` | linked list بتاع الـ `irqaction` (الـ drivers) |
| `status_use_accessors` | flags عامة (IRQ_* bits) |
| `depth` | عمق الـ disable nesting |
| `wake_depth` | عمق الـ wake enable nesting |
| `irq_count` | عداد لاكتشاف الـ stalled IRQs |
| `irqs_unhandled` | عدد الـ spurious interrupts |
| `lock` | `raw_spinlock_t` لحماية الـ descriptor |
| `redirect` | لإعادة توجيه الـ IRQ لـ CPU تاني عبر `irq_work` |
| `affinity_hint` | hint للـ user space بالـ CPU المفضل |
| `pending_mask` | للـ IRQs اللي في طريقها للـ rebalance |
| `threads_oneshot` | bitmask لمتابعة الـ oneshot threads |
| `threads_active` | عدد الـ threads الشغالة حاليًا |
| `wait_for_threads` | wait queue لـ `synchronize_irq()` |
| `request_mutex` | mutex يحمي الـ request/free قبل الـ lock |
| `parent_irq` | الـ IRQ الأب في حالة الـ hierarchy |
| `name` | اسم الـ flow handler |

---

#### 5. `struct irq_chip_regs`

**الغرض:** بيحتفظ بـ offsets الـ registers بالنسبة للـ base address — بيستخدمه الـ Generic IRQ Chip framework.

| Field | الوصف |
|-------|--------|
| `enable` | offset register التفعيل |
| `disable` | offset register التعطيل |
| `mask` | offset register الـ mask |
| `ack` | offset register الـ acknowledge |
| `eoi` | offset register الـ End Of Interrupt |
| `type` | offset register ضبط النوع |

---

#### 6. `struct irq_chip_type`

**الغرض:** instance واحد من الـ Generic IRQ Chip لنوع flow معين (edge أو level).

| Field | الوصف |
|-------|--------|
| `chip` | الـ `irq_chip` الفعلي (الـ vtable) |
| `regs` | الـ register offsets للـ flow type ده |
| `handler` | الـ flow handler المناسب |
| `type` | الـ IRQ_TYPE_* اللي بيتعامل معاه |
| `mask_cache_priv` | cache خاص بالـ chip type |
| `mask_cache` | pointer للـ mask cache المشترك أو الخاص |

---

#### 7. `struct irq_chip_generic`

**الغرض:** بيخلي chip واحد يتحكم في عدة interrupts (حتى 32) بـ registers مشتركة.

| Field | الوصف |
|-------|--------|
| `lock` | `raw_spinlock_t` لحماية الـ register access |
| `reg_base` | العنوان الـ virtual للـ registers |
| `reg_readl` / `reg_writel` | accessors بديلة (للـ BE أو special HW) |
| `suspend` / `resume` | callbacks خاصة بالـ power management |
| `irq_base` | أول رقم IRQ بيتعامل معاه الـ chip |
| `irq_cnt` | عدد الـ IRQs |
| `mask_cache` | cached mask register مشترك |
| `wake_enabled` / `wake_active` | bitmasks للـ wakeup management |
| `num_ct` | عدد الـ `irq_chip_type` instances |
| `private` | data خاص لـ callbacks غير generic |
| `installed` / `unused` | bitmaps لتتبع الـ IRQs المثبتة والغير مستخدمة |
| `domain` | الـ irq_domain المرتبط |
| `list` | لتتبع كل الـ instances |
| `chip_types[]` | flexible array من الـ irq_chip_type instances |

---

#### 8. `struct irq_domain_chip_generic`

**الغرض:** بيدير مجموعة من الـ `irq_chip_generic` على مستوى الـ irq_domain.

| Field | الوصف |
|-------|--------|
| `irqs_per_chip` | عدد الـ IRQs لكل chip |
| `num_chips` | عدد الـ chips |
| `irq_flags_to_set/clear` | flags تتضبط وقت الـ mapping |
| `gc_flags` | flags التهيئة |
| `exit` | callback عند الـ destroy |
| `gc[]` | flexible array من pointers للـ generic chips |

---

#### 9. `struct irq_domain_chip_generic_info`

**الغرض:** معلومات الـ template اللي بتتبنى عليها الـ generic chips وقت إنشاء الـ domain.

| Field | الوصف |
|-------|--------|
| `name` | اسم الـ chip |
| `handler` | الـ flow handler |
| `irqs_per_chip` | حد أقصى 32 |
| `num_ct` | عدد الـ chip types |
| `irq_flags_to_set/clear` | flags لكل IRQ |
| `gc_flags` | flags التهيئة |
| `init` / `exit` | callbacks عند الإنشاء والتدمير |

---

#### 10. `struct irqstat`

**الغرض:** إحصائيات per-CPU لعدد مرات حدوث الـ interrupt.

| Field | الوصف |
|-------|--------|
| `cnt` | العداد الفعلي |
| `ref` | snapshot للمقارنة (مع `CONFIG_GENERIC_IRQ_STAT_SNAPSHOT`) |

---

#### 11. `struct irq_redirect`

**الغرض:** بيسمح بتشغيل الـ IRQ handler على CPU تاني لما الـ CPU الحالي مش في الـ affinity mask.

| Field | الوصف |
|-------|--------|
| `work` | `irq_work` item للتنفيذ المؤجل |
| `target_cpu` | الـ CPU المستهدف |

---

### مخططات العلاقات (ASCII)

#### العلاقة الأساسية بين الـ Structs

```
┌──────────────────────────────────────────────────────────────────┐
│                        struct irq_desc                           │
│  ┌─────────────────────────┐  ┌──────────────────────────────┐  │
│  │  struct irq_common_data │  │      struct irq_data         │  │
│  │  (embedded directly)    │  │      (embedded directly)     │  │
│  │  - state_use_accessors  │◄─┤  - irq (Linux virq number)   │  │
│  │  - affinity (cpumask)   │  │  - hwirq (HW number)         │  │
│  │  - msi_desc *───────────┼─►│  - common * ─────────────────┼──┘
│  │  - handler_data *       │  │  - chip * ──────────────────►│struct irq_chip
│  └─────────────────────────┘  │  - domain * ────────────────►│struct irq_domain
│                                │  - parent_data * ───────────►│struct irq_data
│  - handle_irq() ──────────────►irq_flow_handler_t             │  (parent level)
│  - action * ──────────────────►struct irqaction (chain)       │  - chip_data *
│  - lock (raw_spinlock_t)      └──────────────────────────────┘  │
│  - kstat_irqs (percpu) ──────►struct irqstat                    │
│  - request_mutex                                                 │
└──────────────────────────────────────────────────────────────────┘
```

#### علاقة الـ Generic Chip Hierarchy

```
struct irq_domain
    │
    └──► struct irq_domain_chip_generic
              │  - irqs_per_chip
              │  - num_chips
              │
              └──► gc[] ──► struct irq_chip_generic [0]
                        │        - lock (raw_spinlock_t)
                        │        - reg_base (void __iomem *)
                        │        - irq_base, irq_cnt
                        │        - mask_cache
                        │        - domain *──────────────────────► (back to domain)
                        │        │
                        │        └──► chip_types[] ──► struct irq_chip_type [0]
                        │                  │                 - chip (irq_chip, embedded)
                        │                  │                 - regs (irq_chip_regs)
                        │                  │                 - handler (flow handler)
                        │                  │                 - type, mask_cache
                        │                  └──► struct irq_chip_type [1]
                        │                            (لو فيه edge + level مثلًا)
                        │
                        └──► struct irq_chip_generic [1]
                                  ...
```

#### علاقة الـ IRQ Hierarchy (Multi-level irqchip)

```
                    ┌──────────────────────┐
                    │   struct irq_data    │  ← level 0 (root irqchip, e.g. GIC)
                    │   chip ──────────────┼──► struct irq_chip (GIC ops)
                    │   domain ────────────┼──► GIC irq_domain
                    │   hwirq = 32         │
                    │   parent_data ───────┼──► NULL (root)
                    └──────────────────────┘
                              ▲
                              │ parent_data pointer
                    ┌──────────────────────┐
                    │   struct irq_data    │  ← level 1 (e.g. GPIO irqchip)
                    │   chip ──────────────┼──► struct irq_chip (GPIO ops)
                    │   domain ────────────┼──► GPIO irq_domain
                    │   hwirq = 5          │
                    │   parent_data ───────┘
                    └──────────────────────┘
                              ▲
                              │ (referenced from irq_desc)
                    ┌──────────────────────┐
                    │   struct irq_desc    │  ← الـ descriptor الواحد
                    │   irq_data (embedded)│
                    └──────────────────────┘
```

---

### دورة حياة الـ IRQ — Lifecycle

```
1. ALLOCATION
─────────────
irq_alloc_descs()
  └──► __irq_alloc_descs()
         └──► alloc_descs()
                └──► alloc_desc()
                       ├── kzalloc(sizeof(irq_desc))
                       ├── init irq_common_data (affinity, state=0)
                       ├── init irq_data (irq=N, hwirq=0, chip=&no_irq_chip)
                       ├── init lock (raw_spinlock_t)
                       ├── handle_irq = handle_bad_irq   ← default handler
                       └── insert into irq_desc radix tree (SPARSE_IRQ)
                           or irq_desc[] array

2. REGISTRATION (driver calls request_irq)
──────────────────────────────────────────
irq_set_chip()          → irq_data.chip = chip
irq_set_chip_data()     → irq_data.chip_data = data
irq_set_handler()       → irq_desc.handle_irq = flow_handler
request_irq(irq, handler, flags, name, dev)
  └──► __setup_irq()
         ├── alloc irqaction {handler, flags, name, dev_id}
         ├── irq_desc.action → new irqaction (append to chain)
         ├── irq_startup() → chip->irq_startup() or chip->irq_enable()
         └── irq_desc.depth = 0 (enabled)

3. ACTIVATION (irqdomain path)
───────────────────────────────
irq_domain_activate_irq(irq_data, reserve)
  └──► domain->ops->activate(domain, irq_data, reserve)
         ├── set IRQD_ACTIVATED in state
         ├── configure HW (set affinity, type, etc.)
         └── optionally: chip->irq_set_affinity()

4. RUNTIME (interrupt fires)
─────────────────────────────
CPU receives interrupt signal
  └──► arch interrupt entry (assembly)
         └──► handle_arch_irq(pt_regs)  [set via set_handle_irq()]
                └──► irq_desc->handle_irq(desc)   [e.g. handle_fasteoi_irq]
                       ├── chip->irq_mask() or chip->irq_ack()
                       ├── for each action in desc->action:
                       │     action->handler(irq, action->dev_id)
                       └── chip->irq_eoi() or chip->irq_unmask()

5. TEARDOWN
───────────
free_irq(irq, dev_id)
  └──► __free_irq()
         ├── remove irqaction from chain
         ├── if last action: chip->irq_shutdown() or chip->irq_disable()
         └── synchronize_irq() ← ينتظر الـ threads تنتهي

irq_free_descs(irq, cnt)
  └──► free_desc()
         ├── irq_domain_free_irqs() (if domain)
         └── kfree(desc) (SPARSE_IRQ)
```

---

### مخططات تدفق الاستدعاء — Call Flows

#### 1. تدفق handle_fasteoi_irq (level-triggered, APIC/GIC style)

```
HW raises interrupt line
  └──► CPU traps to interrupt vector
         └──► generic_handle_arch_irq(regs)
                └──► irq_desc->handle_irq(desc)   [= handle_fasteoi_irq]
                       ├── raw_spin_lock(&desc->lock)
                       ├── check IRQD_IRQ_INPROGRESS → set it
                       ├── chip->irq_mask(irq_data)   [if needed]
                       ├── raw_spin_unlock(&desc->lock)
                       ├── handle_irq_event(desc)
                       │     └──► for action in desc->action:
                       │            ret = action->handler(irq, dev_id)
                       │            [if IRQF_ONESHOT: thread is woken]
                       ├── raw_spin_lock(&desc->lock)
                       ├── chip->irq_eoi(irq_data)    [ACK to controller]
                       ├── chip->irq_unmask(irq_data) [re-enable]
                       └── raw_spin_unlock(&desc->lock)
```

#### 2. تدفق irq_set_affinity (SMP)

```
user writes to /proc/irq/N/smp_affinity
  └──► irq_set_affinity_hint() or irq_set_affinity()
         └──► irq_do_set_affinity(irq_data, cpumask, force)
                ├── chip->irq_set_affinity(irq_data, cpumask, force)
                │     └──► HW programs destination register
                │     returns IRQ_SET_MASK_OK / IRQ_SET_MASK_OK_NOCOPY
                ├── [if OK]: cpumask_copy(common->affinity, cpumask)
                └── [if hierarchy]: irq_chip_set_affinity_parent(data, ...)
                       └──► parent->chip->irq_set_affinity(parent_data, ...)
```

#### 3. تدفق Generic IRQ Chip — request path

```
irq_domain_alloc_generic_chips(domain, info)
  └──► alloc irq_domain_chip_generic
         └──► for each chip:
                irq_alloc_generic_chip(name, num_ct, irq_base, reg_base, handler)
                  └──► kzalloc(sizeof(irq_chip_generic) + num_ct * sizeof(irq_chip_type))
                         ├── gc->reg_base = reg_base
                         ├── gc->irq_base = irq_base
                         ├── init raw_spinlock (gc->lock)
                         └── chip_types[i].chip.name = name

irq_setup_generic_chip(gc, mask, flags, clr, set)
  └──► for each IRQ in mask:
         irq_set_chip(irq, &gc->chip_types[0].chip)
         irq_set_chip_data(irq, gc)
         irq_set_handler(irq, gc->chip_types[0].handler)
         irq_modify_status(irq, clr, set)

driver calls request_irq(irq, handler, ...)
  └──► __setup_irq() → appends irqaction → calls irq_startup()
         └──► irq_chip_type->chip->irq_startup(irq_data)
                └──► irq_gc_unmask_enable_reg(d)
                       ├── raw_spin_lock(&gc->lock)
                       ├── mask = irqd_get_trigger_type(d) mask bit
                       ├── gc->mask_cache &= ~mask
                       └──► irq_reg_writel(gc, gc->mask_cache, ct->regs.enable)
```

#### 4. تدفق Hierarchy irqchip — suspend/wakeup path

```
pm_suspend()
  └──► suspend_device_irqs()
         └──► for each irq_desc:
                irq_set_irq_wake(irq, 1) [if wakeup source]
                  └──► chip->irq_set_wake(irq_data, 1)
                         └──► [if hierarchy]: irq_chip_set_wake_parent(data, 1)
                                └──► parent->chip->irq_set_wake(parent_data, 1)
                irq_suspend() [non-wakeup]
                  └──► chip->irq_suspend(irq_data)
                         └──► [if IRQCHIP_MASK_ON_SUSPEND]:
                                chip->irq_mask(irq_data)

Resume:
  └──► chip->irq_resume(irq_data)
         └──► chip->irq_unmask(irq_data)
```

---

### استراتيجية الـ Locking

#### الـ Locks الموجودة وأدوارها

| Lock | النوع | يحمي ماذا؟ |
|------|-------|-------------|
| `irq_desc.lock` | `raw_spinlock_t` | كل محتوى الـ descriptor: الـ action chain، الـ depth، الـ state، الـ chip ops |
| `irq_desc.request_mutex` | `mutex` | الـ request/free path — يحمي قبل الـ lock acquisition |
| `irq_chip_generic.lock` | `raw_spinlock_t` | الـ register access والـ mask_cache في الـ generic chip |
| `irq_lock_sparse()` | internal lock | الـ radix tree للـ SPARSE_IRQ descriptors |

#### ترتيب الـ Locking

```
1. irq_desc.request_mutex    ← أول حاجة في request_irq / free_irq
     └──► 2. irq_desc.lock  ← لتغيير الـ action chain أو الـ chip state
                └──► 3. irq_chip_generic.lock  ← داخل chip callbacks فقط
```

#### قواعد مهمة

- **الـ `irq_desc.lock`** بيتاخد دايمًا بـ `raw_spin_lock_irqsave` في الـ runtime path لأننا بنكون في interrupt context.
- **الـ `request_mutex`** موجود عشان `mutex_lock` مش ممكن يتاخد من interrupt context — بيتاخد في process context بس (request/free path).
- **الـ `irq_chip_generic.lock`** بيتاخد من جوا الـ chip callbacks اللي بتتستدعى من تحت `irq_desc.lock` — **ممنوع منع باتًا** الكود اللي بيمسك `gc->lock` يحاول ياخد `irq_desc.lock` (deadlock).
- الـ `state_use_accessors` في `irq_common_data` دايمًا بيتعدّل من تحت `irq_desc.lock` — لازم تستخدم الـ accessor functions مش تتعامل معاه directly.
- في الـ **SMP pending IRQ move** (CONFIG_GENERIC_PENDING_IRQ): الـ affinity change بيتأجل للـ interrupt context الفعلي لتفادي الـ race مع الـ in-progress handler.
- الـ **bus lock** (`irq_bus_lock` / `irq_bus_sync_unlock`) موجود للـ chips على bus بطيء زي I2C — بيتعمل lock على الـ bus قبل ما يتاخد `irq_desc.lock` في path معين، فلازم يتاخد بالترتيب الصح.
## Phase 4: شرح الـ Functions

---

### ملخص عام — Cheatsheet

#### Group 1: `irqd_*` — State Accessors على `irq_data`

| Function | النوع | الغرض |
|---|---|---|
| `irqd_is_setaffinity_pending(d)` | `bool` | هل في affinity change معلق؟ |
| `irqd_is_per_cpu(d)` | `bool` | هل الـ IRQ per-CPU؟ |
| `irqd_can_balance(d)` | `bool` | هل يقدر الـ balancer يشتغل؟ |
| `irqd_affinity_was_set(d)` | `bool` | هل اتعمله affinity قبل كده؟ |
| `irqd_mark_affinity_was_set(d)` | `void` | سجّل إن affinity اتعمل |
| `irqd_trigger_type_was_set(d)` | `bool` | هل اتعمله trigger type init؟ |
| `irqd_get_trigger_type(d)` | `u32` | اجيب الـ trigger bits |
| `irqd_set_trigger_type(d, type)` | `void` | اكتب الـ trigger type |
| `irqd_is_level_type(d)` | `bool` | هل level-triggered؟ |
| `irqd_set_single_target(d)` | `void` | علّم إنه single-target فقط |
| `irqd_is_single_target(d)` | `bool` | هل single-target فقط؟ |
| `irqd_set_handle_enforce_irqctx(d)` | `void` | فرض إن handler يشتغل من IRQ context |
| `irqd_is_handle_enforce_irqctx(d)` | `bool` | هل الـ enforcement فعّال؟ |
| `irqd_is_enabled_on_suspend(d)` | `bool` | هل enabled في suspend؟ |
| `irqd_is_wakeup_set(d)` | `bool` | هل wakeup source؟ |
| `irqd_irq_disabled(d)` | `bool` | هل disabled؟ |
| `irqd_irq_masked(d)` | `bool` | هل masked؟ |
| `irqd_irq_inprogress(d)` | `bool` | هل في progress دلوقتي؟ |
| `irqd_is_wakeup_armed(d)` | `bool` | هل wakeup armed؟ |
| `irqd_is_forwarded_to_vcpu(d)` | `bool` | هل forwarded لـ vCPU؟ |
| `irqd_set_forwarded_to_vcpu(d)` | `void` | علّم forwarded لـ vCPU |
| `irqd_clr_forwarded_to_vcpu(d)` | `void` | امسح علامة forwarded |
| `irqd_affinity_is_managed(d)` | `bool` | هل kernel managed affinity؟ |
| `irqd_is_activated(d)` | `bool` | هل اتعمله activate؟ |
| `irqd_set_activated(d)` / `irqd_clr_activated(d)` | `void` | set/clr activated |
| `irqd_is_started(d)` | `bool` | هل startup اتعمل؟ |
| `irqd_is_managed_and_shutdown(d)` | `bool` | هل managed IRQ اتقفل بسبب empty affinity؟ |
| `irqd_set_can_reserve(d)` / `irqd_clr_can_reserve(d)` | `void` | set/clr reserve mode |
| `irqd_can_reserve(d)` | `bool` | هل reservation mode مسموح؟ |
| `irqd_set_affinity_on_activate(d)` / `irqd_affinity_on_activate(d)` | `void`/`bool` | افرض affinity وقت activate |
| `irqd_set_resend_when_in_progress(d)` | `void` | علّم resend if in-progress |
| `irqd_needs_resend_when_in_progress(d)` | `bool` | هل محتاج resend؟ |
| `irqd_to_hwirq(d)` | `irq_hw_number_t` | اجيب hwirq number |

#### Group 2: Flow Handlers

| Function | الاستخدام |
|---|---|
| `handle_level_irq` | Level-triggered، بيمسك mask/unmask |
| `handle_edge_irq` | Edge-triggered، بيعمل ack فوري |
| `handle_fasteoi_irq` | Modern EOI-based، common في ARM GIC |
| `handle_fasteoi_nmi` | NMI flow عبر EOI |
| `handle_fasteoi_ack_irq` | Hierarchy: EOI + ack parent |
| `handle_fasteoi_mask_irq` | Hierarchy: EOI + mask parent |
| `handle_simple_irq` | بدون mask/unmask، minimal |
| `handle_untracked_irq` | مش managed من الـ core |
| `handle_percpu_irq` | Per-CPU interrupts |
| `handle_percpu_devid_irq` | Per-CPU مع per-cpu `dev_id` |
| `handle_bad_irq` | Spurious/unhandled |
| `handle_nested_irq` | Nested threaded IRQ |

#### Group 3: Registration & Configuration

| Function | الغرض |
|---|---|
| `irq_set_chip` | ربط `irq_chip` بـ IRQ number |
| `irq_set_chip_and_handler` | ربط chip + flow handler |
| `irq_set_chip_and_handler_name` | زي السابق + اسم |
| `irq_set_handler` | بدّل flow handler بس |
| `irq_set_chained_handler` | handler لـ chained IRQ controller |
| `irq_set_chained_handler_and_data` | chained + private data |
| `irq_set_handler_data` | set `handler_data` |
| `irq_set_chip_data` | set `chip_data` |
| `irq_set_irq_type` | set trigger type (LEVEL/EDGE) |
| `irq_set_msi_desc` | ربط MSI descriptor |
| `irq_set_msi_desc_off` | ربط MSI descriptor بـ offset |
| `irq_set_percpu_devid` | علّم per-cpu devid |
| `irq_set_parent` | set parent IRQ (SW resend) |
| `irq_modify_status` | clear+set IRQ_* flags |
| `irq_set_status_flags` / `irq_clear_status_flags` | wrappers لـ modify |
| `irq_set_noprobe` / `irq_set_probe` | set/clr NOPROBE |
| `irq_set_nothread` / `irq_set_thread` | set/clr NOTHREAD |
| `irq_set_nested_thread` | set/clr NESTED_THREAD |
| `irq_set_percpu_devid_flags` | set مجموعة flags للـ per-cpu devid |

#### Group 4: Data Getters

| Function | الغرض |
|---|---|
| `irq_get_irq_data` | `irq_data*` من irq number |
| `irq_get_chip` | `irq_chip*` من irq number |
| `irq_data_get_irq_chip` | `irq_chip*` من `irq_data` |
| `irq_get_chip_data` | `chip_data` من irq number |
| `irq_data_get_irq_chip_data` | `chip_data` من `irq_data` |
| `irq_get_handler_data` | `handler_data` من irq number |
| `irq_data_get_irq_handler_data` | `handler_data` من `irq_data` |
| `irq_get_msi_desc` | MSI descriptor من irq number |
| `irq_data_get_msi_desc` | MSI descriptor من `irq_data` |
| `irq_get_trigger_type` | trigger type من irq number |
| `irq_get_affinity_mask` | affinity cpumask من irq number |
| `irq_data_get_affinity_mask` | affinity من `irq_data` |
| `irq_data_update_affinity` | اكتب affinity في `irq_data` |
| `irq_data_get_effective_affinity_mask` | effective affinity |
| `irq_data_update_effective_affinity` | اكتب effective affinity |
| `irq_get_effective_affinity_mask` | effective affinity من irq number |
| `irq_common_data_get_node` | NUMA node من `irq_common_data` |
| `irq_data_get_node` | NUMA node من `irq_data` |

#### Group 5: Descriptor Allocation

| Function | الغرض |
|---|---|
| `__irq_alloc_descs` | alloc N descriptors (raw) |
| `irq_alloc_descs` / `irq_alloc_desc` | wrappers سهلة |
| `irq_alloc_desc_at` / `irq_alloc_desc_from` | alloc عند رقم أو من رقم |
| `devm_irq_alloc_descs` | managed نسخة |
| `irq_free_descs` / `irq_free_desc` | تحرير descriptors |

#### Group 6: Generic IRQ Chip (GC)

| Function | الغرض |
|---|---|
| `irq_alloc_generic_chip` | alloc `irq_chip_generic` |
| `irq_setup_generic_chip` | setup وربط بـ IRQs |
| `irq_setup_alt_chip` | switch لـ chip type تاني |
| `irq_remove_generic_chip` | فصل GC من IRQs |
| `irq_destroy_generic_chip` | remove + free |
| `irq_free_generic_chip` | kfree wrapper |
| `irq_map_generic_chip` | irqdomain map callback |
| `irq_unmap_generic_chip` | irqdomain unmap callback |
| `devm_irq_alloc_generic_chip` | managed alloc |
| `devm_irq_setup_generic_chip` | managed setup |
| `irq_get_domain_generic_chip` | GC من domain + hwirq |
| `irq_domain_alloc_generic_chips` | alloc GCs لـ domain كامل |
| `irq_domain_remove_generic_chips` | cleanup domain GCs |
| `irq_gc_noop` | no-op callback |
| `irq_gc_mask_disable_reg` / `irq_gc_unmask_enable_reg` | mask/unmask عبر disable/enable reg |
| `irq_gc_mask_set_bit` / `irq_gc_mask_clr_bit` | mask عبر set/clr bit |
| `irq_gc_ack_set_bit` / `irq_gc_ack_clr_bit` | ack عبر set/clr bit |
| `irq_gc_mask_disable_and_ack_set` | mask + ack مع بعض |
| `irq_gc_eoi` | EOI في GC |
| `irq_gc_set_wake` | wakeup enable/disable في GC |
| `irq_reg_writel` / `irq_reg_readl` | I/O مع support لـ custom accessors |
| `irq_data_get_chip_type` | `irq_chip_type` من `irq_data` |

#### Group 7: irq_chip Hierarchy Helpers

| Function | الغرض |
|---|---|
| `irq_chip_startup_parent` | startup على parent chip |
| `irq_chip_shutdown_parent` | shutdown على parent |
| `irq_chip_enable_parent` | enable على parent |
| `irq_chip_disable_parent` | disable على parent |
| `irq_chip_ack_parent` | ack على parent |
| `irq_chip_mask_parent` / `irq_chip_unmask_parent` | mask/unmask parent |
| `irq_chip_mask_ack_parent` | mask+ack parent |
| `irq_chip_eoi_parent` | EOI على parent |
| `irq_chip_set_affinity_parent` | affinity على parent |
| `irq_chip_set_wake_parent` | wakeup على parent |
| `irq_chip_set_vcpu_affinity_parent` | vCPU affinity على parent |
| `irq_chip_set_type_parent` | trigger type على parent |
| `irq_chip_retrigger_hierarchy` | retrigger كل الـ hierarchy |
| `irq_chip_get_parent_state` / `irq_chip_set_parent_state` | get/set chip state على parent |
| `irq_chip_request_resources_parent` / `irq_chip_release_resources_parent` | resources على parent |
| `irq_chip_pre_redirect_parent` | pre-redirect على parent (SMP) |
| `irq_chip_redirect_set_affinity` | redirect affinity (SMP) |
| `irq_chip_compose_msi_msg` | بناء MSI message عبر hierarchy |
| `irq_chip_pm_get` / `irq_chip_pm_put` | power management reference counting |

#### Group 8: Affinity & Migration

| Function | الغرض |
|---|---|
| `irq_set_affinity_locked` | set affinity وانت شايل الـ lock |
| `irq_set_vcpu_affinity` | set vCPU affinity |
| `irq_migrate_all_off_this_cpu` | نقل كل IRQs من CPU قبل offline |
| `irq_affinity_online_cpu` | restore IRQ affinity عند CPU online |
| `irq_move_irq` | inline check ثم move |
| `__irq_move_irq` | الـ actual move logic |
| `irq_move_masked_irq` | move وهو masked |
| `irq_can_move_in_process_context` | هل ينفع نعمل move هنا؟ |

#### Group 9: Matrix Allocator

| Function | الغرض |
|---|---|
| `irq_alloc_matrix` | alloc matrix struct |
| `irq_matrix_online` / `irq_matrix_offline` | mark CPU online/offline |
| `irq_matrix_assign_system` | حجز bit لـ system use |
| `irq_matrix_reserve_managed` / `irq_matrix_remove_managed` | managed reservation |
| `irq_matrix_alloc_managed` | alloc managed IRQ |
| `irq_matrix_reserve` / `irq_matrix_remove_reserved` | generic reservation |
| `irq_matrix_alloc` / `irq_matrix_free` | alloc/free IRQ vector |
| `irq_matrix_assign` | assign bit manually |
| `irq_matrix_available` / `irq_matrix_allocated` / `irq_matrix_reserved` | stats |
| `irq_matrix_debug_show` | debug dump |

#### Group 10: IPI

| Function | الغرض |
|---|---|
| `ipi_get_hwirq` | hwirq لـ IPI على CPU معين |
| `ipi_send_single` | إرسال IPI لـ CPU واحد |
| `ipi_send_mask` | إرسال IPI لـ cpumask |
| `__ipi_send_single` / `__ipi_send_mask` | internal (من desc) |
| `ipi_mux_create` | إنشاء muxed IPI |
| `ipi_mux_process` | معالجة muxed IPIs |

#### Group 11: Top-level Handler

| Function | الغرض |
|---|---|
| `set_handle_irq` | تسجيل top-level arch IRQ handler |
| `generic_handle_arch_irq` | استدعاء الـ registered handler |
| `handle_arch_irq` | pointer للـ registered handler |

---

### Group 1: `irqd_*` State Accessors

الـ `irq_common_data.state_use_accessors` هو bitmask بيحمل الـ runtime state للـ interrupt. مفيش code خارج الـ IRQ core يلمسه مباشرة — كل الوصول عبر هذه الـ inline accessors.

الـ `__irqd_to_state(d)` هو macro داخلي بيوصل للـ private field، بيتـ `#undef` بعد Section الـ accessors عشان يمنع الاستخدام المباشر من خارج الملف.

---

#### `irqd_is_setaffinity_pending`

```c
static inline bool irqd_is_setaffinity_pending(struct irq_data *d)
```

بتشوف لو فيه affinity change طُلب لكن لسه ما اتطبقش. بيحصل ده لما IRQ بيكون شغّال وقت طلب الـ affinity change على أنظمة `GENERIC_PENDING_IRQ`. الـ pending move هيتعمل في أول فرصة مناسبة.

- **d**: `irq_data` للـ interrupt
- **Returns**: `true` لو `IRQD_SETAFFINITY_PENDING` set
- **Key**: لا lock مطلوب — atomic read على bitmask
- **Caller**: `irq_move_irq()` و affinity management code

---

#### `irqd_is_per_cpu`

```c
static inline bool irqd_is_per_cpu(struct irq_data *d)
```

بتتحقق إن الـ IRQ معلّم `IRQD_PER_CPU`. الـ per-CPU interrupts مش بيتعملهم affinity balancing وبيتعاملوا معاهم بشكل مختلف في الـ core.

- **d**: `irq_data` للـ interrupt
- **Returns**: `true` لو `IRQD_PER_CPU` set
- **Key**: يُستخدم في affinity checks عشان يتجنب محاولة set affinity على per-CPU IRQs

---

#### `irqd_can_balance`

```c
static inline bool irqd_can_balance(struct irq_data *d)
```

بترجع `true` بس لو مش per-CPU ومش explicitly marked no-balancing. هي الـ gating check قبل أي affinity optimization.

- **Returns**: `false` لو `IRQD_PER_CPU` أو `IRQD_NO_BALANCING` set
- **Caller**: `irq_balancing_disabled()` و IRQ affinity management

---

#### `irqd_get_trigger_type` / `irqd_set_trigger_type`

```c
static inline u32 irqd_get_trigger_type(struct irq_data *d)
static inline void irqd_set_trigger_type(struct irq_data *d, u32 type)
```

**الـ `irqd_get_trigger_type`** بترجع الـ 4 bits السفلية من `state_use_accessors` اللي بتمثل الـ trigger type (edge/level).

**الـ `irqd_set_trigger_type`** بتكتب الـ trigger type وبتعمل set لـ `IRQD_DEFAULT_TRIGGER_SET` flag، اللي بيدل إن الـ type اتحدد explicitly مش بالـ default.

- **type**: قيمة من `IRQ_TYPE_*` enum (فقط الـ SENSE_MASK bits)
- **Key**: لازم تتعمل فقط من `irq_chip.irq_set_type()` أو DT/ACPI setup — مش thread-safe بدون lock
- **Side effect**: بيستخدمها `irqd_trigger_type_was_set()` عشان يعرف لو HW ابتدى فعلاً

---

#### `irqd_irq_disabled` / `irqd_irq_masked` / `irqd_irq_inprogress`

```c
static inline bool irqd_irq_disabled(struct irq_data *d)
static inline bool irqd_irq_masked(struct irq_data *d)
static inline bool irqd_irq_inprogress(struct irq_data *d)
```

ثلاث accessors بيقروا الـ runtime state:

| Flag | معناه |
|---|---|
| `IRQD_IRQ_DISABLED` | الـ interrupt اتعمله `disable_irq()` — software disable |
| `IRQD_IRQ_MASKED` | الـ HW mask اتعمل — chip مش هيعمل delivery |
| `IRQD_IRQ_INPROGRESS` | handler شغّال دلوقتي على CPU |

الـ `in_progress` flag مهم لـ spurious detection ولـ `IRQD_RESEND_WHEN_IN_PROGRESS` logic.

---

#### `irqd_set_forwarded_to_vcpu` / `irqd_clr_forwarded_to_vcpu` / `irqd_is_forwarded_to_vcpu`

```c
static inline void irqd_set_forwarded_to_vcpu(struct irq_data *d)
static inline void irqd_clr_forwarded_to_vcpu(struct irq_data *d)
static inline bool irqd_is_forwarded_to_vcpu(struct irq_data *d)
```

دي مجموعة hypervisor-related. لما interrupt بيتحول لـ guest VM (مثلاً GIC ITS في KVM)، بيتعمله set `IRQD_FORWARDED_TO_VCPU`. ده بيأثر على behavior الـ mask/unmask في الـ host.

- **Caller**: KVM و hypervisor IRQ passthrough code
- **Key**: لا يحتاج lock لأنه atomic bitop على private state

---

#### `irqd_set_can_reserve` / `irqd_clr_can_reserve` / `irqd_can_reserve`

```c
static inline void irqd_set_can_reserve(struct irq_data *d)
static inline void irqd_clr_can_reserve(struct irq_data *d)
static inline bool irqd_can_reserve(struct irq_data *d)
```

الـ **reservation mode** بيسمح للـ MSI interrupt يتعمله alloc vector لكن يفضل inactive لحد ما يتعمله activate فعلاً. ده مهم لـ PCI MSI اللي بيتعملها alloc قبل ما الـ device يتعمل enable.

---

#### `irqd_to_hwirq`

```c
static inline irq_hw_number_t irqd_to_hwirq(struct irq_data *d)
```

أبسط accessor — بترجع `d->hwirq` مباشرة. الـ **`hwirq`** هو رقم الـ interrupt في نطاق الـ interrupt controller (مش الـ Linux virtual IRQ number).

---

### Group 2: Flow Handlers

دي الـ handlers اللي بتتحط في `irq_desc->handle_irq`. كل واحدة مناسبة لـ flow type معين.

---

#### `handle_level_irq`

```c
extern void handle_level_irq(struct irq_desc *desc);
```

الـ classic level-triggered handler. بتعمل **mask** للـ interrupt قبل ما تنفّذ الـ action handler، وبعدين **unmask** بعد ما ينتهي. ده ضروري لأن level-triggered interrupts هتفضل تجي طول ما الـ signal موجود.

**Pseudocode:**
```c
handle_level_irq(desc):
    raw_spin_lock(&desc->lock)
    mask_irq(desc)           // منع re-entry
    ack_irq(desc)
    if not IRQ_DISABLED and not IRQ_INPROGRESS:
        handle_irq_event(desc)  // ينفّذ الـ action
    unmask_irq(desc)
    raw_spin_unlock(&desc->lock)
```

- **Caller**: يتحدد عبر `irq_set_chip_and_handler`، بيتنادى من `__handle_irq_event_percpu`
- **Key**: الـ lock بيحمي الـ state — لا تنادي من NMI context

---

#### `handle_edge_irq`

```c
extern void handle_edge_irq(struct irq_desc *desc);
```

لـ edge-triggered interrupts. بتعمل **ack فوري** للـ hardware (عشان يقدر يستقبل edge جديد) وبعدين تشغّل الـ handler. لو interrupt وصل وهو `INPROGRESS`، بتعمل set لـ `IRQD_RESEND` flag عشان ما يضيعش.

**Pseudocode:**
```c
handle_edge_irq(desc):
    raw_spin_lock(&desc->lock)
    if INPROGRESS:
        set_resend_flag()
        mask_ack_irq()       // mask مؤقت
        goto out
    ack_irq(desc)            // ack فوري للـ HW
    mark_INPROGRESS
    handle_irq_event(desc)
    clear_INPROGRESS
    if resend_needed:
        check_and_retrigger()
    raw_spin_unlock(&desc->lock)
```

---

#### `handle_fasteoi_irq`

```c
extern void handle_fasteoi_irq(struct irq_desc *desc);
```

الأكثر استخداماً في الأنظمة الحديثة (ARM GIC، x86 APIC MSI). اسمه "fast EOI" لأنه بيعمل **EOI في الآخر** مش في الأول. مش محتاج mask/unmask — الـ GIC بيمسك الـ interrupt automatically لحد ما يجي الـ EOI.

**Pseudocode:**
```c
handle_fasteoi_irq(desc):
    raw_spin_lock(&desc->lock)
    if DISABLED or (INPROGRESS and !ONESHOT):
        mask_irq(); goto out_eoi
    mark_INPROGRESS
    handle_irq_event(desc)
    clear_INPROGRESS
out_eoi:
    chip->irq_eoi(irq_data)  // EOI في الآخر
    raw_spin_unlock(&desc->lock)
```

- **Key**: الـ EOI بيحصل حتى لو الـ IRQ اتـ mask — ده ضروري لـ GIC عشان ما يـ stall-ش

---

#### `handle_fasteoi_nmi`

```c
extern void handle_fasteoi_nmi(struct irq_desc *desc);
```

نسخة NMI من `handle_fasteoi_irq`. مش بتاخد lock (NMI مش ممكن يتـ preempt)، وبتنادي الـ NMI-safe action handlers مباشرة.

---

#### `handle_simple_irq`

```c
extern void handle_simple_irq(struct irq_desc *desc);
```

أبسط handler — بتنادي `handle_irq_event` مباشرة بدون أي mask/unmask أو ack. مستخدمة لما الـ chip بيدير كل ده بنفسه (مثلاً GPIO expanders بتعمل ack في hardware automatically).

---

#### `handle_percpu_irq` / `handle_percpu_devid_irq`

```c
extern void handle_percpu_irq(struct irq_desc *desc);
extern void handle_percpu_devid_irq(struct irq_desc *desc);
```

**`handle_percpu_irq`** مخصوصة لـ per-CPU interrupts (timers، performance counters). مش بتاخد `desc->lock` لأن per-CPU interrupts مش بتشترك بين CPUs.

**`handle_percpu_devid_irq`** مشابهة بس بتجيب `dev_id` من الـ per-CPU variable عوض الـ shared pointer — ده ضروري لما كل CPU عنده state منفصل (مثلاً per-CPU IRQ chip instances).

---

#### `handle_nested_irq`

```c
extern void handle_nested_irq(unsigned int irq);
```

لـ interrupts اللي بتجي من slow-bus chips (I2C GPIO expanders مثلاً). الـ parent IRQ handler بينادي `handle_nested_irq()` من threaded context، فالـ nested IRQ handler شغّال في نفس thread الـ parent من غير إنشاء thread جديد.

- **irq**: Linux virtual IRQ number (مش `irq_data`)
- **Key**: لازم الـ IRQ يكون معمله `IRQ_NESTED_THREAD` — لو حاولت تشغله من interrupt context هيـ warn

---

#### `handle_bad_irq`

```c
extern void handle_bad_irq(struct irq_desc *desc);
```

الـ default handler للـ spurious interrupts. بتزود counter وبتعمل ack فقط. لو كتير الـ spurious، الـ IRQ debugging code ممكن يعمله disable.

---

### Group 3: Registration & Configuration

---

#### `irq_set_chip`

```c
extern int irq_set_chip(unsigned int irq, const struct irq_chip *chip);
```

بتربط `irq_chip` implementation بـ IRQ descriptor. ده خطوة أساسية في تسجيل أي interrupt controller.

- **irq**: Linux virtual IRQ number
- **chip**: pointer لـ `irq_chip` struct (بيُخزن بالـ pointer، مش copied)
- **Returns**: `0` على success، `-EINVAL` لو `irq` invalid
- **Key**: بيمسك `irq_desc` lock. لو `chip == NULL`، بيحط `no_irq_chip`
- **Caller**: IRQ chip drivers في `probe()` أو irqdomain `map()` callback

---

#### `irq_set_chip_and_handler_name`

```c
extern void irq_set_chip_and_handler_name(unsigned int irq,
                                          const struct irq_chip *chip,
                                          irq_flow_handler_t handle,
                                          const char *name);
```

الـ most complete registration call. بتعمل `irq_set_chip()` وبتحط الـ flow handler و اسمه في خطوة واحدة.

- **handle**: function pointer من نوع `void (*)(struct irq_desc *)` — واحدة من الـ `handle_*_irq` functions
- **name**: ظاهر في `/proc/interrupts` جنب الـ chip name
- **Caller**: أغلب interrupt controllers في `probe()` أو irqdomain `map()`

---

#### `__irq_set_handler` / `irq_set_handler` / `irq_set_chained_handler`

```c
extern void __irq_set_handler(unsigned int irq, irq_flow_handler_t handle,
                               int is_chained, const char *name);
static inline void irq_set_handler(unsigned int irq, irq_flow_handler_t handle)
static inline void irq_set_chained_handler(unsigned int irq,
                                            irq_flow_handler_t handle)
```

**`irq_set_handler`**: بدّل flow handler فقط (بدون chip change)، `is_chained=0`.

**`irq_set_chained_handler`**: لـ IRQ controllers اللي بيكونوا slaves لـ parent IRQ (مثلاً GPIO controller متوصل بـ GIC line). بتعمل `is_chained=1` اللي بيعمل automatically:
- `IRQ_NOREQUEST | IRQ_NOPROBE | IRQ_NOTHREAD` على الـ IRQ
- enable الـ IRQ مباشرة (لأنه محتاج يكون شغّال دايماً يستقبل الـ cascade)

---

#### `irq_set_chained_handler_and_data`

```c
void irq_set_chained_handler_and_data(unsigned int irq,
                                       irq_flow_handler_t handle,
                                       void *data);
```

زي `irq_set_chained_handler` بالإضافة إنها بتعمل set لـ `handler_data`. الـ GPIO controller driver بيحتاج ده عشان يعرف أي controller instance هو من الـ parent handler.

---

#### `irq_modify_status`

```c
void irq_modify_status(unsigned int irq, unsigned long clr, unsigned long set);
```

الـ atomic clear-then-set على الـ `irq_desc->status_use_accessors`. بتعمل clear للـ `clr` bits وبعدين set للـ `set` bits.

- **irq**: Linux IRQ number
- **clr**: bits تتمسح (من `IRQF_MODIFY_MASK`)
- **set**: bits تتحط
- **Key**: بيمسك `irq_desc` lock. فقط bits موجودة في `IRQF_MODIFY_MASK` بتتعدّل
- **Caller**: كل الـ `irq_set_noprobe()`, `irq_set_nothread()` etc. wrappers

---

#### `irq_set_irq_type`

```c
extern int irq_set_irq_type(unsigned int irq, unsigned int type);
```

بتغير الـ trigger type للـ IRQ. بتنادي `irq_chip.irq_set_type()` callback لو موجود، وبتعمل update للـ software flags.

- **type**: `IRQ_TYPE_EDGE_RISING`، `IRQ_TYPE_LEVEL_HIGH`، إلخ
- **Returns**: `0` على success، error code لو الـ chip مش بيدعم التغيير
- **Key**: بيعمل mask للـ IRQ قبل الـ type change لو `IRQCHIP_SET_TYPE_MASKED` flag موجود

---

#### `irq_set_percpu_devid` / `irq_set_percpu_devid_flags`

```c
extern int irq_set_percpu_devid(unsigned int irq);
static inline void irq_set_percpu_devid_flags(unsigned int irq)
```

**`irq_set_percpu_devid`**: بتعمل alloc لـ per-CPU `dev_id` array.

**`irq_set_percpu_devid_flags`**: بتعمل set مجموعة flags دفعة واحدة:
```c
IRQ_NOAUTOEN | IRQ_PER_CPU | IRQ_NOTHREAD | IRQ_NOPROBE | IRQ_PER_CPU_DEVID
```
ده standard setup لأي per-CPU interrupt.

---

### Group 4: Data Getters

---

#### `irq_get_irq_data`

```c
extern struct irq_data *irq_get_irq_data(unsigned int irq);
```

الـ fundamental lookup function. بتترجم Linux virtual IRQ number لـ `irq_data*`. الـ other getters كلها بتبني عليها.

- **irq**: Linux virtual IRQ number
- **Returns**: `irq_data*` أو `NULL` لو invalid IRQ
- **Key**: No lock — الـ descriptor وجوده stable بعد allocation، بس الـ fields جوّاه محتاجة الـ proper locking حسب context

---

#### `irq_get_chip_data` / `irq_data_get_irq_chip_data`

```c
static inline void *irq_get_chip_data(unsigned int irq)
static inline void *irq_data_get_irq_chip_data(struct irq_data *d)
```

بيجيبوا الـ `chip_data` — private data بتاعة الـ irq_chip implementation. مثلاً في GIC، ده بيكون pointer لـ `struct gic_chip_data`.

---

#### `irq_data_get_affinity_mask` / `irq_data_update_affinity`

```c
static inline const struct cpumask *irq_data_get_affinity_mask(struct irq_data *d)
static inline void irq_data_update_affinity(struct irq_data *d,
                                             const struct cpumask *m)
```

الـ **affinity mask** بيحدد أنهي CPUs مسموح تستقبل الـ interrupt. في non-SMP، بيرجع `cpumask_of(0)`.

**`irq_data_update_affinity`** بتعمل `cpumask_copy()` — الـ caller مسؤول عن الـ locking (`irq_desc->lock`).

---

#### `irq_data_get_effective_affinity_mask`

```c
static inline const struct cpumask *
irq_data_get_effective_affinity_mask(struct irq_data *d)
```

الـ **effective affinity** هو الـ actual CPUs اللي بيستقبلوا الـ interrupt فعلاً، ممكن يكون subset من الـ requested affinity (لأن بعض الـ chips مش بتدعم multi-CPU delivery). مهم للـ IRQ balancing monitoring.

---

### Group 5: Descriptor Allocation

---

#### `__irq_alloc_descs`

```c
int __irq_alloc_descs(int irq, unsigned int from, unsigned int cnt,
                       int node, struct module *owner,
                       const struct irq_affinity_desc *affinity);
```

الـ core allocation function للـ IRQ descriptors. بتعمل alloc block من `cnt` descriptors.

- **irq**: لو `-1` = allocate dynamically، غيره = طلب رقم معين
- **from**: أصغر رقم IRQ مقبول
- **cnt**: عدد الـ descriptors المطلوبة
- **node**: NUMA node للـ allocation
- **owner**: الـ module المسؤول (لـ refcounting)
- **affinity**: array من affinity hints (لـ MSI multi-vector)
- **Returns**: أول IRQ number مخصص، أو negative error code
- **Key**: بيمسك global `sparse_irq_lock` spinlock. لا تنادي من interrupt context

---

#### `irq_free_descs`

```c
void irq_free_descs(unsigned int irq, unsigned int cnt);
```

بتحرر block من descriptors. بتتنادى عادةً من platform cleanup أو irqdomain teardown.

- **Key**: بيمسك `sparse_irq_lock`. الـ descriptors لازم تكون free من أي handlers قبل التحرير

---

### Group 6: Generic IRQ Chip (GC)

الـ **Generic IRQ Chip** framework بيوفر abstraction layer فوق الـ register-level operations. بدل ما كل driver يكتب نفس الـ mask/unmask/ack boilerplate، بيقول "الـ mask register offset هو كذا، والـ enable register offset هو كذا" وبيدّيه الـ GC.

---

#### `irq_alloc_generic_chip`

```c
struct irq_chip_generic *
irq_alloc_generic_chip(const char *name, int nr_ct, unsigned int irq_base,
                        void __iomem *reg_base, irq_flow_handler_t handler);
```

بتعمل alloc وتهيئة `irq_chip_generic` struct. بتعمل alloc للـ struct مع `nr_ct` instances من `irq_chip_type` في آخره (flexible array).

- **name**: اسم الـ chip (لـ `/proc/interrupts`)
- **nr_ct**: عدد `irq_chip_type` instances — عادةً 1، لكن ممكن 2 لو محتاج تدعم edge + level بـ registers مختلفة
- **irq_base**: أول Linux IRQ number اللي الـ chip بيديره
- **reg_base**: virtual address للـ register block
- **handler**: default flow handler (`handle_level_irq` مثلاً)
- **Returns**: pointer للـ allocated GC، أو `NULL`
- **Key**: بيعمل alloc بـ `kzalloc` — context يقبل sleep

---

#### `irq_setup_generic_chip`

```c
void irq_setup_generic_chip(struct irq_chip_generic *gc, u32 msk,
                             enum irq_gc_flags flags, unsigned int clr,
                             unsigned int set);
```

بتربط الـ GC بالـ actual IRQ descriptors وبتهيّئ الـ chip_type callbacks.

- **gc**: الـ generic chip
- **msk**: bitmask بيحدد أنهي IRQs (relative لـ `gc->irq_base`) يتعملوا setup
- **flags**: `IRQ_GC_INIT_MASK_CACHE` لقراءة initial mask من HW، `IRQ_GC_BE_IO` لـ big-endian، إلخ
- **clr**: IRQ_* flags يتمسحوا على كل IRQ
- **set**: IRQ_* flags يتحطوا على كل IRQ
- **Key**: بتنادي `irq_set_chip_and_handler` على كل IRQ في الـ mask — context يقبل sleep

**Pseudocode:**
```c
irq_setup_generic_chip(gc, msk, flags, clr, set):
    ct = &gc->chip_types[0]
    if IRQ_GC_INIT_MASK_CACHE:
        gc->mask_cache = irq_reg_readl(gc, ct->regs.mask)

    for each bit in msk:
        irq = gc->irq_base + bit
        irq_set_chip_and_handler(irq, &ct->chip, ct->handler)
        irq_modify_status(irq, clr, set)
        gc->installed |= BIT(bit)
```

---

#### `irq_setup_alt_chip`

```c
int irq_setup_alt_chip(struct irq_data *d, unsigned int type);
```

بتغيّر الـ active `irq_chip_type` للـ interrupt بناءً على الـ flow type. مفيدة لما الـ edge والـ level بيتطلبوا registers مختلفة على نفس الـ controller.

- **type**: الـ trigger type الجديد (`IRQ_TYPE_EDGE_*` أو `IRQ_TYPE_LEVEL_*`)
- **Returns**: `0` على success، `-EINVAL` لو مفيش chip type بيدعم الـ type ده

---

#### `irq_remove_generic_chip`

```c
void irq_remove_generic_chip(struct irq_chip_generic *gc, u32 msk,
                              unsigned int clr, unsigned int set);
```

عكس `irq_setup_generic_chip`. بتفصل الـ GC من الـ IRQ descriptors المحددة بالـ mask. بتمسح الـ `installed` bits وبتعمل reset للـ chip لـ `no_irq_chip`.

---

#### `irq_gc_mask_disable_reg` / `irq_gc_unmask_enable_reg`

```c
void irq_gc_mask_disable_reg(struct irq_data *d);
void irq_gc_unmask_enable_reg(struct irq_data *d);
```

للـ controllers اللي عندهم registers منفصلة للـ disable والـ enable (مش register واحد بيتعمله set/clear bits).

- `irq_gc_mask_disable_reg`: بتكتب الـ IRQ bit في `regs.disable` register
- `irq_gc_unmask_enable_reg`: بتكتب الـ IRQ bit في `regs.enable` register

---

#### `irq_gc_mask_set_bit` / `irq_gc_mask_clr_bit`

```c
void irq_gc_mask_set_bit(struct irq_data *d);
void irq_gc_mask_clr_bit(struct irq_data *d);
```

للـ controllers اللي عندهم register واحد (write 1 to mask, write 0 to unmask):

- `irq_gc_mask_set_bit`: بتعمل set للـ bit في `regs.mask` (= mask = disable)
- `irq_gc_mask_clr_bit`: بتعمل clear للـ bit في `regs.mask` (= unmask = enable)

بيعملوا update للـ `mask_cache` أيضاً.

---

#### `irq_gc_ack_set_bit` / `irq_gc_ack_clr_bit`

```c
void irq_gc_ack_set_bit(struct irq_data *d);
void irq_gc_ack_clr_bit(struct irq_data *d);
```

للـ ack operation:

- `irq_gc_ack_set_bit`: كتابة 1 للـ ack bit (common في edge controllers)
- `irq_gc_ack_clr_bit`: كتابة 0 للـ ack bit

---

#### `irq_gc_set_wake`

```c
int irq_gc_set_wake(struct irq_data *d, unsigned int on);
```

بتعمل enable/disable الـ IRQ كـ wakeup source. بتعمل update للـ `gc->wake_enabled` وبتخزّن mask في `gc->wake_active`.

- **on**: 1 = enable wakeup، 0 = disable
- **Returns**: `0` دايماً (generic implementation)

---

#### `irq_reg_writel` / `irq_reg_readl`

```c
static inline void irq_reg_writel(struct irq_chip_generic *gc,
                                   u32 val, int reg_offset)
static inline u32 irq_reg_readl(struct irq_chip_generic *gc,
                                  int reg_offset)
```

بيعملوا I/O على `gc->reg_base + reg_offset`. لو الـ GC عنده custom `reg_writel`/`reg_readl` callbacks (مثلاً لـ big-endian)، بيستخدموها. غيره بيستخدموا `writel`/`readl` العادية.

---

#### `irq_map_generic_chip` / `irq_unmap_generic_chip`

```c
int irq_map_generic_chip(struct irq_domain *d, unsigned int virq,
                          irq_hw_number_t hw_irq);
void irq_unmap_generic_chip(struct irq_domain *d, unsigned int virq);
```

callbacks جاهزة تحطها في `irq_domain_ops` للـ domains اللي بتستخدم GC. `irq_map_generic_chip` بتلاقي الـ correct `irq_chip_generic` من الـ `hw_irq` وبتعمل setup الـ descriptor.

---

#### `irq_domain_alloc_generic_chips`

```c
int irq_domain_alloc_generic_chips(struct irq_domain *d,
                                    const struct irq_domain_chip_generic_info *info);
```

الـ modern API لـ GC مع irqdomain. بتاخد `irq_domain_chip_generic_info` struct وبتعمل alloc وبتربط كل الـ GC instances للـ domain.

- **info**: struct بتحدد اسم الـ chip، handler، عدد IRQs per chip، init/exit callbacks، إلخ
- **Returns**: `0` على success

---

### Group 7: irq_chip Hierarchy Helpers

في الـ hierarchical irqdomain (مثلاً: GIC → ITS → device، أو GPIO → GIC)، كل level له `irq_data` خاص. هذه الـ functions بتعمل "forward" للـ operations للـ parent level.

---

#### `irq_chip_enable_parent` / `irq_chip_disable_parent`

```c
extern void irq_chip_enable_parent(struct irq_data *data);
extern void irq_chip_disable_parent(struct irq_data *data);
```

بيستدعيوا `irq_chip.irq_enable` أو `irq_chip.irq_disable` على `data->parent_data`. يُستخدموا من child irq_chip بيريد يعمل enable/disable على الـ parent hardware.

---

#### `irq_chip_ack_parent`

```c
extern void irq_chip_ack_parent(struct irq_data *data);
```

بتعمل forward للـ ack لـ parent chip. ضروري في cascade setups لما الـ child chip يعمل ack بس الـ parent كمان محتاج ack.

---

#### `irq_chip_retrigger_hierarchy`

```c
extern int irq_chip_retrigger_hierarchy(struct irq_data *data);
```

بتعمل retrigger على كل levels في الـ hierarchy. بتمشي لفوق في الـ `parent_data` chain ولما تلاقي `irq_chip.irq_retrigger`، بتنادية.

- **Returns**: `1` لو retrigger نجح، `0` لو مفيش chip قدر يعمله

---

#### `irq_chip_set_affinity_parent`

```c
extern int irq_chip_set_affinity_parent(struct irq_data *data,
                                         const struct cpumask *dest,
                                         bool force);
```

بتعمل forward لـ affinity set request للـ parent. مهمة في setups زي GIC ITS حيث الـ affinity بيتحدد على مستوى أعلى.

---

#### `irq_chip_compose_msi_msg`

```c
extern int irq_chip_compose_msi_msg(struct irq_data *data,
                                     struct msi_msg *msg);
```

بتمشي في الـ hierarchy وبتنادي `irq_chip.irq_compose_msi_msg` على أول chip عنده الـ callback. ده بيسمح لـ MSI message تتبنى بطريقة صح للـ irqchip المناسب في الـ chain.

---

#### `irq_chip_pm_get` / `irq_chip_pm_put`

```c
extern int irq_chip_pm_get(struct irq_data *data);
extern void irq_chip_pm_put(struct irq_data *data);
```

بيعملوا `pm_runtime_get_sync`/`pm_runtime_put` على الـ device المرتبط بالـ irq_chip. ضروري لـ chips محتاجة power management قبل الوصول للـ registers.

- **Returns** (`pm_get`): `0` على success، negative error لو الـ device مش available
- **Caller**: الـ IRQ core قبل ما تنادي أي chip callback

---

### Group 8: Affinity & Migration

---

#### `irq_set_affinity_locked`

```c
extern int irq_set_affinity_locked(struct irq_data *data,
                                    const struct cpumask *cpumask, bool force);
```

بتعمل set للـ affinity وأنت شايل `irq_desc->lock`. الـ caller مسؤول عن الـ locking.

- **force**: لو `true`، بتتجاوز sanity checks (used لـ CPU hotplug)
- **Returns**: `0` على success، error لو الـ chip رفض

---

#### `irq_migrate_all_off_this_cpu`

```c
extern void irq_migrate_all_off_this_cpu(void);
```

بتنادى من CPU hotplug teardown (على الـ CPU اللي بيتـ offline). بتمشي على كل الـ IRQs المحددة للـ CPU ده وبتنقلهم لـ CPU تاني. بتنادى من migration thread قبل الـ CPU يتوقف.

---

#### `irq_move_irq` / `__irq_move_irq`

```c
static inline void irq_move_irq(struct irq_data *data)
extern void __irq_move_irq(struct irq_data *data);
```

**`irq_move_irq`** هي inline check بتشوف `IRQD_SETAFFINITY_PENDING` وبتنادي `__irq_move_irq` بس لو لازم. بيتنادوا من flow handlers بعد الـ ack وقبل الـ handler عشان يطبقوا الـ pending affinity changes.

---

#### `irq_move_masked_irq`

```c
void irq_move_masked_irq(struct irq_data *data);
```

زي `irq_move_irq` بس يتنادى لما الـ IRQ معمول mask. بيُستخدم في `handle_edge_irq` حيث الـ IRQ ممكن يكون masked في وقت الـ migration.

---

#### `irq_can_move_in_process_context`

```c
bool irq_can_move_in_process_context(struct irq_data *data);
```

بتتحقق إن الـ affinity move ينفع يحصل في process context. لو الـ chip معمله `IRQCHIP_MOVE_DEFERRED`، الـ move لازم يحصل من interrupt context فقط.

---

### Group 9: IRQ Matrix Allocator

الـ **irq_matrix** بيدير الـ vector allocation على x86 والـ architectures اللي بتطلب mapping بين IRQ numbers وـ CPU-specific hardware vectors (مثلاً APIC vectors).

---

#### `irq_alloc_matrix`

```c
struct irq_matrix *irq_alloc_matrix(unsigned int matrix_bits,
                                     unsigned int alloc_start,
                                     unsigned int alloc_end);
```

بتعمل alloc لـ matrix struct بيتتبع الـ vector allocation لكل CPU.

- **matrix_bits**: إجمالي bits في الـ matrix (عدد الـ possible vectors)
- **alloc_start** / **alloc_end**: range مسموح بالـ dynamic allocation فيه
- **Returns**: `struct irq_matrix*` أو `NULL`

---

#### `irq_matrix_online` / `irq_matrix_offline`

```c
void irq_matrix_online(struct irq_matrix *m);
void irq_matrix_offline(struct irq_matrix *m);
```

بيعملوا mark لـ per-CPU matrix entry كـ available أو unavailable. لازم يتنادوا من CPU hotplug callbacks.

---

#### `irq_matrix_alloc` / `irq_matrix_free`

```c
int irq_matrix_alloc(struct irq_matrix *m, const struct cpumask *msk,
                      bool reserved, unsigned int *mapped_cpu);
void irq_matrix_free(struct irq_matrix *m, unsigned int cpu,
                      unsigned int bit, bool managed);
```

**`irq_matrix_alloc`**: بتعمل alloc لـ vector على أحسن CPU في الـ mask.
- **reserved**: لو `true`، بتستخدم الـ reserved slot مش free slot
- **mapped_cpu**: output — أنهي CPU اتـ allocate عليه
- **Returns**: vector number أو negative error

**`irq_matrix_free`**: بترجع الـ vector للـ pool. الـ `managed` flag بيحدد هل ده managed IRQ (لأنه بيأثر على counters).

---

#### `irq_matrix_reserve_managed` / `irq_matrix_alloc_managed`

```c
int irq_matrix_reserve_managed(struct irq_matrix *m,
                                 const struct cpumask *msk);
int irq_matrix_alloc_managed(struct irq_matrix *m,
                               const struct cpumask *msk,
                               unsigned int *mapped_cpu);
```

للـ **managed interrupts** (MSI-X vectors اللي بيتعملهم affinity management تلقائي). الـ reservation بتحجز space، والـ allocation بتعمل assign الـ actual vector.

---

### Group 10: IPI Functions

---

#### `ipi_send_single` / `ipi_send_mask`

```c
int ipi_send_single(unsigned int virq, unsigned int cpu);
int ipi_send_mask(unsigned int virq, const struct cpumask *dest);
```

بيبعتوا **Inter-Processor Interrupt** لـ CPU واحد أو cpumask. بيستخدموا `irq_chip.ipi_send_single` أو `ipi_send_mask` callbacks.

- **virq**: Linux virtual IRQ number للـ IPI
- **Returns**: `0` على success، negative error

---

#### `ipi_get_hwirq`

```c
irq_hw_number_t ipi_get_hwirq(unsigned int irq, unsigned int cpu);
```

بتجيب الـ hwirq المقابل للـ IPI على CPU معين. الـ IPI لكل CPU ممكن يكون له hwirq مختلف (حسب الـ offset المحدد في `irq_common_data.ipi_offset`).

---

#### `ipi_mux_create` / `ipi_mux_process`

```c
int ipi_mux_create(unsigned int nr_ipi, void (*mux_send)(unsigned int cpu));
void ipi_mux_process(void);
```

الـ **IPI multiplexer** بيسمح لـ single hardware IPI تحمل multiple software IPI types. مفيد جداً على architectures اللي عندها hardware IPIs محدودة.

**`ipi_mux_create`**: بتسجّل `nr_ipi` software IPIs فوق hardware IPI واحدة. الـ `mux_send` callback هو اللي بيبعت الـ physical IPI.

**`ipi_mux_process`**: بتنادى من الـ hardware IPI handler عشان تعالج كل الـ pending software IPIs.

---

### Group 11: Top-Level Architecture Handler

---

#### `set_handle_irq`

```c
int __init set_handle_irq(void (*handle_irq)(struct pt_regs *));
```

بتسجّل الـ top-level IRQ entry point. ده بيكون أول C code بيتنادى من الـ assembly IRQ entry stub.

- **handle_irq**: function pointer للـ top-level handler (مثلاً `gic_handle_irq`)
- **Returns**: `0` على success، `-EBUSY` لو handler اتسجّل قبل كده
- **Key**: بيتنادى مرة واحدة فقط في `__init` phase
- **Caller**: interrupt controller driver في `init()` function

---

#### `generic_handle_arch_irq`

```c
asmlinkage void generic_handle_arch_irq(struct pt_regs *regs);
```

بتنادي الـ registered `handle_arch_irq` function pointer. `asmlinkage` لأنها بتتنادى من assembly code مع specific calling convention.

**الـ `handle_arch_irq`** هو `__ro_after_init` pointer — بيتحط مرة واحدة في boot وبعدين بيبقى read-only للحماية من kernel exploits.

---

### ملاحظات جوهرية — Key Cross-Cutting Concerns

#### Locking Model

```
irq_desc->lock (raw_spinlock_t)
    └── يحمي: chip pointer, handler, status flags, action list
    └── يتمسك من: flow handlers, irq_set_* functions, enable/disable

irq_chip_generic->lock (raw_spinlock_t)
    └── يحمي: register cache (mask_cache)
    └── يتمسك من: irq_gc_* callbacks

sparse_irq_lock (mutex)
    └── يحمي: IRQ descriptor allocation/free
    └── context: process only
```

#### Context Safety Summary

| Function Category | IRQ Context | Process Context | NMI |
|---|---|---|---|
| `irqd_*` accessors | ✅ | ✅ | ✅ |
| Flow handlers | ✅ | ❌ | فقط NMI handlers |
| `irq_set_*` config | ❌ | ✅ | ❌ |
| `irq_alloc_descs` | ❌ | ✅ | ❌ |
| `irq_gc_*` callbacks | ✅ | ✅ | ❌ |
| `ipi_send_*` | ✅ | ✅ | ❌ |

#### `no_irq_chip` vs `dummy_irq_chip`

- **`no_irq_chip`**: بيترجع warning لو أي callback اتنادى — للـ uninitialized IRQs
- **`dummy_irq_chip`**: no-op callbacks — للـ IRQs اللي مش محتاجة chip operations
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — entries خاصة بالـ IRQ

الـ kernel بيعمل expose لحالة كل IRQ في debugfs لو `CONFIG_GENERIC_IRQ_DEBUGFS=y`:

```bash
# Mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل entries الـ IRQ
ls /sys/kernel/debug/irq/

# شوف state IRQ معين (مثلاً IRQ 42)
cat /sys/kernel/debug/irq/42/irq_domain
cat /sys/kernel/debug/irq/42/actions
cat /sys/kernel/debug/irq/42/hwirq
cat /sys/kernel/debug/irq/42/chip_name
cat /sys/kernel/debug/irq/42/type
cat /sys/kernel/debug/irq/42/wakeup
```

**تفسير الـ output:**
- `irq_domain` → الـ domain المسؤول عن الـ mapping بين hwirq و linux irq
- `actions` → list بأسماء الـ handlers المُسجَّلين على الـ IRQ ده
- `hwirq` → رقم الـ interrupt على الـ hardware controller نفسه
- `chip_name` → اسم الـ `irq_chip` struct المستخدم
- `type` → نوع الـ trigger: EDGE_RISING, LEVEL_HIGH, إلخ

```bash
# اعرض كل descriptors الـ IRQ وحالتها
cat /sys/kernel/debug/irq/domains/default
```

---

#### 2. sysfs — entries خاصة بالـ IRQ

```bash
# كل IRQ ليه directory في /proc/irq/
ls /proc/irq/

# affinity الـ CPU لـ IRQ معين
cat /proc/irq/42/smp_affinity        # bitmask hex
cat /proc/irq/42/smp_affinity_list   # human-readable (e.g. 0-3)

# الـ effective affinity (بعد ما الـ chip يطبقها)
cat /proc/irq/42/effective_affinity
cat /proc/irq/42/effective_affinity_list

# node في NUMA
cat /proc/irq/42/node

# عدد الـ spurious interrupts
cat /proc/irq/42/spurious

# اسم الـ handler
cat /proc/irq/42/*/node   # per-action info

# إحصائيات كل الـ IRQs per-CPU
cat /proc/interrupts
```

**مثال /proc/interrupts output:**
```
           CPU0       CPU1       CPU2       CPU3
  42:      1523       4821        983       2104   GICv3  42 Edge  eth0
```
الأعمدة: irq_number → per-cpu counts → chip name → hwirq → type → handler name

---

#### 3. ftrace — tracepoints للـ IRQ subsystem

الـ IRQ subsystem عنده tracepoints جاهزة تحت `CONFIG_TRACE_IRQFLAGS` و `CONFIG_IRQSOFF_TRACER`:

```bash
# شوف كل events متاحة
grep -r "irq" /sys/kernel/debug/tracing/available_events

# أهم events:
# irq:irq_handler_entry   → عند بداية تنفيذ handler
# irq:irq_handler_exit    → عند انتهاء handler + return value
# irq:softirq_entry/exit  → للـ softirq
# irq:irq_enable/disable  → تتبع enable/disable cycles

# تفعيل irq handler tracing
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اقرأ النتائج
cat /sys/kernel/debug/tracing/trace

# تتبع latency الـ IRQs disabled (يكشف lock-ups)
echo irqsoff > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل الـ workload ...
cat /sys/kernel/debug/tracing/trace
```

**تفسير الـ output:**
```
kworker/0:1-42    [000] d... 1234.567: irq_handler_entry: irq=42 name=eth0
kworker/0:1-42    [000] d... 1234.568: irq_handler_exit:  irq=42 ret=handled
```
- `d...` = IRQ disabled أثناء التنفيذ
- `ret=unhandled` = spurious interrupt — مشكلة!

```bash
# تتبع affinity changes
echo 1 > /sys/kernel/debug/tracing/events/irq_matrix/enable

# function tracer لتتبع دوال معينة زي irq_set_affinity
echo function > /sys/kernel/debug/tracing/current_tracer
echo irq_set_affinity > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. printk و dynamic debug

```bash
# تفعيل dynamic debug لملفات الـ IRQ core
echo 'file kernel/irq/chip.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/irq/manage.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/irq/spurious.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/irq/irqdesc.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/irq/irqdomain.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل الـ IRQ subsystem debug messages
echo 'module irqdomain +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages
dmesg -w | grep -i irq

# رفع loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk

# إيقاف irq debug (noirqdebug)
# في kernel cmdline:
# noirqdebug  → يوقف note_interrupt() من report الـ spurious IRQs
```

---

#### 5. Kernel Config options للـ debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_GENERIC_IRQ_DEBUGFS` | يعمل expose لكل IRQ state في debugfs |
| `CONFIG_GENERIC_IRQ_STAT_SNAPSHOT` | يحفظ snapshot لعدد الـ interrupts عشان يكشف stalls |
| `CONFIG_IRQSOFF_TRACER` | يقيس أقصى فترة IRQs disabled |
| `CONFIG_PREEMPT_TRACER` | يقيس preemption latency |
| `CONFIG_TRACE_IRQFLAGS` | يضيف tracing لـ enable/disable calls |
| `CONFIG_SPARSE_IRQ` | يخلي الـ IRQ descriptors dynamic مع kobj support |
| `CONFIG_DEBUG_SHIRQ` | يتحقق من الـ shared IRQ handlers عند الـ free |
| `CONFIG_HARDIRQS_SW_RESEND` | يسمح بـ software resend للـ edge IRQs |
| `CONFIG_GENERIC_PENDING_IRQ` | يدعم تتبع pending affinity moves |
| `CONFIG_GENERIC_IRQ_EFFECTIVE_AFF_MASK` | يحفظ الـ effective affinity بعد ما الـ chip تطبقها |
| `CONFIG_IRQ_DOMAIN_HIERARCHY` | يدعم stacked irqchips وهيراركية الـ domains |
| `CONFIG_LOCKDEP` | يكشف deadlocks في الـ IRQ locking |
| `CONFIG_DEBUG_SPINLOCK` | يكشف مشاكل الـ spinlock في `irq_desc->lock` |

```bash
# تحقق من الـ config الحالية
zcat /proc/config.gz | grep -E "IRQ_DEBUGFS|IRQSOFF|TRACE_IRQ|DEBUG_SHIRQ"
```

---

#### 6. أدوات خاصة بالـ IRQ subsystem

```bash
# irqbalance — daemon بيعمل distribute الـ IRQs على الـ CPUs
systemctl status irqbalance
irqbalance --debug --foreground

# تعطيل irqbalance مؤقتاً لـ debugging
systemctl stop irqbalance

# ضبط affinity يدوياً (مثلاً IRQ 42 على CPU 2 فقط)
echo 4 > /proc/irq/42/smp_affinity   # بيت 2 = CPU 2

# راقب معدل الـ interrupts في real-time
watch -n1 cat /proc/interrupts

# كمّ أتباع التغيير في الـ interrupts كل ثانية
sar -I 42 1 10

# أداة tuna لضبط thread و IRQ affinity
tuna --irqs=42 --cpus=2 --move

# perf لقياس interrupt-related events
perf stat -e irq_vectors:local_timer_entry,irq_vectors:call_function_entry \
     -a sleep 5
```

---

#### 7. جدول رسائل الـ error الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---------------------|--------|------|
| `irq N: nobody cared` | IRQ بييجي بس مفيش handler بيتعامل معاه (spurious) | تحقق من `irqs_unhandled` في `/proc/irq/N/spurious`، راجع الـ hardware connections |
| `Disabling IRQ #N` | الـ kernel عطّل الـ IRQ بعد كتير من الـ spurious | راجع الـ hardware، restart الـ driver |
| `IRQ: IRQF_SHARED` mismatch | driver بيحاول يشارك IRQ غير shareable | ضيف `IRQF_SHARED` في كل الـ drivers اللي بتستخدم نفس الـ IRQ |
| `enable_irq(N) unbalanced` | `enable_irq` اتنادت أكتر من `disable_irq` | فتش عن missing `disable_irq` في error paths |
| `Unbalanced enable for IRQ N` | نفس المشكلة الفوقانية من irq_desc->depth < 0 | راجع كل `disable_irq`/`enable_irq` pairs |
| `BUG: IRQ handler called in wrong context` | handler اتنادى من outside interrupt context وـ`IRQD_HANDLE_ENFORCE_IRQCTX` set | الـ driver لازم يتأكد إنه بيتنادى من irq context بس |
| `irq_set_affinity failed` | الـ chip مش قادر يعمل set affinity (hardware limitation) | اتحقق من `irq_chip->irq_set_affinity` return value |
| `IRQ N: requested but not present` | الـ IRQ اتطلب من الـ driver بس الـ hardware مش موجود | تحقق من الـ Device Tree أو ACPI tables |
| `kernel BUG at kernel/irq/chip.c` | `irq_chip` operations اتنادت بـ NULL pointers | الـ `irq_chip` struct ناقص fields مطلوبة |
| `mask_irq failed for irq N` | `irq_chip->irq_mask` فشل | راجع base address الـ interrupt controller وصلاحيات الـ I/O |

---

#### 8. نقاط استراتيجية لـ dump_stack() / WARN_ON()

```c
/* في irq_chip->irq_enable() — للتحقق من الـ state قبل enable */
void my_chip_enable(struct irq_data *d)
{
    WARN_ON(!irqd_irq_disabled(d));   /* لازم يكون disabled قبل enable */
    /* ... */
}

/* في irq_chip->irq_set_type() — للتحقق من الـ type صحيح */
int my_chip_set_type(struct irq_data *d, unsigned int type)
{
    WARN_ON(type & ~IRQ_TYPE_SENSE_MASK);
    /* ... */
}

/* في الـ flow handler — للتحقق من الـ desc مش NULL */
void my_handle_irq(struct irq_desc *desc)
{
    struct irq_chip *chip = irq_desc_get_chip(desc);
    WARN_ON(!chip);
    if (WARN_ON(irqd_irq_inprogress(&desc->irq_data)))
        return;   /* IRQ in progress — race condition */
    /* ... */
}

/* عند alloc descriptor — تحقق من الـ domain */
static int my_domain_map(struct irq_domain *d, unsigned int irq,
                          irq_hw_number_t hwirq)
{
    if (WARN_ON(hwirq >= MY_MAX_IRQ)) {
        dump_stack();
        return -EINVAL;
    }
    /* ... */
}
```

**أماكن مفيدة لـ `dump_stack()`:**
- في `irq_chip->irq_mask()` لو الـ mask register ما اتغيرش بعد الكتابة
- في الـ spurious handler (`note_interrupt`) لو `irqs_unhandled` وصل threshold
- في `handle_bad_irq` — ده بيعمل dump تلقائياً

---

### Hardware Level

#### 1. التحقق من حالة الـ Hardware مقابل الـ Kernel

```bash
# قارن hwirq في الـ kernel مع الـ interrupt controller registers
# مثلاً لـ GIC (ARM Generic Interrupt Controller):

# شوف hwirq لكل linux IRQ
for irq in /sys/kernel/debug/irq/*/; do
    echo "Linux IRQ $(basename $irq): hwirq=$(cat $irq/hwirq) chip=$(cat $irq/chip_name)"
done

# تحقق من الـ IRQ state في irq_common_data (عبر debugfs)
cat /sys/kernel/debug/irq/42/irq_domain    # المفروض يطابق الـ DT node

# تحقق إن الـ irq_chip المسجل matches الـ hardware
cat /sys/kernel/debug/irq/42/chip_name
# المفروض يطابق الاسم في irq_chip->name في الـ driver
```

---

#### 2. Register Dump للـ Interrupt Controller

```bash
# devmem2 — لقراءة registers مباشرة من userspace
# مثال: GIC Distributor على Cortex-A (base عادةً 0x08000000)

# GICD_ISENABLER — بيشوف IRQs enabled
devmem2 0x08000100 w   # Enable Set Register 0 (IRQs 0-31)
devmem2 0x08000104 w   # Enable Set Register 1 (IRQs 32-63)

# GICD_ISPENDR — بيشوف IRQs pending
devmem2 0x08000200 w

# GICD_ISACTIVER — بيشوف IRQs active (in-service)
devmem2 0x08000300 w

# GICD_ITARGETSR — بيشوف CPU targets لكل IRQ
devmem2 0x08000800 w   # IRQs 0-3

# لو devmem2 مش متاح، استخدم /dev/mem
python3 -c "
import mmap, struct
with open('/dev/mem','rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0x08000100)
    val = struct.unpack('<I', m.read(4))[0]
    print(f'GICD_ISENABLER0 = 0x{val:08x}')
    m.close()
"

# io utility (من package iotools)
io -4 0x08000100   # قراءة 32-bit register
```

**لـ x86 (APIC/IOAPIC):**
```bash
# قراءة IOAPIC entries
cat /proc/iomem | grep -i apic

# باستخدام rdmsr/wrmsr لـ MSR-based interrupts
rdmsr 0x1B   # APIC_BASE MSR
```

---

#### 3. Logic Analyzer / Oscilloscope

عشان تتحقق من الـ IRQ signals على مستوى الـ hardware:

**نقاط القياس:**
- قس الـ signal على الـ IRQ line مباشرة (بين الـ device والـ interrupt controller)
- تحقق إن الـ signal matches الـ configured trigger type (`IRQ_TYPE_EDGE_RISING` مثلاً)

**إعدادات الـ Logic Analyzer:**
```
Channel:    IRQ_LINE (GPIO pin أو dedicated IRQ pin)
Threshold:  1.8V أو 3.3V حسب الـ SoC
Trigger:    Rising Edge (لو configured كـ EDGE_RISING)
Sample Rate: 10x أسرع من أسرع interrupt متوقع
```

**علامات المشاكل على الـ scope:**
- الـ signal بيرجع low قبل الـ kernel handler يعمل ACK → edge lost
- glitches قصيرة جداً → الـ filter الـ hardware مش مضبوط
- الـ signal مش بيرجع high بعد الـ EOI → الـ chip مش بيعمل EOI صح
- الـ pulse width أقل من 10ns على level-triggered → hardware noise

---

#### 4. Hardware Issues الشائعة وأنماطها في الـ kernel log

| المشكلة | النمط في الـ dmesg |
|---------|-------------------|
| Float على الـ IRQ line (no pull-up) | `irq N: nobody cared` متكرر → Disabling IRQ |
| EOI مش بيحصل (edge IRQ) | `irq_handler_exit: ret=handled` بس الـ IRQ بييجي تاني فوراً |
| Level IRQ مش بيتعمل deassert | `irq N: nobody cared` + الـ CPU load بـ 100% |
| Shared IRQ mismatch في الـ hardware | `Unhandled IRQ` من driver تاني على نفس الـ line |
| Clock إلى الـ interrupt controller مش شغال | `irq_chip->irq_mask failed` أو no interrupts at all |
| Power domain للـ device إيه قبل driver | `request_irq` ينجح بس لا interrupts بتيجي |
| Polarity مغلوط (HIGH بدل LOW) | IRQ بييجي كل الوقت بدل عند الـ event |

---

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT interrupt spec
# الـ kernel بيعمل parse للـ interrupts property في الـ DT

# شوف الـ compiled DT
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "interrupt"

# أو اقرأ الـ properties مباشرة
hexdump -C /sys/firmware/devicetree/base/soc/ethernet@ff0e0000/interrupts

# تحقق من الـ interrupt-parent
cat /sys/firmware/devicetree/base/soc/ethernet@ff0e0000/interrupt-parent | hexdump -C

# قارن مع الـ actual IRQ في kernel
grep eth0 /proc/interrupts
# IRQ رقمه المفروض يطابق الـ hwirq + offset الـ interrupt controller

# تحقق من الـ interrupt-cells (عدد cells لوصف interrupt في الـ DT)
cat /sys/firmware/devicetree/base/interrupt-controller@8000000/\#interrupt-cells | hexdump -C
# GICv3: 3 cells = (type, hwirq, flags)
```

**خطأ شائع في الـ DT:**
```dts
/* خطأ: trigger type غلط */
interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
/* الـ hardware فعلاً edge — ده بيخلي الـ IRQ يفضل active ومايروحش */

/* صح: */
interrupts = <GIC_SPI 42 IRQ_TYPE_EDGE_RISING>;
```

```bash
# تحقق إن الـ irq_domain بيعمل translate صح
cat /sys/kernel/debug/irq/domains/*/name
# المفروض يشوف اسم الـ interrupt-controller من الـ DT
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للـ Debugging

**1. تشخيص سريع لـ IRQ مشكوك فيه:**

```bash
IRQ=42

# الحالة الكاملة
echo "=== IRQ $IRQ Status ==="
grep "^ *$IRQ:" /proc/interrupts
echo "--- Affinity ---"
cat /proc/irq/$IRQ/smp_affinity_list
echo "--- Spurious ---"
cat /proc/irq/$IRQ/spurious
echo "--- debugfs ---"
for f in /sys/kernel/debug/irq/$IRQ/*; do
    echo "$(basename $f): $(cat $f 2>/dev/null)"
done
```

**تفسير الـ output:**
```
=== IRQ 42 Status ===
  42:      1523       4821   GICv3  42 Edge  eth0
--- Affinity ---
0-3
--- Spurious ---
count 0 unhandled 0 last_unhandled 0 ms
--- debugfs ---
actions: eth0
chip_name: GICv3
hwirq: 42
type: edge-rising
wakeup: disabled
```
لو `unhandled > 100` → مشكلة hardware أو driver.

---

**2. مراقبة الـ IRQ rate في real-time:**

```bash
# الفرق في عدد الـ interrupts كل ثانية لكل CPU
watch -d -n1 'cat /proc/interrupts | grep -E "^ *(42|43|44):"'
```

---

**3. تشخيص مشاكل الـ affinity:**

```bash
# شوف كل IRQs وأين بتتنفذ
awk '/^[[:space:]]*[0-9]/' /proc/interrupts | \
  awk '{printf "IRQ %-5s CPU-counts: ", $1; for(i=2;i<=NF-3;i++) printf "%s ", $i; print ""}'

# حدد IRQ مش متوزع صح (كل الـ load على CPU واحد)
awk '/^[[:space:]]*[0-9]/{
    irq=$1; sum=0; max=0;
    for(i=2;i<=NF-3;i++){sum+=$i; if($i>max)max=$i}
    if(sum>0 && max/sum>0.9) print "Imbalanced IRQ: "irq
}' /proc/interrupts
```

---

**4. تفعيل IRQ tracing كامل وجمع البيانات:**

```bash
#!/bin/bash
TRACE=/sys/kernel/debug/tracing

# reset
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace

# enable IRQ events
echo 1 > $TRACE/events/irq/irq_handler_entry/enable
echo 1 > $TRACE/events/irq/irq_handler_exit/enable

# filter على IRQ معين بس (مثلاً irq=42)
echo 'irq == 42' > $TRACE/events/irq/irq_handler_entry/filter
echo 'irq == 42' > $TRACE/events/irq/irq_handler_exit/filter

# ابدأ
echo 1 > $TRACE/tracing_on

# استنى 10 ثواني
sleep 10

# إيقاف وجمع
echo 0 > $TRACE/tracing_on
cat $TRACE/trace > /tmp/irq42_trace.txt
echo "Results saved to /tmp/irq42_trace.txt"

# تحليل بسيط: كام مرة اتنادى الـ handler؟
grep irq_handler_entry /tmp/irq42_trace.txt | wc -l
# كام مرة كانت unhandled؟
grep "ret=unhandled" /tmp/irq42_trace.txt | wc -l
```

---

**5. تشخيص spurious IRQs:**

```bash
# الـ kernel بيكتب في dmesg لما يلاقي spurious
dmesg | grep -E "nobody cared|Disabling IRQ|spurious"

# شوف الـ unhandled count لكل IRQ
for dir in /proc/irq/*/; do
    irq=$(basename $dir)
    if [ -f "$dir/spurious" ]; then
        unhandled=$(awk '/unhandled/{print $4}' $dir/spurious 2>/dev/null)
        [ "$unhandled" -gt 0 ] 2>/dev/null && \
            echo "IRQ $irq: unhandled=$unhandled"
    fi
done
```

---

**6. التحقق من الـ irq_chip flags:**

```bash
# الـ flags موجودة في debugfs
cat /sys/kernel/debug/irq/42/chip_name

# لمعرفة الـ flags المضبوطة، محتاج kernel module صغير أو crash/gdb:
# في crash:
# crash> struct irq_chip.flags (addr_of_chip)

# بديل: dmesg لو الـ driver بيطبع flags
dmesg | grep -i "irqchip\|irq_chip" | head -20
```

---

**7. تتبع IRQ disable/enable latency:**

```bash
# irqsoff tracer — يلتقط أطول فترة IRQs كانت disabled
echo irqsoff > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 30   # شغّل الـ workload
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# تفسير الـ output:
# # Duration: 123.4 us   ← ده أقصى وقت IRQs كانت disabled
# # Entries-In-Buffer: 456
# kworker/0:1   [000]  1234.5: <stack trace of who disabled IRQs>
```

**لو الـ duration أكبر من 1ms → مشكلة latency حقيقية تستاهل تحقيق.**
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — interrupt storm بسبب trigger type غلط

#### العنوان
**GPIO interrupt storm** يوقف الـ industrial gateway بالكامل بعد تشغيل ساعتين

#### السياق
شركة تبني industrial gateway على بورد مبنية على **RK3562**. الـ gateway بتقرأ بيانات من sensor خارجي عبر GPIO line. الـ sensor بيرفع الـ signal لما تيجي بيانات جديدة. المنتج بيشتغل عادي في الـ lab، بس في الـ field بيتعلق بعد ساعتين وبيحتاج hard reset.

#### المشكلة
الـ kernel log مليان بـ:
```
irq 47: nobody cared (try booting with the "irqpoll" option)
...
handlers:
Disabling IRQ #47
```
الـ IRQ اتعطل خالص والـ sensor بطّل شغل. الـ `/proc/interrupts` بيظهر العداد رايح في السما (millions per second).

#### التحليل
الـ engineer بيفتح الـ device tree:

```dts
sensor_irq: sensor@0 {
    interrupt-parent = <&gpio2>;
    interrupts = <3 IRQ_TYPE_LEVEL_HIGH>;   /* <-- المشكلة هنا */
};
```

الـ sensor بيرفع الـ line لـ ~5ms بس — لكن الـ code استخدم `IRQ_TYPE_LEVEL_HIGH` بدل `IRQ_TYPE_EDGE_RISING`.

بيراجع `irq.h`:

```c
IRQ_TYPE_LEVEL_HIGH  = 0x00000004,
IRQ_TYPE_EDGE_RISING = 0x00000001,
```

مع `IRQ_TYPE_LEVEL_HIGH`، طالما الـ line عالية والـ interrupt مش masked، الـ core بيفضل يدق الـ handler مرار. الـ `handle_level_irq()` المعرف في الـ header بيشتغل كالآتي: بيعمل mask للـ IRQ → بيدي للـ handler → بيعمل unmask. لو الـ handler اتأخر أو الـ signal فضل high، الـ interrupt بيعود فوراً. ده بيخلق **interrupt storm** وبعد عدد معين من الـ spurious interrupts، الـ kernel بيكال `note_interrupt()` (المعرف في الـ header) وبيعمل disable للـ IRQ نهائياً.

```c
/* kernel/irq/spurious.c: note_interrupt يُعدّ الـ spurious */
extern void note_interrupt(struct irq_desc *desc, irqreturn_t action_ret);
```

#### الحل
**تغيير الـ DT:**

```dts
sensor_irq: sensor@0 {
    interrupt-parent = <&gpio2>;
    interrupts = <3 IRQ_TYPE_EDGE_RISING>;  /* edge, مش level */
};
```

لو الـ sensor بيحتاج acknowledgment صريح قبل ما يخفض الـ line، بيستخدم:

```c
/* في الـ driver */
irq_set_irq_type(irq_num, IRQ_TYPE_EDGE_RISING);

/* أو لو lazy disable مشكلة: */
irq_set_status_flags(irq_num, IRQ_DISABLE_UNLAZY);
```

`IRQ_DISABLE_UNLAZY` موجود في الـ `irq.h`:
```c
IRQ_DISABLE_UNLAZY = (1 << 19),
```
بيضمن إن الـ IRQ يتعطل فوراً مش بشكل lazy.

#### الدرس المستفاد
**الفرق بين `IRQ_TYPE_LEVEL_HIGH` و `IRQ_TYPE_EDGE_RISING` مش cosmetic** — هو بيحدد متى بيفتح الـ hardware الـ interrupt. الـ level trigger يحتاج الـ handler يكون متزامن مع الـ signal وإلا بيحصل storm.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — HDMI interrupt مش بيصحّي من الـ suspend

#### العنوان
الـ HDMI بيفضل أسود بعد wake-from-suspend في Android TV box

#### السياق
منتج Android TV box مبني على **Allwinner H616**. المستخدم بيضغط على الـ remote، الـ box بيصحى من الـ suspend، لكن الشاشة بتفضل سودة. الـ HDMI output مش بيرجع. المشكلة بتحصل 100% من الـ suspend.

#### المشكلة
الـ engineer يراجع الـ HDMI driver ويلاقي إن الـ wakeup interrupt مش متفعّل:

```bash
$ cat /proc/interrupts | grep hdmi
# hdmi interrupt موجود بس count بيفضل zero بعد الصحيان
```

الـ dmesg:
```
PM: Syncing filesystems...
Disabling non-boot CPUs ...
[suspend ok]
...
[resume] hdmi: no interrupt received on wake
```

#### التحليل
الـ engineer يفتح الـ HDMI driver ويلاقي:

```c
static int sun50i_hdmi_probe(struct platform_device *pdev)
{
    irq = platform_get_irq(pdev, 0);
    request_irq(irq, hdmi_irq_handler, 0, "hdmi", priv);
    /* missing: enable_irq_wake() */
}
```

بيراجع `irq.h` للـ state bits:

```c
IRQD_WAKEUP_STATE  = BIT(14),   /* interrupt configured for wakeup */
IRQD_WAKEUP_ARMED  = BIT(19),   /* wakeup mode armed */
```

الـ accessor المعرفة في الـ header:
```c
static inline bool irqd_is_wakeup_set(struct irq_data *d)
{
    return __irqd_to_state(d) & IRQD_WAKEUP_STATE;
}
```

الـ `irq_chip` بيحتوي على:
```c
int (*irq_set_wake)(struct irq_data *data, unsigned int on);
```

الـ suspend path في الـ kernel بيفحص `IRQD_WAKEUP_STATE` لكل interrupt — لو مش set، بيعمله mask أثناء الـ suspend. الـ HDMI interrupt اتعمله mask وماحدش صحّاه.

#### الحل
في الـ HDMI driver:

```c
static int sun50i_hdmi_probe(struct platform_device *pdev)
{
    irq = platform_get_irq(pdev, 0);
    request_irq(irq, hdmi_irq_handler, 0, "hdmi", priv);

    /* enable wakeup: يضبط IRQD_WAKEUP_STATE عبر irq_set_wake() */
    enable_irq_wake(irq);
}

static void sun50i_hdmi_remove(struct platform_device *pdev)
{
    disable_irq_wake(irq);
    free_irq(irq, priv);
}
```

لو الـ irqchip مش بيدعم `irq_set_wake`، بيضيف الـ flag:

```c
IRQCHIP_SKIP_SET_WAKE = (1 << 4),
```

في الـ `irq_chip.flags` عشان الـ core يتجاوز الـ error.

يتحقق:
```bash
$ cat /sys/bus/platform/devices/hdmi/power/wakeup
enabled
```

#### الدرس المستفاد
**كل interrupt بيحتاج wakeup صريح** عبر `enable_irq_wake()`. الـ `IRQD_WAKEUP_STATE` في `irq_common_data.state_use_accessors` هو الـ guard الوحيد — لو مش set، الـ suspend path بيكمه.

---

### السيناريو الثالث: Custom Board Bring-up على STM32MP1 — I2C GPIO expander interrupt مش بيشتغل

#### العنوان
الـ interrupt من GPIO expander (PCA9555) عبر I2C على STM32MP1 مش بيصل للـ driver

#### السياق
فريق bring-up بيشغّل custom board على **STM32MP1**. الـ board فيها PCA9555 GPIO expander متوصل بـ I2C. الـ expander بيجمع interrupts من عدة sensors وبيوصّلها لـ pin واحد على الـ SoC. الـ driver مكتوب ومبيشتغلش.

#### المشكلة
```bash
$ cat /proc/interrupts | grep pca9555
# لا شيء — الـ interrupt مش registered أصلاً
```

الـ kernel log:
```
pca9555: failed to request irq: -22
```

#### التحليل
الـ engineer بيلاقي إن الـ PCA9555 driver بيستخدم `irq_bus_lock` / `irq_bus_sync_unlock` لأن الـ I2C slow bus ومحتاج synchronization. بيراجع `irq.h`:

```c
void (*irq_bus_lock)(struct irq_data *data);
void (*irq_bus_sync_unlock)(struct irq_data *data);
```

الـ `irq_chip` للـ expander محتاج يعرّف الاتنين. بيراجع الكود:

```c
static struct irq_chip pca9555_irq_chip = {
    .name           = "pca9555",
    .irq_mask       = pca9555_irq_mask,
    .irq_unmask     = pca9555_irq_unmask,
    /* missing: irq_bus_lock, irq_bus_sync_unlock */
    /* missing: irq_set_wake for wakeup support */
};
```

من غير `irq_bus_lock`، الـ kernel لما بيحاول يضبط الـ interrupt state بيعدي كود اللي بيتوقع الـ locking ومش بيلاقيه.

المشكلة التانية: الـ DT بيعرّف nested interrupt ومحتاج:

```c
IRQ_NESTED_THREAD = (1 << 15),
```

بس الـ driver مش بيضبط الـ flag ده عبر:
```c
irq_set_nested_thread(child_irq, true);
```

#### الحل
الـ engineer بيضيف للـ irq_chip:

```c
static struct irq_chip pca9555_irq_chip = {
    .name                   = "pca9555",
    .irq_mask               = pca9555_irq_mask,
    .irq_unmask             = pca9555_irq_unmask,
    .irq_bus_lock           = pca9555_irq_bus_lock,       /* lock I2C */
    .irq_bus_sync_unlock    = pca9555_irq_bus_sync_unlock, /* flush to hw */
    .irq_set_wake           = pca9555_irq_set_wake,
};
```

وفي الـ probe:
```c
/* mark all child irqs as nested threads */
for (i = 0; i < PCA9555_NGPIO; i++) {
    int child_irq = irq_find_mapping(chip->irq_domain, i);
    irq_set_nested_thread(child_irq, true);
    irq_set_noprobe(child_irq);      /* IRQ_NOPROBE */
}
```

`irq_set_noprobe()` معرف في `irq.h`:
```c
static inline void irq_set_noprobe(unsigned int irq)
{
    irq_modify_status(irq, 0, IRQ_NOPROBE);
}
```

#### الدرس المستفاد
**الـ slow-bus irqchips** (I2C, SPI) لازم تعرّف `irq_bus_lock`/`irq_bus_sync_unlock` — ده مش optional. والـ nested thread interrupts لازم يتعلم عليهم صراحةً بـ `IRQ_NESTED_THREAD`.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — IRQ affinity بيكسر real-time latency

#### العنوان
الـ CAN bus interrupt latency بتتجاوز الـ deadline الـ real-time على i.MX8 ECU

#### السياق
شركة automotive بتبني ECU على **i.MX8**. المعالج رباعي النواة. الـ CAN bus driver بيتعامل مع messages بـ deadline صارم (1ms max latency). الـ testing بيظهر إن latency بتوصل 5ms أحياناً.

#### المشكلة
الـ engineer يراجع الـ IRQ affinity:

```bash
$ cat /proc/interrupts | grep can
 42:   1234567        0        0        0   GIC-400   42  Edge  can0
```

الـ CAN interrupt شغّال على CPU0 بس — وده نفس الـ CPU اللي فيه الـ networking stack والـ system IRQs التانية.

#### التحليل
الـ engineer بيفتح `irq.h` ويفهم الـ affinity system:

```c
struct irq_common_data {
    ...
#ifdef CONFIG_SMP
    cpumask_var_t  affinity;           /* requested affinity */
#endif
#ifdef CONFIG_GENERIC_IRQ_EFFECTIVE_AFF_MASK
    cpumask_var_t  effective_affinity; /* actual affinity after hw constraints */
#endif
};
```

والـ chip callback:
```c
int (*irq_set_affinity)(struct irq_data *data,
                        const struct cpumask *dest, bool force);
```

الـ state bit المهم:
```c
IRQD_NO_BALANCING = BIT(10),  /* balancing disabled */
IRQD_AFFINITY_MANAGED = BIT(21), /* auto-managed by kernel */
```

الـ IRQ balancer الخاص بالـ kernel (irqbalance) بيحرك الـ CAN interrupt على CPUs مختلفة بشكل عشوائي. المشكلة إن `IRQD_NO_BALANCING` مش مضبوط، فالـ kernel حر يحرك الـ interrupt.

#### الحل
**خطوة 1:** Pin الـ CAN interrupt على CPU3 (الـ RT core) وامنع الـ balancing:

```bash
# اعمل pin للـ interrupt على CPU3
echo 8 > /proc/irq/42/smp_affinity   # bitmask: CPU3

# أو عبر systemd-irqbalance ban
echo 42 >> /etc/irqbalance_banned_irqs
```

**خطوة 2:** في الـ driver نفسه، اضبط `IRQ_NO_BALANCING`:

```c
static int flexcan_probe(struct platform_device *pdev)
{
    ...
    irq = platform_get_irq(pdev, 0);
    request_irq(irq, flexcan_irq_handler, IRQF_NO_THREAD, "can0", priv);

    /* منع الـ kernel من إعادة توزيع الـ interrupt */
    irq_set_status_flags(irq, IRQ_NO_BALANCING);

    /* اضبط الـ affinity على CPU3 */
    irq_set_affinity(irq, cpumask_of(3));
}
```

`irq_set_status_flags()` معرف في `irq.h`:
```c
static inline void irq_set_status_flags(unsigned int irq, unsigned long set)
{
    irq_modify_status(irq, 0, set);
}
```

وبيضبط `IRQ_NO_BALANCING_MASK`:
```c
#define IRQ_NO_BALANCING_MASK (IRQ_PER_CPU | IRQ_NO_BALANCING)
```

**التحقق:**
```bash
$ cat /proc/irq/42/smp_affinity_list
3
$ cat /proc/irq/42/affinity_hint
# should show CPU3 pinned
```

#### الدرس المستفاد
**الـ IRQ affinity في `irq_common_data` بيفرق بين `affinity` (requested) و`effective_affinity` (actual)**. الـ hardware ممكن ميعرفش multi-CPU targets فبيستخدم `effective_affinity` subset. لازم دايماً تحدد `IRQ_NO_BALANCING` للـ RT interrupts وتتحقق من `effective_affinity` مش `affinity` بس.

---

### السيناريو الخامس: IoT Sensor Hub على AM62x — Generic IRQ Chip setup فيه mask cache corruption

#### العنوان
الـ GPIO interrupts على AM62x بيتفعلوا عشوائياً بعد الـ resume من deep sleep

#### السياق
منتج IoT sensor hub مبني على **AM62x** من Texas Instruments. الـ hub بيجمع بيانات من 8 sensors عبر GPIO lines. الـ system بيدخل deep sleep كل 30 ثانية. بعد أسبوع من الـ testing، بيلاحظوا إن بعض الـ sensors بيبدأ يبعت interrupts وهي مش المفروض.

#### المشكلة
```bash
$ cat /proc/interrupts
  89:     45230   GPIO  89  Edge  sensor0   # طبيعي
  90:    891023   GPIO  90  Edge  sensor1   # عدد مبالغ فيه!
  91:         0   GPIO  91  Edge  sensor2   # صفر مع إن sensor بيشتغل
```

sensor1 بيولد interrupts وهو في idle، وsensor2 مش بيولد خالص.

#### التحليل
الـ engineer بيراجع الـ GPIO driver للـ AM62x اللي بيستخدم **Generic IRQ Chip** (`irq_chip_generic`):

```c
struct irq_chip_generic {
    raw_spinlock_t   lock;
    void __iomem    *reg_base;
    u32              mask_cache;    /* <-- الـ cached mask register */
    u32              wake_enabled;
    u32              wake_active;
    unsigned int     irq_base;
    unsigned int     irq_cnt;
    ...
    struct irq_chip_type chip_types[];
};
```

الـ `mask_cache` المفروض يتزامن مع الـ hardware register. بيلاقي إن الـ `resume` callback مش بيعيد كتابة الـ cache للـ hardware:

```c
static void am62x_gpio_gc_resume(struct irq_chip_generic *gc)
{
    /* BUG: بيعيد قراءة mask_cache من الـ hw بعد resume */
    /* لكن الـ hw اتـ reset أثناء deep sleep! */
    /* النتيجة: mask_cache بيفضل قديم، الـ hw mask = 0 (unmask all) */
}
```

الـ `irq_gc_flags`:
```c
IRQ_GC_INIT_MASK_CACHE = 1 << 0,  /* initialize mask_cache by reading mask reg */
```

الـ flag ده بيقرأ الـ mask register **مرة واحدة** عند الـ init. بعد deep sleep، الـ hardware register بيتـ reset لـ 0 (كل الـ interrupts unmasked) لكن الـ `mask_cache` في الـ RAM بيفضل بيحمل القيمة القديمة. الـ driver بيعتمد على الـ cache ومش بيعيد كتابة الـ register.

الـ `irq_reg_writel()` و`irq_reg_readl()` معرفين في `irq.h`:

```c
static inline void irq_reg_writel(struct irq_chip_generic *gc,
                                  u32 val, int reg_offset)
{
    if (gc->reg_writel)
        gc->reg_writel(val, gc->reg_base + reg_offset);
    else
        writel(val, gc->reg_base + reg_offset);
}
```

#### الحل
الـ engineer بيضيف `resume` callback صحيح:

```c
static void am62x_gpio_gc_resume(struct irq_chip_generic *gc)
{
    struct irq_chip_type *ct = gc->chip_types;

    /* بعد deep sleep، الـ hw reset — اكتب الـ mask_cache للـ hardware */
    irq_reg_writel(gc, ~gc->mask_cache, ct->regs.mask);

    /* لو في wakeup interrupts، أعد تفعيلها */
    if (gc->wake_active)
        irq_reg_writel(gc, gc->wake_active, ct->regs.enable);
}

static int am62x_gpio_probe(struct platform_device *pdev)
{
    struct irq_chip_generic *gc;

    gc = devm_irq_alloc_generic_chip(dev, "am62x-gpio", 1, irq_base,
                                     base, handle_level_irq);
    gc->resume = am62x_gpio_gc_resume;  /* register resume callback */

    devm_irq_setup_generic_chip(dev, gc, IRQ_MSK(ngpio),
                                IRQ_GC_INIT_MASK_CACHE,
                                IRQ_NOREQUEST, 0);
}
```

**التحقق:**
```bash
# بعد resume
$ devmem2 0x601000C0 w   # اقرأ mask register من hardware
# المفروض تتطابق مع mask_cache
```

#### الدرس المستفاد
**الـ `mask_cache` في `irq_chip_generic` هو SW state فقط** — الـ hardware ممكن ينسى كل حاجة بعد power cycle أو deep sleep. الـ `resume` callback لازم يعيد كتابة كل الـ registers من الـ cache للـ hardware، مش العكس. استخدم `irq_reg_writel()` من `irq.h` مباشرة داخل الـ callback.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**LWN.net** هو المرجع الأول لمتابعة تطور الـ Linux kernel، وفيما يلي أهم المقالات المتعلقة بالـ IRQ subsystem:

| المقال | الوصف |
|--------|-------|
| [A new generic IRQ layer](https://lwn.net/Articles/184750/) | المقال الأساسي اللي يشرح فلسفة الـ **genirq** وليه اتبنى بدل `__do_IRQ()` القديم |
| [Generic IRQ Subsystem: -V4](https://lwn.net/Articles/184260/) | نسخة كاملة من الـ patch set بتاع Thomas Gleixner وIingo Molnar اللي أدخل genirq |
| [generic irq subsystem, core](https://lwn.net/Articles/104845/) | الـ core patches الأولانية للـ generic IRQ layer |
| [generic irq subsystem, x64 port](https://lwn.net/Articles/104847/) | port الـ generic IRQ على معمارية x86-64 |
| [generic irq subsystem, ppc port](https://lwn.net/Articles/104848/) | port الـ generic IRQ على PowerPC |
| [IRQ threads](https://lwn.net/Articles/95387/) | مناقشة فكرة الـ **threaded IRQ handlers** وإزاي بيشتغلوا |
| [A question about threaded IRQs](https://lwn.net/Articles/311016/) | نقاش تفصيلي عن الـ `request_threaded_irq()` ومتى تستخدمه |
| [IRQ-domain.txt](https://lwn.net/Articles/487684/) | مقال عن الـ **irq_domain** library ومشكلة mapping بين HW IRQ numbers و virtual IRQ numbers |
| [irq: Track the interrupt timings](https://lwn.net/Articles/691297/) | تحسينات على الـ IRQ subsystem لتتبع توقيتات الـ interrupts |
| [Add support for a new IMS interrupt mechanism](https://lwn.net/Articles/799181/) | مثال حديث على إضافة آلية interrupt جديدة للـ kernel |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة في `Documentation/` في الـ kernel source tree:

```
Documentation/core-api/genericirq.rst       ← الشرح الكامل للـ generic IRQ layer
Documentation/core-api/irq/concepts.rst     ← تعريفات أساسية: ما هو IRQ؟
Documentation/core-api/irq/irq-domain.rst   ← شرح الـ irq_domain library
Documentation/IRQ-domain.txt                ← نسخة قديمة من وثائق irq_domain
```

**الـ Online Documentation:**

| الرابط | المحتوى |
|--------|---------|
| [What is an IRQ?](https://docs.kernel.org/core-api/irq/concepts.html) | المفاهيم الأساسية للـ IRQ في الـ kernel |
| [Linux generic IRQ handling](https://docs.kernel.org/core-api/genericirq.html) | الوثيقة الرسمية الكاملة للـ genirq framework |
| [The irq_domain Interrupt Number Mapping Library](https://docs.kernel.org/core-api/irq/irq-domain.html) | كل حاجة عن الـ `irq_domain` وhierarchical domains |
| [Linux generic IRQ handling (v4.18)](https://www.kernel.org/doc/html/v4.18/core-api/genericirq.html) | نسخة v4.18 — مفيدة للمقارنة مع الإصدارات القديمة |

---

### Kernel Source Files المهمة

الملفات دي هي قلب الـ IRQ subsystem في الـ kernel:

```
include/linux/irq.h           ← تعريفات irq_desc, irq_chip, irq_data (الملف الحالي)
include/linux/irqdesc.h       ← struct irq_desc الكاملة
include/linux/irqdomain.h     ← struct irq_domain و irq_domain_ops
include/linux/interrupt.h     ← IRQF_* flags و request_irq() API
kernel/irq/chip.c             ← تنفيذ irq_chip operations
kernel/irq/manage.c           ← request_irq(), free_irq(), enable/disable
kernel/irq/irqdomain.c        ← تنفيذ الـ irq_domain library
kernel/irq/handle.c           ← الـ flow handlers (edge, level, fasteoi)
kernel/irq/debugfs.c          ← debugfs interface للـ IRQ subsystem
```

---

### Kernel Commits المهمة

| الـ Commit | الوصف |
|-----------|-------|
| [genirq: Implement a generic interrupt chip](https://github.com/torvalds/linux/commit/7d8280624797bbe2f5170bd3c85c75a8c9c74242) | Thomas Gleixner بيضيف الـ **generic irq chip** اللي بيعمل خدمة للـ irq_chip implementations الشائعة |
| [kernel/irq/chip.c](https://github.com/torvalds/linux/blob/master/kernel/irq/chip.c) | الكود الحالي لتنفيذ الـ chip-level operations |
| [kernel/irq/manage.c](https://github.com/torvalds/linux/blob/master/kernel/irq/manage.c) | الكود الحالي لإدارة الـ IRQ lifecycle |

**لمتابعة الـ commits على الـ tip tree بتاع Thomas Gleixner:**
```bash
git log --oneline --all | grep -i "genirq\|irq/core\|irq_chip\|irq_domain"
```

---

### Mailing List Discussions

| الرابط | الموضوع |
|--------|---------|
| [genirq: Provide interrupt injection mechanism](https://lore.kernel.org/linux-pci/20200306130623.990928309@linutronix.de/) | نقاش Thomas Gleixner على lore.kernel.org لـ interrupt injection |
| [genirq: Allow irqchip state save/restore](https://www.uwsg.indiana.edu/hypermail/linux/kernel/1504.1/00715.html) | تحسين على الـ irqchip state management |
| [genirq: Introduce request_any_context_irq()](https://lkml.indiana.edu/hypermail/linux/kernel/1004.1/01913.html) | إضافة `request_any_context_irq()` للـ kernel |
| [genirq/debugfs: Add proper debugfs interface](https://lore.kernel.org/lkml/tip-087cdfb662ae50e3826e7cd2e54b6519d07b60f0@git.kernel.org/) | إضافة واجهة debugfs لفحص حالة الـ IRQs |

**الـ LKML (Linux Kernel Mailing List) كـ archive:**
- [lore.kernel.org/linux-kernel](https://lore.kernel.org/linux-kernel/) — ابحث عن `irq_chip`, `irq_domain`, `genirq`

---

### KernelNewbies.org

| الرابط | المحتوى |
|--------|---------|
| [Internals of Interrupt Handling](https://kernelnewbies.org/KernelHacking-HOWTO/Overview_of_the_Kernel_Source_Code/Internals_of_Interrupt_Handling) | نظرة عامة على آليات معالجة الـ interrupts |
| [Details of do_IRQ() function](https://kernelnewbies.org/KernelHacking-HOWTO/Overview_of_the_Kernel_Source_Code/Internals_of_Interrupt_Handling/Details_of_do_IRQ()_function) | تفاصيل `do_IRQ()` القديمة وكيف اتطورت |
| [Exceptions and Interrupts Handling](https://kernelnewbies.org/New_Kernel_Hacking_HOWTO/Subsystems/Exceptions_and_Interrupts_Handling) | من الـ New Kernel Hacking HOWTO — شرح شامل |
| [Details of do_IRQ() (New HOWTO)](https://kernelnewbies.org/New_Kernel_Hacking_HOWTO/Subsystems/Exceptions_and_Interrupts_Handling/Details_of_do_IRQ()_function) | نسخة محدثة من شرح `do_IRQ()` |
| [Linux 2.6.18 — genirq introduction](https://kernelnewbies.org/Linux_2_6_18) | changelog الـ kernel 2.6.18 اللي أدخل الـ genirq |

---

### eLinux.org

| الرابط | المحتوى |
|--------|---------|
| [Soft IRQ Threads](https://elinux.org/Soft_IRQ_Threads) | شرح الـ **softirq threads** وعلاقتها بالـ real-time priority |
| [Fix IRQ Domain DT support issues and GPIO IRQ](https://elinux.org/Fix_IRQ_Domain_DT_support_issues_and_gpio_IRQ) | حل مشاكل الـ irq_domain مع Device Tree وGPIO |
| [Real Time Working Group](https://elinux.org/Real_Time_Working_Group) | تأثير الـ PREEMPT_RT patch على الـ IRQ handling |
| [Ftrace — IRQ tracing](https://elinux.org/Ftrace) | استخدام ftrace لتتبع أحداث الـ IRQ في الـ embedded Linux |

---

### كتب مُوصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصول المهمة:**
  - Chapter 10: *Interrupt Handling* — الأساسيات من `request_irq()` لحد `free_irq()`
  - Chapter 6: *Advanced Char Driver Operations* — blocking I/O وعلاقتها بالـ interrupts
- **متاح مجانًا:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل المهم:**
  - Chapter 7: *Interrupts and Interrupt Handlers* — شرح كامل للـ flow من HW interrupt لحد الـ handler
  - Chapter 8: *Bottom Halves and Deferring Work* — softirqs, tasklets, workqueues

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل المهم:**
  - Chapter 9: *File Systems* — بيتكلم عن device drivers وinterrupt handling في الـ embedded context
  - Chapter 14: *Kernel Debugging Techniques* — debugging الـ IRQ problems

#### Understanding the Linux Kernel — Bovet & Cesati
- Chapter 4: *Interrupts and Exceptions* — أعمق شرح للـ x86 interrupt handling على مستوى الـ hardware وthe kernel

---

### Search Terms للبحث عن مزيد من المعلومات

استخدم الـ terms دي للبحث في Google أو LKML أو LWN:

```
genirq linux kernel
irq_chip linux driver
irq_domain mapping hwirq
request_threaded_irq tutorial
linux interrupt flow handler edge level
irq_desc struct kernel
IRQF_SHARED linux driver
linux MSI interrupt setup
irq affinity SMP linux
PREEMPT_RT threaded IRQ
linux interrupt controller driver irq_chip
devicetree interrupt-parent interrupts property linux
```

**للبحث في الـ kernel source مباشرة:**
```bash
# ابحث عن implementations لـ irq_chip
grep -r "irq_chip" drivers/ --include="*.c" -l

# شوف كل الـ irq_domain registrations
grep -r "irq_domain_add\|irq_domain_create" drivers/ --include="*.c" -l

# تتبع flow handlers
grep -r "handle_edge_irq\|handle_level_irq\|handle_fasteoi_irq" kernel/irq/ --include="*.c"
```

**للبحث في الـ git log:**
```bash
git log --oneline --all --grep="genirq" -- kernel/irq/
git log --oneline --all --grep="irq_domain" -- include/linux/irqdomain.h
```
## Phase 8: Writing simple module

### الفكرة: Hook على `handle_fasteoi_irq`

**`handle_fasteoi_irq`** هي الـ flow handler الأكتر استخداماً في الـ Linux kernel — بتتولى الـ interrupt delivery لأي IRQ بيشتغل بنظام Fast-EOI (وده الـ default للـ PCI interrupts والـ APIC على x86). كل ما interrupt اتفعّل، الـ kernel بيـcall الـ function دي ويمرر ليها الـ `irq_desc` اللي فيه كل المعلومات عن الـ interrupt: رقمه، الـ chip اللي بيتحكم فيه، وعدد الـ kstat.

هنستخدم **kprobe** عشان نعمل hook على الـ function دي من غير ما نغيّر أي كود في الـ kernel.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * fasteoi_probe.c
 * Hooks handle_fasteoi_irq() via kprobe and prints IRQ info on each call.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/irqdesc.h>     /* struct irq_desc, irq_desc_get_chip */
#include <linux/irq.h>         /* struct irq_chip */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc-arabic");
MODULE_DESCRIPTION("kprobe on handle_fasteoi_irq to log IRQ chip info");

/* ------------------------------------------------------------------ */
/* pre_handler: يتنفذ قبل ما handle_fasteoi_irq تشتغل                 */
/* ------------------------------------------------------------------ */
static int fasteoi_pre_handler(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64: first argument (struct irq_desc *desc) lives in rdi.
     * On arm64:  it lives in x0.
     * We cast the register to get our desc pointer.
     */
#if defined(CONFIG_X86_64)
    struct irq_desc *desc = (struct irq_desc *)regs->di;
#elif defined(CONFIG_ARM64)
    struct irq_desc *desc = (struct irq_desc *)regs->regs[0];
#else
    /* fallback — skip logging on unsupported arch */
    return 0;
#endif

    /* Guard against NULL before touching any field */
    if (!desc)
        return 0;

    {
        struct irq_chip *chip = irq_desc_get_chip(desc); /* get hardware chip */
        unsigned int irq      = irq_desc_get_irq(desc);  /* linux IRQ number  */
        const char  *name     = (chip && chip->name) ? chip->name : "unknown";

        pr_info("[fasteoi_probe] irq=%u  chip=%s\n", irq, name);
    }

    return 0; /* 0 = don't modify execution flow */
}

/* ------------------------------------------------------------------ */
/* تعريف الـ kprobe نفسه                                              */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name   = "handle_fasteoi_irq", /* اسم الـ symbol في الـ kernel */
    .pre_handler   = fasteoi_pre_handler,
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
static int __init fasteoi_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[fasteoi_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[fasteoi_probe] planted at %s (%p)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit fasteoi_probe_exit(void)
{
    unregister_kprobe(&kp); /* لازم نشيل الـ hook قبل ما الـ module يتشال */
    pr_info("[fasteoi_probe] removed\n");
}

module_init(fasteoi_probe_init);
module_exit(fasteoi_probe_exit);
```

---

### Makefile للتجميع

```makefile
obj-m += fasteoi_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

تشغيل:
```bash
make
sudo insmod fasteoi_probe.ko
sudo dmesg | grep fasteoi_probe   # شوف الـ log
sudo rmmod fasteoi_probe
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | يعرّف `struct kprobe` وكل الـ API بتاعته |
| `linux/irqdesc.h` | يعرّف `struct irq_desc` والـ accessors زي `irq_desc_get_chip()` |
| `linux/irq.h`     | يعرّف `struct irq_chip` اللي فيه `chip->name` |

**الـ `linux/irqdesc.h`** محتاجينه عشان `struct irq_desc` مش معرّف جوه `linux/irq.h` — الـ irq.h بيـforward-declare بس `struct irq_desc` وبيـinclude الـ irqdesc.h داخلياً، لكن نص الـ include guards بيمنع double-inclusion.

---

#### الـ `pre_handler` والـ arguments

الـ kprobe بيـcall الـ `pre_handler` قبل ما الـ kernel ينفّذ أول instruction في `handle_fasteoi_irq`. الـ argument الوحيد للـ function الأصلية هو `struct irq_desc *desc`، وده بيتمرر في:
- **`regs->di`** على x86-64 (الـ System V ABI بيحط الـ first arg في `rdi`)
- **`regs->regs[0]`** على ARM64 (الـ AAPCS64 بيحط الـ first arg في `x0`)

الـ `#if defined(...)` ده ضروري عشان الـ `pt_regs` struct بيختلف من architecture لأخرى.

---

#### ليه `irq_desc_get_chip()` ومش `desc->irq_data.chip` مباشرة؟

`irq_desc_get_chip()` هي **accessor** رسمية — بتضمن إننا ما بنلمسش الـ internal layout وبتعمل لنا NULL-safety. الـ kernel بيشجع استخدام الـ accessors بدل الـ direct field access عشان الـ layout ممكن يتغير.

---

#### الـ `module_init` / `module_exit`

الـ `register_kprobe()` بتـpatch الـ first byte من `handle_fasteoi_irq` بـ breakpoint instruction (int3 على x86). لو الـ registration فشل (مثلاً الـ symbol مش موجود أو الـ CONFIG_KPROBES مش مفعّل)، الـ module بيرجع error.

الـ `unregister_kprobe()` في الـ `exit` **إلزامية** — لو الـ module اتشال من غيرها، الـ kernel هيظل يـcall pointer لـ code مش موجود ويحصل kernel panic فوراً.

---

### مثال على الـ output

```
[fasteoi_probe] planted at handle_fasteoi_irq (ffffffffb234a1c0)
[fasteoi_probe] irq=28  chip=IR-IO-APIC
[fasteoi_probe] irq=28  chip=IR-IO-APIC
[fasteoi_probe] irq=40  chip=IR-PCI-MSI
[fasteoi_probe] irq=16  chip=IR-IO-APIC
...
[fasteoi_probe] removed
```

الـ `IR-IO-APIC` هو الـ chip اللي بيتحكم في الـ APIC interrupts على x86، والـ `IR-PCI-MSI` للـ PCIe devices — الـ output ده بيوضح إزاي الـ kernel بيوزّع الـ interrupts على chips مختلفة في نفس الوقت.
