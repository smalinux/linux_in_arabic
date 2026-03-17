## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ `irqhandler.h` جزء من **IRQ SUBSYSTEM** في Linux kernel، المسؤول عنه Thomas Gleixner، وكوده موجود في `kernel/irq/`.

---

### القصة من الأول

تخيل إنك شغّال في مكتب كبير، وعندك زر جرس على مكتبك. لما حد يضغط الجرس ده، أنت محتاج تعرف:

1. **مين ضغط الجرس؟** (أي جهاز أرسل الـ interrupt؟)
2. **إيه طبيعة الجرس ده؟** (هل بيظل مضغوط لحد ما تيجي تعالجه؟ ولا بيبعت نبضة وبس؟)
3. **مين المسؤول إنه يتعامل مع كل نوع من الأجراس دي؟**

الـ **interrupt** في الكومبيوتر هو نفس الفكرة — جهاز hardware (زي network card أو keyboard) بيقول للـ CPU "أنا عندي كلام مهم!". الـ CPU بيوقف اللي هو بيعمله، ويروح يتعامل مع الموضوع.

المشكلة إن مش كل الأجراس بتشتغل بنفس الطريقة — في فرق جوهري في **طبيعة الإشارة الكهربية** نفسها:

| النوع | الوصف | مثال |
|---|---|---|
| **Level-triggered** | الإشارة فاضلة عالية طول ما المشكلة موجودة | قديم ISA |
| **Edge-triggered** | بتبعت نبضة لحظية عند التغيير | معظم PCI devices |
| **FastEOI** | بعد المعالجة بتبعت EOI للـ controller | APIC في x86 |

كل نوع محتاج **سلوك مختلف** من الـ kernel — متى يعمل mask؟ متى يعمل unmask؟ متى يبعت EOI؟

---

### دور الـ `irqhandler.h` تحديداً

الملف ده **صغير جداً** — 14 سطر بس. هدفه الوحيد هو تعريف **نوع واحد فقط**:

```c
typedef void (*irq_flow_handler_t)(struct irq_desc *desc);
```

الـ **`irq_flow_handler_t`** هو pointer لـ function، بتاخد `struct irq_desc *` وترجع `void`. ده هو **توقيع** أي function ممكن تكون مسؤولة عن **تدفق** معالجة الـ interrupt — يعني الـ flow handler.

**ليه الـ typedef منفصل في ملف لوحده؟**

السبب واضح في كومنت الملف نفسه:

> *"Interrupt flow handler typedefs are defined here to avoid circular include dependencies."*

الـ `struct irq_desc` محتاج الـ `irq_flow_handler_t` (لأنه فيه field اسمه `handle_irq` من النوع ده)، والـ `irq_flow_handler_t` محتاجة الـ `struct irq_desc`. لو الاتنين في نفس الملف هيبقى عندنا **circular include** — كل واحد محتاج الثاني قبل ما يتعرّف.

الحل: الـ `typedef` اتنقل لملف خاص صغير، واللي يحتاجه بس يـ include الملف الصغير ده بدل الـ header الكبير، وبكده الدايرة المفرغة بتنكسر.

---

### رحلة الـ Interrupt من البداية للنهاية

```
Hardware Device
      │
      │ (يرسل إشارة كهربية)
      ▼
Interrupt Controller (e.g., APIC, GIC)
      │
      │ (يحدد رقم الـ IRQ ويوصّله للـ CPU)
      ▼
CPU Exception Entry
      │
      │ (يجيب struct irq_desc الخاص بالـ IRQ ده)
      ▼
desc->handle_irq(desc)   ◄─── هنا بتتنادى irq_flow_handler_t
      │
      ├─ handle_level_irq()   ← لو level-triggered
      ├─ handle_edge_irq()    ← لو edge-triggered
      └─ handle_fasteoi_irq() ← لو modern APIC/GIC
                │
                ▼
         action->handler()   ← كود الـ driver نفسه
```

الـ `irq_flow_handler_t` هو بالظبط النوع الخاص بالـ `desc->handle_irq` — يعني دي الـ function اللي بتقرر **كيف** يتعامل مع تدفق الـ interrupt قبل ما يوصل للـ driver.

---

### ليه الملف ده مهم رغم صغره؟

- **كل** الـ flow handlers (level, edge, fasteoi, bad, per-CPU) نوعها `irq_flow_handler_t`.
- **كل** الـ IRQ chip drivers اللي بتكتب custom flow handler محتاجة الـ typedef ده.
- الـ GPIO subsystem بيستخدمه (`gpio/driver.h`).
- الـ IRQ domain بيستخدمه (`irqdomain.h`).
- لو الـ typedef مش موجود في ملف منفصل، الكود كله كان هيبقى مستحيل يـ compile بسبب circular deps.

---

### الملفات المهمة في الـ Subsystem

#### Core Headers
| الملف | الدور |
|---|---|
| `include/linux/irqhandler.h` | تعريف `irq_flow_handler_t` فقط (الملف ده) |
| `include/linux/irq.h` | الـ API الكامل للـ IRQ subsystem |
| `include/linux/irqdesc.h` | تعريف `struct irq_desc` (قلب الـ subsystem) |
| `include/linux/irqdomain.h` | mapping بين hardware IRQ numbers و Linux IRQ numbers |
| `include/linux/irqreturn.h` | تعريف `irqreturn_t` (IRQ_HANDLED, IRQ_NONE, ...) |
| `include/linux/irqchip.h` | واجهة الـ irqchip drivers |

#### Core Implementation
| الملف | الدور |
|---|---|
| `kernel/irq/chip.c` | تنفيذ الـ flow handlers (level, edge, fasteoi, ...) |
| `kernel/irq/handle.c` | الـ generic interrupt handling entry points |
| `kernel/irq/manage.c` | إدارة الـ IRQ actions (request_irq, free_irq, ...) |
| `kernel/irq/irqdesc.c` | إنشاء وإدارة `struct irq_desc` |
| `kernel/irq/irqdomain.c` | تنفيذ الـ IRQ domain mapping |

#### Hardware Drivers
| الملف | الدور |
|---|---|
| `drivers/irqchip/` | كل الـ interrupt controllers (GIC, APIC, ...) |
| `include/linux/gpio/driver.h` | GPIO chips اللي بتستخدم IRQ flow handlers |
## Phase 2: شرح الـ IRQ Flow Handler Framework

### المشكلة — ليه الـ Subsystem ده موجود أصلاً؟

لما hardware interrupt بيحصل، السؤال مش بس "مين الـ driver اللي هيتعامل معاه؟" — السؤال الأعمق هو **"إزاي الـ interrupt نفسه بيتحكم فيه على مستوى الـ hardware قبل ما يوصل للـ driver?"**

الـ interrupt hardware مش كله شغّال بنفس الطريقة:

| النوع | الوصف | مثال |
|---|---|---|
| **Edge-triggered** | بيحصل لحظة واحدة عند حافة الـ signal | زرار، GPIO |
| **Level-triggered** | بيفضل active طول ما الـ signal موجود | PCI interrupt line |
| **Percpu** | كل CPU عندها نسختها الخاصة من الـ interrupt | local timer |
| **Chained / Nested** | interrupt controller جوه interrupt controller | GPIO expander على I2C |
| **Fasteoi** | hardware بيعمل EOI تلقائي بعد الـ handler | GIC v3 على ARM |

لو الـ kernel عمل نفس الكود لكل الحالات دي، هيبقى فيه `if/else` بلا نهاية في المسار الأكتر حساسية في الـ kernel — مسار الـ interrupt handling.

**الحل:** فصل **كيفية التحكم في الـ interrupt hardware** (flow) عن **ما يعمله الـ driver** (action). الـ `irq_flow_handler_t` هي التجسيد لهذا الفصل.

---

### الحل — المقاربة اللي اتخدت

الـ kernel عرّف نوع واحد بسيط جداً:

```c
typedef void (*irq_flow_handler_t)(struct irq_desc *desc);
```

**الـ `irq_flow_handler_t`** هي pointer لـ function بتاخد `irq_desc` وبترجع void. الـ kernel بيجهز مجموعة من الـ flow handlers الجاهزة، وكل `irq_desc` بيحمل واحدة منهم في الـ field الاسمه `handle_irq`.

لما interrupt بيحصل، الـ architecture code بتعمل حاجة واحدة بس:

```c
/* من irqdesc.h */
static inline void generic_handle_irq_desc(struct irq_desc *desc)
{
    desc->handle_irq(desc);  /* استدعاء الـ flow handler المناسب */
}
```

الـ flow handler هي اللي بتقرر:
1. هل تعمل mask/unmask للـ interrupt؟
2. هل تعمل acknowledge للـ chip؟
3. هل توصل للـ `irqaction` handlers؟
4. هل تعمل EOI (End of Interrupt)؟

---

### الـ Big Picture — معمار الـ Subsystem

```
+--------------------------------------------------+
|              Hardware Interrupt Event            |
|         (GPIO line goes high, timer fires, ...)  |
+--------------------------------------------------+
                          |
                          v
+--------------------------------------------------+
|          Architecture Entry Point (arch/)        |
|   ARM64: vectors.S → el1_irq → handle_arch_irq  |
+--------------------------------------------------+
                          |
                          v
+--------------------------------------------------+
|         Interrupt Controller Driver (irqchip/)   |
|    GIC, NVIC, PIC ...                            |
|    يحدد hwirq → يترجمه لـ Linux irq number      |
+--------------------------------------------------+
                          |
                          v  generic_handle_domain_irq()
+--------------------------------------------------+
|           irq_desc  (Generic IRQ Core)           |
|  +--------------------------------------------+  |
|  |  handle_irq  → [irq_flow_handler_t]        |  |  ← هنا موضوعنا
|  |  action      → irqaction → irqaction → ... |  |
|  |  irq_data    → irq_chip ops               |  |
|  +--------------------------------------------+  |
+--------------------------------------------------+
         |                        |
         v                        v
  [Flow Handler]           [irq_chip ops]
  handle_fasteoi_irq       chip->irq_mask()
  handle_edge_irq          chip->irq_ack()
  handle_level_irq         chip->irq_eoi()
  handle_percpu_irq        chip->irq_unmask()
         |
         v
  [irqaction chain]
  driver_handler_1()
  driver_handler_2()  ← request_irq() بيضيف هنا
```

---

### التشبيه الواقعي — مركز اتصالات (Call Center)

تخيل **مركز اتصالات** فيه:

| جزء في التشبيه | يقابله في الـ kernel |
|---|---|
| **الخط التليفوني** | الـ interrupt line (GPIO, IRQ pin) |
| **سنترال الشركة** | الـ interrupt controller (GIC/NVIC) |
| **بروتوكول استقبال المكالمة** | الـ `irq_flow_handler_t` |
| **موظف الـ customer service** | الـ `irqaction` handler (driver) |
| **قواعد التحويل والإيقاف** | الـ `irq_chip` ops (mask/ack/eoi) |

**الـ flow handler هي البروتوكول** — مش الموظف نفسه.

مثلاً:
- **Edge call (مكالمة واحدة):** السنترال بيلاحظ إن الخط رنّ مرة واحدة، بيحجبه (mask) عشان ما يرنش تاني، بيوديه للموظف، وبعد ما الموظف يخلص بيفتحه تاني ← هذا `handle_edge_irq`
- **Level call (حد صابر على الخط):** الخط فاضل مشغول طول ما المشكلة مش اتحلت، الموظف لازم يحلها أولاً عشان الخط يتحرر ← هذا `handle_level_irq`
- **Fasteoi call:** السنترال الحديث بيعمل ACK تلقائي بعد ما الموظف يرد، مفيش تدخل يدوي ← هذا `handle_fasteoi_irq` (المستخدم في GIC على ARM)

---

### الـ Core Abstraction — الفكرة المركزية

الـ `irq_flow_handler_t` بتجسّد مبدأ **"فصل الـ flow عن الـ action"**:

```
irq_desc
├── handle_irq  ←── [FLOW] كيف يُدار الـ interrupt على مستوى الـ hardware؟
│                          (mask? ack? eoi? order of operations?)
└── action      ←── [ACTION] ماذا يفعل الـ software بعد ما الـ hardware اتعالج؟
                             (driver callbacks)
```

الـ **flow** بيخص الـ chip وطبيعة الـ trigger — ده شغل الـ `irq_flow_handler_t`.
الـ **action** بيخص الـ subsystem أو الـ driver — ده شغل الـ `irqaction`.

---

### الـ `irq_desc` — قلب الـ Framework

الـ `struct irq_desc` هو الـ "ملف الكامل" لكل interrupt في النظام. يربط كل الأجزاء:

```
struct irq_desc
│
├── irq_common_data  ←── بيانات مشتركة بين كل الـ irqchips في الـ hierarchy
│   ├── affinity         (على أنهي CPUs يشتغل؟)
│   └── handler_data     (private data للـ flow handler)
│
├── irq_data         ←── بيانات الـ chip layer
│   ├── irq              (Linux IRQ number — logical)
│   ├── hwirq            (hardware IRQ number — physical)
│   ├── chip      ───────────────────────────────────────┐
│   ├── domain           (الـ translation map)           │
│   └── parent_data      (للـ hierarchy domains)         │
│                                                        │
├── handle_irq  ←── [irq_flow_handler_t]                 │
│   (مثلاً: handle_fasteoi_irq)                          │
│                                                        v
├── action ──────► irqaction ──► irqaction ──► NULL   irq_chip
│                  (driver 1)    (driver 2)            ├── irq_ack()
│                                                      ├── irq_mask()
├── kstat_irqs   ←── per-CPU counter                   ├── irq_unmask()
├── depth        ←── nested disable count              ├── irq_eoi()
├── lock         ←── SMP protection                    └── irq_set_type()
└── name         ←── /proc/interrupts entry
```

**ملاحظة مهمة:** الـ `irq_data` بيحمل الـ `hwirq` (رقم الـ interrupt في الـ hardware) والـ `irq` (الرقم اللي الـ Linux بيشوفه). الـ translation بيتعمل عن طريق الـ **IRQ domain** subsystem — ده subsystem مستقل مهمته تحويل `hwirq → linux irq`.

---

### الـ Flow Handlers الموجودة في الـ Kernel

الـ kernel بيجهز مجموعة من الـ handlers الجاهزة في `kernel/irq/chip.c`:

| Handler | متى يُستخدم؟ | الترتيب |
|---|---|---|
| `handle_edge_irq` | Edge-triggered (GPIO, buttons) | ack → run action → unmask |
| `handle_level_irq` | Level-triggered (legacy PCI) | mask → ack → run action → unmask |
| `handle_fasteoi_irq` | Modern GIC (ARM), APIC | run action → eoi |
| `handle_percpu_irq` | Per-CPU interrupts (local timer) | run action (no locking needed) |
| `handle_nested_irq` | Threaded/chained (GPIO expander) | run action in thread context |
| `handle_bad_irq` | Unknown/unconfigured IRQ | warn and drop |

---

### ماذا يملك الـ Subsystem وماذا يفوّض؟

**الـ Generic IRQ Core يملك:**
- الـ `irq_desc` table وإدارة الذاكرة
- تسلسل استدعاء الـ `irqaction` chain
- إحصاءات الـ interrupts (`kstat_irqs`)
- كشف الـ spurious interrupts
- الـ SMP balancing و affinity logic
- الـ threaded IRQ mechanism
- الـ `/proc/interrupts` و `/proc/irq/` interface

**يفوّض للـ `irq_chip` (الـ driver):**
- `irq_ack()` — كيف تعمل acknowledge للـ interrupt في الـ hardware
- `irq_mask()` / `irq_unmask()` — كيف تحجب/تفتح الـ interrupt
- `irq_eoi()` — كيف تعمل End-of-Interrupt
- `irq_set_type()` — كيف تضبط edge/level على الـ hardware registers
- `irq_set_affinity()` — كيف توجّه الـ interrupt لـ CPU معين

**يفوّض للـ driver عبر `irqaction`:**
- ما يفعله الـ software بعد وصول الـ interrupt
- هل الـ interrupt مشترك أم حصري؟

---

### مثال حي — GIC على ARM64

```
ARM64 Exception Vector
        |
        v
gic_handle_irq()       ← GIC driver في drivers/irqchip/irq-gic-v3.c
        |
        | يقرأ hwirq من GIC_CPU_INTACK
        v
generic_handle_domain_irq(domain, hwirq)
        |
        | يحول hwirq → linux irq عبر الـ IRQ domain
        v
irq_desc->handle_irq()  ← handle_fasteoi_irq (الـ default لـ GIC)
        |
        | 1. يستدعي irqaction handlers
        | 2. بعد كل الـ handlers يعمل chip->irq_eoi()
        v
driver_interrupt_handler()  ← الكود بتاعك في الـ driver
```

الـ GIC بيضبط الـ flow handler في `irq_set_chip_and_handler()` لما بيعمل map للـ interrupt:

```c
/* GIC driver يعمل ده لكل interrupt */
irq_set_chip_and_handler(irq, &gic_chip, handle_fasteoi_irq);
```

ده بالضبط معناه: "الـ chip اللي بيتحكم في الـ hardware هو `gic_chip`، والـ flow protocol المناسب هو `handle_fasteoi_irq`".

---

### لماذا الـ `irqhandler.h` صغير جداً؟

الملف بيحتوي على typedef واحد بس، وده مقصود. التعليق في الملف بيقول:

> "Interrupt flow handler typedefs are defined here to avoid **circular include dependencies**."

الـ `irq_flow_handler_t` بتحتاجها كلٌّ من `irqdesc.h` و `irq.h`، واللي بتحتاجوا بعض. عشان تكسر الـ circular dependency، الـ typedef اتحط في ملف minimal مستقل مش بيـ include أي حاجة تانية.

ده نمط شائع في الـ kernel لما يكون عندك type بيتشارك بين headers كثيرة — تعزله في header صغير خاص بيه.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الفكرة الأساسية

الـ `irqhandler.h` نفسه صغير جداً — بيعرّف حاجة واحدة بس:

```c
typedef void (*irq_flow_handler_t)(struct irq_desc *desc);
```

بس ده مش بسيط. الـ `irq_flow_handler_t` هو العمود الفقري لكل الـ interrupt flow control في الـ kernel. عشان نفهمه صح، لازم نفهم الـ `struct irq_desc` اللي بيُمرَّر ليه، واللي معرّف في `irqdesc.h` و `irq.h`.

---

### 0. Flags & Enums — Cheatsheet

#### IRQ Type Flags (`irq.h`) — نوع الـ trigger

| Flag | Value | المعنى |
|------|-------|---------|
| `IRQ_TYPE_NONE` | `0x0` | مفيش type محدد |
| `IRQ_TYPE_EDGE_RISING` | `0x1` | trigger على الـ rising edge |
| `IRQ_TYPE_EDGE_FALLING` | `0x2` | trigger على الـ falling edge |
| `IRQ_TYPE_EDGE_BOTH` | `0x3` | الـ edge الاتنين |
| `IRQ_TYPE_LEVEL_HIGH` | `0x4` | level trigger — high |
| `IRQ_TYPE_LEVEL_LOW` | `0x8` | level trigger — low |
| `IRQ_TYPE_SENSE_MASK` | `0xf` | mask لكل أنواع الـ trigger |
| `IRQ_TYPE_PROBE` | `0x10` | جاري الـ probing |

#### IRQ Status Flags (`irq.h`) — حالة الـ IRQ

| Flag | Bit | المعنى |
|------|-----|---------|
| `IRQ_LEVEL` | 8 | interrupt من نوع level |
| `IRQ_PER_CPU` | 9 | خاص بـ CPU واحد |
| `IRQ_NOPROBE` | 10 | ممنوع autoprobing |
| `IRQ_NOREQUEST` | 11 | ممنوع `request_irq()` |
| `IRQ_NOAUTOEN` | 12 | مش بيتفعّل تلقائياً |
| `IRQ_NO_BALANCING` | 13 | ممنوع affinity balancing |
| `IRQ_NESTED_THREAD` | 15 | بيتعشش جوه thread تاني |
| `IRQ_NOTHREAD` | 16 | ممنوع threading |
| `IRQ_PER_CPU_DEVID` | 17 | الـ dev_id هو per-cpu variable |
| `IRQ_IS_POLLED` | 18 | بيتعمله poll من interrupt تاني |
| `IRQ_DISABLE_UNLAZY` | 19 | إيقاف الـ lazy disable |
| `IRQ_HIDDEN` | 20 | مش بيظهر في `/proc/interrupts` |
| `IRQ_NO_DEBUG` | 21 | مستثنى من `note_interrupt()` |

#### IRQD State Bits (`irq_common_data.state_use_accessors`)

| Flag | Bit | المعنى |
|------|-----|---------|
| `IRQD_TRIGGER_MASK` | 0-3 | نوع الـ trigger الحالي |
| `IRQD_SETAFFINITY_PENDING` | 8 | في affinity change معلّق |
| `IRQD_ACTIVATED` | 9 | تم تفعيله |
| `IRQD_NO_BALANCING` | 10 | بالانسينج ممنوع |
| `IRQD_PER_CPU` | 11 | per-cpu |
| `IRQD_IRQ_DISABLED` | 16 | معطّل |
| `IRQD_IRQ_MASKED` | 17 | masked |
| `IRQD_IRQ_INPROGRESS` | 18 | شغّال دلوقتي |
| `IRQD_WAKEUP_ARMED` | 19 | مجهّز للـ wakeup |
| `IRQD_FORWARDED_TO_VCPU` | 20 | محوّل لـ VCPU |
| `IRQD_AFFINITY_MANAGED` | 21 | الـ kernel بيدير الـ affinity تلقائياً |
| `IRQD_RESEND_WHEN_IN_PROGRESS` | 30 | إعادة الإرسال لو الـ IRQ شغّال |

#### IRQ Set Affinity Return Values

| Value | المعنى |
|-------|---------|
| `IRQ_SET_MASK_OK` | تمام — الـ core بيحدّث الـ affinity |
| `IRQ_SET_MASK_OK_NOCOPY` | تمام — الـ chip حدّث الـ affinity بنفسه |
| `IRQ_SET_MASK_OK_DONE` | زي OK بس يتجاهل الـ child chips |

---

### 1. الـ Structs المهمة

---

#### `irq_flow_handler_t` — القلب

```c
typedef void (*irq_flow_handler_t)(struct irq_desc *desc);
```

**الغرض:** function pointer بيمثّل الـ **flow handler** — اللي بيتحكم في تسلسل معالجة الـ interrupt (mask → handle → unmask أو EOI أو غيرها).

**مش هو الـ ISR الخاص بالـ driver** — ده layer أعلى. الـ flow handler هو اللي بيحدد *إزاي* الـ interrupt بيتعالج (edge vs level vs oneshot…)، وبعدين بيشغّل الـ `irqaction` chain بتاع الـ driver.

**أمثلة على الـ flow handlers الـ built-in:**

| Handler | الاستخدام |
|---------|-----------|
| `handle_edge_irq` | edge-triggered |
| `handle_level_irq` | level-triggered |
| `handle_fasteoi_irq` | modern controllers (EOI بعد الـ handler) |
| `handle_percpu_irq` | per-CPU interrupts |
| `handle_nested_irq` | nested/threaded |
| `handle_bad_irq` | IRQ مجهول أو خاطئ |

---

#### `struct irq_desc` — الـ Descriptor الرئيسي

**الغرض:** يمثّل **كل interrupt line** في النظام. ده الـ struct الأكبر والأهم — كل interrupt رقم في الـ kernel عنده `irq_desc` خاص بيه.

| Field | النوع | الدور |
|-------|-------|-------|
| `irq_common_data` | `struct irq_common_data` | بيانات مشتركة بين كل الـ irqchips في الـ hierarchy |
| `irq_data` | `struct irq_data` | بيانات الـ chip الـ bottom-level (الـ hardware الفعلي) |
| `kstat_irqs` | `struct irqstat __percpu *` | إحصائيات per-CPU |
| `handle_irq` | `irq_flow_handler_t` | **الـ flow handler** — اللي معرّفناه في `irqhandler.h` |
| `action` | `struct irqaction *` | linked list لكل الـ ISR handlers المسجّلة |
| `status_use_accessors` | `unsigned int` | flags حالة الـ IRQ |
| `depth` | `unsigned int` | عدد nested `irq_disable()` calls |
| `wake_depth` | `unsigned int` | عدد nested `irq_set_irq_wake()` calls |
| `irq_count` | `unsigned int` | لكشف الـ stalled IRQs |
| `irqs_unhandled` | `unsigned int` | عدد الـ spurious interrupts |
| `lock` | `raw_spinlock_t` | الـ lock الأساسي للحماية في SMP |
| `redirect` | `struct irq_redirect` | إعادة توجيه الـ IRQ لـ CPU تاني |
| `affinity_hint` | `const struct cpumask *` | hint للـ userspace |
| `threads_oneshot` | `unsigned long` | bitmask لإدارة الـ oneshot threads |
| `threads_active` | `atomic_t` | عدد الـ threads الشغّالة دلوقتي |
| `wait_for_threads` | `wait_queue_head_t` | لـ `synchronize_irq()` |
| `request_mutex` | `struct mutex` | بيحمي `request_irq()` / `free_irq()` |
| `name` | `const char *` | اسم الـ flow handler في `/proc/interrupts` |

---

#### `struct irq_common_data` — البيانات المشتركة

**الغرض:** بيانات بتتشارك بين كل الـ irqchips في الـ hierarchy (مش بس الـ leaf chip).

| Field | الدور |
|-------|-------|
| `state_use_accessors` | IRQD_* flags — الحالة الداخلية |
| `node` | NUMA node للـ balancing |
| `handler_data` | pointer لبيانات الـ driver الخاصة |
| `msi_desc` | pointer لـ MSI descriptor لو كان MSI interrupt |
| `affinity` | الـ CPUs المسموح بتشغيل الـ interrupt عليها |
| `effective_affinity` | الـ affinity الفعلية (بعض الـ chips مش بتسمح بـ multi-CPU) |
| `ipi_offset` | offset الـ IPI في الـ affinity mask |

---

#### `struct irq_data` — بيانات الـ Chip

**الغرض:** البيانات اللي بتتمرر للـ `irq_chip` functions — بتمثّل الـ interrupt على مستوى الـ hardware.

| Field | الدور |
|-------|-------|
| `mask` | bitmask محسوب مسبقاً للـ chip register |
| `irq` | رقم الـ IRQ المنطقي في الـ Linux |
| `hwirq` | رقم الـ IRQ في الـ hardware (local to domain) |
| `common` | pointer لـ `irq_common_data` |
| `chip` | pointer لـ `irq_chip` — الـ hardware operations |
| `domain` | الـ IRQ domain المسؤول عن الـ mapping |
| `parent_data` | pointer لـ parent في الـ hierarchy (لـ stacked domains) |
| `chip_data` | بيانات خاصة بالـ chip implementation |

---

#### `struct irqstat` — الإحصائيات

```c
struct irqstat {
    unsigned int cnt;   /* العداد الفعلي للـ interrupts */
    unsigned int ref;   /* snapshot للمقارنة (لو CONFIG_GENERIC_IRQ_STAT_SNAPSHOT) */
};
```

per-CPU — كل CPU عنده نسخته الخاصة لكل `irq_desc`.

---

#### `struct irq_redirect` — إعادة التوجيه

```c
struct irq_redirect {
    struct irq_work work;       /* irq_work item — بيتنفّذ على CPU تاني */
    unsigned int    target_cpu; /* الـ CPU المستهدف */
};
```

**الغرض:** لو الـ CPU الحالي مش في الـ affinity mask، بيستخدم الـ `irq_work` mechanism لإعادة تشغيل الـ handler على الـ CPU الصح.

---

### 2. مخطط العلاقات بين الـ Structs

```
irqhandler.h defines:
┌─────────────────────────────────────┐
│  irq_flow_handler_t                 │
│  = void (*)(struct irq_desc *desc)  │
└───────────────────┬─────────────────┘
                    │ parameter
                    ▼
┌──────────────────────────────────────────────────────────────┐
│  struct irq_desc          (____cacheline_internodealigned)   │
│                                                              │
│  ┌─────────────────────┐  .irq_common_data                  │
│  │ struct irq_common_  │◄─────────────────                  │
│  │ data                │                                     │
│  │  .state_use_access  │                                     │
│  │  .affinity ──────────────► cpumask                       │
│  │  .handler_data ─────────► driver private data            │
│  │  .msi_desc ─────────────► struct msi_desc                │
│  └─────────────────────┘                                     │
│                                                              │
│  ┌─────────────────────┐  .irq_data                         │
│  │ struct irq_data     │◄─────────────────                  │
│  │  .irq  (linux #)    │                                     │
│  │  .hwirq (hw #)      │                                     │
│  │  .chip ─────────────────► struct irq_chip                │
│  │  .domain ───────────────► struct irq_domain              │
│  │  .common ───────────────► irq_common_data (back ptr)     │
│  │  .parent_data ──────────► struct irq_data (hierarchy)    │
│  │  .chip_data ────────────► chip private data              │
│  └─────────────────────┘                                     │
│                                                              │
│  .handle_irq ──────────────► irq_flow_handler_t (fn ptr)   │
│  .action ───────────────────► struct irqaction              │
│                                  └─► .next ──► irqaction   │
│  .kstat_irqs ───────────────► struct irqstat [per_cpu]     │
│  .lock (raw_spinlock_t)                                      │
│  .request_mutex (mutex)                                      │
│                                                              │
│  [SMP only]                                                  │
│  .redirect.work ───────────► irq_work (deferred execution)  │
│  .affinity_hint ───────────► cpumask (userspace hint)       │
│  .affinity_notify ─────────► struct irq_affinity_notify     │
│                                                              │
│  [SPARSE_IRQ]                                                │
│  .rcu ──────────────────────► rcu_head (delayed free)       │
│  .kobj ─────────────────────► kobject (sysfs)               │
└──────────────────────────────────────────────────────────────┘
```

---

### 3. مخطط دورة الحياة — من التسجيل للـ teardown

```
Boot / Driver Init
      │
      ▼
irq_alloc_desc()                   ← يخصّص irq_desc جديد
      │                              (أو من irq_desc[] الثابتة لو !SPARSE_IRQ)
      ▼
irq_domain_alloc_irqs()            ← يربط الـ hwirq بـ linux irq number
      │                              عبر struct irq_domain
      ▼
irq_set_chip_and_handler()         ← يضبط irq_data.chip
      │                              ويضبط irq_desc.handle_irq (الـ flow handler)
      ▼
request_irq() / request_threaded_irq()
      │                            ← يخصّص struct irqaction
      │                              يضيفها لـ irq_desc.action chain
      │                              (محمي بـ request_mutex أولاً، ثم desc->lock)
      ▼
enable_irq()                       ← يعمل unmask للـ hardware
      │                              (عبر irq_chip.irq_unmask)
      ▼
══════════════════════════════════
      IRQ حصل في الـ Hardware
══════════════════════════════════
      │
      ▼
CPU يستقبل interrupt vector
      │
      ▼
arch interrupt entry (assembly)
      │
      ▼
generic_handle_irq_desc(desc)      ← inline: desc->handle_irq(desc)
      │
      ▼
handle_edge_irq() / handle_level_irq() / ...
      │                            ← الـ flow handler يتحكم في:
      │                              1. mask الـ interrupt (لو level)
      │                              2. ACK الـ chip
      │                              3. تشغيل الـ action chain
      ▼
handle_irq_event(desc)
      │
      ▼
for each irqaction in desc->action:
      │
      ▼
action->handler(irq, dev_id)       ← الـ ISR الفعلي للـ driver
      │
      ▼
unmask / EOI                       ← الـ flow handler ينهي الدورة
══════════════════════════════════
      Teardown
══════════════════════════════════
      │
      ▼
free_irq()                         ← يشيل الـ irqaction من الـ chain
      │                              ينتظر الـ threads تخلص (synchronize_irq)
      ▼
irq_domain_free_irqs()             ← يلغي الـ mapping
      ▼
irq_free_desc()                    ← يحرر الـ irq_desc
                                     (RCU delayed لو SPARSE_IRQ)
```

---

### 4. مخطط تدفق الـ Call Flow

#### من الـ Hardware لحد الـ ISR

```
Hardware raises interrupt
  │
  └─► CPU exception entry (arch/arm64/kernel/entry.S أو x86 equivalent)
        │
        └─► handle_domain_irq(domain, hwirq, regs)
              │
              ├─► irq_find_mapping(domain, hwirq)
              │     └─► يرجع linux irq number
              │
              └─► generic_handle_irq(irq)
                    │
                    └─► irq_to_desc(irq)       ← يجيب irq_desc
                          │
                          └─► generic_handle_irq_desc(desc)
                                │
                                └─► desc->handle_irq(desc)
                                      │
                                      ├─[edge]─► handle_edge_irq(desc)
                                      │            ├─ chip->irq_ack()
                                      │            ├─ handle_irq_event()
                                      │            │    └─ action->handler()
                                      │            └─ (unmask if needed)
                                      │
                                      ├─[level]─► handle_level_irq(desc)
                                      │            ├─ chip->irq_mask_ack()
                                      │            ├─ handle_irq_event()
                                      │            │    └─ action->handler()
                                      │            └─ chip->irq_unmask()
                                      │
                                      └─[fasteoi]► handle_fasteoi_irq(desc)
                                                   ├─ handle_irq_event()
                                                   │    └─ action->handler()
                                                   └─ chip->irq_eoi()
```

#### ضبط الـ Flow Handler من الـ Driver/Board Code

```
irq_set_irq_type(irq, IRQ_TYPE_EDGE_RISING)
  │
  └─► chip->irq_set_type(irq_data, IRQ_TYPE_EDGE_RISING)
        │
        └─► [inside chip driver] irq_set_handler_locked(data, handle_edge_irq)
              │
              └─► desc->handle_irq = handle_edge_irq
```

#### إعادة التوجيه في SMP (irq_redirect)

```
interrupt arrives on CPU-X
  │
  └─► CPU-X not in affinity mask?
        │
        ├─[No]─► handle normally
        │
        └─[Yes]─► irq_redirect.target_cpu = correct CPU
                    │
                    └─► irq_work_queue_on(target_cpu, &redirect.work)
                          │
                          └─► target CPU runs handler via irq_work mechanism
```

---

### 5. استراتيجية الـ Locking

الـ `irq_desc` بيستخدم **طبقتين من الـ locks** بترتيب ثابت لمنع الـ deadlock:

```
┌──────────────────────────────────────────────────────────────┐
│                   Lock Hierarchy                             │
│                                                              │
│  1. desc->request_mutex  (struct mutex)                      │
│     ├─ يحمي: request_irq() و free_irq()                     │
│     ├─ السبب: عمليات الـ setup/teardown بتاخد وقت طويل     │
│     └─ lazily acquired قبل desc->lock                        │
│                                                              │
│  2. desc->lock  (raw_spinlock_t)                             │
│     ├─ يحمي: كل fields الـ irq_desc تقريباً                 │
│     │   - handle_irq (الـ flow handler pointer)             │
│     │   - action chain                                       │
│     │   - status/depth/wake_depth                           │
│     │   - chip configuration                                │
│     └─ interrupt context safe (raw_spinlock)                 │
└──────────────────────────────────────────────────────────────┘

الترتيب الإلزامي:
  request_mutex → desc->lock

خطأ شائع: محاولة أخد desc->lock ثم request_mutex → deadlock مضمون
```

#### جدول تفصيلي

| الـ Resource المحمي | الـ Lock المستخدم | السياق |
|---------------------|-------------------|---------|
| `desc->handle_irq` | `desc->lock` | hardirq-safe |
| `desc->action` chain | `desc->lock` | hardirq-safe |
| `desc->depth` / `status` | `desc->lock` | hardirq-safe |
| `irq_common_data.affinity` | `desc->lock` | hardirq-safe |
| `request_irq` / `free_irq` flow | `request_mutex` ثم `desc->lock` | process context only |
| `irq_desc` allocation (SPARSE_IRQ) | `irq_lock_sparse()` | process context |
| `irq_desc` pointer read | RCU | read-side: rcu_read_lock() |
| per-CPU `kstat_irqs` | implicit (per-CPU) | لا يحتاج lock |
| `threads_active` | `atomic_t` | lock-free atomic |
| `wait_for_threads` | `wait_queue_head_t` | built-in locking |

#### ملاحظة على RCU

في `CONFIG_SPARSE_IRQ`، الـ `irq_desc` بيتحرر بـ RCU (الـ `rcu` field). يعني:
- الـ lookup بيحتاج `rcu_read_lock()` مش lock أثقل
- الـ `irq_free_desc()` بتستخدم `call_rcu()` — التحرير الفعلي بيحصل بعد كل الـ readers يخلصوا

```
irq_to_desc(irq):
  rcu_read_lock()
    │
    └─► radix_tree_lookup(irq_desc_tree, irq)  ← O(log n) آمن
  rcu_read_unlock()
```
## Phase 4: شرح الـ Functions

---

### ملخص عام

**الـ `irqhandler.h`** ملف صغير جداً — محتواه الوحيد هو تعريف **`irq_flow_handler_t`**، وهو الـ typedef الأساسي اللي يمثّل الـ flow handler لأي interrupt في الـ generic IRQ subsystem. الهدف منه فصل الـ circular includes بين ملفات الـ IRQ المختلفة.

---

### Overview Table — كل الـ APIs في الملف

| الاسم | النوع | الغرض |
|---|---|---|
| `irq_flow_handler_t` | `typedef` | نوع الـ pointer للـ flow handler function |
| `struct irq_desc` | forward declaration | الـ interrupt descriptor — مُعرَّف في `irqdesc.h` |

> الملف نفسه لا يحتوي على functions — كل محتواه هو تعريف نوع واحد وـ forward declaration واحد.

---

### Category 1: Type Definitions — الأنواع الأساسية

#### الغرض من المجموعة

الـ `irq_flow_handler_t` هو العمود الفقري لنظام الـ interrupts في Linux. كل interrupt يملك **flow handler** مُخزَّن في `irq_desc->handle_irq`، وده الـ handler اللي بيتحكم في **كيفية تسليم الـ interrupt** للـ driver — مش الـ driver نفسه. الفصل ده (ملف منفصل للـ typedef) موجود لتجنب الـ circular includes بين `irq.h` و `irqdesc.h` وملفات الـ architecture.

---

### `irq_flow_handler_t`

```c
typedef void (*irq_flow_handler_t)(struct irq_desc *desc);
```

**الـ `irq_flow_handler_t`** هو نوع الـ function pointer اللي بيمثّل الـ high-level flow control handler لأي interrupt line. بيُستدعى من الـ interrupt entry path بعد ما الـ architecture بتحدد رقم الـ IRQ وتجيب الـ `irq_desc` الخاص بيه. الـ flow handler هو المسؤول عن التحكم في سلوك الـ IRQ line (masking، acking، running actions، إلخ) — مش تنفيذ الـ driver logic مباشرةً.

| البارامتر | النوع | الوصف |
|---|---|---|
| `desc` | `struct irq_desc *` | الـ interrupt descriptor الكامل للـ IRQ المُحدَّد — بيحتوي على الـ chip، الـ action chain، الـ status، إلخ |

**القيمة المُرجَّعة:** `void` — الـ flow handler مش بيرجع حاجة.

**الـ Key Details:**

- الـ flow handler **مش هو الـ ISR** (الـ `irqaction->handler`) — هو طبقة أعلى بتتحكم في flow الـ interrupt على مستوى الـ line، مش على مستوى الـ device.
- الـ kernel بيوفّر implementations جاهزة: `handle_level_irq`، `handle_edge_irq`، `handle_fasteoi_irq`، `handle_percpu_irq`، إلخ — كل واحدة بتتعامل مع نوع مختلف من الـ interrupt topology.
- الـ handler ده بيتخزَّن في `irq_desc->handle_irq` ويتغير ديناميكياً عبر `irq_set_handler()` أو `irq_set_handler_locked()`.
- **Locking:** الـ `irq_desc->lock` (raw_spinlock) لازم يكون مضبوط قبل ما تغيّر الـ handler — ولو بتغيّره من `irq_set_type()` callback، يبقى اللوك مضبوط بالفعل.
- **Caller context:** بيُستدعى دايماً من **hardirq context** — مباشرةً من `generic_handle_irq_desc()` اللي بتتكلّمه architecture entry points.

**من بيستدعيه:**

```
arch IRQ entry (e.g., handle_domain_irq / __handle_domain_irq)
    └─> generic_handle_irq_desc(desc)
            └─> desc->handle_irq(desc)   ← هنا بيتنادى الـ flow handler
                    └─> handle_irq_event(desc)
                            └─> irqaction->handler(irq, dev_id)  ← الـ driver ISR
```

**الـ Flow Handlers الجاهزة في الـ Kernel:**

| Handler | متى تُستخدم |
|---|---|
| `handle_level_irq` | Level-triggered IRQs — mask أثناء الـ processing |
| `handle_edge_irq` | Edge-triggered IRQs — ack مبكر قبل الـ processing |
| `handle_fasteoi_irq` | Modern GIC-style — EOI بعد الـ processing |
| `handle_percpu_irq` | Per-CPU interrupts (timers، IPIs) |
| `handle_bad_irq` | IRQs بدون action أو غير configured |
| `handle_nested_irq` | Nested/chained IRQs (e.g., GPIO expander على I2C) |

---

### Category 2: Forward Declaration — الـ `struct irq_desc`

```c
struct irq_desc;
```

**الـ forward declaration** للـ `struct irq_desc` موجود في `irqhandler.h` فقط عشان يخلّي التعريف `irq_flow_handler_t` صالح من غير ما يحتاج يـ include ملف كبير زي `irqdesc.h`. ده نمط شائع في الـ kernel لكسر الـ circular dependencies.

**الـ `struct irq_desc` الحقيقي** مُعرَّف في `include/linux/irqdesc.h` وبيحتوي على:

| الفيلد المهم | النوع | الغرض |
|---|---|---|
| `irq_common_data` | `struct irq_common_data` | بيانات مشتركة بين الـ chip والـ handler |
| `irq_data` | `struct irq_data` | بيانات الـ chip على مستوى الـ hierarchy |
| `kstat_irqs` | `struct irqstat __percpu *` | إحصائيات per-CPU |
| `handle_irq` | `irq_flow_handler_t` | **الـ flow handler** — قلب موضوعنا |
| `action` | `struct irqaction *` | سلسلة الـ ISR handlers المسجَّلين |
| `lock` | `raw_spinlock_t` | حماية الـ descriptor في الـ SMP |
| `depth` | `unsigned int` | عدد nested `disable_irq()` calls |
| `name` | `const char *` | اسم الـ flow handler في `/proc/interrupts` |

---

### العلاقة بين الملفات — Big Picture

```
irqhandler.h
├── typedef irq_flow_handler_t   ← النوع الأساسي
└── forward: struct irq_desc

irqdesc.h
├── #includes irqhandler.h (implicitly via irq.h)
├── struct irq_desc { handle_irq: irq_flow_handler_t; ... }
└── static inline generic_handle_irq_desc() → desc->handle_irq(desc)

irq.h
├── irq_set_handler(irq, handler)
├── irq_set_chip_and_handler(irq, chip, handler)
└── handle_level_irq / handle_edge_irq / handle_fasteoi_irq ...
```

**الـ `irqhandler.h` بيحل مشكلة:** `irqdesc.h` محتاج يعرف `irq_flow_handler_t` عشان يحطّه في الـ struct، و`irq.h` محتاج يعرف `struct irq_desc` — لو كل واحد include التاني هيحصل circular include. الحل: ملف صغير منفصل بيعرّف بس الـ typedef والـ forward declaration.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem اللي بنتكلم عنه هو **IRQ flow handler** — تحديداً الـ `irq_flow_handler_t` اللي بتتعامل مع الـ `struct irq_desc`. ده قلب الـ interrupt handling في Linux kernel.

---

### Software Level

#### 1. Debugfs Entries

لازم يكون `CONFIG_GENERIC_IRQ_DEBUGFS=y` عشان تظهر الـ entries دي.

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# browse الـ IRQ debug entries
ls /sys/kernel/debug/irq/

# كل IRQ ليه directory خاص بيه
ls /sys/kernel/debug/irq/0/
# المخرجات:
# actions  chip  chip_data  handler  hwirq  irqdata  name  per_cpu_count  resend  spurious  threaded  type

# اقرأ الـ flow handler الحالي لأي IRQ (مثلاً IRQ 27)
cat /sys/kernel/debug/irq/27/handler
# handle_fasteoi_irq

# اقرأ stats الـ spurious
cat /sys/kernel/debug/irq/27/spurious
# irq_count:        1000
# irqs_unhandled:      0
# irq_threshold:       0

# اقرأ معلومات الـ chip
cat /sys/kernel/debug/irq/27/chip
# name: GIC-0
# irq_mask: gic_mask_irq
# irq_unmask: gic_unmask_irq
# irq_eoi: gic_eoi_irq

# شوف الـ actions (handlers) المسجلة
cat /sys/kernel/debug/irq/27/actions
# [0:eth0-rx, 0:eth0-tx]

# threaded handler info
cat /sys/kernel/debug/irq/27/threaded
```

#### 2. Sysfs Entries

```bash
# IRQ affinity لكل interrupt
cat /proc/irq/27/affinity_hint
cat /proc/irq/27/smp_affinity
cat /proc/irq/27/smp_affinity_list
cat /proc/irq/27/effective_affinity
cat /proc/irq/27/effective_affinity_list

# اسم الـ node
cat /proc/irq/27/node

# نظرة عامة على كل الـ interrupts
cat /proc/interrupts
# Column headers: CPU0 CPU1 ... IRQ# Type Chip Name [actions]

# IRQ stats ملخصة
cat /proc/softirqs

# kobject لكل IRQ (لو CONFIG_SPARSE_IRQ)
ls /sys/kernel/irq/27/
cat /sys/kernel/irq/27/type
cat /sys/kernel/irq/27/chip_name
cat /sys/kernel/irq/27/hwirq
cat /sys/kernel/irq/27/name
cat /sys/kernel/irq/27/wakeup
cat /sys/kernel/irq/27/actions
cat /sys/kernel/irq/27/per_cpu_count
```

#### 3. Ftrace — Tracepoints والـ Events

```bash
# شوف الـ events المتاحة للـ IRQ subsystem
ls /sys/kernel/debug/tracing/events/irq/

# irq_handler_entry: اتفاكت handler
# irq_handler_exit:  انتهى handler
# softirq_entry / softirq_exit / softirq_raise
# irq_matrix_*: CPU affinity matrix events

# Enable كل الـ IRQ events
echo 1 > /sys/kernel/debug/tracing/events/irq/enable

# تتبع handler معين بس (مثلاً eth0)
echo 'name == "eth0"' > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/filter
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable

# شغل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe

# مثال مخرجات:
# eth0-irq-0  [002] d..1  1234.567890: irq_handler_entry: irq=27 name=eth0
# eth0-irq-0  [002] d..1  1234.567891: irq_handler_exit: irq=27 ret=handled

# تتبع flow handler نفسه باستخدام function tracer
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'handle_fasteoi_irq' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تتبع call graph كامل
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'generic_handle_irq_desc' > /sys/kernel/debug/tracing/set_graph_function
```

#### 4. Printk والـ Dynamic Debug

```bash
# تفعيل dynamic debug لكل ملفات الـ IRQ core
echo 'file kernel/irq/*.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/irq/handle.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/irq/spurious.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لـ module معين
echo 'module my_driver +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug المفعل حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep irq

# رفع مستوى الـ printk مؤقتاً
echo 8 > /proc/sys/kernel/printk

# أو عن طريق kernel cmdline
# loglevel=8 debug

# متابعة kernel messages فورياً
dmesg -w
journalctl -k -f
```

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_GENERIC_IRQ_DEBUGFS` | يفعّل `/sys/kernel/debug/irq/` entries |
| `CONFIG_SPARSE_IRQ` | يفعّل kobject لكل IRQ في sysfs |
| `CONFIG_GENERIC_IRQ_STAT_SNAPSHOT` | يحتفظ بـ snapshot للـ interrupt count |
| `CONFIG_GENERIC_PENDING_IRQ` | يفعّل pending_mask للـ IRQ balancing |
| `CONFIG_IRQ_DOMAIN_DEBUG` | debug info للـ IRQ domain mapping |
| `CONFIG_DEBUG_SHIRQ` | يستدعي handler وقت free لاكتشاف bugs |
| `CONFIG_LOCKDEP` | يتتبع locking order للـ irq_desc->lock |
| `CONFIG_PROVE_LOCKING` | يثبت correctness الـ locking |
| `CONFIG_HARDIRQS_SW_RESEND` | يفعّل soft resend للـ IRQs |
| `CONFIG_IRQ_DOMAIN` | domain mapping — لازم للـ domain debug |
| `CONFIG_GENERIC_IRQ_CHIP` | generic chip support + debug |
| `CONFIG_PM_SLEEP` | يفعّل nr_actions/no_suspend_depth fields |

#### 6. أدوات متخصصة للـ IRQ Subsystem

```bash
# irqtop — مراقبة الـ IRQs real-time (زي top بس للـ IRQs)
irqtop

# watch على /proc/interrupts
watch -n 1 cat /proc/interrupts

# اكتشاف الـ IRQ الأكثر نشاطاً
awk 'NR>1 {sum=0; for(i=2;i<=NF-2;i++) sum+=$i; print sum, $0}' /proc/interrupts | sort -rn | head -20

# irqbalance — الـ daemon المسؤول عن توزيع الـ IRQs
systemctl status irqbalance
irqbalance --debug --foreground 2>&1 | head -50

# قفل IRQ على CPU معين (تعطيل irqbalance أولاً)
systemctl stop irqbalance
echo 4 > /proc/irq/27/smp_affinity   # CPU 2 فقط (bitmask)
echo "2" > /proc/irq/27/smp_affinity_list

# perf stat لقياس الـ IRQ overhead
perf stat -e irq:irq_handler_entry,irq:irq_handler_exit sleep 5

# perf record للـ IRQ handlers
perf record -e irq:irq_handler_entry -a sleep 10
perf report
```

#### 7. جدول رسائل الـ Kernel الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|---|---|---|
| `irq X: nobody cared` | الـ IRQ بيحصل spurious كتير، مفيش handler بيـ handle | تحقق من الـ driver، شوف `irqreturn_t`، تأكد من الـ shared IRQ |
| `Disabling IRQ #X` | الـ kernel قرر يعطل الـ IRQ بعد كتير من الـ spurious | راجع الـ hardware، تحقق من الـ handler يرجع `IRQ_HANDLED` صح |
| `WARN: IRQ X was not disabled` | الـ handler اتنادى وهو المفروض يكون الـ IRQ معطل | bug في الـ chip driver أو flow handler |
| `do_IRQ: X.Y No irq handler for vector` | الـ IRQ vector مش عنده handler مسجل | مشكلة في الـ IRQ domain mapping |
| `irq_flow_handler_t handle_irq is NULL` | الـ `irq_desc->handle_irq` = NULL | الـ IRQ مش مـ configure صح، اتحقق من `irq_set_handler()` |
| `unexpected IRQ trap at vector X` | hardware بعت IRQ مش متوقع | مشكلة firmware أو BIOS |
| `irq X: affinity broken` | فشل تطبيق الـ affinity المطلوب | الـ CPU المستهدف offline، راجع `cpumask` |
| `IRQ X: controller Y stuck` | الـ interrupt controller مش بيستجاب | hardware fault، restart الـ controller |
| `genirq: Flags mismatch irq X` | تعارض في الـ flags عند `request_irq` | `IRQF_SHARED` مش متطابق بين الـ drivers |
| `WARNING: IRQ Y has no chip!` | الـ `irq_data.chip` = NULL | الـ IRQ domain لم يسجل chip صح |

#### 8. نقاط استراتيجية لـ `dump_stack()` و `WARN_ON()`

```c
/* في generic_handle_irq_desc() — تحقق إن الـ handler مش NULL */
static inline void generic_handle_irq_desc(struct irq_desc *desc)
{
    WARN_ON_ONCE(!desc->handle_irq);  /* اكتشف NULL handler */
    desc->handle_irq(desc);
}

/* في irq_set_handler_locked() — تحقق من الـ locking */
static inline void irq_set_handler_locked(struct irq_data *data,
                                          irq_flow_handler_t handler)
{
    struct irq_desc *desc = irq_data_to_desc(data);
    /* ضع WARN_ON هنا لو handler = NULL في context مش متوقع */
    WARN_ON(!handler && desc->action);
    desc->handle_irq = handler;
}

/* في الـ flow handler نفسه — لو irqs_unhandled تجاوز حد */
void handle_fasteoi_irq(struct irq_desc *desc)
{
    /* نقطة جيدة لـ dump_stack لو depth > 1 بشكل غير متوقع */
    WARN_ON_ONCE(desc->depth && !irqd_irq_disabled(&desc->irq_data));
    /* ... */
}

/* في الـ driver عند request_irq — تحقق من الـ return */
ret = request_irq(irq, my_handler, IRQF_SHARED, "my_dev", dev);
WARN_ON(ret < 0);
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware تطابق حالة الـ Kernel

```bash
# قارن الـ IRQ numbers في kernel مع الـ hardware map
cat /proc/interrupts          # kernel view
cat /sys/kernel/debug/irq/*/hwirq  # HW IRQ numbers

# تحقق من الـ IRQ domain mapping
cat /sys/kernel/debug/irq_domain_mapping 2>/dev/null
# أو
ls /sys/kernel/debug/irq/*/

# لـ GIC (ARM): تحقق من الـ GIC registers
cat /sys/kernel/debug/irq/*/chip  # اسم الـ chip

# تحقق إن الـ IRQ مفعّل في الـ hardware
cat /sys/kernel/debug/irq/27/chip  # شوف irq_mask/irq_unmask

# تحقق من الـ affinity الفعلية مقابل المطلوبة
diff <(cat /proc/irq/27/smp_affinity_list) \
     <(cat /proc/irq/27/effective_affinity_list)
```

#### 2. Register Dump Techniques

```bash
# devmem2: اقرأ register من physical address
# مثال: GIC Distributor base على ARM (GICD_ISENABLER)
devmem2 0x08000100 w   # GICD_ISENABLER0 - enabled IRQs بitmask

# /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((0x08000100 / 4)) 2>/dev/null | xxd

# io utility (x86)
# اقرأ APIC registers
io -4 -r 0xFEE00020   # Local APIC ID register (x86)
io -4 -r 0xFEE00080   # Task Priority Register

# لو بتشتغل على ARM مع GIC-v3
# GICD_CTLR (Distributor Control)
devmem2 0x08000000 w
# GICD_ISENABLER<n> (IRQ enable status)
devmem2 0x08000100 w
# GICD_IPRIORITYR<n> (Priority registers)
devmem2 0x08000400 w

# كشف الـ memory-mapped registers من DT/ACPI
cat /proc/iomem | grep -i gic
cat /proc/iomem | grep -i apic
```

#### 3. Logic Analyzer والـ Oscilloscope

```
نصائح عملية:

1. Probe نقاط:
   - الـ IRQ line بين الـ device والـ interrupt controller
   - الـ CS (Chip Select) للـ device
   - الـ interrupt acknowledge signal لو موجود

2. Trigger Setup:
   - Trigger على الـ falling edge (active-low IRQs شائعة)
   - Trigger على الـ rising edge للـ edge-triggered
   - استخدم الـ protocol decoder (SPI/I2C) لتحديد متى الـ device بيبعت IRQ

3. Timing Measurements:
   - وقت الـ IRQ latency: من حدوث الـ event لأول استجابة GPIO
   - وقت الـ handler execution: GPIO toggle في بداية ونهاية الـ handler
   - الـ IRQ pulse width: تحديد edge-triggered vs level-triggered

4. الـ GPIO Toggle في الـ Handler (للـ timing):
```

```c
/* في بداية الـ irq_flow_handler أو الـ action handler */
gpio_set_value(DEBUG_GPIO, 1);  /* rising edge = handler start */
/* ... handler code ... */
gpio_set_value(DEBUG_GPIO, 0);  /* falling edge = handler end */
```

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| مشكلة الـ Hardware | Pattern في الـ Kernel Log |
|---|---|
| IRQ line عالق على active | `irq X: nobody cared` + `Disabling IRQ #X` |
| Level-triggered بدل Edge | IRQ بيتكرر باستمرار، `irqs_unhandled` بيزيد |
| Shared IRQ بدون صحيح clear | Handler بيرجع `IRQ_NONE` دايماً |
| Interrupt Controller مش مـ configure | `genirq: Flags mismatch` أو `No irq handler for vector` |
| Power issue (glitch) | IRQ spurious متفرق بدون pattern |
| الـ device مش بيـ clear الـ interrupt | الـ handler بيتنادى لانهاية، system يـ hang |

#### 5. Device Tree Debugging

```bash
# شوف الـ IRQ specs في الـ DT المُحمَّل
cat /proc/device-tree/soc/interrupt-controller@8000000/compatible
cat /proc/device-tree/soc/ethernet@1000000/interrupts | xxd

# استخدم dtc لعرض الـ DT الحالي
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "interrupt"

# تحقق من الـ IRQ domain المرتبط بكل device
ls /sys/kernel/debug/irq/*/  | head -30

# fdtdump لملف DTB
fdtdump /boot/dtb-$(uname -r) 2>/dev/null | grep -B2 -A5 "interrupt"

# تحقق إن الـ IRQ number في الـ DT صح
# مثلاً: interrupts = <GIC_SPI 27 IRQ_TYPE_LEVEL_HIGH>
# ده بيعني: SPI interrupt, HW IRQ 27, level-high
# في /proc/interrupts هيظهر كـ IRQ = 27 + GIC_SPI_OFFSET

# فحص الـ interrupt-parent
cat /proc/device-tree/soc/ethernet@1000000/interrupt-parent | xxd
# قارن مع phandle الـ GIC
cat /proc/device-tree/soc/interrupt-controller@8000000/phandle | xxd

# تحقق من الـ interrupt-cells
cat /proc/device-tree/soc/interrupt-controller@8000000/#interrupt-cells | xxd
```

---

### Practical Commands — جاهزة للـ Copy

#### مجموعة أوامر سريعة للتشخيص الأولي

```bash
#!/bin/bash
# IRQ Quick Diagnostic Script

TARGET_IRQ=${1:-27}

echo "=== IRQ $TARGET_IRQ Quick Diagnosis ==="

echo -e "\n[1] Interrupt Count per CPU:"
grep "^ *$TARGET_IRQ:" /proc/interrupts

echo -e "\n[2] IRQ Handler (flow handler):"
cat /sys/kernel/debug/irq/$TARGET_IRQ/handler 2>/dev/null || echo "debugfs not available"

echo -e "\n[3] Registered Actions:"
cat /sys/kernel/debug/irq/$TARGET_IRQ/actions 2>/dev/null

echo -e "\n[4] Spurious Stats:"
cat /sys/kernel/debug/irq/$TARGET_IRQ/spurious 2>/dev/null

echo -e "\n[5] Chip Info:"
cat /sys/kernel/debug/irq/$TARGET_IRQ/chip 2>/dev/null

echo -e "\n[6] Affinity:"
echo "  Requested: $(cat /proc/irq/$TARGET_IRQ/smp_affinity_list 2>/dev/null)"
echo "  Effective: $(cat /proc/irq/$TARGET_IRQ/effective_affinity_list 2>/dev/null)"

echo -e "\n[7] IRQ Type:"
cat /sys/kernel/irq/$TARGET_IRQ/type 2>/dev/null

echo -e "\n[8] Recent kernel messages for this IRQ:"
dmesg | grep -E "irq $TARGET_IRQ|IRQ $TARGET_IRQ" | tail -10
```

#### تتبع الـ IRQ handler بـ ftrace

```bash
#!/bin/bash
# Trace specific IRQ handler execution

TARGET_IRQ=${1:-27}
DURATION=${2:-5}

TRACEDIR=/sys/kernel/debug/tracing

# Reset
echo nop > $TRACEDIR/current_tracer
echo 0 > $TRACEDIR/tracing_on
echo > $TRACEDIR/trace

# Set filter on IRQ number
echo "irq == $TARGET_IRQ" > $TRACEDIR/events/irq/irq_handler_entry/filter
echo "irq == $TARGET_IRQ" > $TRACEDIR/events/irq/irq_handler_exit/filter

# Enable events
echo 1 > $TRACEDIR/events/irq/irq_handler_entry/enable
echo 1 > $TRACEDIR/events/irq/irq_handler_exit/enable

# Start tracing
echo 1 > $TRACEDIR/tracing_on
sleep $DURATION
echo 0 > $TRACEDIR/tracing_on

# Show results
cat $TRACEDIR/trace | grep -v "^#" | head -50

# Cleanup
echo 0 > $TRACEDIR/events/irq/irq_handler_entry/enable
echo 0 > $TRACEDIR/events/irq/irq_handler_exit/enable
```

**مثال مخرجات التتبع:**

```
# الـ output هيكون بالشكل ده:
        eth0-rx-2  [002] d..1  5678.123456: irq_handler_entry: irq=27 name=eth0
        eth0-rx-2  [002] d..1  5678.123489: irq_handler_exit:  irq=27 ret=handled

# التفسير:
# eth0-rx-2  = اسم الـ thread
# [002]      = CPU رقم 2
# d..1       = interrupt context flags
# 5678.12... = timestamp بالثواني
# ret=handled = Handler أرجع IRQ_HANDLED (صح)
# ret=unhandled = مشكلة! الـ handler مش بيتعرف على الـ interrupt
```

#### مراقبة الـ IRQ rate في real-time

```bash
#!/bin/bash
# Monitor IRQ rate (interrupts per second)

TARGET_IRQ=${1:-27}
INTERVAL=1

get_irq_count() {
    awk -v irq="$TARGET_IRQ:" '$1==irq {sum=0; for(i=2;i<=NF-2;i++) sum+=$i; print sum}' /proc/interrupts
}

prev=$(get_irq_count)
while true; do
    sleep $INTERVAL
    curr=$(get_irq_count)
    rate=$((curr - prev))
    echo "$(date +%H:%M:%S) IRQ $TARGET_IRQ rate: $rate/s (total: $curr)"
    prev=$curr
done
```

#### فحص الـ spurious IRQs

```bash
#!/bin/bash
# Detect spurious IRQs across all interrupts

echo "=== Spurious IRQ Report ==="
for irq_dir in /sys/kernel/debug/irq/*/; do
    irq=$(basename $irq_dir)
    spurious_file="$irq_dir/spurious"
    if [ -f "$spurious_file" ]; then
        unhandled=$(grep irqs_unhandled $spurious_file | awk '{print $2}')
        if [ "$unhandled" -gt 0 ] 2>/dev/null; then
            echo "IRQ $irq: $unhandled unhandled interrupts"
            cat $spurious_file
            echo "  Handler: $(cat $irq_dir/handler 2>/dev/null)"
            echo "  Actions: $(cat $irq_dir/actions 2>/dev/null)"
            echo "---"
        fi
    fi
done
```

**مثال مخرجات:**

```
=== Spurious IRQ Report ===
IRQ 45: 12 unhandled interrupts
irq_count:        1500
irqs_unhandled:     12
irq_threshold:       0
  Handler: handle_fasteoi_irq
  Actions: [0:my_device]
---

# التفسير:
# 12 spurious من 1500 = 0.8% — مقبول
# لو الـ irqs_unhandled > 99999 → الـ kernel هيعطل الـ IRQ تلقائياً
# الحل: تحقق من الـ driver يرجع IRQ_NONE بدل IRQ_HANDLED
#        أو الـ device مش بيـ clear الـ interrupt flag صح
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — IRQ Flood يوقف الـ UART

#### العنوان
**Runaway UART IRQ يعطّل الـ gateway بالكامل**

#### السياق
شركة بتبني industrial gateway بـ **RK3562** لجمع بيانات من sensors عبر UART. المنتج بيشتغل في بيئة مصنع فيها noise كتير على الـ RS-485 lines.

#### المشكلة
بعد ساعات من الشغل، الـ gateway بيبقى unresponsive. الـ CPU usage بيوصل 100% وكل الـ cores بتبقى stuck في interrupt context. الـ `/proc/interrupts` بيظهر عداد UART IRQ بيزيد بالملايين في الثانية.

#### التحليل
الـ **`irq_flow_handler_t`** هي typedef بسيطة:

```c
typedef void (*irq_flow_handler_t)(struct irq_desc *desc);
```

الـ flow handler المسجّلة على الـ UART IRQ line — غالباً `handle_level_irq` — بتتسمى بشكل متواصل لأن:

1. الـ hardware بيرفع الـ interrupt line (level-triggered).
2. الـ flow handler بتتسمى مع الـ `irq_desc` بتاع الـ UART.
3. الـ handler بتستدعي الـ action handler اللي المفروض يـ clear الـ interrupt في الـ UART FIFO.
4. بسبب الـ noise، الـ RX line مش بترجع low، فالـ interrupt بيفضل active.
5. الـ kernel بيلف في loop: `irq_flow_handler_t` → action → check → ما اتmaskش → loop تاني.

لو الـ flow handler كانت `handle_edge_irq` بدل `handle_level_irq`، كان المشكلة هتختلف — بس هنا الـ DT غلط بيحدد `IRQ_TYPE_LEVEL_HIGH` وده بيتوافق مع noise مستمر.

#### الحل
**1. فحص الـ DT binding:**

```bash
# شوف إيه الـ trigger type المستخدم
cat /proc/device-tree/serial@feba0000/interrupts | xxd
```

**2. تغيير الـ interrupt type في الـ DT:**

```dts
/* قبل */
interrupts = <GIC_SPI 35 IRQ_TYPE_LEVEL_HIGH>;

/* بعد — لو الـ IP بيدعم edge */
interrupts = <GIC_SPI 35 IRQ_TYPE_EDGE_RISING>;
```

**3. لو لازم يفضل level-triggered، نضيف debounce على الـ hardware وnoise filter:**

```bash
# تحقق إن الـ IRQ مش stuck بـ
watch -n 1 'cat /proc/interrupts | grep uart'
```

**4. في worst case، نـ mask الـ IRQ مؤقتاً:**

```c
/* في الـ driver */
disable_irq(irq);
/* process backlog */
enable_irq(irq);
```

#### الدرس المستفاد
الـ `irq_flow_handler_t` هي الـ entry point للـ IRQ flow logic. اختيار الـ flow handler (level vs edge) لازم يتطابق مع طبيعة الـ hardware signal فعلياً، مش بس مع الـ spec. في بيئات noisy، الـ edge-triggered أأمن من level-triggered.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI HPD IRQ مش بيشتغل

#### العنوان
**Hot-Plug Detection للـ HDMI مش بيتفعّل بعد تركيب custom kernel**

#### السياق
فريق بيعمل Android TV box بـ **Allwinner H616**. بعد upgrade الـ kernel من 5.15 لـ 6.6، الـ HDMI hot-plug detection بطل يشتغل — الشاشة بتتعرف بس لو عملت reboot كامل.

#### المشكلة
الـ HPD (Hot-Plug Detect) interrupt مش بيشتغل dynamically. الـ `/proc/interrupts` بيورّي إن الـ IRQ line مسجّل بس العداد صفر حتى لو انت بتوصّل وتفصل الكابل.

#### التحليل
الـ `irq_flow_handler_t` المسجّلة على الـ HPD line بتتحدد من خلال `irq_set_handler()` في الـ driver أو IRQ chip setup.

```c
/* الـ typedef اللي بيتبنى عليها كل ده */
typedef void (*irq_flow_handler_t)(struct irq_desc *desc);
```

المشكلة إن في الـ kernel الجديد، الـ sun8i HDMI driver اتغيّر فيه كيف بيسجّل الـ IRQ chip. الـ `irq_set_chained_handler_and_data()` بقت بتتسمى بدل `irq_set_handler()`، وده غيّر الـ flow handler المستخدمة.

الـ chained handler بيتوقع إن الـ parent IRQ controller يـ demultiplex الـ child IRQs — بس الـ H616 HPD line مش chained، هي dedicated line. فالـ flow handler الجديدة مش بتـ invoke الـ action handlers الصح.

#### الحل
**1. تتبع الـ handler المسجّل:**

```bash
# شوف الـ IRQ descriptor details
cat /sys/kernel/debug/irq/irqs/<irq_num>/actions
cat /sys/kernel/debug/irq/irqs/<irq_num>/handler
```

**2. في الـ driver، التأكد من استخدام الـ flow handler الصح:**

```c
/* للـ dedicated HPD line — مش chained */
irq_set_handler(irq, handle_edge_irq);  /* مش irq_set_chained_handler */

/* وتسجيل الـ action handler عادي */
request_irq(irq, hdmi_hpd_irq_handler, IRQF_TRIGGER_BOTH,
            "hdmi-hpd", hdmi);
```

**3. تحقق من الـ IRQ trigger type في الـ DT:**

```dts
hdmi_hpd_irq: interrupt-controller {
    interrupts = <GIC_SPI 58 IRQ_TYPE_EDGE_BOTH>;
};
```

#### الدرس المستفاد
الفرق بين `irq_set_handler()` و `irq_set_chained_handler_and_data()` جوهري. كلاهما بيستخدم `irq_flow_handler_t` كـ type، بس السياق مختلف تماماً. الـ chained handler بيبلع الـ interrupt ومش بيـ pass لـ action handlers عادية.

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — I2C IRQ بيطلع في الـ wrong context

#### العنوان
**`BUG: scheduling while atomic` في I2C interrupt handler**

#### السياق
فريق بيبني IoT environmental sensor بـ **STM32MP1**. الـ board فيها I2C sensor بيشتغل بـ interrupt-driven mode. في الـ production، kernel بيطلع panic مع message: `BUG: scheduling while atomic`.

#### المشكلة
الـ I2C IRQ handler بيحاول يعمل `mutex_lock()` — وده ممنوع في interrupt context. الـ crash بيحصل randomly تحت load.

#### التحليل
الـ **`irq_flow_handler_t`** بتتسمى في hardirq context:

```c
typedef void (*irq_flow_handler_t)(struct irq_desc *desc);
```

الـ flow handler (مثلاً `handle_fasteoi_irq`) بتشغّل الـ action handler (`irq_thread_fn` أو `handler_fn`) مباشرةً في interrupt context إلا لو الـ IRQ مسجّل بـ `IRQF_THREAD`.

في الكود ده، الـ developer نسي يحط `IRQF_ONESHOT | IRQF_THREAD` لما سجّل الـ interrupt:

```c
/* غلط — handler بيشتغل في hardirq context */
request_irq(irq, stm32_i2c_isr, 0, "stm32-i2c", i2c_dev);

/* صح — handler بيشتغل في kernel thread */
request_threaded_irq(irq, stm32_i2c_isr_quick, stm32_i2c_isr_thread,
                     IRQF_ONESHOT, "stm32-i2c", i2c_dev);
```

الـ flow handler نفسها مش المشكلة — هي بتعمل شغلها صح. المشكلة إن الـ action handler اتكتبت على افتراض إنها هتشتغل في thread context.

#### الحل
**1. تحويل لـ threaded IRQ:**

```c
static irqreturn_t stm32_i2c_isr_quick(int irq, void *dev_id)
{
    /* سريع جداً — بس check وreturn IRQ_WAKE_THREAD */
    struct stm32_i2c *i2c = dev_id;
    if (!(readl(i2c->base + I2C_ISR) & I2C_ISR_MASK))
        return IRQ_NONE;
    return IRQ_WAKE_THREAD;
}

static irqreturn_t stm32_i2c_isr_thread(int irq, void *dev_id)
{
    /* هنا تقدر تعمل mutex_lock بدون مشكلة */
    struct stm32_i2c *i2c = dev_id;
    mutex_lock(&i2c->lock);
    /* ... process data ... */
    mutex_unlock(&i2c->lock);
    return IRQ_HANDLED;
}
```

**2. تحقق من الـ context في الـ handler:**

```c
/* للـ debugging */
WARN_ON(in_interrupt());
```

#### الدرس المستفاد
الـ `irq_flow_handler_t` وفلسفة الـ flow handlers في الـ kernel مبنية على إن الـ action handler ممكن تشتغل في hardirq context. لو الـ handler محتاجة sleeping operations، لازم تستخدم `request_threaded_irq()` وده بيحول الـ action handler لـ kernel thread منفصل.

---

### السيناريو 4: Automotive ECU على i.MX8 — SPI IRQ بيتـ share بشكل غلط

#### العنوان
**IRQ sharing بين SPI controller وسنسور خارجي بيسبب data corruption**

#### السياق
فريق automotive بيبني ECU بـ **i.MX8MP** للتحكم في نظام الفرامل. الـ SPI controller والـ external safety sensor بيـ share نفس الـ IRQ line (بسبب قيود الـ hardware).

#### المشكلة
بيحصل data corruption في الـ SPI transfers بشكل عشوائي. الـ logs بتظهر إن الـ SPI transaction بتتقطع في النص.

#### التحليل
الـ `irq_flow_handler_t` المستخدمة هنا هي `handle_fasteoi_irq` أو `handle_level_irq`. لما الـ shared IRQ بيتفعّل:

```c
typedef void (*irq_flow_handler_t)(struct irq_desc *desc);
```

الـ flow handler بتلف على كل الـ action handlers المسجّلة على الـ IRQ line دي (لأنه shared). كل handler بترجع `IRQ_NONE` أو `IRQ_HANDLED`.

المشكلة: الـ safety sensor handler بتـ access الـ SPI bus عشان تقرأ الـ sensor data — وده بيحصل في نفس الوقت اللي الـ SPI controller handler شغّالة transaction تاني. ما فيش mutual exclusion بين الـ two handlers لأن كلاهم في interrupt context.

```c
/* الـ flow handler بتعمل كده داخلياً لـ shared IRQs */
list_for_each_entry(action, &desc->action, list) {
    res = action->handler(irq, action->dev_id);  /* الاتنين بيتسموا */
    if (res == IRQ_HANDLED)
        break;
}
```

#### الحل
**1. فصل الـ IRQ lines على مستوى الـ hardware لو ممكن (الحل المثالي).**

**2. لو الـ sharing إجباري، نستخدم spinlock:**

```c
static DEFINE_SPINLOCK(spi_irq_lock);

static irqreturn_t imx8_spi_isr(int irq, void *dev_id)
{
    unsigned long flags;
    spin_lock_irqsave(&spi_irq_lock, flags);
    /* ... handle SPI ... */
    spin_unlock_irqrestore(&spi_irq_lock, flags);
    return IRQ_HANDLED;
}

static irqreturn_t safety_sensor_isr(int irq, void *dev_id)
{
    unsigned long flags;
    spin_lock_irqsave(&spi_irq_lock, flags);
    /* ... read sensor via SPI ... */
    spin_unlock_irqrestore(&spi_irq_lock, flags);
    return IRQ_HANDLED;
}
```

**3. تسجيل الاتنين بـ `IRQF_SHARED`:**

```c
request_irq(irq, imx8_spi_isr, IRQF_SHARED, "imx8-spi", spi_dev);
request_irq(irq, safety_sensor_isr, IRQF_SHARED, "safety-sensor", sensor);
```

#### الدرس المستفاد
الـ `irq_flow_handler_t` في حالة الـ shared IRQs بتستدعي كل الـ action handlers بالترتيب. الـ developers لازم يفهموا إن الـ handlers دي ممكن تتعارض مع بعض. في السياق الـ automotive، ده critical بشكل خاص لأن data corruption ممكن يعني failure في safety systems.

---

### السيناريو 5: Custom Board Bring-Up على AM62x — IRQ مش بيوصل للـ driver خالص

#### العنوان
**IRQ controller جديد على AM62x مش بيوصّل الـ interrupts للـ drivers**

#### السياق
مهندس BSP بيعمل bring-up لـ custom board بـ **TI AM62x** فيها custom FPGA متوصّل بـ GPIO interrupt line. الـ FPGA عنده 8 internal interrupt sources محتاجة تتـ demultiplex.

#### المشكلة
الـ drivers المتوصّلة بالـ FPGA مش بتستلم أي interrupts. `/proc/interrupts` بيظهر الـ parent GPIO IRQ بيزيد، بس الـ child IRQs عداداتها صفر.

#### التحليل
المشكلة في الـ custom IRQ chip implementation. الـ FPGA driver محتاج يعمل IRQ domain ويسجّل `irq_flow_handler_t` مناسبة لكل child IRQ.

```c
/* الـ typedef المحورية */
typedef void (*irq_flow_handler_t)(struct irq_desc *desc);
```

الـ flow handler للـ parent GPIO IRQ (الـ chained handler) لازم تعمل الآتي:
1. تقرأ الـ FPGA interrupt status register.
2. تحدد أنهي child IRQ اتفعّل.
3. تستدعي `generic_handle_irq()` للـ child IRQ المناسب.

المشكلة إن الـ developer كتب الـ chained handler غلط — بيستدعي `handle_nested_irq()` بدل `generic_handle_irq()`:

```c
/* غلط */
static void fpga_irq_handler(struct irq_desc *desc)
{
    /* ... read status ... */
    handle_nested_irq(child_irq);  /* ده للـ threaded IRQs فقط */
}

/* صح */
static void fpga_irq_handler(struct irq_desc *desc)
{
    struct irq_chip *chip = irq_desc_get_chip(desc);
    u32 status;

    chained_irq_enter(chip, desc);  /* mask parent، save state */

    status = readl(fpga->base + FPGA_IRQ_STATUS);
    while (status) {
        int bit = __ffs(status);
        generic_handle_irq(irq_find_mapping(fpga->domain, bit));
        status &= ~BIT(bit);
    }

    chained_irq_exit(chip, desc);  /* unmask parent */
}
```

**تسجيل الـ chained handler:**

```c
/* بيستخدم irq_flow_handler_t داخلياً */
irq_set_chained_handler_and_data(gpio_irq, fpga_irq_handler, fpga);
```

#### الحل
**1. استبدال `handle_nested_irq` بـ `generic_handle_irq` مع `chained_irq_enter/exit`.**

**2. التأكد من الـ IRQ domain setup:**

```c
fpga->domain = irq_domain_add_linear(np, FPGA_NUM_IRQS,
                                      &fpga_irq_domain_ops, fpga);
```

**3. Debug بـ:**

```bash
# شوف الـ IRQ domain
cat /sys/kernel/debug/irq/domains/
# شوف الـ mapping
cat /sys/kernel/debug/irq/domains/fpga-irq/
```

**4. تفعيل IRQ debugging في الـ kernel config:**

```bash
CONFIG_GENERIC_IRQ_DEBUGFS=y
CONFIG_IRQ_DOMAIN_DEBUG=y
```

#### الدرس المستفاد
الـ `irq_flow_handler_t` في الـ chained IRQ scenario هي المفتاح لـ demultiplexing. الفرق بين `handle_nested_irq` و `generic_handle_irq` مش cosmetic — الأول للـ threaded IRQ emulation والتاني للـ real hardware IRQ cascading. اختيار الغلط بيخلي الـ child interrupts ما توصلش للـ drivers خالص، وده بيضيّع وقت طويل في الـ bring-up.
## Phase 7: مصادر ومراجع

الـ `irqhandler.h` ملف صغير جداً — سطرين فعليين — لكن اللي بيعرّفه (`irq_flow_handler_t`) هو حجر الأساس في كل subsystem الـ generic IRQ في الكيرنل. المصادر دي بتغطي السياق الكامل من تصميم الـ subsystem لحد التطبيق العملي.

---

### مقالات LWN.net

| المقال | الأهمية |
|--------|----------|
| [A new generic IRQ layer](https://lwn.net/Articles/184750/) | الأهم — بيشرح إزاي Thomas Gleixner وIngo Molnar صمموا الـ generic IRQ layer اللي أدخل مفهوم `irq_flow_handler_t` |
| [generic irq subsystem, x64 port](https://lwn.net/Articles/104847/) | بيتكلم عن porting الـ generic IRQ subsystem على x86-64 |
| [Moving interrupts to threads](https://lwn.net/Articles/302043/) | بيشرح إزاي الـ threaded IRQs اشتغلت مع الـ flow handler model |
| [IRQ threads](https://lwn.net/Articles/95387/) | نقاش مبكر عن ideaالـ IRQ threading |
| [Improving lost and spurious IRQ handling](https://lwn.net/Articles/392136/) | بيتناول edge cases في الـ flow handlers — spurious وlost interrupts |
| [Handling interrupts in user space](https://lwn.net/Articles/127698/) | زاوية مختلفة: إزاي الـ UIO framework بيستخدم نفس الـ flow abstraction |

---

### التوثيق الرسمي للكيرنل

**الأهم على الإطلاق** — التوثيق الرسمي للـ generic IRQ subsystem:

- **[Linux generic IRQ handling — kernel.org](https://docs.kernel.org/core-api/genericirq.html)**
  ده الـ canonical reference. بيشرح `irq_flow_handler_t`، الـ flow handlers المدمجة (`handle_level_irq`، `handle_edge_irq`، إلخ)، وكيفية تعيينها.

- **`Documentation/core-api/genericirq.rst`** — نفس المحتوى في الكيرنل source مباشرة.

- **`kernel/irq/`** — المجلد اللي بيحتوي كل implementation:
  - `kernel/irq/chip.c` — الـ chip-level operations
  - `kernel/irq/handle.c` — الـ flow handler dispatch
  - `kernel/irq/manage.c` — `request_irq` وأخواتها
  - `kernel/irq/internals.h` — الـ internal structures

- **`include/linux/irq.h`** — الهيدر الرئيسي اللي بيستخدم `irq_flow_handler_t` المعرّف في `irqhandler.h`.

- **`include/linux/irqdesc.h`** — تعريف `struct irq_desc` كاملاً — الـ struct اللي بتاخده الـ `irq_flow_handler_t` كـ argument.

---

### Commits مهمة في تاريخ الكيرنل

الـ commits دي هي اللي أسست الـ model الحالي:

- **الـ genirq patchset الأصلي** (كيرنل 2.6.17-2.6.18) — Thomas Gleixner وIngo Molnar، اللي فيه تم فصل الـ flow handler عن الـ chip-level operations وظهر نوع `irq_flow_handler_t` لأول مرة.

- للبحث في تاريخ الملف:
  ```bash
  git log --follow --oneline include/linux/irqhandler.h
  git log --follow --oneline include/linux/irq.h | head -20
  ```

- الـ commit اللي فصل `irqhandler.h` كـ header مستقل (لكسر الـ circular includes) موجود في تاريخ الكيرنل بـ message قريب من "irq: avoid circular include dependencies".

---

### نقاشات Mailing List

- **[Threaded IRQ handler question — LKML](https://lists.gt.net/linux/kernel/1216325)** — نقاش عملي عن الـ flow handlers مع الـ threaded IRQs وعلاقة `IRQ_WAKE_THREAD` بالـ flow.

- **[Question pertaining to request_threaded_irq — LKML](https://linux-kernel.vger.kernel.narkive.com/1GSN2fNl/question-pertaining-to-request-threaded-irq)** — بيوضح إزاي الـ flow handler بيتعامل مع الـ oneshot semantics.

- **[LKML.ORG Archive](https://lkml.org/)** — للبحث عن `irq_flow_handler_t` أو `handle_irq` بالـ keywords دي مباشرة.

- **[The Great IRQ Debate — linux.com](https://www.linux.com/news/great-irq-debate-linux-kernel/)** — ملخص للنقاشات التاريخية حول تصميم الـ IRQ subsystem.

---

### كتب مُوصى بيها

#### Linux Device Drivers 3 (LDD3)
- **الفصل 10: Interrupt Handling** — الأكثر صلة مباشرة.
  - [PDF رسمي من LWN](https://static.lwn.net/images/pdf/LDD3/ch10.pdf)
  - بيشرح `request_irq()`، الـ handler prototype، وفلسفة الـ top-half/bottom-half.
- ملاحظة: LDD3 قديم (كيرنل 2.6.10) — الـ `irq_flow_handler_t` نفسه جه بعده، لكن المفاهيم الأساسية ثابتة.

#### Linux Kernel Development — Robert Love (الطبعة 3)
- **الفصل 7: Interrupts and Interrupt Handlers** — بيغطي:
  - تسجيل الـ handlers
  - الـ interrupt context وقيوده
  - الـ bottom halves (softirqs، tasklets، workqueues)
- أحدث من LDD3 وبيغطي الـ flow handler model.

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 14: Kernel Debugging Techniques** + أجزاء من الفصل 8 — بيتكلم عن الـ IRQ handling في السياق الـ embedded، مفيد جداً لفهم إزاي الـ `irq_flow_handler_t` بيتعمل assign على ARM وPowerPC.

---

### KernelNewbies.org

- **[Internals of Interrupt Handling](https://kernelnewbies.org/KernelHacking-HOWTO/Overview_of_the_Kernel_Source_Code/Internals_of_Interrupt_Handling)** — walkthrough للـ interrupt path من أول ما الـ CPU يستقبل الـ signal لحد ما يتم استدعاء الـ flow handler.

- **[Details of do_IRQ() function](https://kernelnewbies.org/KernelHacking-HOWTO/Overview_of_the_Kernel_Source_Code/Internals_of_Interrupt_Handling/Details_of_do_IRQ()_function)** — بيشرح الـ old `__do_IRQ()` super-handler اللي اتستبدل بالـ per-descriptor flow handler model الحالي.

- **[Exceptions and Interrupts Handling](https://kernelnewbies.org/New_Kernel_Hacking_HOWTO/Subsystems/Exceptions_and_Interrupts_Handling)** — أحدث — من الـ New Kernel Hacking HOWTO.

---

### eLinux.org

- **[Soft IRQ Threads](https://elinux.org/Soft_IRQ_Threads)** — بيتكلم عن تشغيل الـ softirq bottom-halves في kernel threads — مرتبط بمفهوم الـ flow handler في سياق الـ real-time systems.

---

### مصادر إضافية

- **[Linux Interrupts: The Basic Concepts — embetronicx.com](https://embetronicx.com/tutorials/linux/device-drivers/linux-device-driver-tutorial-part-13-interrupt-example-program-in-linux-kernel/)** — مثال عملي كامل بكود يشرح `request_irq()` والـ handler.

- **[Linux Insides — Interrupts Chapter 10](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-10.html)** — تحليل عميق للـ interrupt path على مستوى الـ assembly والـ C code.

- **[TLDP: Interrupt Handlers](https://tldp.org/LDP/lkmpg/2.6/html/x1256.html)** — مرجع قديم لكن مفيد للمفاهيم الأساسية.

---

### مصطلحات البحث الموصى بيها

للبحث عن معلومات أكتر:

```
irq_flow_handler_t linux kernel
irq_desc handle_irq linux
generic IRQ subsystem linux Thomas Gleixner
handle_level_irq handle_edge_irq linux kernel
request_irq linux kernel driver
IRQF_SHARED IRQF_ONESHOT linux
linux kernel interrupt bottom half softirq tasklet
threaded IRQ linux kernel request_threaded_irq
linux IRQ chip descriptor irq_chip
```
## Phase 8: Writing simple module

### الفكرة

الـ `generic_handle_irq` هي function مُصدَّرة بـ `EXPORT_SYMBOL` بتأخد رقم الـ IRQ وتشغّل الـ flow handler المرتبط بيه. هنعمل **kprobe** عليها عشان نطبع رقم الـ IRQ كل ما حد يستدعيها — ده بيخلّينا نشوف أي interrupts بتتعامل معاها عبر المسار ده في real-time من غير ما نلمس أي interrupt handler حقيقي.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_generic_handle_irq.c
 *
 * Hooks generic_handle_irq() via kprobe and logs the IRQ number
 * each time it is called. Useful for observing which IRQs pass
 * through the generic flow-handler dispatch path.
 */

#include <linux/kernel.h>       /* pr_info, pr_err                  */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit  */
#include <linux/kprobes.h>      /* kprobe, register_kprobe, ...      */
#include <linux/irqdesc.h>      /* irq_desc, irq_desc_get_irq        */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe on generic_handle_irq() to log IRQ numbers");

/* ------------------------------------------------------------------ */
/* pre-handler: يتنفذ قبل ما generic_handle_irq تبدأ                  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument (unsigned int irq) lives in
     * the di register.  On ARM64 it lives in regs->regs[0].
     * pt_regs_param0() is a portable helper that picks the right
     * register automatically.
     */
    unsigned int irq = (unsigned int)regs->di; /* x86-64 first arg */

    pr_info("kprobe: generic_handle_irq() called with irq=%u\n", irq);

    return 0; /* 0 = continue normal execution */
}

/* ------------------------------------------------------------------ */
/* post-handler: يتنفذ بعد ما الـ instruction الـ probed تخلص         */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* نطبع رسالة بسيطة تأكيد إن الـ function خلصت                    */
    pr_info("kprobe: generic_handle_irq() returned\n");
}

/* ------------------------------------------------------------------ */
/* تعريف الـ kprobe struct                                              */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "generic_handle_irq",  /* الاسم اللي هيتحلّه        */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
static int __init kp_irq_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe: register_kprobe failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("kprobe: planted on generic_handle_irq at %p\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit kp_irq_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe: removed from generic_handle_irq\n");
}

module_init(kp_irq_init);
module_exit(kp_irq_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل الـ API بتاعت التسجيل والإلغاء |
| `linux/irqdesc.h` | بيعرّف `struct irq_desc` اللي بنستخدمه لو احتجنا نجيب بيانات إضافية من الـ descriptor |
| `linux/module.h` | لازم لأي kernel module — بيعرّف `module_init/exit` والـ macros |
| `linux/kernel.h` | بيديك `pr_info` و `pr_err` للـ logging |

---

#### الـ `handler_pre`

الـ `pre_handler` بيتنفذ **قبل** الـ instruction الـ probed. بييجيله `pt_regs` اللي فيه حالة الـ CPU registers في لحظة الـ probe — ومنه بنجيب أول argument للـ function (الـ `irq` number) من `regs->di` على x86-64. لو الكود محتاج يشتغل على ARM64 يبدّل ده بـ `regs->regs[0]`.

---

#### الـ `handler_post`

الـ `post_handler` بيتنفذ **بعد** ما الـ probed instruction تخلص. هنا بنطبع تأكيد إن الـ function خلصت. ده مفيد في الـ debugging لو عايز تقيس timing أو تتأكد إن الـ return سليم.

---

#### الـ `struct kprobe`

- **`symbol_name`**: الـ kernel بيحلّ الاسم ده لـ address وقت الـ `register_kprobe` — مش لازم تعرف الـ address يدوياً.
- **`pre_handler` / `post_handler`**: الـ callbacks اللي اتشرحوا فوق.

---

#### الـ `module_init` — `kp_irq_init`

بيسجّل الـ kprobe. لو فشل (مثلاً الـ symbol مش موجود أو الـ kprobes مش enabled في الـ kernel config) بيرجع الـ error code ومش بيكمل.

---

#### الـ `module_exit` — `kp_irq_exit`

**لازم** تعمل `unregister_kprobe` في الـ exit عشان الـ kernel ميستدعيش callback موجود في module اتفك تحميله — ده كان هيعمل kernel panic فوري.

---

### Makefile للتجربة

```makefile
obj-m += kprobe_generic_handle_irq.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# تحميل الـ module
sudo insmod kprobe_generic_handle_irq.ko

# مراقبة الـ output
sudo dmesg -w | grep kprobe

# إزالة الـ module
sudo rmmod kprobe_generic_handle_irq
```

---

### ملاحظة على الـ portability

الـ `regs->di` خاص بـ x86-64. لو الكود هيشتغل على ARM64 استخدم:

```c
unsigned int irq = (unsigned int)regs->regs[0];
```

أو استخدم الـ macro `regs_get_kernel_argument(regs, 0)` اللي متاح في بعض kernel versions وبيعمل abstraction على الـ ABI.
