## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبع له الملف

الملف `include/linux/irqreturn.h` جزء من **IRQ Subsystem** في Linux kernel، المسؤول عنه Thomas Gleixner. ده الـ subsystem اللي بيتحكم في كل حاجة بتخص **Interrupts** — من لحظة ما الـ hardware بيبعت إشارة للـ CPU لحد ما الـ driver بيعمل معالجة للحدث.

---

### القصة من الأول: ليه محتاجين IRQ أصلاً؟

تخيل إنك شغال على كمبيوتر وعايز تعرف امتى الكيبورد اتضغط عليه زر. عندك طريقتين:

**الطريقة الغلط (Polling):** الـ CPU يسأل الكيبورد كل ميللي ثانية "في حاجة جديدة؟" — ده هيضيع وقت الـ CPU خالص حتى لو مفيش حاجة بتحصل.

**الطريقة الصح (Interrupt):** الكيبورد نفسه يصرخ للـ CPU "هوه! في زر اتضغط!" — الـ CPU يوقف شغله، يتعامل مع الكيبورد، وبعدين يرجع لشغله.

الصرخة دي اسمها **IRQ** (Interrupt Request). كل device في النظام — كيبورد، نت كارت، هارد، USB — عنده خط IRQ يقدر يبعت بيه إشارة للـ CPU.

---

### الـ Driver بيرد إيه بعد ما بيتعامل مع الـ Interrupt؟

لما الـ CPU يستقبل IRQ، الـ kernel بيسأل الـ driver "انت اللي بعت الإشارة دي؟ وعملت إيه؟"

الـ driver لازم يرد بواحدة من 3 إجابات — وده بالظبط اللي `irqreturn.h` بيعرّفه:

```c
enum irqreturn {
    IRQ_NONE        = (0 << 0),  // مش أنا اللي بعته، مش شغلتي
    IRQ_HANDLED     = (1 << 0),  // أيوه أنا عملته وخلصت
    IRQ_WAKE_THREAD = (1 << 1),  // أنا شفت الإشارة، بس المعالجة الحقيقية محتاجة thread تاني
};

typedef enum irqreturn irqreturn_t;
#define IRQ_RETVAL(x) ((x) ? IRQ_HANDLED : IRQ_NONE)
```

---

### تفاصيل كل قيمة

| القيمة | المعنى | امتى بتتبعت |
|--------|--------|------------|
| `IRQ_NONE` | الـ interrupt مش بتاعي | الـ driver اكتشف إن الإشارة جت من device تاني |
| `IRQ_HANDLED` | اتعمل المطلوب | الـ driver قرأ البيانات، مسح الـ flag، وخلص |
| `IRQ_WAKE_THREAD` | محتاج thread | الشغل التقيل (زي network packet processing) محتاج context مش interrupt context |

---

### ليه `IRQ_WAKE_THREAD` مهمة جداً؟

الـ interrupt handlers بتشتغل في **interrupt context** — محظور فيه النوم، المنتظرة، وأي حاجة بطيئة. لو الـ driver محتاج يعمل حاجة بطيئة (زي يقرأ من disk أو يبعت network packet)، بيقول للـ kernel "صحّي الـ threaded handler" — وده بيشتغل في process context عادي.

```
[ Hardware ] --IRQ--> [ CPU ] --يستدعي--> [ hard IRQ handler ]
                                                    |
                                         IRQ_WAKE_THREAD
                                                    |
                                         [ kernel thread ] (شغل تقيل هنا)
```

---

### ليه الملف ده صغير جداً بس مهم؟

`irqreturn.h` هو **contract** بين الـ kernel وكل driver في النظام. أي driver بيسجّل interrupt handler لازم ترجع دالته `irqreturn_t`. لو مفيش تعريف موحد للـ return type ده، كل driver هيعمل convention خاص بيه والـ kernel مش هيعرف يفهم الرد.

ده تعريف صغير بيخدم الآلاف من الـ drivers في الـ kernel.

---

### مثال عملي: Network Card Interrupt

```c
// مثال مبسط لـ interrupt handler في network driver
irqreturn_t my_nic_irq_handler(int irq, void *dev_id)
{
    struct net_device *dev = dev_id;

    // اتحقق إن الإشارة من الكارت ده فعلاً
    if (!my_nic_check_interrupt(dev))
        return IRQ_NONE;  // مش أنا، تجاهل

    // وصل باكيت — صحّي الـ NAPI thread يعالجه
    napi_schedule(&priv->napi);

    return IRQ_HANDLED;  // خلصت الجزء الأساسي
}
```

---

### الملفات المرتبطة اللي المفروض تعرفها

**ملفات الـ IRQ Subsystem الأساسية:**

| الملف | الدور |
|-------|-------|
| `include/linux/irqreturn.h` | تعريف `irqreturn_t` — الملف ده |
| `include/linux/irq.h` | تعريفات الـ `irq_desc`, `irqaction`, وكل الـ structs |
| `include/linux/irqhandler.h` | تعريف `irq_flow_handler_t` |
| `include/linux/interrupt.h` | `request_irq()`, `free_irq()`, `IRQF_*` flags |
| `kernel/irq/handle.c` | الكود الحقيقي اللي بيستدعي الـ handlers ويفحص الـ return value |
| `kernel/irq/manage.c` | تسجيل وإلغاء تسجيل الـ interrupt handlers |
| `kernel/irq/spurious.c` | كشف الـ spurious interrupts (اللي مرجعتش `IRQ_HANDLED`) |
| `kernel/irq/irqdesc.c` | إدارة الـ `irq_desc` structures |

**ملفات الـ IRQ Domains (للـ hardware mapping):**

| الملف | الدور |
|-------|-------|
| `include/linux/irqdomain.h` | mapping بين الـ hardware IRQ numbers والـ Linux IRQ numbers |
| `kernel/irq/irqdomain.c` | تنفيذ الـ IRQ domain |

**drivers الـ IRQ Controllers:**

| المسار | الدور |
|--------|-------|
| `drivers/irqchip/` | drivers للـ GIC (ARM), APIC (x86), وغيرهم |
| `include/linux/irqchip.h` | واجهة تسجيل الـ IRQ chip drivers |
## Phase 2: شرح الـ IRQ Handling Framework

### المشكلة — ليه الـ subsystem ده موجود؟

لما **hardware interrupt** بتحصل، الـ CPU بتوقف كل حاجة وبتجري على الـ **Interrupt Service Routine (ISR)**. المشكلة الأساسية: الـ kernel محتاج يعرف — بعد ما الـ handler خلص — هل فعلاً الجهاز ده هو اللي عمل الـ interrupt ولا لأ؟

في الأنظمة الحديثة، ممكن **أكتر من device** تشارك نفس الـ IRQ line (shared interrupts). الـ kernel بيكال على كل الـ handlers المسجلين على نفس الـ line واحد واحد، ومحتاج كل handler يقوله: "أيوه أنا عملتها"، أو "مش أنا"، أو "أنا شوفتها بس محتاج thread تاني يكمل الشغل".

من غير آلية return value موحدة:
- الـ kernel مش هيعرف يعمل **shared IRQ demultiplexing** صح
- هيفضل يستنى handlers تانية مش محتاجها
- الـ **threaded IRQ** mechanism مش هتشتغل

---

### الحل — الـ kernel بيعمل إيه؟

الـ kernel عرّف **type موحد** يرجعه كل interrupt handler:

```c
enum irqreturn {
    IRQ_NONE        = (0 << 0),  /* مش أنا اللي عمل الـ interrupt */
    IRQ_HANDLED     = (1 << 0),  /* أنا عملتها وخلصت */
    IRQ_WAKE_THREAD = (1 << 1),  /* أنا شوفتها، بس عايز thread تكمل */
};

typedef enum irqreturn irqreturn_t;
#define IRQ_RETVAL(x) ((x) ? IRQ_HANDLED : IRQ_NONE)
```

ده بسيط جداً في الكود، لكن وراه نظام كامل للـ **interrupt dispatching**.

---

### Big Picture Architecture

```
+---------------------------+
|        Hardware           |
|   [Device A] [Device B]   |
|      |           |        |
|      +-----+-----+        |
|            |              |
|        IRQ Line N         |
+------------|------------------+
             |
     +--------v--------+
     |   GIC / APIC    |  <-- الـ Interrupt Controller (مثلاً ARM GIC)
     | (irq_chip layer)|
     +--------+--------+
              |
     +--------v--------+
     |  irq_desc[N]    |  <-- الـ kernel's IRQ descriptor table
     |  (kernel core)  |
     +--------+--------+
              |
     +--------v-----------------------------------+
     |          IRQ Core Dispatcher               |
     |  handle_irq_event() / handle_fasteoi_irq() |
     +------+-------------------+----------------+
            |                   |
   +--------v------+   +--------v----------+
   | handler_A()   |   | handler_B()       |
   | returns:      |   | returns:          |
   | IRQ_NONE      |   | IRQ_HANDLED  OR   |
   |               |   | IRQ_WAKE_THREAD   |
   +---------------+   +-------------------+
                                |
                       +--------v----------+
                       |  IRQ Thread       |
                       |  (kthread)        |
                       |  thread_fn()      |
                       +-------------------+
```

---

### التشبيه من الواقع

تخيل **بواب عمارة** في عمارة فيها 10 شقق، وكل الشقق بيرن على نفس الجرس الرئيسي (shared IRQ line).

لما الجرس يرن:
- البواب بيدق على **كل شقة** واحدة واحدة (الـ kernel بيكال كل handler)
- كل ساكن بيرد: **"مش أنا" (IRQ_NONE)** أو **"أيوه أنا" (IRQ_HANDLED)** أو **"أيوه أنا، بس محتاج حد تاني يستلم الطرد" (IRQ_WAKE_THREAD)**

| العمارة | الـ Kernel |
|---------|-----------|
| الجرس الرئيسي | IRQ line مشتركة |
| البواب | IRQ core dispatcher |
| الشقق | الـ drivers المسجلين على الـ IRQ |
| رد الساكن | `irqreturn_t` |
| "مش أنا" | `IRQ_NONE` |
| "أيوه أنا" | `IRQ_HANDLED` |
| "أنا بس محتاج حد تاني" | `IRQ_WAKE_THREAD` — بيصحّي الـ IRQ thread |
| الموظف اللي بييجي يستلم الطرد | الـ `irq_thread` kthread |

الفرق الجوهري: لو **كل الشقق قالت "مش أنا"** — البواب هيعرف في الـ kernel إن ده **spurious interrupt** وبيعمل log ليه.

---

### الـ Core Abstraction

الـ central idea هنا هي **"handler contract"**: كل interrupt handler في الـ kernel **لازم** يوقّع على عقد — يقول بوضوح إيه اللي عمله. ده مش optional، ده **نظام**.

```c
/* الـ signature القياسية لكل interrupt handler */
irqreturn_t my_driver_isr(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;

    /* هل الـ interrupt من جهازنا؟ */
    if (!(readl(dev->base + STATUS_REG) & IRQ_PENDING))
        return IRQ_NONE;  /* مش أنا */

    /* أعمل الشغل السريع (clear interrupt, read data) */
    handle_fast_stuff(dev);

    return IRQ_HANDLED;  /* أنا عملتها */
}
```

أو في حالة الـ **threaded IRQ**:

```c
/* الـ top-half: سريع جداً */
irqreturn_t my_driver_isr(int irq, void *dev_id)
{
    /* disable الـ interrupt في الـ hardware بسرعة */
    disable_hw_irq(dev_id);
    return IRQ_WAKE_THREAD;  /* صحّي الـ thread */
}

/* الـ bottom-half: بتشتغل في context عادي */
irqreturn_t my_driver_thread_fn(int irq, void *dev_id)
{
    /* هنا ممكن تعمل حاجات تاخد وقت: I2C, SPI, mutex, etc. */
    process_data(dev_id);
    return IRQ_HANDLED;
}
```

---

### الـ IRQ_RETVAL Macro

```c
#define IRQ_RETVAL(x) ((x) ? IRQ_HANDLED : IRQ_NONE)
```

ده macro utility بسيط للـ drivers اللي عندها boolean condition:

```c
/* بدل ما تكتب */
if (handled)
    return IRQ_HANDLED;
else
    return IRQ_NONE;

/* تكتب */
return IRQ_RETVAL(handled);
```

---

### الـ Subsystem بيمتلك إيه vs بيفوّض إيه للـ drivers؟

| الـ IRQ Core يمتلك | الـ Driver يمتلك |
|-------------------|-----------------|
| تسجيل الـ handlers (irq_desc table) | منطق "هل الـ interrupt من جهازي؟" |
| ترتيب استدعاء الـ handlers | قراءة الـ hardware registers |
| قرار الـ spurious interrupt | تحديد إيه اللي يحصل في الـ thread |
| إدارة الـ IRQ thread (kthread) | متى يرجع `IRQ_WAKE_THREAD` |
| الـ IRQ affinity و CPU routing | الـ device-specific bottom-half logic |
| الـ IRQ statistics (`/proc/interrupts`) | اختيار الـ return value المناسب |

---

### Subsystems تانية محتاج تعرفها

- **الـ irq_chip layer**: الـ abstraction للـ interrupt controller hardware (GIC, APIC). بيتحكم في enable/disable/ack على مستوى الـ hardware. لازم تعرفه قبل ما تتعمق في `irq_desc`.
- **الـ softirq / tasklet / workqueue**: ده الـ "bottom-half" framework — البديل لـ `IRQ_WAKE_THREAD` في السياقات الأقدم. الـ threaded IRQ هو الطريقة الحديثة الموصى بيها.
- **الـ genirq**: الـ generic IRQ layer اللي بيجمع كل ده تحت سقف واحد (`kernel/irq/`).
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### ملاحظة على الملف

الملف `irqreturn.h` صغير جداً ومتخصص — بيحتوي على **enum واحد بس** (`irqreturn`) وـ `typedef` واحد وـ `macro` واحد. مفيش structs، مفيش locking، مفيش call graphs معقدة. الشرح هيكون cheatsheet-style مركّز على الـ enum والـ macro وسياق استخدامهم في الـ IRQ subsystem.

---

### Enum: `irqreturn` — الـ Flags الكاملة

| القيمة | الـ Value (binary) | المعنى |
|---|---|---|
| `IRQ_NONE` | `0b00` (0) | الـ interrupt مش من الجهاز ده أو ما اتعملشيء |
| `IRQ_HANDLED` | `0b01` (1) | الـ interrupt اتعالج بنجاح في الـ hardirq handler |
| `IRQ_WAKE_THREAD` | `0b10` (2) | الـ handler طلب إيقاظ الـ threaded IRQ handler |

> الـ values بيتم تعريفها كـ bit-shifts صريحة (`1 << 0`, `1 << 1`) عشان ممكن مستقبلاً يتضافوا أو يتوسعوا بـ bitmask جديد.

---

### الـ typedef والـ macro

```c
typedef enum irqreturn irqreturn_t;
#define IRQ_RETVAL(x)   ((x) ? IRQ_HANDLED : IRQ_NONE)
```

| الاسم | النوع | الغرض |
|---|---|---|
| `irqreturn_t` | `typedef` | الاسم المستخدم فعلياً في كل الكود — shorthand للـ enum |
| `IRQ_RETVAL(x)` | `macro` | يحوّل أي boolean expression لـ `irqreturn_t` — مفيد لو الـ handler بيرجع `int` |

**مثال عملي للـ `IRQ_RETVAL`:**

```c
/* بدل ما تكتب الـ ternary بنفسك */
return IRQ_RETVAL(handled);  /* نفس: return handled ? IRQ_HANDLED : IRQ_NONE; */
```

---

### مفيش Structs في الملف ده

الملف `irqreturn.h` مش بيعرّف أي struct — دوره الوحيد هو تعريف **نوع القيمة الراجعة** من الـ interrupt handler. الـ structs الأساسية في الـ IRQ subsystem موجودة في ملفات تانية:

| الـ Struct | الملف | العلاقة بـ `irqreturn_t` |
|---|---|---|
| `struct irq_desc` | `include/linux/irqdesc.h` | بيخزن pointer للـ handler اللي بيرجع `irqreturn_t` |
| `struct irqaction` | `include/linux/interrupt.h` | بيحتوي `irq_handler_t` اللي return type بتاعها `irqreturn_t` |
| `irq_handler_t` | `include/linux/interrupt.h` | `typedef irqreturn_t (*irq_handler_t)(int, void *)` |

---

### مكانة `irqreturn_t` في الـ IRQ Subsystem

```
┌─────────────────────────────────────────────────────┐
│                   struct irqaction                  │
│  ┌──────────────────────────────────────────────┐  │
│  │  irq_handler_t   handler  ──────────────────────┼──► يرجع irqreturn_t
│  │  irq_handler_t   thread_fn ────────────────────┼──► يرجع irqreturn_t
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
              ▲
              │ مربوط بـ
              ▼
┌─────────────────────────────────────────────────────┐
│                   struct irq_desc                   │
│   بيمثل الـ IRQ line كامل في النظام                 │
└─────────────────────────────────────────────────────┘
```

---

### Lifecycle Diagram — من الـ Interrupt للـ Return Value

```
Hardware Interrupt Fires
        │
        ▼
  CPU jumps to IDT entry
        │
        ▼
  kernel/irq/handle.c: handle_irq_event()
        │
        ▼
  handle_irq_event_percpu()
        │
        ├─► يلف على كل irqaction مسجّلة
        │
        ▼
  action->handler(irq, dev_id)
        │
        ├── returns IRQ_NONE        ──► kernel يجرّب الـ handler الجاي
        │
        ├── returns IRQ_HANDLED     ──► kernel يسجّل الـ interrupt كـ handled
        │                               ويخلص
        │
        └── returns IRQ_WAKE_THREAD ──► kernel يصحّى الـ thread_fn
                                        (threaded IRQ — يشتغل في process context)
```

---

### Call Flow — إزاي الـ Kernel يستخدم القيمة الراجعة

```
handle_irq_event_percpu()
  │
  ├─► action->handler(irq, dev_id)
  │       └─► returns irqreturn_t (res)
  │
  ├─► if (res == IRQ_WAKE_THREAD)
  │       └─► __irq_wake_thread(desc, action)
  │               └─► wake_up_process(action->thread)
  │
  └─► note_interrupt(irq, desc, res)
          └─► يحسب spurious interrupts
              لو IRQ_NONE كتير ► يعطّل الـ IRQ تلقائياً
```

---

### لا يوجد Locking في هذا الملف

الملف بيعرّف enum و typedef بس — مفيش shared state، مفيش لزوم لـ locking. الـ locking الخاص بالـ IRQ subsystem موجود في:

| الـ Lock | الموقع | بيحمي إيه |
|---|---|---|
| `desc->lock` (`raw_spinlock_t`) | `struct irq_desc` | الـ irqaction list وحالة الـ IRQ descriptor |
| `irq_controller_lock` | بعض الـ irqchips | الـ hardware register access |

---

### خلاصة — Cheatsheet سريع

```
irqreturn.h
├── enum irqreturn
│   ├── IRQ_NONE        = 0   → مش أنا اللي عمله أو ما عملتش حاجة
│   ├── IRQ_HANDLED     = 1   → خلصت وبخير
│   └── IRQ_WAKE_THREAD = 2   → صحّى الـ thread handler
│
├── typedef irqreturn_t       → الاسم المستخدم في كل الكود
│
└── macro IRQ_RETVAL(x)       → bool → irqreturn_t
```

**كل IRQ handler في الـ kernel** لازم يرجع `irqreturn_t` — ده الـ contract الأساسي بين الـ driver والـ kernel IRQ core.
## Phase 4: شرح الـ Functions

---

### ملخص — Cheatsheet

| الـ Symbol | النوع | القيمة | الاستخدام |
|---|---|---|---|
| `IRQ_NONE` | `enum irqreturn` | `0` | الـ interrupt مش من الـ device ده أو ما اتعالجش |
| `IRQ_HANDLED` | `enum irqreturn` | `1` | الـ interrupt اتعالج بنجاح |
| `IRQ_WAKE_THREAD` | `enum irqreturn` | `2` | اطلب إيقاظ الـ threaded handler |
| `irqreturn_t` | `typedef` | — | الـ type المستخدم في كل الـ ISR signatures |
| `IRQ_RETVAL(x)` | `macro` | `IRQ_HANDLED` أو `IRQ_NONE` | تحويل boolean expression لـ `irqreturn_t` |

---

### التصنيف — Logical Groups

الـ file ده صغير جداً بس محوري — فيه **نوع واحد (enum + typedef)** وـ **macro واحد**، كلهم بيخدموا غرض واحد: تعريف القيم اللي بترجعها الـ **interrupt handler (ISR)** للـ kernel.

---

### Group 1: الـ Return Type Definition

#### `enum irqreturn` و `irqreturn_t`

```c
enum irqreturn {
    IRQ_NONE        = (0 << 0),  /* 0b00 */
    IRQ_HANDLED     = (1 << 0),  /* 0b01 */
    IRQ_WAKE_THREAD = (1 << 1),  /* 0b10 */
};

typedef enum irqreturn irqreturn_t;
```

الـ `enum irqreturn` هو الـ contract بين الـ ISR والـ kernel interrupt core. كل **interrupt handler** مسجل بـ `request_irq()` أو `request_threaded_irq()` لازم يرجع واحدة من القيم دي. الـ kernel بيستخدم القيمة دي عشان يقرر هيعمل إيه بعد ما الـ handler يخلص.

**القيم بالتفصيل:**

| القيمة | البت | المعنى |
|---|---|---|
| `IRQ_NONE` | `bit 0 = 0` | الـ handler شاف إن الـ interrupt مش بتاعه — مش من الـ device اللي هو مسؤول عنه، أو جه والـ device مش في حالة interrupt. شائع في الـ shared IRQ lines. |
| `IRQ_HANDLED` | `bit 0 = 1` | الـ handler عالج الـ interrupt بنجاح وخلص. ده الـ return value العادي. |
| `IRQ_WAKE_THREAD` | `bit 1 = 1` | الـ primary (hardirq) handler خلص شغله السريع، وطلب من الـ kernel إنه يصحّى الـ threaded handler المسجل بـ `request_threaded_irq()` عشان يكمل الشغل الثقيل في process context. |

**ليه bit-shift وليس أرقام عادية؟**

الـ design بالـ bit flags بيفتح الباب إن القيم دي تتعامل كـ bitmask داخل الـ kernel. الـ core code في `kernel/irq/handle.c` بيعمل فحص على البتات مش على القيمة كـ integer عادي، وده بيخلي الـ checks أسرع وأوضح.

**الـ typedef:**

الـ `irqreturn_t` هو الـ type الرسمي اللي بيظهر في كل الـ kernel APIs والـ driver signatures. وجوده كـ typedef بدل enum مباشر بيسمح للـ kernel بإنه يغير الـ underlying type لو احتاج في المستقبل من غير ما يكسر الـ ABI.

**السياق — من بيستخدمه؟**

كل ISR في الـ kernel بيرجع `irqreturn_t`. مثلاً:

```c
/* مثال: handler بسيط لـ GPIO interrupt */
static irqreturn_t my_gpio_isr(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;

    if (!my_dev_has_interrupt(dev))
        return IRQ_NONE;   /* مش interrupt بتاعنا — shared IRQ */

    my_dev_ack_interrupt(dev);
    return IRQ_HANDLED;    /* خلصنا */
}
```

```c
/* مثال: threaded handler */
static irqreturn_t my_threaded_isr(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;

    my_dev_ack_interrupt(dev);   /* اعمل الـ ACK في hardirq context */
    return IRQ_WAKE_THREAD;       /* صحّى الـ thread handler */
}

static irqreturn_t my_thread_fn(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;

    my_dev_process_data(dev);    /* شغل تقيل في process context */
    return IRQ_HANDLED;
}
```

---

### Group 2: الـ Helper Macro

#### `IRQ_RETVAL(x)`

```c
#define IRQ_RETVAL(x)   ((x) ? IRQ_HANDLED : IRQ_NONE)
```

**الـ macro ده بيعمل إيه:**
بيحول أي boolean expression لـ `irqreturn_t` صح. لو `x` non-zero (يعني "حصل حاجة") → يرجع `IRQ_HANDLED`. لو `x` zero → يرجع `IRQ_NONE`. الغرض منه تبسيط الكود في الحالات اللي فيها الـ handler عنده condition واحدة بتحدد هو عالج ولا لأ.

**Parameters:**

| البارامتر | النوع | الشرح |
|---|---|---|
| `x` | أي expression | أي قيمة integer أو boolean — بيتقيّم كـ truthy/falsy |

**Return value:**
`irqreturn_t` — إما `IRQ_HANDLED` (1) أو `IRQ_NONE` (0).

**Key details:**
- الـ macro مش بيدعم `IRQ_WAKE_THREAD` — لو محتاج تصحّى thread handler استخدم القيمة مباشرة.
- الـ expansion بسيط جداً — zero overhead بعد الـ compilation.
- الـ parentheses حوالين `(x)` ضرورية لتجنب الـ operator precedence bugs لو `x` expression مركب.

**مثال استخدام:**

```c
static irqreturn_t my_isr(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;
    int handled;

    handled = my_dev_handle(dev);   /* returns 1 if handled, 0 if not */
    return IRQ_RETVAL(handled);
}
```

**مقارنة مع الكتابة الصريحة:**

```c
/* بدون الـ macro — verbose */
return handled ? IRQ_HANDLED : IRQ_NONE;

/* معاه — أوضح في النية */
return IRQ_RETVAL(handled);
```

---

### Flow — كيف الـ kernel بيستخدم الـ return value

```
ISR يرجع irqreturn_t
         │
         ▼
  kernel/irq/handle.c
  __handle_irq_event_percpu()
         │
         ├─ IRQ_NONE        → continue loop (shared IRQ — جرّب الـ handler الجاي)
         │
         ├─ IRQ_HANDLED     → set IRQS_AUTODETECT flag + exit loop
         │
         └─ IRQ_WAKE_THREAD → irq_wake_thread() → wake_up_process(action->thread)
                                                    (الـ thread handler يشتغل في process context)
```

**الـ `IRQ_NONE` في الـ shared IRQs:**

في الـ systems اللي فيها **shared interrupt lines** (IRQ line واحدة لأكتر من device)، الـ kernel بيلف على كل الـ handlers المسجلين للـ IRQ ده. لو كل الـ handlers رجّعوا `IRQ_NONE`، الـ kernel يعتبر ده spurious interrupt ويعمل log وربما يـ disable الـ IRQ لو اتكرر كتير (`irq_spurious_count` threshold).

---

### ملاحظات مهمة للـ Driver Developer

- **لازم** ترجع `IRQ_NONE` لو الـ interrupt مش بتاعك في الـ shared IRQ scenarios — ده مش optional.
- **ممنوع** ترجع `IRQ_WAKE_THREAD` من الـ threaded handler نفسه (الـ `thread_fn`) — بس من الـ primary handler.
- الـ `IRQ_WAKE_THREAD` من غير ما تسجّل `thread_fn` في `request_threaded_irq()` → الـ kernel بيتعامل معاها كـ `IRQ_HANDLED` automatically (الـ core code بيتحقق من وجود الـ thread).
- من الـ kernel v2.6.30+ الـ threaded IRQs أصبحت standard والـ `IRQ_WAKE_THREAD` بقى في common use.
## Phase 5: دليل الـ Debugging الشامل

الـ `irqreturn_t` هو enum بسيط جداً بس موجود في قلب كل interrupt handler في الكيرنل — أي مشكلة فيه بتتجلى في سلوك الـ interrupt handling كله. الـ debugging هنا بيتركز على تتبع قيمة الـ return من الـ handler ومعرفة إيه اللي بيحصل بعدها.

---

### Software Level

#### 1. debugfs — المداخل المتعلقة بالـ IRQ

الـ debugfs بيوفر معلومات تفصيلية عن كل interrupt في النظام:

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل الـ IRQ descriptors والـ handlers المسجلين
ls /sys/kernel/debug/irq/

# تفاصيل IRQ معين (مثلاً IRQ 10)
cat /sys/kernel/debug/irq/irqs/10

# الـ irq_domain — بيربط hardware IRQ بـ Linux IRQ number
ls /sys/kernel/debug/irq/domains/
cat /sys/kernel/debug/irq/domains/default
```

**تفسير الـ output:**
```
name:   gpio-irq
  domain: /soc/gpio@1234
  hwirq: 5  -> linux irq: 42
  chip: gpiochip0
  flags: IRQD_IRQ_MASKED | IRQD_ACTIVATED
```

الـ `hwirq` هو رقم الـ interrupt في الـ hardware، والـ `linux irq` هو الرقم اللي بيستخدمه الكيرنل داخلياً.

---

#### 2. sysfs — المداخل المتعلقة بالـ IRQ

```bash
# عدد مرات fire كل IRQ — العمود الأول هو العداد لكل CPU
cat /proc/interrupts

# مثال output:
#            CPU0       CPU1
#  42:        1523       2041   GIC-0  42  Edge  my-device
#  ^^^         ^^         ^^    ^^^^   ^^  ^^^^  ^^^^^^^^^
#  IRQ#     cnt-c0     cnt-c1  chip  hwirq type  handler-name

# لو IRQ_NONE بيرجع دايماً — العداد مش بيزيد أو بيزيد ببطء غريب
# شوف /proc/stat برضو
grep intr /proc/stat

# معلومات عن كل interrupt per-CPU
cat /proc/irq/42/smp_affinity        # CPU affinity mask (hex)
cat /proc/irq/42/smp_affinity_list   # CPU affinity (human-readable)
cat /proc/irq/42/spurious            # عدد الـ spurious interrupts
```

**الـ spurious counter:** لو `count` في `/proc/irq/N/spurious` بيزيد — ده معناه إن الـ handler بيرجع `IRQ_NONE` لـ interrupt مش بتاعه، وده ممكن يؤدي لـ `irq N: nobody cared` في الـ kernel log.

```bash
# شوف الـ irqbalance distribution
cat /proc/irq/42/node   # NUMA node
```

---

#### 3. ftrace — تتبع الـ interrupt handlers

```bash
# تفعيل tracing للـ IRQ events
cd /sys/kernel/debug/tracing

# الـ events المتاحة للـ IRQ subsystem
ls events/irq/

# irq_handler_entry  — عند دخول الـ handler
# irq_handler_exit   — عند خروج الـ handler (بيحتوي على القيمة المرجعة!)
# softirq_entry / softirq_exit / softirq_raise

# تفعيل تتبع دخول وخروج الـ handler
echo 1 > events/irq/irq_handler_entry/enable
echo 1 > events/irq/irq_handler_exit/enable
echo 1 > tracing_on
cat trace
echo 0 > tracing_on
```

**مثال output:**
```
   my-device-123   [001] d... 12345.678: irq_handler_entry: irq=42 name=my_handler
   my-device-123   [001] d... 12345.679: irq_handler_exit:  irq=42 ret=handled
#                                                                       ^^^^^^^
#                                              هنا: handled أو unhandled
```

لو شفت `ret=unhandled` كتير — الـ handler بيرجع `IRQ_NONE` لـ interrupts مش بتاعته.

```bash
# تتبع function محددة باستخدام function_graph tracer
echo function_graph > current_tracer
echo my_irq_handler > set_graph_function
echo 1 > tracing_on
# ... trigger interrupt ...
cat trace
```

---

#### 4. printk / dynamic debug

```bash
# تفعيل dynamic debug لـ IRQ core
echo 'file kernel/irq/*.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/*.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug المفعلة حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep irq

# رفع مستوى الـ loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk   # KERN_DEBUG مرئي

# أو عبر dmesg
dmesg -n 8   # show all levels including debug
dmesg -w     # watch mode — بيتابع في real-time
dmesg | grep -E "irq|IRQ|interrupt"
```

**في الكود** — إضافة debug prints استراتيجية:

```c
static irqreturn_t my_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;

    /* تحقق إن الـ interrupt بتاعنا فعلاً */
    if (!my_device_irq_pending(dev)) {
        pr_debug("IRQ %d: not ours, returning IRQ_NONE\n", irq);
        return IRQ_NONE;  /* <-- ده اللي بنتتبعه */
    }

    /* معالجة الـ interrupt */
    my_device_clear_irq(dev);

    pr_debug("IRQ %d: handled successfully\n", irq);
    return IRQ_HANDLED;
}
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_GENERIC_IRQ_DEBUGFS` | يفعّل `/sys/kernel/debug/irq/` بالكامل |
| `CONFIG_IRQ_DOMAIN_DEBUG` | debug info للـ IRQ domain mapping |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في interrupt context |
| `CONFIG_LOCKDEP` | تتبع lock ordering violations |
| `CONFIG_DEBUG_SHIRQ` | يستدعي الـ shared IRQ handlers عند disable للتحقق |
| `CONFIG_SPARSE_IRQ` | أحسن لـ systems بـ IRQ numbers كبيرة |
| `CONFIG_HARDIRQS_SW_RESEND` | يعيد إرسال الـ IRQ لو اتفوّت |
| `CONFIG_IRQ_FORCED_THREADING` | يحول الـ handlers لـ threads للـ debugging |
| `CONFIG_TRACE_IRQFLAGS` | يتتبع enable/disable الـ IRQ flags |
| `CONFIG_DEBUG_IRQFLAGS` | يتحقق من سلامة الـ IRQ flags |

```bash
# تحقق من الـ config الحالية
grep -E "CONFIG_(GENERIC_IRQ_DEBUGFS|IRQ_DOMAIN_DEBUG|DEBUG_SHIRQ|LOCKDEP|PROVE_LOCKING)" /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ IRQ Subsystem

```bash
# irqtop — مراقبة الـ IRQ rates في real-time (زي top بس للـ IRQs)
irqtop   # لو مثبت، أو:
watch -n 0.5 cat /proc/interrupts

# استخدام perf لتتبع الـ IRQ events
perf stat -e irq:irq_handler_entry,irq:irq_handler_exit sleep 5
perf record -e irq:irq_handler_entry -a sleep 10
perf report

# تعطيل الـ IRQ balancing مؤقتاً للـ debugging
systemctl stop irqbalance

# pin الـ IRQ على CPU معين
echo 2 > /proc/irq/42/smp_affinity   # CPU 1 فقط (bit mask)

# استخدام stress-ng لـ trigger IRQs
stress-ng --iomix 4 --timeout 30s
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `irq N: nobody cared (try booting with the "irqpoll" option)` | كل الـ handlers رجّعوا `IRQ_NONE` لـ IRQ N لفترة طويلة — الكيرنل هيعمل disable للـ IRQ | تحقق إن الـ handler بيتعرف على الـ interrupt بتاعه صح |
| `Disabling IRQ #N` | الكيرنل عمل disable للـ IRQ بعد spurious interrupts متكررة | فحص الـ hardware line، وراجع منطق `IRQ_NONE` |
| `unexpected IRQ trap at vector N` | IRQ وصل بدون handler مسجل | تأكد إن `request_irq()` اتنفذت قبل enable الـ hardware |
| `enable_irq(N) unbalanced` | `enable_irq` اتنادت أكتر من `disable_irq` | راجع كل الـ enable/disable pairs في الكود |
| `Unhandled IRQ or irq disabled` | الـ IRQ handler مش موجود أو الـ IRQ معطّل | تأكد من تسجيل الـ handler وحالة الـ IRQ |
| `IRQ N: IRQF_SHARED is required` | محاولة مشاركة IRQ line بدون `IRQF_SHARED` flag | أضف `IRQF_SHARED` لـ `request_irq()` |
| `irq_to_desc(N) is NULL` | IRQ number غير صالح | تحقق من الـ IRQ number اللي بيجي من الـ DT أو ACPI |
| `handler N enabled interrupts` | الـ handler فعّل الـ interrupts داخلياً — ممنوع | شيل أي `local_irq_enable()` من داخل الـ handler |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
static irqreturn_t my_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;

    /* تحقق إن مش في wrong context */
    WARN_ON(!in_interrupt());

    /* تحقق إن dev_id مش NULL — لازم يكون valid */
    if (WARN_ON(!dev))
        return IRQ_NONE;

    /* تحقق من state consistency */
    if (WARN_ON(dev->state == DEV_STATE_UNINITIALIZED)) {
        dump_stack();   /* عشان نشوف call chain كامل */
        return IRQ_NONE;
    }

    /* لو الـ IRQ مش pending والمفروض يكون — مشكلة hardware */
    if (!my_device_irq_status(dev)) {
        WARN_ONCE(1, "IRQ %d fired but no status bit set!\n", irq);
        return IRQ_NONE;
    }

    return IRQ_HANDLED;
}
```

---

### Hardware Level

#### 1. التحقق إن Hardware State بيتطابق مع Kernel State

```bash
# شوف الـ IRQ state في الكيرنل
cat /sys/kernel/debug/irq/irqs/42

# output مهم:
# flags: 0x2000  -> IRQD_IRQ_DISABLED (الكيرنل عطّله)
# state: 0x1     -> IRQS_AUTODETECT or similar
# depth: 1       -> عدد مرات disable بدون enable مقابل

# تحقق إن الـ IRQ line enabled في الـ GIC (ARM Generic Interrupt Controller)
# عبر /sys/kernel/debug/irq/
cat /sys/kernel/debug/irq/domains/GIC/
```

**قاعدة ذهبية:** لو `/proc/interrupts` بيُظهر إن الـ IRQ counter مش بيزيد — إما الـ hardware مش بيبعت الـ interrupt أو الكيرنل عطّله.

---

#### 2. Register Dump Techniques

```bash
# devmem2 — قراءة hardware registers مباشرة
# مثال: قراءة interrupt status register لجهاز على عنوان 0x40010000
devmem2 0x40010010 w    # read word (32-bit)
devmem2 0x40010014 w    # read interrupt enable register

# /dev/mem — للـ systems اللي بتسمح بيه
dd if=/dev/mem bs=4 count=1 skip=$((0x40010010 / 4)) 2>/dev/null | xxd

# io utility (من package ioport)
io -4 0x40010010   # x86 فقط

# لو MMIO mapped في sysfs
ls /sys/bus/platform/devices/my-device/
xxd /sys/bus/platform/devices/my-device/resource0 | head -20
```

**مثال — قراءة GIC registers على ARM:**
```bash
# GIC Distributor base عادةً على 0x08000000 في QEMU virt
devmem2 0x08000100 w   # GICD_ISENABLER0 — interrupt enable register
devmem2 0x08000200 w   # GICD_ISPENDR0   — interrupt pending register
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس المهمة:**

```
IRQ Line ──────────────────────────────────────
         ____      ____
        |    |    |    |
________↑    ↓____↑    ↓____________________
        ^    ^
        |    └── Handler returns IRQ_HANDLED (يمسح الـ status)
        └──────── Interrupt fires (rising/falling edge حسب الـ polarity)
```

- **Edge-triggered:** المهم هو حافة الـ signal — Rising أو Falling. لو الـ handler اتأخر، ممكن يتفوت interrupt تاني.
- **Level-triggered:** الـ signal لازم يرجع لـ inactive قبل ما الكيرنل يعمل re-enable للـ IRQ — لو الـ hardware مش بيمسح الـ status، هيحصل **spurious interrupt loop**.

```
المشكلة الشائعة — Level-triggered بدون clear:
─────────────────────────────────────────────
IRQ Line: ████████████████████████  (مش بيرجع)
Handler:  ↑IRQ_HANDLED↑IRQ_HANDLED↑IRQ_HANDLED  (loop لا نهائي)
```

**ضبط الـ oscilloscope:**
- Trigger على rising/falling edge حسب الـ IRQ polarity
- قيس الـ latency بين fire الـ interrupt ورد الـ handler
- تأكد إن الـ IRQ line بيرجع لـ inactive بعد الـ handler

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log |
|---|---|
| الـ IRQ line stuck high (level-triggered) | `irq N: nobody cared` + IRQ disabled |
| الـ hardware مش بيبعت interrupt خالص | `/proc/interrupts` counter frozen |
| الـ IRQ line مش connected (floating) | Spurious interrupts عشوائية |
| خطأ في الـ IRQ polarity | Handler بيشتغل بس مفيش interrupt flag set |
| مشاركة IRQ line بين devices (shared IRQ) | Handler الغلط بيرجع `IRQ_HANDLED` وبيخفي interrupts التانية |
| Clock/Power issue للـ device | Handler بيشتغل بعد delay غير منتظم |

---

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT interrupts property
# في الـ DTS:
# my-device@40010000 {
#     interrupts = <GIC_SPI 42 IRQ_TYPE_EDGE_RISING>;
#                              ^^  ^^^^^^^^^^^^^^^^
#                              |   النوع: edge/level + polarity
#                              IRQ number في الـ GIC

# شوف الـ DT المُفسَّر بعد boot
cat /proc/device-tree/soc/my-device@40010000/interrupts | xxd
# أو
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "my-device"

# تحقق إن الـ IRQ mapping صح
cat /sys/kernel/debug/irq/domains/GIC/
# هيُظهر: hwirq 42 -> linux irq 75 (مثلاً)

# تحقق من الـ IRQ type
cat /sys/kernel/debug/irq/irqs/75
# flags: IRQD_TRIGGER_EDGE أو IRQD_TRIGGER_LEVEL
```

**المشاكل الشائعة في الـ DT:**

```
1. غلط في الـ interrupt-parent → الـ IRQ بيتروح للـ wrong controller
2. غلط في الـ IRQ type (edge vs level) → spurious أو missed interrupts
3. غلط في الـ IRQ number (off-by-one بين hardware manual والـ DT) → wrong IRQ fires
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للـ Copy-Paste

```bash
#!/bin/bash
# ==========================================
# IRQ Debugging Toolkit
# استخدم: chmod +x irq_debug.sh && ./irq_debug.sh 42
# ==========================================

IRQ=${1:-42}

echo "=== IRQ $IRQ Status ==="
grep "^ *$IRQ:" /proc/interrupts

echo ""
echo "=== Spurious Count ==="
cat /proc/irq/$IRQ/spurious 2>/dev/null || echo "Not available"

echo ""
echo "=== CPU Affinity ==="
cat /proc/irq/$IRQ/smp_affinity_list 2>/dev/null

echo ""
echo "=== debugfs Info ==="
cat /sys/kernel/debug/irq/irqs/$IRQ 2>/dev/null || echo "Enable CONFIG_GENERIC_IRQ_DEBUGFS"

echo ""
echo "=== Recent IRQ Messages in dmesg ==="
dmesg | grep -i "irq $IRQ\|irq#$IRQ" | tail -20
```

```bash
# مراقبة الـ IRQ rate في real-time (interrupts per second)
watch -n 1 'awk -v irq=42 "NR==1{for(i=1;i<=NF;i++) cpu[i]=\$i} \
    \$1==irq\":\" {for(i=2;i<=NF;i++) printf cpu[i-1]\": \"\$i\" \"}" \
    /proc/interrupts; echo'

# تفعيل ftrace للـ IRQ handler بسرعة
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo irq:irq_handler_entry > set_event
echo irq:irq_handler_exit >> set_event
echo 1 > tracing_on
sleep 5
echo 0 > tracing_on
grep "irq=42" trace | head -50

# تشخيص "nobody cared" error
dmesg | grep -A20 "nobody cared"
# هيُظهر stack trace بيبيّن مين register الـ handler وإيه اللي حصل
```

**تفسير output الـ ftrace:**

```
# الـ output الطبيعي:
kworker/1:2-45    [001] d... 1234.000: irq_handler_entry: irq=42 name=my_drv
kworker/1:2-45    [001] d... 1234.001: irq_handler_exit:  irq=42 ret=handled
#                                                                      ^^^^^^^
#                                                           IRQ_HANDLED = "handled"
#                                                           IRQ_NONE    = "unhandled"

# الـ output الـ problematic:
kworker/1:2-45    [001] d... 1234.000: irq_handler_entry: irq=42 name=my_drv
kworker/1:2-45    [001] d... 1234.001: irq_handler_exit:  irq=42 ret=unhandled
#                                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                 ده معناه IRQ_NONE — الـ handler مش بيتعرف على الـ interrupt
#                 لو بيحصل كتير → "nobody cared" → IRQ disabled
```

```bash
# تفعيل IRQ_FORCED_THREADING لعزل مشاكل الـ context
# (بيحول hard IRQ handlers لـ kernel threads للـ debugging)
# في kernel cmdline:
# threadirqs

# أو في runtime (بعض kernels):
echo 1 > /sys/kernel/debug/irq/threaded

# بعد كده تقدر تعمل strace على الـ IRQ thread:
ps aux | grep irq/42
strace -p <PID>
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ UART interrupt مش بيتعالج

#### العنوان
**الـ IRQ_NONE بيسبب starvation في RS-485 driver على industrial gateway**

#### السياق
شركة بتبني industrial IoT gateway بيستخدم RK3562، المنتج بيجمع data من sensors عن طريق RS-485 (UART3). الـ driver اتكتب بسرعة لمطابقة deadline التسليم.

#### المشكلة
في production، الـ gateway بيفوّت packets بشكل عشوائي. الـ `/proc/interrupts` بيظهر إن عدد الـ IRQs على UART3 line بيزيد بشكل طبيعي، بس الـ data اللي بتوصل ناقصة. مفيش kernel warnings ومفيش crashes.

#### التحليل
الـ engineer فتح الـ driver وقاله:

```c
static irqreturn_t rs485_isr(int irq, void *dev_id)
{
    struct rs485_dev *dev = dev_id;

    if (!(readl(dev->base + UART_IIR) & UART_IIR_NOINT)) {
        /* طلعت interrupt مش من الـ UART ده */
        return IRQ_NONE;   /* <-- هنا المشكلة */
    }

    handle_rx_data(dev);
    return IRQ_HANDLED;
}
```

المشكلة: الـ `UART_IIR_NOINT` bit logic اتعكس. لما الـ UART بيطلع interrupt حقيقي، الـ register bit بيكون 0 (not "no interrupt")، فالـ condition اتقلب والـ handler بيرجع `IRQ_NONE` دايمًا للـ real interrupts ويدخل للـ `handle_rx_data` بس لما مفيش data.

الـ kernel IRQ subsystem لما بيشوف `IRQ_NONE` بيتجاهل الـ handler ده، بس الـ shared IRQ line لو موجود، بيجرّب handlers تانية. لو الـ line مش shared، الـ interrupt بيتعالج بس الـ data مش بتتقرأ.

من `irqreturn.h`:
```c
IRQ_NONE    = (0 << 0),  /* = 0 */
IRQ_HANDLED = (1 << 0),  /* = 1 */
```

الـ kernel's `handle_irq_event_percpu()` بيحسب كام مرة الـ handler رجع `IRQ_NONE` — لو كتير أوي بيظهر `irq N: nobody cared` وبيعمل disable للـ IRQ. لو الـ system وصل لحد الـ spurious_count، الـ UART3 interrupt بيتقفل تمامًا.

#### الحل

```c
static irqreturn_t rs485_isr(int irq, void *dev_id)
{
    struct rs485_dev *dev = dev_id;
    u32 iir = readl(dev->base + UART_IIR);

    /* UART_IIR_NOINT = 1 means NO interrupt pending */
    if (iir & UART_IIR_NOINT)
        return IRQ_NONE;   /* فعلًا مش interrupt منا */

    handle_rx_data(dev);
    return IRQ_HANDLED;
}
```

التحقق بعد الحل:
```bash
# شوف الـ spurious counter قبل وبعد
cat /proc/interrupts | grep "uart3\|UART"
# شوف لو في irq_disabled events
dmesg | grep "nobody cared\|Disabling IRQ"
```

#### الدرس المستفاد
الـ `IRQ_NONE` مش مجرد "أنا مش عارف" — الـ kernel بيعدّها ولو كتّرت بيعمل disable للـ IRQ line كلها. اتأكد دايمًا إن الـ polarity بتاع الـ "interrupt pending" bit صح قبل ما ترجع `IRQ_NONE`.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ HDMI HPD interrupt بيعمل kernel panic

#### العنوان
**الـ IRQ_WAKE_THREAD بيسبب deadlock في HDMI hot-plug handler**

#### السياق
منتج Android TV box بيستخدم Allwinner H616. الـ HDMI hot-plug detection (HPD) بيشغّل interrupt لما تتوصل شاشة. الـ team عايزة تعمل EDID read في الـ interrupt handler عشان تعرف capabilities الشاشة.

#### المشكلة
لما المستخدم بيوصل ويفصل الشاشة بسرعة، الـ system بيعمل hang كامل. الـ magic sysrq بيظهر إن في kernel thread stuck في `schedule()`.

#### التحليل
الـ code الأصلي:

```c
static irqreturn_t hdmi_hpd_hardirq(int irq, void *data)
{
    struct hdmi_dev *hdmi = data;

    /* قراءة EDID بتاخد وقت — عايزين نعملها في thread */
    if (hdmi_hpd_connected(hdmi)) {
        schedule_work(&hdmi->edid_work);
        return IRQ_HANDLED;   /* <-- المشكلة هنا */
    }
    return IRQ_NONE;
}
```

الـ developer استخدم `schedule_work()` في الـ hard IRQ context وبعدين رجع `IRQ_HANDLED`. المشكلة إن الـ `edid_work` بتاخد mutex لقراءة I2C (DDC channel للـ EDID) — وفي نفس الوقت الـ I2C controller ممكن يكون محتاج IRQ هو كمان، وده بيسبب deadlock.

الحل الصح كان `IRQ_WAKE_THREAD`:

```c
/* من irqreturn.h */
IRQ_WAKE_THREAD = (1 << 1),  /* = 2 */
```

الـ `IRQ_WAKE_THREAD` بيخلي الـ kernel يصحّي الـ threaded handler اللي بيتشغل في process context عادي — ممكن ياخد mutex وينام وكل حاجة.

```c
static irqreturn_t hdmi_hpd_hardirq(int irq, void *data)
{
    struct hdmi_dev *hdmi = data;

    if (!hdmi_hpd_connected(hdmi))
        return IRQ_NONE;

    /* صحّي الـ thread — هو اللي هيعمل EDID read */
    return IRQ_WAKE_THREAD;
}

static irqreturn_t hdmi_hpd_thread(int irq, void *data)
{
    struct hdmi_dev *hdmi = data;

    /* هنا process context — ممكن ناخد mutex */
    hdmi_read_edid(hdmi);       /* بتاخد I2C mutex */
    hdmi_update_modes(hdmi);
    return IRQ_HANDLED;
}

/* في الـ probe */
devm_request_threaded_irq(dev, irq,
    hdmi_hpd_hardirq,   /* hard IRQ handler */
    hdmi_hpd_thread,    /* threaded handler */
    IRQF_TRIGGER_BOTH,
    "hdmi-hpd", hdmi);
```

#### الحل التشخيصي
```bash
# شوف الـ thread اللي بيخدم الـ IRQ
ps aux | grep irq
# مثلًا: irq/45-hdmi-hpd

# شوف الـ IRQ flags
cat /proc/irq/45/actions
```

#### الدرس المستفاد
الـ `IRQ_WAKE_THREAD` موجود بالظبط عشان الـ operations اللي محتاجة sleeping context زي I2C و SPI reads. استخدمه بدل `schedule_work` في الـ hard IRQ handler — أنظف وأأمن.

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — الـ SPI interrupt مش بيتسجّل

#### العنوان
**الـ IRQ_RETVAL macro بيخفي bug في SPI sensor driver**

#### السياق
embedded team بتشتغل على IoT board بيستخدم STM32MP1 مع IMU sensor (accelerometer/gyroscope) متوصل بـ SPI. الـ sensor بيطلع data-ready interrupt لما يكون عنده قراءة جديدة.

#### المشكلة
الـ driver بيشتغل بس الـ sensor data بتتأخر أحيانًا. الـ `/proc/interrupts` بيظهر إن الـ IRQ counter بيزيد ببطء مش بالـ rate المتوقعة (100Hz).

#### التحليل
الـ code:

```c
static int data_ready;  /* global flag — مش thread-safe */

static irqreturn_t imu_data_ready_isr(int irq, void *dev)
{
    data_ready = spi_check_drdy(dev);  /* بترجع 0 أو 1 */
    return IRQ_RETVAL(data_ready);
}
```

من `irqreturn.h`:
```c
#define IRQ_RETVAL(x)   ((x) ? IRQ_HANDLED : IRQ_NONE)
```

يعني لو `spi_check_drdy()` رجعت 0 (مفيش data ready)، الـ handler بيرجع `IRQ_NONE`. بس الـ interrupt وصل فعلًا! المشكلة إن الـ sensor بيطلع interrupt، الـ STM32MP1 بيستقبله، بس الـ handler بيكذّب إنه مش منه — فالـ kernel بيعدّها spurious.

الـ bug الأعمق: `spi_check_drdy()` نفسها بتعمل SPI transaction في hard IRQ context — ده ممنوع لو الـ SPI controller بيستخدم DMA أو interrupts هو كمان.

الحل الصح:

```c
static irqreturn_t imu_data_ready_isr(int irq, void *dev)
{
    /* الـ interrupt وصل = data ready بالتعريف */
    /* مش محتاجين نعمل SPI read هنا */
    return IRQ_WAKE_THREAD;
}

static irqreturn_t imu_data_thread(int irq, void *dev)
{
    struct imu_dev *imu = dev;

    /* هنا نقرأ الـ SPI في process context */
    imu_spi_read_fifo(imu);
    iio_trigger_notify_done(imu->trig);
    return IRQ_HANDLED;
}
```

أو لو محتاج `IRQ_RETVAL`:

```c
static irqreturn_t imu_data_ready_isr(int irq, void *dev)
{
    struct imu_dev *imu = dev;
    /* تحقق من flag مش من SPI */
    bool mine = gpio_get_value(imu->drdy_gpio) == imu->drdy_active;
    return IRQ_RETVAL(mine);
}
```

#### Debug commands
```bash
# شوف الـ spurious IRQ count
grep "ERR\|spurious" /proc/interrupts
# شوف latency الـ SPI
trace-cmd record -e spi\* &
sleep 5
trace-cmd report | grep imu
```

#### الدرس المستفاد
الـ `IRQ_RETVAL(x)` مش magic — لو الـ `x` بتجي من operation ممكن تفشل، بترجع `IRQ_NONE` على interrupts حقيقية وتسبب مشكلة. اتأكد إن الـ condition اللي بتديها لـ `IRQ_RETVAL` بتعبّر فعلًا عن "هل الـ interrupt ده منا؟".

---

### السيناريو 4: Automotive ECU على i.MX8 — الـ CAN bus interrupt بيتعمل disable

#### العنوان
**الـ spurious IRQ detection بيعمل disable للـ CAN controller interrupt في automotive ECU**

#### السياق
automotive ECU بيستخدم i.MX8QM للتحكم في نظام infotainment. الـ CAN bus driver بيستقبل messages من sensors تانية في السيارة. بعد ساعات من التشغيل، الـ CAN bus بيبطّل يستقبل messages بدون أي warning واضح.

#### المشكلة
الـ `dmesg` بيظهر:
```
irq 78: nobody cared (try booting with the "irqpoll" option)
Disabling IRQ #78
```

الـ CAN controller interrupt اتعطّل تمامًا بعد إن الـ kernel شاف كتير من `IRQ_NONE` responses.

#### التحليل
الـ kernel IRQ subsystem عنده protection mechanism:
- لو handler رجع `IRQ_NONE` أكتر من `99,900` مرة في آخر `100,000` invocations
- الـ kernel بيعتبر الـ IRQ "nobody cared" وبيعمله disable

الـ driver:

```c
static irqreturn_t can_isr(int irq, void *data)
{
    struct can_dev *can = data;
    u32 status = readl(can->base + CAN_INT_STATUS);

    if (!status)
        return IRQ_NONE;  /* مفيش interrupts pending */

    /* clear الـ interrupts */
    writel(status, can->base + CAN_INT_CLEAR);
    can_handle_messages(can, status);
    return IRQ_HANDLED;
}
```

الـ code يبدو صح، بس المشكلة في الـ i.MX8 CAN controller hardware: بعد `writel` للـ clear register، في latency صغيرة قبل ما الـ `CAN_INT_STATUS` يتصفّر فعلًا. لو في interrupt تاني وصل في نفس الوقت وقرأنا `status` قبل ما الـ clear يتأثر، بنشوف 0 ونرجع `IRQ_NONE` وإحنا المفروض نـ handle الـ interrupt ده.

بالإضافة، الـ i.MX8 CAN controller عنده level-triggered interrupt — يعني طول ما في message جديدة، الـ interrupt line بتفضل active. لو في race condition في الـ clear، ممكن الـ handler يتكلم مرتين على نفس الـ interrupt — المرة التانية يلاقي status = 0 ويرجع `IRQ_NONE`.

#### الحل

```c
static irqreturn_t can_isr(int irq, void *data)
{
    struct can_dev *can = data;
    u32 status = readl(can->base + CAN_INT_STATUS);

    if (!status)
        return IRQ_NONE;

    /* clear أولًا، بعدين اقرأ تاني للتأكد */
    writel(status, can->base + CAN_INT_CLEAR);
    /* memory barrier عشان نضمن إن الـ write وصل */
    wmb();

    can_handle_messages(can, status);
    return IRQ_HANDLED;
}
```

وفي الـ Device Tree، تأكد من الـ interrupt trigger type:
```dts
/* في DT الـ i.MX8 CAN node */
interrupts = <GIC_SPI 78 IRQ_TYPE_LEVEL_HIGH>;
```

```bash
# تحقق من الـ IRQ type
cat /proc/irq/78/chip_name
cat /sys/kernel/irq/78/type
# شوف الـ spurious counter
cat /proc/irq/78/spurious
```

#### الدرس المستفاد
في الـ level-triggered interrupts، الـ clear sequence مهمة جدًا. الـ `IRQ_NONE` المتكرر بيأدي لـ disable تلقائي من الـ kernel. استخدم `wmb()` بعد الـ clear وراجع الـ DT trigger type يطابق الـ hardware spec.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ I2C interrupt مش بيشتغل خالص

#### العنوان
**الـ IRQ_NONE من shared interrupt handler بيخفي bring-up bug على AM62x custom board**

#### السياق
hardware team بتعمل bring-up لـ custom board بيستخدم TI AM62x SoC. الـ board عنده I2C bus متوصل بيه PMIC و RTC و temperature sensor. الـ ثلاثة أجهزة بيشتركوا في نفس الـ IRQ line (shared interrupt).

#### المشكلة
أثناء الـ boot، الـ RTC driver بيفشل في الـ initialization ودايمًا بيطلع:
```
rtc-ds1307 2-0068: unable to read RTC, waiting...
```
الـ PMIC والـ temperature sensor شغّالين تمام.

#### التحليل
الـ shared interrupt setup:

```c
/* كل device بيسجّل handler على نفس الـ IRQ */
request_irq(irq, pmic_isr,   IRQF_SHARED, "pmic",   pmic);
request_irq(irq, rtc_isr,    IRQF_SHARED, "rtc",    rtc);
request_irq(irq, temp_isr,   IRQF_SHARED, "temp",   temp);
```

لما الـ IRQ يحصل، الـ kernel بيجرّب الـ handlers بالترتيب:

```c
static irqreturn_t rtc_isr(int irq, void *data)
{
    struct rtc_dev *rtc = data;
    u8 status;

    /* بنقرأ الـ interrupt status من الـ RTC عن طريق I2C */
    if (i2c_smbus_read_byte_data(rtc->client, RTC_INT_STATUS, &status))
        return IRQ_NONE;  /* I2C read فشلت */

    if (!(status & RTC_INT_ALARM))
        return IRQ_NONE;  /* مش interrupt بتاعنا */

    rtc_update_irq(rtc->rtc_dev, 1, RTC_AF | RTC_IRQF);
    return IRQ_HANDLED;
}
```

المشكلة: أثناء الـ bring-up، الـ I2C bus address للـ RTC اتكتب غلط في الـ Device Tree:

```dts
/* غلط — الـ DS1307 address المفروض 0x68 */
rtc@69 {
    compatible = "dallas,ds1307";
    reg = <0x69>;   /* <-- خطأ، المفروض 0x68 */
    interrupt-parent = <&gpio0>;
    interrupts = <5 IRQ_TYPE_LEVEL_LOW>;
};
```

لأن الـ I2C address غلط، كل `i2c_smbus_read_byte_data` بترجع error، فالـ `rtc_isr` دايمًا بيرجع `IRQ_NONE`. الـ kernel مش بيشكي لأن الـ interrupt اتعالج من الـ PMIC أو الـ temp sensor (اللي عندهم interrupts حقيقية). الـ RTC نفسه مش بيشتغل لأن الـ driver initialization بتعتمد على الـ first successful interrupt.

#### التشخيص خطوة بخطوة

```bash
# 1. تأكد إن الـ RTC موجود على الـ I2C bus
i2cdetect -y 2
# المفروض تشوف 0x68 مش 0x69

# 2. شوف الـ IRQ handlers المسجّلة
cat /proc/interrupts | grep "rtc\|pmic\|temp"

# 3. شوف الـ IRQ_NONE count
cat /proc/irq/42/spurious

# 4. جرّب تقرأ من الـ RTC مباشرة
i2cget -y 2 0x68 0x00
```

#### الحل

تصحيح الـ DT:
```dts
rtc@68 {
    compatible = "dallas,ds1307";
    reg = <0x68>;   /* الـ address الصح */
    interrupt-parent = <&gpio0>;
    interrupts = <5 IRQ_TYPE_LEVEL_LOW>;
};
```

وتحسين الـ driver للـ logging:

```c
static irqreturn_t rtc_isr(int irq, void *data)
{
    struct rtc_dev *rtc = data;
    u8 status;
    int ret;

    ret = i2c_smbus_read_byte_data(rtc->client, RTC_INT_STATUS, &status);
    if (ret < 0) {
        /* log مرة واحدة بس عشان منـfloodش الـ log */
        dev_err_ratelimited(rtc->dev, "I2C read failed: %d\n", ret);
        return IRQ_NONE;
    }

    if (!(status & RTC_INT_ALARM))
        return IRQ_NONE;

    rtc_update_irq(rtc->rtc_dev, 1, RTC_AF | RTC_IRQF);
    return IRQ_HANDLED;
}
```

#### الدرس المستفاد
في الـ shared interrupts، الـ `IRQ_NONE` بيتم بصمت — الـ kernel مش بيشكي لأن device تاني بيعالج الـ interrupt. دايمًا أثناء الـ bring-up، تحقق من الـ I2C/SPI addresses في الـ DT قبل ما تبدأ تشخّص الـ interrupt behavior. استخدم `i2cdetect` كأول خطوة لأي I2C device مش بيشتغل.
## Phase 7: مصادر ومراجع

### مصادر LWN.net

دي أهم المقالات على LWN.net اللي بتتكلم عن آلية الـ interrupt handling وعلاقتها بـ `irqreturn_t`:

| المقال | الأهمية |
|--------|---------|
| [Moving interrupts to threads](https://lwn.net/Articles/302043/) | المقال الأساسي اللي شرح فكرة الـ threaded IRQ وازاي `IRQ_WAKE_THREAD` اتضاف |
| [A new generic IRQ layer](https://lwn.net/Articles/184750/) | شرح طبقة الـ generic IRQ اللي بتستخدم `irqreturn_t` كـ contract بين الـ handler والـ kernel |
| [IRQ threads](https://lwn.net/Articles/95387/) | أول نقاش عن فكرة تشغيل الـ interrupt handlers في threads منفصلة |
| [A question about threaded IRQs](https://lwn.net/Articles/311016/) | نقاش تفصيلي عن سيناريوهات الاستخدام الصح لـ `IRQ_WAKE_THREAD` |
| [Interrupts, threads, and lockdep](https://lwn.net/Articles/321663/) | بيشرح تفاعل الـ threaded IRQs مع أدوات الـ locking وأهمية القيم المرجعة الصح |
| [genirq: Forced threaded interrupt handlers](https://lwn.net/Articles/429690/) | التحديث اللي خلّى الـ threaded handlers إجبارية في بعض السيناريوهات |
| [Improving lost and spurious IRQ handling](https://lwn.net/Articles/392136/) | بيشرح ازاي الـ kernel بيراقب الـ `IRQ_NONE` عشان يكتشف الـ spurious interrupts |
| [Software interrupts and realtime](https://lwn.net/Articles/520076/) | العلاقة بين الـ softirq والـ hard IRQ return values |
| [Handling interrupts in user space](https://lwn.net/Articles/127698/) | تجربة معالجة الـ interrupts من الـ user space وليه الـ `irqreturn_t` موجودة في الـ kernel فقط |
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مثال عملي على استخدام `irqreturn_t` في الـ GPIO interrupt handlers |

---

### التوثيق الرسمي للـ Kernel

**الـ `irqreturn_t` نفسها موثقة في:**

- **الملف المصدري:** `include/linux/irqreturn.h`
- **Generic IRQ handling:** `Documentation/core-api/genericirq.rst`
  - الرابط أونلاين: [Linux generic IRQ handling](https://docs.kernel.org/core-api/genericirq.html)
- **Interrupt management:** `Documentation/driver-api/` — بيشرح `request_irq()` و`request_threaded_irq()`

**مسارات مهمة في الـ kernel source:**

```
include/linux/irqreturn.h       ← التعريف الأساسي لـ irqreturn_t
include/linux/interrupt.h       ← request_irq, free_irq, IRQ flags
kernel/irq/handle.c             ← handle_irq_event_percpu() اللي بتفحص القيمة المرجعة
kernel/irq/manage.c             ← request_threaded_irq() وإدارة IRQ_WAKE_THREAD
```

---

### كومتس مهمة في تاريخ `irqreturn_t`

| الكومت | الوصف |
|--------|-------|
| [`irqreturn_t` introduction (2.5.x era)](https://github.com/torvalds/linux/blob/master/include/linux/irqreturn.h) | أول إدخال للـ `irqreturn_t` بدل الـ `void` عشان يتحكم في الـ spurious IRQs |
| [`kernel/irq/manage.c`](https://github.com/torvalds/linux/blob/master/kernel/irq/manage.c) | الكود اللي بيتعامل مع `IRQ_WAKE_THREAD` وبيصحّى الـ thread |
| [`kernel/irq/handle.c`](https://github.com/torvalds/linux/blob/master/kernel/irq/handle.c) | الـ `handle_irq_event_percpu()` اللي بتفحص القيمة المرجعة وتتخد قرار |

---

### نقاشات Mailing List

- **LKML (Linux Kernel Mailing List Archive):** [https://lkml.org/](https://lkml.org/)
  - ابحث عن: `irqreturn`, `IRQ_WAKE_THREAD`, `request_threaded_irq`
- **Threaded IRQ priority discussion:** [PATCH: request_threaded_irq with specified priority](https://www.spinics.net/lists/kernel/msg3904118.html)
- **KernelNewbies mailing list — threaded IRQ:** [Regarding threaded irq](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2011-March/001118.html)

---

### الكتب الموصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل:** Chapter 10 — *Interrupt Handling*
- بيشرح `irqreturn_t` وقيمها ومتى ترجع كل واحدة بأمثلة كاملة
- الـ PDF متاح مجاناً: [LDD3 Chapter 10 — Interrupt Handling](https://static.lwn.net/images/pdf/LDD3/ch10.pdf)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل:** Chapter 7 — *Interrupts and Interrupt Handlers*
- بيغطي `irqreturn_t`، الـ `request_irq()`، وفلسفة الـ top half/bottom half
- مرجع: [Writing an Interrupt Handler (mirror)](https://litux.nl/mirror/kerneldevelopment/0672327201/ch06lev1sec4.html)

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل:** الفصل المخصص لـ BSP والـ device drivers
- بيشرح تكامل الـ interrupt handlers مع الـ embedded hardware وأهمية القيمة المرجعة الصح

#### Linux Device Drivers Development — John Madieu (PacktPub)
- **الفصل:** Chapter on Interrupt Management
- بيشرح `IRQ_WAKE_THREAD` بالتفصيل مع أمثلة threaded IRQ عملية
- مرجع: [Threaded IRQs — O'Reilly](https://www.oreilly.com/library/view/linux-device-drivers/9781785280009/44550685-5e57-4b2c-a482-d05d9da6c647.xhtml)

---

### صفحات KernelNewbies و eLinux

- **KernelNewbies — Internals of Interrupt Handling:**
  [https://kernelnewbies.org/KernelHacking-HOWTO/Overview_of_the_Kernel_Source_Code/Internals_of_Interrupt_Handling](https://kernelnewbies.org/KernelHacking-HOWTO/Overview_of_the_Kernel_Source_Code/Internals_of_Interrupt_Handling)
  - بيشرح الـ flow الداخلي من لما الـ hardware يبعت interrupt لحد ما الـ handler يرجع `IRQ_HANDLED`

- **eLinux — Soft IRQ Threads:**
  [https://elinux.org/Soft_IRQ_Threads](https://elinux.org/Soft_IRQ_Threads)
  - بيشرح الـ softirq threads وعلاقتها بقيم الـ `irqreturn_t`

---

### موارد تعليمية إضافية

- **Linux Kernel Labs — I/O access and Interrupts:**
  [https://linux-kernel-labs.github.io/refs/heads/master/labs/interrupts.html](https://linux-kernel-labs.github.io/refs/heads/master/labs/interrupts.html)
  — تمارين عملية بتستخدم `irqreturn_t` في كتابة الـ interrupt handlers

- **EmbeTronicX — Interrupt Example Program:**
  [https://embetronicx.com/tutorials/linux/device-drivers/linux-device-driver-tutorial-part-13-interrupt-example-program-in-linux-kernel/](https://embetronicx.com/tutorials/linux/device-drivers/linux-device-driver-tutorial-part-13-interrupt-example-program-in-linux-kernel/)
  — مثال كامل بالكود على handler بيستخدم `IRQ_HANDLED` و`IRQ_NONE`

- **EmbeTronicX — Threaded IRQ:**
  [https://embetronicx.com/tutorials/linux/device-drivers/threaded-irq-in-linux-kernel/](https://embetronicx.com/tutorials/linux/device-drivers/threaded-irq-in-linux-kernel/)
  — شرح تفصيلي مع كود كامل لـ `IRQ_WAKE_THREAD` و`request_threaded_irq()`

- **The Linux Kernel Module Programming Guide — Interrupt Handlers:**
  [https://tldp.org/LDP/lkmpg/2.6/html/x1256.html](https://tldp.org/LDP/lkmpg/2.6/html/x1256.html)

---

### Search Terms للبحث عن معلومات أكتر

لو عايز تعمق أكتر، استخدم الـ search terms دي:

```
irqreturn_t linux kernel
IRQ_WAKE_THREAD request_threaded_irq
IRQ_NONE spurious interrupt detection
linux interrupt handler return value
handle_irq_event_percpu kernel source
irq_retval macro linux
linux threaded interrupt handler bottom half
generic IRQ layer linux kernel
```

**على LKML:**
```
site:lkml.org irqreturn
site:lkml.org "IRQ_WAKE_THREAD"
site:lkml.org "request_threaded_irq"
```

**على kernel.org:**
```
site:docs.kernel.org irqreturn
site:elixir.bootlin.com irqreturn_t
```
## Phase 8: Writing simple module

### الفكرة

**`irqreturn.h`** بيحدد `enum irqreturn` والـ `typedef irqreturn_t` اللي كل interrupt handler في الـ kernel بيرجعها. مفيش exported functions مباشرة في الملف ده — هو header-only — بس الـ interrupt handlers اللي بترجع `irqreturn_t` بتتسجل عبر `request_irq()` وبتتنفذ تحت interrupt context.

أحسن hook عملي هنا: نستخدم **kprobe** على `handle_irq_event_percpu` — الدالة الجوهرية اللي بتنفذ كل interrupt handler على الـ CPU وبتستقبل `irqreturn_t` منه وتحكم على النتيجة. ده يخلينا نشوف في الـ real-time إيه اللي الـ handlers بترجعه.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_irqreturn.c
 * Hook handle_irq_event_percpu to observe irqreturn_t values live.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit          */
#include <linux/kernel.h>       /* pr_info, pr_fmt                           */
#include <linux/kprobes.h>      /* kprobe struct, register/unregister_kprobe */
#include <linux/irqreturn.h>    /* irqreturn_t, IRQ_HANDLED, IRQ_NONE, ...   */
#include <linux/interrupt.h>    /* irqaction struct (handler pointer inside) */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs Project");
MODULE_DESCRIPTION("kprobe on handle_irq_event_percpu to observe irqreturn_t");

/*
 * handle_irq_event_percpu signature (kernel >= 5.x):
 *   irqreturn_t handle_irq_event_percpu(struct irq_desc *desc);
 *
 * We probe the RETURN point (kretprobe) to capture the irqreturn_t value
 * the function hands back — that value is the OR of all chained handlers.
 */

/* ------------------------------------------------------------------ */
/*  kretprobe — fires AFTER handle_irq_event_percpu returns           */
/* ------------------------------------------------------------------ */

static int ret_handler(struct kretprobe_instance *ri, struct pt_regs *regs)
{
    /* regs_return_value() gives us the function's return value (AX on x86) */
    irqreturn_t ret = (irqreturn_t)regs_return_value(regs);

    const char *retname;

    switch (ret) {
    case IRQ_NONE:
        retname = "IRQ_NONE";
        break;
    case IRQ_HANDLED:
        retname = "IRQ_HANDLED";
        break;
    case IRQ_WAKE_THREAD:
        retname = "IRQ_WAKE_THREAD";
        break;
    default:
        retname = "UNKNOWN";
        break;
    }

    /* Print the raw numeric value and its symbolic name */
    pr_info("kprobe_irqreturn: handle_irq_event_percpu returned %d (%s)\n",
            ret, retname);

    return 0; /* must return 0 — non-zero aborts the kretprobe */
}

/* ------------------------------------------------------------------ */
/*  kretprobe descriptor                                               */
/* ------------------------------------------------------------------ */

static struct kretprobe my_kretprobe = {
    .handler     = ret_handler,    /* called on function return      */
    .maxactive   = 20,             /* max concurrent instances       */
    .kp = {
        .symbol_name = "handle_irq_event_percpu",
    },
};

/* ------------------------------------------------------------------ */
/*  module_init                                                        */
/* ------------------------------------------------------------------ */

static int __init kprobe_irqreturn_init(void)
{
    int ret;

    ret = register_kretprobe(&my_kretprobe);
    if (ret < 0) {
        pr_err("kprobe_irqreturn: register_kretprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_irqreturn: planted kretprobe on %s\n",
            my_kretprobe.kp.symbol_name);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                        */
/* ------------------------------------------------------------------ */

static void __exit kprobe_irqreturn_exit(void)
{
    unregister_kretprobe(&my_kretprobe);
    pr_info("kprobe_irqreturn: kretprobe removed. missed=%d\n",
            my_kretprobe.nmissed);
}

module_init(kprobe_irqreturn_init);
module_exit(kprobe_irqreturn_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | لازم لأي module — بيوفر `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info`, `pr_err` |
| `linux/kprobes.h` | تعريف `kretprobe`, `register_kretprobe`, `unregister_kretprobe` |
| `linux/irqreturn.h` | `irqreturn_t` والـ constants `IRQ_NONE`, `IRQ_HANDLED`, `IRQ_WAKE_THREAD` |
| `linux/interrupt.h` | context إضافي عن الـ interrupt subsystem |

---

#### الـ `ret_handler` callback

**الـ `kretprobe`** بيشتغل بعد ما الدالة المستهدفة ترجع، فـ `regs_return_value(regs)` بيجيب قيمة الـ return (register `rax` على x86-64) اللي هي الـ `irqreturn_t`.

الـ `switch` بيحول القيمة الرقمية لاسم symbolic مقروء، وبعدين `pr_info` بيطبعهم مع بعض — ده بيسمح لك تتابع في `dmesg` كل interrupt handler رجع إيه.

---

#### الـ `kretprobe` struct

**`maxactive = 20`** معناها إن الـ kprobe يقدر يشتغل على 20 CPU في نفس الوقت بدون ما يفوّت أي event. لو عدد الـ CPUs أكبر، الـ `nmissed` counter بيتزود وبنطبعه في الـ exit عشان نعرف اتفوّت أد إيه.

---

#### الـ `module_init`

`register_kretprobe` بتزرع الـ hook وبترجع error code لو الـ symbol مش موجود أو الـ kernel مش بيدعم kprobes — لازم نتعامل مع الـ error ونرجعه عشان الـ module ما يفضلش loaded بحالة مكسورة.

---

#### الـ `module_exit`

`unregister_kretprobe` ضروري جداً — لو ما اتعملش والـ module اتـ unload، الـ kernel هيحاول ينفذ كود اتشال من الذاكرة وده بيعمل kernel panic فوري. طباعة `nmissed` بتساعد في debugging لو الـ system كان تحت load عالي.

---

### Makefile وطريقة التشغيل

```makefile
obj-m += kprobe_irqreturn.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# بناء وتحميل الـ module
make
sudo insmod kprobe_irqreturn.ko

# مراقبة الـ output
sudo dmesg -w | grep kprobe_irqreturn

# إزالة الـ module
sudo rmmod kprobe_irqreturn
```

---

### مثال على الـ output المتوقع

```
[  123.456] kprobe_irqreturn: planted kretprobe on handle_irq_event_percpu
[  123.789] kprobe_irqreturn: handle_irq_event_percpu returned 1 (IRQ_HANDLED)
[  123.790] kprobe_irqreturn: handle_irq_event_percpu returned 0 (IRQ_NONE)
[  124.100] kprobe_irqreturn: handle_irq_event_percpu returned 2 (IRQ_WAKE_THREAD)
...
[  200.000] kprobe_irqreturn: kretprobe removed. missed=0
```

**الـ `IRQ_NONE`** بيظهر لما interrupt وصل للـ handler بس مش بتاعه (shared IRQ lines).
**الـ `IRQ_WAKE_THREAD`** بيظهر مع الـ threaded interrupts اللي بتحتاج processing تانية خارج interrupt context.
