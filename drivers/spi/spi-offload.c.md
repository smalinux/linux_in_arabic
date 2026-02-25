## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ SPI Offload** subsystem — جزء من SPI SUBSYSTEM في الـ Linux kernel، لكنه subsystem مستقل بـ maintainer خاص (David Lechner من BayLibre). الملف ده هو **قلب** هذا الـ subsystem.

---

### الـ SPI Protocol — المقدمة النظرية

**الـ SPI (Serial Peripheral Interface)** هو بروتوكول تواصل بين الـ CPU وأجهزة خارجية زي الـ ADC sensors والـ DAC والـ flash chips. الأساس إن الـ CPU هو اللي يتحكم في كل حاجة — يبعت بيانات، يستقبل، يتحكم في التوقيت.

المشكلة: في بعض التطبيقات الـ CPU بيكون مشغول جداً أو التطبيق محتاج سرعة عالية جداً. تخيل sensor بيقرأ بيانات **مليون مرة في الثانية** — الـ CPU مش هيعرف يواكب ده بدون ما يبوظ أي حاجة تانية في النظام.

---

### القصة — المشكلة اللي بتحلها

تخيل عندك **oscilloscope رقمي** أو **ADC سريع** (زي اللي بتعمله Analog Devices) محتاج تجمع بيانات بسرعة 10 ميغا sample في الثانية عبر SPI. كل sample محتاج SPI transfer:

```
CPU بدون offload:
┌─────┐   interrupt   ┌─────┐   SPI transfer   ┌─────────┐
│ CPU │ ←────────────│Timer│ ─────────────────→│ADC Chip │
│     │   (1MHz rate) └─────┘                  └─────────┘
│     │ ← يصحى كل microsecond → يبعت أمر SPI → ينتظر → يقرأ
└─────┘
النتيجة: CPU مشغول 100% بس بـ SPI transfers فقط!
```

الحل — **الـ SPI Offload**:

```
CPU مع offload:
┌─────┐  "ابدأ التشغيل"  ┌─────────────┐  SPI transfers  ┌─────────┐
│ CPU │ ─────────────────→│ SPI Hardware│ ────────────────→│ADC Chip │
│     │                   │  Controller │                  └─────────┘
│     │                   │  (يشتغل     │       ↓
│     │                   │  لوحده!)    │  DMA Stream
└─────┘                   └─────────────┘       ↓
CPU حر يعمل حاجات تانية              ┌──────────────┐
                                      │ Memory/Buffer│
                                      └──────────────┘
```

الـ SPI controller الذكي بيعمل الـ transfers لوحده بدون CPU intervention — إما بـ hardware trigger أو بـ MCU processor تاني جوا الـ chip.

---

### هدف الملف `spi-offload.c`

الملف ده هو **الـ core infrastructure** — يوفر الـ API المشتركة اللي:

1. **الـ Providers** (SPI controller drivers) يستخدموها عشان يسجّلوا قدرتهم على الـ offload.
2. **الـ Consumers** (peripheral drivers زي ADC driver) يستخدموها عشان يطلبوا الـ offload ويتحكموا فيه.

الملف بيعمل **وسيط (broker)** بين الطرفين:

```
┌──────────────────────────────────────────────────────┐
│              spi-offload.c (الـ Core)                │
│                                                      │
│  Provider API          Consumer API                  │
│  ───────────           ────────────                  │
│  devm_spi_offload_alloc()    devm_spi_offload_get()  │
│  devm_spi_offload_trigger_register()                 │
│                        devm_spi_offload_trigger_get()│
│                        spi_offload_trigger_enable()  │
│                        spi_offload_trigger_disable() │
│                        devm_spi_offload_*_dma_chan()  │
└──────────────────────────────────────────────────────┘
         ↑                              ↑
┌────────────────┐            ┌─────────────────────┐
│  SPI Controller│            │  Peripheral Driver  │
│  Driver        │            │  (مثلاً ADC driver) │
│  (Provider)    │            │  (Consumer)         │
└────────────────┘            └─────────────────────┘
```

---

### المفاهيم الأساسية في الملف

#### 1. الـ Offload Instance (`struct spi_offload`)
ده الكيان الأساسي — بيمثل "فرصة offload واحدة" جوا الـ SPI controller. الـ controller ممكن يدعم أكتر من offload instance واحدة في نفس الوقت.

#### 2. الـ Trigger (`struct spi_offload_trigger`)
مين اللي بيقول للـ offload "ابدأ transfer دلوقتي"؟ ده الـ trigger. في نوعين:
- **`SPI_OFFLOAD_TRIGGER_DATA_READY`**: الـ sensor نفسه بيقول "أنا جاهز، اقرأني."
- **`SPI_OFFLOAD_TRIGGER_PERIODIC`**: clock منتظم بيضرب كل X microseconds.

الـ triggers بتتسجل في **global list** (`spi_offload_triggers`) وبتتبحث عنها بالـ `fwnode` (Device Tree node).

#### 3. الـ DMA Streams
لما بتجمع بيانات بسرعة، مش عايز الـ CPU يحرّك كل byte. الـ offload بيدعم:
- **TX Stream**: بيانات بتيجي من DMA مباشرة للـ SPI للـ chip.
- **RX Stream**: بيانات بتيجي من الـ chip بتروح لـ DMA مباشرة للـ memory.

---

### الـ Capabilities Flags

```c
SPI_OFFLOAD_CAP_TRIGGER        // الـ hardware يقدر يتشغّل بـ external trigger
SPI_OFFLOAD_CAP_TX_STATIC_DATA // يقدر يسجّل TX data ويرددها
SPI_OFFLOAD_CAP_TX_STREAM_DMA  // TX بيجي من DMA stream
SPI_OFFLOAD_CAP_RX_STREAM_DMA  // RX بيروح لـ DMA stream
```

الـ consumer بيقول "عايز offload بيدعم كذا" عن طريق `spi_offload_config.capability_flags`.

---

### إزاي الـ Consumer بيستخدم الـ Offload — سيناريو كامل

```c
// 1. الـ ADC driver بيطلب offload
struct spi_offload *offload = devm_spi_offload_get(dev, spi, &config);

// 2. بيجيب الـ trigger (مثلاً: periodic clock من PWM)
struct spi_offload_trigger *trigger =
    devm_spi_offload_trigger_get(dev, offload, SPI_OFFLOAD_TRIGGER_PERIODIC);

// 3. بيعمل validate إن الـ hardware يقدر يشتغل بالـ frequency المطلوبة
spi_offload_trigger_validate(trigger, &trigger_config);

// 4. بيجيب الـ DMA channel عشان يستقبل البيانات
struct dma_chan *rx_chan =
    devm_spi_offload_rx_stream_request_dma_chan(dev, offload);

// 5. بيشغّل الـ trigger → الـ hardware يبدأ يشتغل لوحده
spi_offload_trigger_enable(offload, trigger, &trigger_config);

// ... الـ CPU حر يعمل حاجات تانية، البيانات بتيجي تلقائياً ...

// 6. لما خلص
spi_offload_trigger_disable(offload, trigger);
```

---

### الـ Provider/Consumer Architecture

الملف بيستخدم نفس فكرة الـ Linux frameworks التانية (زي clk framework وregulator framework):

| الجانب | الدور | مثال |
|--------|-------|-------|
| **Provider** | يعلن عن قدرته على الـ offload | AXI SPI Engine driver |
| **Consumer** | يطلب الـ offload ويستخدمه | ADC peripheral driver |
| **Core** (الملف ده) | يوصّل بينهم | `spi-offload.c` |

---

### الملفات المكوِّنة للـ Subsystem

#### الـ Core والـ Headers

| الملف | الدور |
|-------|-------|
| `drivers/spi/spi-offload.c` | الـ core — الملف ده نفسه |
| `include/linux/spi/offload/types.h` | الـ structs والـ enums المشتركة |
| `include/linux/spi/offload/consumer.h` | الـ API للـ consumer drivers |
| `include/linux/spi/offload/provider.h` | الـ API للـ provider drivers |

#### الـ Trigger Drivers

| الملف | الدور |
|-------|-------|
| `drivers/spi/spi-offload-trigger-pwm.c` | Trigger بناءً على PWM (periodic clock) |
| `drivers/spi/spi-offload-trigger-adi-util-sigma-delta.c` | Trigger خاص بـ Analog Devices Sigma-Delta converters |

#### الـ SPI Controllers اللي بتدعم الـ Offload (Providers)

| الملف | الدور |
|-------|-------|
| `drivers/spi/spi-axi-spi-engine.c` | AXI SPI Engine من Analog Devices — أهم provider |

#### الـ SPI Subsystem الأصلي (للخلفية)

| الملف | الدور |
|-------|-------|
| `drivers/spi/spi.c` | الـ SPI core الأساسي |
| `include/linux/spi/spi.h` | الـ main SPI types (فيها `spi_controller` و`spi_device`) |

---

### ملاحظة على الـ Design

الملف بيستخدم **`devm_` pattern** في كل حاجة — يعني أي resource بيتخصّص بيتحرر أوتوماتيك لما الـ device يتشال. ده يمنع الـ memory leaks في الـ driver lifecycle.

كمان الـ triggers بتستخدم **`kref`** (reference counting) عشان لو الـ provider راح بينما الـ consumer لسه شايل reference، النظام مش بيكرش — بس كل العمليات بتفشل بـ `-ENODEV`.
## Phase 2: شرح الـ SPI Offload Framework

### المشكلة — ليه الـ Framework ده موجود أصلاً؟

في الـ embedded systems، بعض الـ peripherals بتتطلب قراءة بيانات بشكل مستمر وبـ rate عالية جداً من الـ SPI bus — زي ADC بيشتغل على 10 MHz أو أكتر. الـ flow العادي بيبقى كده:

```
CPU → kernel SPI driver → transfer → interrupt → CPU يقرأ البيانات → يكررها
```

**المشكلة الحقيقية:** الـ CPU بيضيع جزء كبير من وقته في:
- scheduling الـ SPI transfers
- معالجة الـ interrupts لكل transfer
- نقل البيانات من/إلى الـ buffers

لو الـ ADC بيشتغل على 1 MSPS وكل sample محتاجة SPI transfer → الـ CPU بيتعامل مع **مليون interrupt في الثانية**. ده مستحيل في أي نظام real-time أو حتى نظام عادي.

بعض الـ SPI controllers الحديثة (زي اللي في Analog Devices أو FPGA-based designs) بتدعم **hardware offload**: الـ controller نفسه بيعمل الـ transfers بشكل autonomous من غير تدخل الـ CPU، ويبعت البيانات مباشرةً لـ DMA buffer.

---

### الحل — مدخل الـ Kernel

**الـ SPI Offload Framework** هو abstraction layer بيتيح لـ SPI controller drivers إنها تعلن عن قدراتها الـ offload، وبيتيح لـ peripheral drivers (الـ consumers) إنهم يستخدموا الـ offload دي بطريقة موحدة.

الـ framework بيحل ثلاث مشاكل:

1. **Capability negotiation** — الـ peripheral driver بيطلب offload بخصائص معينة، والـ framework بيجيب اللي يناسب
2. **Trigger management** — مين يبدأ الـ transfer؟ hardware event؟ periodic timer? الـ framework بيعزل ده عن الـ consumer
3. **Data streaming** — الـ framework بيوصّل الـ consumer بـ DMA channels الخاصة بالـ offload

---

### Real-World Analogy — خط التجميع في المصنع

تخيل **مصنع بيتعامل مع طلبيات**:

| عنصر المصنع | المقابل في الـ Kernel |
|---|---|
| **المدير** (الـ CPU) | يشتغل طول الوقت في كل طلبية = مشكلة | CPU يعمل كل SPI transfer = مشكلة |
| **خط تجميع أوتوماتيكي** | بيشتغل لوحده من غير المدير | SPI Offload Engine في الـ controller |
| **إشارة الشيفت** (trigger) | بتوقيت محدد أو عند الطلب | `spi_offload_trigger` |
| **الـ conveyor belt للإنتاج** | بينقل المنتج النهائي للمخزن | DMA RX stream |
| **مواصفات الطلبية** | لازم تتوافق مع قدرة الخط | `spi_offload_config` |
| **عقد العمل مع المصنع** | يتجدد وينتهي بشكل منظم | `devm_spi_offload_get` / `put` |
| **مشرف الخط** | حلقة الوصل بين المدير والخط | `spi_offload` struct |

التفصيل العميق للـ analogy:
- **المدير (CPU)** بيوقّع عقد مع خط التجميع مرة واحدة (`devm_spi_offload_get`)، وبعدين يقول "ابدأ" (`trigger_enable`) ويمشي
- **إشارة الشيفت (trigger)** ممكن تكون:
  - ساعة دقيقة بتدق كل X ميلي ثانية → `SPI_OFFLOAD_TRIGGER_PERIODIC`
  - رنة تليفون من العميل نفسه لما البضاعة جاهزة → `SPI_OFFLOAD_TRIGGER_DATA_READY`
- **الـ conveyor belt (DMA stream)** بيشيل المنتج (البيانات) مباشرة للمخزن (RAM buffer) من غير ما المدير يلمسه
- لو **المصنع اتباع (driver unregistered)** والخط لسه شغال، الـ framework بيشيل الـ `ops` ويرجع `-ENODEV` لأي call جديدة — زي ما المصنع يقول "مافيش عقد جديد"

---

### البنية العامة — Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     PERIPHERAL DRIVER                           │
│              (e.g., ADC driver: ad7768)                         │
│                                                                 │
│   devm_spi_offload_get()        devm_spi_offload_trigger_get()  │
│   devm_spi_offload_rx_stream_request_dma_chan()                 │
└──────────────┬───────────────────────────┬──────────────────────┘
               │   Consumer API            │
               │  (consumer.h)             │
               ▼                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                  SPI OFFLOAD FRAMEWORK CORE                      │
│                   (spi-offload.c)                                │
│                                                                  │
│  ┌──────────────────┐      ┌────────────────────────────────┐   │
│  │ spi_offload       │      │ spi_offload_trigger            │   │
│  │ ┌──────────────┐ │      │ ┌──────────────────────────┐   │   │
│  │ │ provider_dev  │ │      │ │ fwnode                   │   │   │
│  │ │ priv          │ │      │ │ ops (match/req/enable..) │   │   │
│  │ │ ops           │ │      │ │ kref (reference count)   │   │   │
│  │ │ xfer_flags    │ │      │ │ mutex (concurrency safe) │   │   │
│  │ └──────────────┘ │      │ └──────────────────────────┘   │   │
│  └──────────────────┘      └────────────────────────────────┘   │
│                                    │                             │
│              Global list: spi_offload_triggers (linked list)     │
└──────────────────┬────────────────────────────┬─────────────────┘
                   │   Provider API             │
                   │  (provider.h)              │
                   ▼                            ▼
┌─────────────────────────────┐   ┌─────────────────────────────┐
│   SPI CONTROLLER DRIVER      │   │   TRIGGER PROVIDER DRIVER   │
│   (e.g., spi-adi-3wire.c)   │   │   (e.g., PWM, timer, GPIO)  │
│                              │   │                             │
│  .get_offload()              │   │  devm_spi_offload_trigger   │
│  .put_offload()              │   │  _register()                │
│  spi_offload_ops:            │   │  spi_offload_trigger_ops:   │
│    .trigger_enable()         │   │    .match()                 │
│    .trigger_disable()        │   │    .request()               │
│    .rx_stream_request_dma_chan│   │    .validate()              │
│    .tx_stream_request_dma_chan│   │    .enable()                │
└──────────────┬───────────────┘   │    .disable()               │
               │                   └─────────────┬───────────────┘
               ▼                                 ▼
┌──────────────────────────┐      ┌──────────────────────────────┐
│   SPI HARDWARE ENGINE    │      │   TRIGGER HARDWARE           │
│   (FIFO, state machine,  │      │   (PWM block, timer, DRQ pin)│
│    DMA interface)        │      │                              │
└──────────────────────────┘      └──────────────────────────────┘
               │                                 │
               └──────────────┬──────────────────┘
                              ▼
                 ┌───────────────────────┐
                 │   DMA ENGINE          │
                 │  (dmaengine framework)│
                 │  TX/RX streams        │
                 └───────────┬───────────┘
                             ▼
                    ┌─────────────────┐
                    │   RAM BUFFER    │
                    │  (user-space or │
                    │   kernel buffer)│
                    └─────────────────┘
```

**ملاحظة:** الـ **DMA Engine framework** هو subsystem منفصل في الـ kernel بيوفر abstraction للـ DMA controllers — الـ SPI Offload بيستخدمه لنقل البيانات من/إلى الـ offload engine من غير تدخل الـ CPU.

---

### الـ Core Abstraction — الفكرة المحورية

الـ framework بيبني على فكرة **فصل ثلاث مسؤوليات** بشكل تام:

```
┌──────────────┐   ┌─────────────────┐   ┌──────────────────┐
│  WHAT to     │   │  WHEN to        │   │  WHERE data      │
│  transfer    │   │  transfer       │   │  goes            │
│              │   │                 │   │                  │
│  spi_message │   │ spi_offload     │   │ dma_chan         │
│  (prepared   │   │ _trigger        │   │ (TX/RX stream)   │
│  by consumer)│   │                 │   │                  │
└──────────────┘   └─────────────────┘   └──────────────────┘
      ▲                    ▲                       ▲
      │                    │                       │
      └────────────────────┼───────────────────────┘
                           │
              ┌────────────────────────┐
              │   SPI OFFLOAD ENGINE   │
              │   (hardware + driver)  │
              └────────────────────────┘
```

الـ **central abstraction** هي إن الـ CPU بيعمل **setup مرة واحدة** ثم:
- **يسلّم التحكم للـ hardware** عن طريق `trigger_enable`
- الـ hardware بيعمل الـ transfers لوحده
- البيانات بتتدفق مباشرة عبر DMA لـ RAM
- الـ CPU يتدخل بس لما يعوز **يوقف** الـ offload

---

### الـ Structs وعلاقتهاببعض

```
spi_offload_config               spi_offload_trigger_config
┌───────────────────┐            ┌──────────────────────────────┐
│ capability_flags  │            │ type (DATA_READY / PERIODIC) │
│ (CAP_TRIGGER,     │            │ union {                      │
│  CAP_TX_STREAM,   │            │   periodic: {                │
│  CAP_RX_STREAM,   │            │     frequency_hz             │
│  CAP_TX_STATIC)   │            │     offset_ns                │
└─────────┬─────────┘            │   }                          │
          │ used in              │ }                            │
          ▼                      └──────────┬───────────────────┘
┌─────────────────────────┐                 │ used in
│    spi_offload           │                 ▼
│ ┌─────────────────────┐ │     ┌──────────────────────────────┐
│ │ provider_dev         │ │     │   spi_offload_trigger        │
│ │ priv (hw state)      │ │     │ ┌──────────────────────────┐ │
│ │ ops ──────────────┐  │ │     │ │ list (global linked list)│ │
│ │ xfer_flags        │  │ │     │ │ kref (refcount)          │ │
│ └───────────────────│──┘ │     │ │ fwnode (DT node match)   │ │
└────────────────────┬│────┘     │ │ lock (mutex)             │ │
                     ││          │ │ ops ──────────────────┐  │ │
                     ▼▼          │ │ priv (hw state)       │  │ │
         ┌───────────────────┐   │ └───────────────────────│──┘ │
         │ spi_offload_ops   │   └───────────────────────┬─│────┘
         │ .trigger_enable() │                           │ │
         │ .trigger_disable()│                           ▼ ▼
         │ .tx_stream_dma()  │           ┌──────────────────────────┐
         │ .rx_stream_dma()  │           │ spi_offload_trigger_ops  │
         └───────────────────┘           │ .match()  (فين الـtrigger)│
                                         │ .request() (احجز)        │
                                         │ .release() (حرّر)        │
                                         │ .validate() (اتأكد)      │
                                         │ .enable()  (شغّل)        │
                                         │ .disable() (وقّف)        │
                                         └──────────────────────────┘
```

---

### الـ Two-Tier Ops Design

الـ framework عنده مستويين من الـ ops لأن الـ trigger والـ offload ممكن يكونوا في drivers مختلفة:

**المستوى الأول — `spi_offload_ops`** (بتنفذها الـ SPI controller):
- بتعمل enable/disable للـ offload engine الخاص بالـ controller
- بتديك الـ DMA channels للـ TX/RX streams

**المستوى الثاني — `spi_offload_trigger_ops`** (بتنفذها الـ trigger provider):
- `match()` — لازم يكون موجود دايماً: بيقول "أنا الـ trigger المطلوب ده"
- `request()` / `release()` — احجز/حرّر resources الـ trigger
- `validate()` — تحقق من الـ config وعدّلها لو محتاج (مثلاً round الـ frequency للأقرب)
- `enable()` / `disable()` — شغّل/وقّف الـ trigger فعلياً

الـ `spi_offload_trigger_enable()` بتعمل **كالاتالي بالترتيب**:
```c
// 1. شغّل الـ offload engine في الـ SPI controller
offload->ops->trigger_enable(offload);

// 2. شغّل الـ trigger الخارجي
trigger->ops->enable(trigger, config);
```

لو الخطوة 2 فشلت، بيعمل rollback تلقائي ويعمل `trigger_disable` على الـ offload. ده **atomic-safe** بالنسبة للـ consumer.

---

### الـ Trigger Discovery — إزاي الـ Consumer بيلاقي الـ Trigger؟

ده بيحصل عبر **Device Tree (Open Firmware)** — وهو subsystem بيوصف الـ hardware topology في شكل tree من الـ nodes. كل node ليها properties.

```
// مثال Device Tree
adc@0 {
    compatible = "adi,ad7768";
    // الـ ADC بيشير للـ timer اللي هيعمل trigger
    trigger-sources = <&timer0 2>;
    #trigger-source-cells = <1>;
};

timer0: timer@40000000 {
    compatible = "adi,axi-pwm-gen";
    #trigger-source-cells = <1>;
};
```

الـ `devm_spi_offload_trigger_get()` بتعمل:
1. `fwnode_property_get_reference_args()` — تقرأ الـ `trigger-sources` property من الـ Device Tree
2. بتجيب الـ `fwnode` والـ `args` (البيانات الإضافية زي channel number)
3. بتعمل iteration على `spi_offload_triggers` list
4. لكل trigger بتسأله `ops->match()`: "أنت مطابق لـ fwnode ده وبالـ args دي؟"
5. لو اتطابق → بتعمل `ops->request()` وترجع الـ trigger للـ consumer

لو مالقتش مطابق → `-EPROBE_DEFER` — الـ kernel بيفهم إن الـ trigger driver مش registered لسه، وبيحاول تاني بعد كده.

---

### الـ Reference Counting والـ Lifecycle

الـ `spi_offload_trigger` بتستخدم `kref` لأن:
- الـ consumer بياخد reference
- الـ provider driver ممكن يتـ unregister في أي وقت

```
Provider registers                Consumer gets trigger
      │                                    │
      ▼                                    ▼
kref_init(&trigger->ref)        kref_get(&trigger->ref)
(ref = 1)                       (ref = 2)
      │                                    │
      │                          Consumer releases:
      │                          spi_offload_trigger_put()
      │                               │
      │                          ops->release()  ← لو provider لسه موجود
      │                          kref_put()      ← (ref = 1)
      │
Provider unregisters:
spi_offload_trigger_unregister()
      │
      ├── list_del()           ← مش هيتلاقى في discovery جديد
      ├── trigger->ops = NULL  ← أي call جديدة هترجع -ENODEV
      ├── trigger->priv = NULL
      └── kref_put()           ← لو ref وصلت صفر → kfree()
                                  لو لسه فيه consumer → بيستنى هو يعمل kfree
```

الـ `mutex` على الـ trigger نفسه بيحمي الـ race condition بين:
- `ops->enable()` بيتنفذ
- `spi_offload_trigger_unregister()` بيمسح الـ ops في نفس الوقت

---

### ملكية الـ Framework — ما يملكه وما يفوّضه

| المسؤولية | الـ Framework يملكها | بتفوّضها لـ |
|---|---|---|
| **Lifecycle management** (alloc/free) | نعم — `devm_*` wrappers | — |
| **Trigger discovery** عبر Device Tree | نعم — `fwnode` matching | — |
| **Reference counting** للـ triggers | نعم — `kref` | — |
| **Thread safety** للـ trigger ops | نعم — `mutex` | — |
| **Hardware-specific offload setup** | لا | SPI controller driver |
| **Trigger hardware control** (PWM, timer) | لا | Trigger provider driver |
| **DMA channel allocation** | لا | SPI controller driver ops |
| **SPI transfer content** | لا | Peripheral (consumer) driver |
| **Data processing بعد الـ DMA** | لا | Consumer driver / IIO subsystem |

---

### مثال عملي — ADC يشتغل على 1 MSPS

```c
/* في الـ ADC driver (consumer) */

static int ad7768_probe(struct spi_device *spi)
{
    struct spi_offload_config config = {
        /* نحتاج trigger + RX stream عبر DMA */
        .capability_flags = SPI_OFFLOAD_CAP_TRIGGER |
                            SPI_OFFLOAD_CAP_RX_STREAM_DMA,
    };

    /* اطلب offload instance يدعم الـ capabilities دي */
    offload = devm_spi_offload_get(&spi->dev, spi, &config);
    if (IS_ERR(offload))
        return PTR_ERR(offload); /* -ENODEV لو الـ controller مش بيدعم */

    /* اطلب الـ trigger من الـ Device Tree */
    trigger = devm_spi_offload_trigger_get(&spi->dev, offload,
                                           SPI_OFFLOAD_TRIGGER_PERIODIC);
    if (IS_ERR(trigger))
        return PTR_ERR(trigger);

    /* اطلب الـ DMA channel لاستقبال البيانات */
    rx_chan = devm_spi_offload_rx_stream_request_dma_chan(&spi->dev, offload);
    if (IS_ERR(rx_chan))
        return PTR_ERR(rx_chan);

    /* ... setup الـ DMA descriptor ... */
}

static int ad7768_start_sampling(struct ad7768_dev *adc, u64 rate_hz)
{
    struct spi_offload_trigger_config config = {
        .type = SPI_OFFLOAD_TRIGGER_PERIODIC,
        .periodic = {
            .frequency_hz = rate_hz,
            .offset_ns = 0,
        },
    };

    /* تحقق إن الـ hardware يدعم الـ frequency دي (ممكن يعدّلها) */
    ret = spi_offload_trigger_validate(adc->trigger, &config);
    if (ret)
        return ret;

    /*
     * من هنا الـ CPU مش محتاج يتدخل:
     * الـ PWM/timer هيبعت trigger → الـ SPI engine هيعمل transfer
     * → البيانات تروح مباشرةً عبر DMA للـ buffer
     */
    return spi_offload_trigger_enable(adc->offload, adc->trigger, &config);
}
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### Transfer Flags — `SPI_OFFLOAD_XFER_*` (بيتحط في `spi_transfer.offload_flags`)

| Flag | Value | المعنى |
|------|-------|--------|
| `SPI_OFFLOAD_XFER_TX_STREAM` | `BIT(0)` | الـ TX data مش من `tx_buf` — بييجي من external DMA stream |
| `SPI_OFFLOAD_XFER_RX_STREAM` | `BIT(1)` | الـ RX data مش في `rx_buf` — بيتبعت لـ external DMA stream |

#### Capability Flags — `SPI_OFFLOAD_CAP_*` (بيتحط في `spi_offload_config.capability_flags`)

| Flag | Value | المعنى |
|------|-------|--------|
| `SPI_OFFLOAD_CAP_TRIGGER` | `BIT(0)` | الـ offload بيدعم hardware trigger خارجي |
| `SPI_OFFLOAD_CAP_TX_STATIC_DATA` | `BIT(1)` | الـ controller يقدر يحفظ TX data ويعيد تشغيلها |
| `SPI_OFFLOAD_CAP_TX_STREAM_DMA` | `BIT(2)` | الـ TX data بيجي من external DMA stream |
| `SPI_OFFLOAD_CAP_RX_STREAM_DMA` | `BIT(3)` | الـ RX data بيتبعت لـ external DMA stream |

#### Trigger Types — `enum spi_offload_trigger_type`

| Value | المعنى |
|-------|--------|
| `SPI_OFFLOAD_TRIGGER_DATA_READY` | الـ peripheral يشير إن في data جاهزة للقراءة |
| `SPI_OFFLOAD_TRIGGER_PERIODIC` | trigger دوري — زي clock بتردد ثابت |

---

### الـ Structs المهمة

#### 1. `struct spi_offload` — القلب الأساسي

الـ **offload instance** — بيمثل "صلاحية" واحدة للاستخدام بدون CPU.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `provider_dev` | `struct device *` | الـ device بتاع الـ provider (SPI controller) — بيُستخدم لـ get/put reference |
| `priv` | `void *` | private data خاصة بالـ provider driver — بيُخصص مع الـ offload في `devm_spi_offload_alloc` |
| `ops` | `const struct spi_offload_ops *` | callback table — الـ provider يحدد اللي بيعمله |
| `xfer_flags` | `u32` | الـ `SPI_OFFLOAD_XFER_*` flags اللي الـ provider بيدعمها |

---

#### 2. `struct spi_offload_ops` — الـ vtable بتاع الـ offload provider

| Callback | متى بيتاستخدم |
|----------|--------------|
| `trigger_enable(offload)` | لما الـ consumer يطلب تفعيل الـ hardware trigger |
| `trigger_disable(offload)` | لما الـ consumer يوقف الـ trigger |
| `tx_stream_request_dma_chan(offload)` | لطلب الـ DMA channel اللي بيوفر TX data |
| `rx_stream_request_dma_chan(offload)` | لطلب الـ DMA channel اللي بياخد RX data |

---

#### 3. `struct spi_offload_trigger` — الـ trigger instance (internal)

**بيتعرّف جوا `spi-offload.c` بس** — مش exposed للـ consumers أو الـ providers مباشرة.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `list` | `struct list_head` | بيربطه بالـ global list `spi_offload_triggers` |
| `ref` | `struct kref` | reference counting — لما يوصل صفر بيتحرر |
| `fwnode` | `struct fwnode_handle *` | الـ firmware node — بيُستخدم لـ match الـ trigger بالـ consumer |
| `lock` | `struct mutex` | بيحمي `ops` و`priv` من الـ concurrent access |
| `ops` | `const struct spi_offload_trigger_ops *` | الـ vtable بتاع الـ trigger provider — بيتعمل `NULL` لما الـ provider يمشي |
| `priv` | `void *` | private data خاصة بالـ trigger provider — بيتعمل `NULL` لما الـ provider يمشي |

---

#### 4. `struct spi_offload_trigger_ops` — الـ vtable بتاع الـ trigger provider

| Callback | الـ signature | الوظيفة |
|----------|--------------|---------|
| `match` | `bool (*)(trigger, type, args, nargs)` | **إجبارية** — بيحدد لو الـ trigger ده مناسب للـ consumer |
| `request` | `int (*)(trigger, type, args, nargs)` | اختيارية — بيتاخد resources عند أول get |
| `release` | `void (*)(trigger)` | اختيارية — بيحرر resources عند الـ put |
| `validate` | `int (*)(trigger, config)` | اختيارية — بيتحقق وممكن يعدّل الـ config |
| `enable` | `int (*)(trigger, config)` | اختيارية — بيفعّل الـ hardware trigger فعلياً |
| `disable` | `void (*)(trigger)` | اختيارية — بيوقف الـ hardware trigger |

---

#### 5. `struct spi_offload_trigger_info` — بيتبعته الـ provider عند التسجيل

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `fwnode` | `struct fwnode_handle *` | الـ firmware node بتاع الـ provider |
| `ops` | `const struct spi_offload_trigger_ops *` | الـ vtable — `match` إجبارية |
| `priv` | `void *` | private data هيتحط في `trigger->priv` |

---

#### 6. `struct spi_offload_config` — طلب الـ consumer

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `capability_flags` | `u32` | الـ `SPI_OFFLOAD_CAP_*` اللي الـ consumer محتاجها — الـ controller يختار الـ offload instance المناسب |

---

#### 7. `struct spi_offload_trigger_config` — config الـ trigger

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `type` | `enum spi_offload_trigger_type` | discriminator للـ union |
| `periodic.frequency_hz` | `u64` | التردد للـ periodic trigger |
| `periodic.offset_ns` | `u64` | delay بالنانوثانية بين triggers متعددة |

---

#### 8. `struct spi_controller_and_offload` — internal resource wrapper

بيتعمل مؤقتاً جوا `devm_spi_offload_get` عشان `devm` يقدر يعمل cleanup صح.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `controller` | `struct spi_controller *` | عشان نعرف مين اللي عنده `put_offload` |
| `offload` | `struct spi_offload *` | الـ instance اللي محتاج نرجعه |

---

### علاقات الـ Structs (ASCII Diagram)

```
                    ┌─────────────────────────────────┐
                    │        spi_offload_triggers      │  ← global list (protected by
                    │     (LIST_HEAD, global)          │    spi_offload_triggers_lock)
                    └────────────┬────────────────────-┘
                                 │  list_head
                    ┌────────────▼────────────────────-┐
                    │      spi_offload_trigger          │
                    │  ┌──────────────────────────┐    │
                    │  │ list      (list_head)     │    │
                    │  │ ref       (kref)          │    │
                    │  │ fwnode ──────────────────────► fwnode_handle
                    │  │ lock      (mutex)         │    │
                    │  │ ops  ────────────────────────► spi_offload_trigger_ops
                    │  │ priv ────────────────────────► [provider private data]
                    │  └──────────────────────────┘    │
                    └──────────────────────────────────┘

                    ┌──────────────────────────────────┐
                    │         spi_offload               │
                    │  ┌──────────────────────────┐    │
                    │  │ provider_dev ───────────────► struct device (SPI controller)
                    │  │ priv ────────────────────────► [provider private data]
                    │  │ ops  ────────────────────────► spi_offload_ops
                    │  │ xfer_flags                │    │
                    │  └──────────────────────────┘    │
                    └──────────────────────────────────┘
                              ▲
                              │ offload->ops->trigger_enable/disable
                              │
                    ┌─────────┴────────────────────────┐
                    │  spi_controller_and_offload       │  ← temporary devm wrapper
                    │  controller → spi_controller      │
                    │  offload   → spi_offload          │
                    └──────────────────────────────────┘

spi_offload_ops:
  trigger_enable()  ─────────────► hardware register (SPI controller HW)
  trigger_disable() ─────────────► hardware register
  tx_stream_request_dma_chan() ──► dma_chan
  rx_stream_request_dma_chan() ──► dma_chan
```

---

### Lifecycle Diagrams

#### Lifecycle الـ `spi_offload` (Provider Side)

```
Provider Driver probe()
        │
        ▼
devm_spi_offload_alloc(dev, priv_size)
        │  kzalloc(spi_offload) + kzalloc(priv)
        │  offload->provider_dev = dev
        │  offload->priv = priv
        ▼
[Provider fills offload->ops, offload->xfer_flags]
        │
        ▼
spi_controller->get_offload = my_get_offload_fn   ← registered in controller
        │
        ▼
[Device is live — consumers can call devm_spi_offload_get]
        │
        ▼
Driver remove() / device unbind
        │
        ▼
devm cleanup → kfree(priv) + kfree(offload)   ← devm_kzalloc cleanup
```

#### Lifecycle الـ `spi_offload` (Consumer Side)

```
Consumer Driver probe()
        │
        ▼
devm_spi_offload_get(dev, spi, config)
        │  spi->controller->get_offload(spi, config)  ← provider allocates/returns
        │  kzalloc(spi_controller_and_offload) as devm resource
        │  devm_add_action_or_reset(dev, spi_offload_put, resource)
        ▼
[Use offload — pass to spi_optimize_message, etc.]
        │
        ▼
Consumer Driver remove() / device unbind
        │
        ▼
devm cleanup → spi_offload_put()
        │  controller->put_offload(offload)
        │  kfree(resource)
        ▼
[Done]
```

#### Lifecycle الـ `spi_offload_trigger` (Provider Side)

```
Trigger Provider Driver probe()
        │
        ▼
devm_spi_offload_trigger_register(dev, info)
        │  info->fwnode must be set
        │  info->ops->match must be set
        │  kzalloc(spi_offload_trigger)
        │  kref_init(&trigger->ref)         ← ref = 1
        │  mutex_init(&trigger->lock)
        │  trigger->fwnode = fwnode_handle_get(info->fwnode)
        │  trigger->ops = info->ops
        │  trigger->priv = info->priv
        │  list_add_tail → spi_offload_triggers  (under triggers_lock)
        │  devm_add_action_or_reset(dev, spi_offload_trigger_unregister, trigger)
        ▼
[Trigger is visible globally — consumers can find it]
        │
        ▼
Provider Driver remove() / device unbind
        │
        ▼
devm cleanup → spi_offload_trigger_unregister()
        │  list_del (under triggers_lock)       ← removed from global list
        │  trigger->priv = NULL                 ← under trigger->lock
        │  trigger->ops  = NULL                 ← under trigger->lock
        │  kref_put → spi_offload_trigger_free() IF ref == 0
        ▼
[If consumer still holds ref: ops/priv are NULL, all calls return -ENODEV]
[When consumer calls put: kref drops to 0 → free]
```

#### Lifecycle الـ `spi_offload_trigger` (Consumer Side)

```
Consumer Driver probe()
        │
        ▼
devm_spi_offload_trigger_get(dev, offload, type)
        │  fwnode_property_get_reference_args(provider_dev, "trigger-sources", ...)
        │  → spi_offload_trigger_get(type, &args)
        │      guard(mutex) on triggers_lock
        │      list_for_each_entry → find matching trigger (by fwnode + ops->match)
        │      guard(mutex) on trigger->lock
        │      ops->request() if exists
        │      kref_get(&trigger->ref)      ← ref++
        │  devm_add_action_or_reset(dev, spi_offload_trigger_put, trigger)
        ▼
spi_offload_trigger_validate(trigger, config)   ← optional
        │  guard(mutex) on trigger->lock
        │  ops->validate(trigger, config)
        ▼
spi_offload_trigger_enable(offload, trigger, config)
        │  guard(mutex) on trigger->lock
        │  offload->ops->trigger_enable(offload)   ← enable SPI controller side
        │  trigger->ops->enable(trigger, config)   ← enable trigger HW
        │  [bus is now exclusively owned by offload]
        ▼
[Offload running autonomously — no CPU needed for each transfer]
        │
        ▼
spi_offload_trigger_disable(offload, trigger)
        │  offload->ops->trigger_disable(offload)  ← no lock needed here
        │  guard(mutex) on trigger->lock
        │  trigger->ops->disable(trigger)
        │  [bus released]
        ▼
Consumer Driver remove()
        │
        ▼
devm cleanup → spi_offload_trigger_put()
        │  guard(mutex) on trigger->lock
        │  ops->release(trigger) if exists
        │  kref_put(&trigger->ref, spi_offload_trigger_free)
        ▼
[If ref == 0: mutex_destroy, fwnode_handle_put, kfree]
```

---

### Call Flow Diagrams

#### `devm_spi_offload_get` — Consumer يطلب offload

```
consumer driver
  devm_spi_offload_get(dev, spi, config)
    → check spi->controller->get_offload exists
      → controller->get_offload(spi, config)        [provider vtable]
          → provider selects matching offload instance
          → returns spi_offload *
    → kzalloc(spi_controller_and_offload)            [devm wrapper]
    → devm_add_action_or_reset(dev, spi_offload_put) [cleanup registration]
    → return offload
```

#### `devm_spi_offload_trigger_get` — Consumer يطلب trigger

```
consumer driver
  devm_spi_offload_trigger_get(dev, offload, type)
    → fwnode_property_get_reference_args(provider_dev, "trigger-sources", ...)
        → parses DT/ACPI phandle + args
    → spi_offload_trigger_get(type, &args)
        → lock(spi_offload_triggers_lock)             [global list lock]
          → list_for_each_entry(trigger, &spi_offload_triggers)
              → match fwnode
              → trigger->ops->match(trigger, type, args, nargs)
          → lock(trigger->lock)                       [per-trigger lock]
            → trigger->ops->request(...)              [optional]
            → kref_get(&trigger->ref)
    → fwnode_handle_put(args.fwnode)
    → devm_add_action_or_reset(dev, spi_offload_trigger_put)
    → return trigger
```

#### `spi_offload_trigger_enable` — تفعيل الـ offload

```
consumer driver
  spi_offload_trigger_enable(offload, trigger, config)
    → lock(trigger->lock)
      → check trigger->ops != NULL                    [provider still alive?]
      → offload->ops->trigger_enable(offload)         [SPI controller HW enable]
          → controller programs HW registers
      → trigger->ops->enable(trigger, config)         [trigger source HW enable]
          → e.g. start PWM / enable IRQ line
          → on failure: offload->ops->trigger_disable(offload)  [rollback]
    → return 0 on success
```

#### `spi_offload_trigger_disable` — إيقاف الـ offload

```
consumer driver
  spi_offload_trigger_disable(offload, trigger)
    → offload->ops->trigger_disable(offload)          [NO lock — disable first]
        → controller stops accepting HW triggers
    → lock(trigger->lock)
      → check trigger->ops != NULL
      → trigger->ops->disable(trigger)                [disable trigger source]
```

#### DMA Stream Request Flow

```
consumer driver
  devm_spi_offload_rx_stream_request_dma_chan(dev, offload)
    → check offload->ops->rx_stream_request_dma_chan != NULL
      → offload->ops->rx_stream_request_dma_chan(offload)   [provider vtable]
          → dma_request_chan() or similar
          → return dma_chan *
    → devm_add_action_or_reset(dev, spi_offload_release_dma_chan, chan)
    → return chan

  [same flow for tx_stream_request_dma_chan]
```

---

### Locking Strategy

#### الـ Locks الموجودة

| Lock | النوع | بيحمي إيه |
|------|-------|-----------|
| `spi_offload_triggers_lock` | `DEFINE_MUTEX` — global static | الـ global list `spi_offload_triggers` — قراءة وكتابة |
| `trigger->lock` | `struct mutex` — per-trigger | الـ `trigger->ops` و`trigger->priv` — write على registration/unregistration، read على كل operation |

#### Lock Ordering

```
1. spi_offload_triggers_lock   (global — دايما أول)
2. trigger->lock               (per-instance — دايما تاني)
```

**ممنوع** تعكس الترتيب ده عشان deadlock.

في `spi_offload_trigger_get`:
```c
guard(mutex)(&spi_offload_triggers_lock);   // 1st: global lock
  // find trigger in list
  guard(mutex)(&trigger->lock);             // 2nd: per-trigger lock (nested, safe)
    trigger->ops->request(...)
    kref_get(...)
```

في `spi_offload_trigger_unregister`:
```c
scoped_guard(mutex, &spi_offload_triggers_lock)
    list_del(&trigger->list);               // remove from list first

scoped_guard(mutex, &trigger->lock) {       // then null out ops/priv
    trigger->priv = NULL;
    trigger->ops  = NULL;
}
```

#### الـ "Provider Disappears" Pattern

لو الـ provider driver اتفكّ (unbound) وفي consumer لسه شايل reference:

```
Provider unregisters:
  list_del (trigger no longer findable by new consumers)
  trigger->ops  = NULL   ← under trigger->lock
  trigger->priv = NULL   ← under trigger->lock
  kref_put (ref stays > 0 because consumer holds it)

Consumer calls any API:
  guard(mutex) on trigger->lock
  check trigger->ops == NULL → return -ENODEV

Consumer eventually calls put:
  ops->release() skipped (ops is NULL)
  kref_put → reaches 0 → spi_offload_trigger_free()
  mutex_destroy + fwnode_handle_put + kfree
```

ده pattern الـ **safe provider teardown** — مفيش use-after-free لأن الـ kref بيمنع التحرير المبكر، والـ NULL check بيمنع استخدام pointer مش valid.

#### الـ kref Lifecycle

```
devm_spi_offload_trigger_register:   kref_init → ref = 1
devm_spi_offload_trigger_get:        kref_get  → ref++
devm cleanup (consumer put):         kref_put  → ref--  [→ free if 0]
devm cleanup (provider unregister):  kref_put  → ref--  [→ free if 0]
```

الـ trigger مش بيتحرر إلا لما **الاتنين** (provider + كل consumers) يعملوا put.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions (Cheatsheet)

#### Consumer API

| Function | Signature Summary | Purpose |
|---|---|---|
| `devm_spi_offload_get` | `(dev, spi, config) → spi_offload*` | Consumer يطلب offload instance يطابق config معين |
| `devm_spi_offload_trigger_get` | `(dev, offload, type) → spi_offload_trigger*` | Consumer يطلب trigger instance من firmware node |
| `spi_offload_trigger_validate` | `(trigger, config) → int` | يتحقق إن الـ trigger hardware يقدر يعمل بالـ config المطلوب |
| `spi_offload_trigger_enable` | `(offload, trigger, config) → int` | يفعّل الـ trigger ويحجز الـ SPI bus للـ offload |
| `spi_offload_trigger_disable` | `(offload, trigger)` | يوقف الـ trigger ويحرر الـ SPI bus |
| `devm_spi_offload_tx_stream_request_dma_chan` | `(dev, offload) → dma_chan*` | يجيب الـ DMA channel للـ TX stream |
| `devm_spi_offload_rx_stream_request_dma_chan` | `(dev, offload) → dma_chan*` | يجيب الـ DMA channel للـ RX stream |

#### Provider API

| Function | Signature Summary | Purpose |
|---|---|---|
| `devm_spi_offload_alloc` | `(dev, priv_size) → spi_offload*` | Provider يخصص offload instance جديد |
| `devm_spi_offload_trigger_register` | `(dev, info) → int` | Provider يسجل trigger في global list |
| `spi_offload_trigger_get_priv` | `(trigger) → void*` | Provider يجيب الـ private data بتاعته من الـ trigger |

#### Static / Internal Helpers

| Function | Purpose |
|---|---|
| `spi_offload_put` | devm cleanup callback — يستدعي `put_offload` ويحرر الـ resource struct |
| `spi_offload_trigger_free` | kref release callback — يدمر mutex ويحرر fwnode وكمان الـ trigger struct |
| `spi_offload_trigger_put` | devm cleanup callback — يستدعي `ops->release` ويحط الـ kref |
| `spi_offload_trigger_get` | internal lookup — يبحث في global list ويطلب الـ trigger |
| `spi_offload_trigger_unregister` | devm cleanup callback — يشيل الـ trigger من الـ list ويصفّر الـ ops/priv |
| `spi_offload_release_dma_chan` | devm cleanup callback — يستدعي `dma_release_channel` |

---

### المجموعة 1: Provider Allocation

الهدف من المجموعة دي إن الـ SPI controller driver (provider) يقدر يخصص `spi_offload` instance بطريقة devm-managed بدون ما يحتاج يعمل manual cleanup.

---

#### `devm_spi_offload_alloc`

```c
struct spi_offload *devm_spi_offload_alloc(struct device *dev,
                                           size_t priv_size);
```

بتخصص `struct spi_offload` و private data block بحجم `priv_size` باستخدام `devm_kzalloc`، وبتربطهم ببعض عن طريق `offload->priv`. الـ `provider_dev` بتتحدد بالـ `dev` اللي اتعمل بيه الـ alloc.

**Parameters:**
- `dev` — الـ device بتاع الـ SPI controller (provider)؛ بيُستخدم كـ devm owner وبيُخزَّن في `offload->provider_dev`
- `priv_size` — حجم الـ private state اللي محتاجه الـ provider driver في الـ offload instance

**Return:** pointer لـ `spi_offload` جديد أو `ERR_PTR(-ENOMEM)` لو الـ alloc فشل.

**Key details:**
- بيعمل **اتنين** كولات منفصلين لـ `devm_kzalloc`: واحد للـ `spi_offload` نفسه وواحد للـ `priv`؛ لو الـ second alloc فشل الـ first هيتحرر تلقائيًا بالـ devm.
- مفيش error path تانية — الـ devm بيضمن الـ cleanup.
- **Exported** بـ `EXPORT_SYMBOL_GPL` تحت namespace `SPI_OFFLOAD`.

**Caller context:** provider driver في `probe()`.

---

### المجموعة 2: Consumer Acquisition & Release

الـ consumer driver (الـ peripheral driver زي ADC مثلاً) بيستخدم الـ functions دي يطلب offload instance ويربطه بـ device lifetime.

---

#### `devm_spi_offload_get`

```c
struct spi_offload *devm_spi_offload_get(struct device *dev,
                                         struct spi_device *spi,
                                         const struct spi_offload_config *config);
```

بتطلب offload instance من الـ SPI controller المرتبط بالـ `spi` device. بتستدعي `controller->get_offload(spi, config)` وبتسجل cleanup action عشان `put_offload` يتنادى لما الـ `dev` يتهد.

**Parameters:**
- `dev` — الـ consumer device؛ بيُستخدم كـ devm owner للـ cleanup
- `spi` — الـ SPI device اللي هيتعمل عليه الـ transfers
- `config` — الـ capabilities المطلوبة (مثلاً `SPI_OFFLOAD_CAP_TRIGGER`)

**Return:** pointer لـ `spi_offload` أو `ERR_PTR` في حالة:
- `-EINVAL` لو `spi` أو `config` هو NULL
- `-ENODEV` لو الـ controller مش بيدعم offload
- أي error من `get_offload` callback

**Key details:**
- بتخصص `spi_controller_and_offload` helper struct داخليًا عشان تحتفظ بالـ controller + offload معًا للـ cleanup.
- `devm_add_action_or_reset` بتضمن إن لو التسجيل فشل يتنادى `put_offload` فورًا.
- الـ locking على مستوى الـ controller نفسه — مش من مسؤولية الدالة دي.

**Caller context:** consumer driver في `probe()`.

**Pseudocode flow:**
```
validate(spi, config)
  → check controller->get_offload exists
  → alloc spi_controller_and_offload resource
  → call controller->get_offload(spi, config)
  → devm_add_action_or_reset(spi_offload_put)
  → return offload
```

---

#### `spi_offload_put` (static)

```c
static void spi_offload_put(void *data);
```

الـ devm cleanup callback المرتبط بـ `devm_spi_offload_get`. بتستدعي `controller->put_offload(offload)` وبتحرر الـ helper struct.

**Key details:** بتنادى automatically لما الـ consumer device يتهد — مش للاستخدام المباشر.

---

### المجموعة 3: Trigger Provider Registration

الـ trigger provider (مثلاً PWM controller أو clock source) بيسجل triggers في global list عشان الـ consumers يقدروا يلاقوها.

---

#### `devm_spi_offload_trigger_register`

```c
int devm_spi_offload_trigger_register(struct device *dev,
                                      struct spi_offload_trigger_info *info);
```

بتخصص `spi_offload_trigger` جديد وبتملاه من الـ `info` struct وبتضيفه لـ global list `spi_offload_triggers`. بتسجل `spi_offload_trigger_unregister` كـ devm cleanup action.

**Parameters:**
- `dev` — الـ trigger provider device؛ devm owner
- `info` — بيحتوي على:
  - `fwnode`: الـ firmware node اللي بيُستخدم لمطابقة الـ consumer references
  - `ops`: الـ callback table — **لازم** يحتوي على `match` على الأقل
  - `priv`: private state للـ provider

**Return:** `0` على النجاح أو `-EINVAL` لو الـ info ناقص أو `-ENOMEM` لو الـ alloc فشل.

**Key details:**
- `kref_init(&trigger->ref)` — الـ initial count هو 1 (للـ provider نفسه).
- الإضافة للـ list بتتعمل تحت `spi_offload_triggers_lock` mutex.
- `fwnode_handle_get` بتعمل reference count على الـ fwnode — الـ cleanup بتحررها.
- **Locking:** الـ global list محمية بـ `spi_offload_triggers_lock`؛ الـ trigger نفسه محمي بـ `trigger->lock`.
- **Exported** بـ `EXPORT_SYMBOL_GPL`.

**Caller context:** trigger provider driver في `probe()`.

---

#### `spi_offload_trigger_unregister` (static)

```c
static void spi_offload_trigger_unregister(void *data);
```

الـ devm cleanup callback للـ registration. بتعمل ثلاث حاجات بالترتيب:
1. بتشيل الـ trigger من الـ global list (تحت `spi_offload_triggers_lock`).
2. بتصفّر `trigger->ops` و `trigger->priv` (تحت `trigger->lock`) — ده بيضمن إن أي caller شايل reference هيشوف `ops == NULL` ويرجع `-ENODEV`.
3. بتعمل `kref_put` — لو مفيش consumers شايلين reference، الذاكرة بتتحرر فورًا.

---

### المجموعة 4: Trigger Consumer Acquisition

---

#### `devm_spi_offload_trigger_get`

```c
struct spi_offload_trigger
*devm_spi_offload_trigger_get(struct device *dev,
                               struct spi_offload *offload,
                               enum spi_offload_trigger_type type);
```

بتقرأ الـ `trigger-sources` property من الـ firmware node بتاع الـ `offload->provider_dev`، وبتبحث في الـ global list عن trigger يطابق الـ fwnode والـ type، وبتسجل cleanup action.

**Parameters:**
- `dev` — الـ consumer device؛ devm owner
- `offload` — الـ offload instance المرتبط؛ بيُستخدم عشان نجيب `provider_dev->fwnode`
- `type` — نوع الـ trigger المطلوب: `SPI_OFFLOAD_TRIGGER_DATA_READY` أو `SPI_OFFLOAD_TRIGGER_PERIODIC`

**Return:** pointer لـ `spi_offload_trigger` أو `ERR_PTR` في حالة:
- أي error من `fwnode_property_get_reference_args` (مثلاً `-ENOENT`)
- `-EPROBE_DEFER` لو الـ trigger provider لسه ما سجلش
- أي error من `ops->request`

**Key details:**
- بتستخدم `fwnode_property_get_reference_args` مع property `"trigger-sources"` وـ `"#trigger-source-cells"` — ده standard **trigger binding** في device tree.
- الـ `fwnode_handle_put(args.fwnode)` بتتنادى بعد الـ `spi_offload_trigger_get` حتى لو رجع error، عشان تحرر الـ temporary fwnode reference.
- `-EPROBE_DEFER` معناه الـ trigger provider driver لسه ما اتحمّلش — الـ framework هيعيد الـ probe تاني.

**Pseudocode flow:**
```
fwnode_property_get_reference_args("trigger-sources", ...)
  → lock spi_offload_triggers_lock
  → for each trigger in list:
      if fwnode matches AND ops->match() returns true:
          call ops->request() if exists
          kref_get()
          return trigger
  → if no match: return -EPROBE_DEFER
  → devm_add_action_or_reset(spi_offload_trigger_put)
```

---

#### `spi_offload_trigger_get` (static)

```c
static struct spi_offload_trigger
*spi_offload_trigger_get(enum spi_offload_trigger_type type,
                          struct fwnode_reference_args *args);
```

الـ internal helper اللي بيعمل الـ actual lookup في global list. بتستخدم `guard(mutex)` macro من `linux/cleanup.h` اللي بيضمن release automatic للـ lock لما الـ scope ينتهي.

**Key details:**
- الـ global `spi_offload_triggers_lock` بيتحجز طول فترة الـ search.
- لما يلاقي trigger، بيحجز `trigger->lock` كمان — ده بيمنع race مع الـ unregister.
- لو `ops->request` موجودة وفشلت، بيرجع error بدون ما يعمل `kref_get` — ده صح لأن الـ trigger مش محجوز.

---

#### `spi_offload_trigger_put` (static)

```c
static void spi_offload_trigger_put(void *data);
```

الـ devm cleanup callback للـ trigger consumer. بتستدعي `ops->release` لو موجودة (تحت lock) وبعدين بتعمل `kref_put`.

**Key details:** الـ lock تحت `scoped_guard(mutex, &trigger->lock)` — ده بيحمي من race مع الـ unregister اللي بيصفّر الـ ops. لو الـ provider راح قبل الـ consumer، الـ `ops` هتبقى NULL والـ release مش هيتنادى — ده safe.

---

### المجموعة 5: Trigger Runtime Control

---

#### `spi_offload_trigger_validate`

```c
int spi_offload_trigger_validate(struct spi_offload_trigger *trigger,
                                 struct spi_offload_trigger_config *config);
```

بتتحقق إن الـ trigger hardware يقدر يعمل بالـ configuration المطلوبة. الـ provider ممكن يعدّل الـ `config` (مثلاً يقرّب الـ frequency للأقرب مدعوم).

**Parameters:**
- `trigger` — الـ trigger instance
- `config` — الـ configuration المطلوبة؛ ممكن تتغير بعد الكول كـ in/out parameter

**Return:** `0` على النجاح؛ `-ENODEV` لو الـ provider راح؛ `-EOPNOTSUPP` لو الـ validate callback مش موجود.

**Key details:**
- بتحجز `trigger->lock` — بتحمي من concurrent unregister.
- الـ `config` modification مهم: الـ caller **لازم** يتحقق من القيم الجديدة بعد الكول لأن الـ hardware ممكن ما يدعمش بالظبط اللي طلبه.

**Caller context:** consumer driver قبل ما يستدعي `spi_offload_trigger_enable`.

---

#### `spi_offload_trigger_enable`

```c
int spi_offload_trigger_enable(struct spi_offload *offload,
                                struct spi_offload_trigger *trigger,
                                struct spi_offload_trigger_config *config);
```

بتفعّل الـ offload-side trigger أولاً (`offload->ops->trigger_enable`)، وبعدين بتفعّل الـ trigger source نفسه (`trigger->ops->enable`). لو الـ trigger enable فشل، بتعمل rollback وتستدعي `offload->ops->trigger_disable`.

**Parameters:**
- `offload` — الـ offload instance — بيُفعّل side الـ SPI controller
- `trigger` — الـ trigger source instance — بيُفعّل الـ hardware trigger نفسه
- `config` — الـ trigger configuration (نوع + parameters)

**Return:** `0` على النجاح أو negative error code.

**Key details:**
- **Locking:** `trigger->lock` بيتحجز طول الدالة — ده بيمنع concurrent disable أو unregister.
- **Rollback:** لو `trigger->ops->enable` فشل وكان `offload->ops->trigger_disable` موجود، بيتنادى تلقائيًا — ده يضمن consistency.
- الكول ده بيحجز الـ SPI bus للـ offload — أي `spi_sync` تاني هيرجع `-EBUSY`.
- **لازم** يتنادى `spi_offload_trigger_disable` بعد كل successful call.

**Pseudocode flow:**
```
lock trigger->lock
  if offload->ops->trigger_enable:
      ret = offload->ops->trigger_enable(offload)
      if ret: return ret
  if trigger->ops->enable:
      ret = trigger->ops->enable(trigger, config)
      if ret:
          if offload->ops->trigger_disable:
              offload->ops->trigger_disable(offload)  // rollback
          return ret
return 0
```

---

#### `spi_offload_trigger_disable`

```c
void spi_offload_trigger_disable(struct spi_offload *offload,
                                  struct spi_offload_trigger *trigger);
```

بتوقف الـ trigger source أولاً (تحت `trigger->lock`)، وبعدين بتوقف الـ offload side في الـ SPI controller. ترتيب الـ disable عكس الـ enable.

**Parameters:**
- `offload` — الـ offload instance
- `trigger` — الـ trigger source instance

**Return:** void — مفيش error return.

**Key details:**
- الـ `offload->ops->trigger_disable` بتتنادى **قبل** حجز الـ lock — لأنها مش محتاجة تحمي من unregister race.
- `trigger->ops->disable` بتتنادى تحت `trigger->lock` — لو الـ provider راح (`ops == NULL`) الدالة بتخرج بأمان.
- **Context:** can sleep (من الـ kernel-doc).

---

### المجموعة 6: DMA Stream Support

الـ SPI offload ممكن يكون عنده streams مباشرة لـ DMA channels — ده بيخلي الـ data يتنقل بين الـ peripheral والـ memory بدون CPU intervention.

---

#### `devm_spi_offload_tx_stream_request_dma_chan`

```c
struct dma_chan
*devm_spi_offload_tx_stream_request_dma_chan(struct device *dev,
                                              struct spi_offload *offload);
```

بتطلب الـ DMA channel الخاص بالـ TX stream من الـ offload provider، وبتسجل `dma_release_channel` كـ devm cleanup action. الـ channel ده بيُغذي بيانات للـ SPI transfers اللي عليها `SPI_OFFLOAD_XFER_TX_STREAM` flag.

**Parameters:**
- `dev` — الـ consumer device؛ devm owner
- `offload` — الـ offload instance

**Return:** pointer لـ `dma_chan` أو `ERR_PTR(-EOPNOTSUPP)` لو الـ provider مش بيدعم TX stream.

**Key details:**
- بتتحقق من وجود `offload->ops->tx_stream_request_dma_chan` قبل ما تستدعيه.
- الـ DMA channel بيتحرر automatically لما الـ `dev` يتهد.

---

#### `devm_spi_offload_rx_stream_request_dma_chan`

```c
struct dma_chan
*devm_spi_offload_rx_stream_request_dma_chan(struct device *dev,
                                              struct spi_offload *offload);
```

نفس `devm_spi_offload_tx_stream_request_dma_chan` بالظبط لكن للـ RX direction. بيطلب الـ DMA channel اللي بيستقبل بيانات من الـ SPI transfers اللي عليها `SPI_OFFLOAD_XFER_RX_STREAM` flag.

**Parameters:** نفس TX.

**Return:** pointer لـ `dma_chan` أو `ERR_PTR(-EOPNOTSUPP)`.

---

#### `spi_offload_release_dma_chan` (static)

```c
static void spi_offload_release_dma_chan(void *chan);
```

devm cleanup callback بسيط — بتستدعي `dma_release_channel(chan)` فقط. مش للاستخدام المباشر.

---

### المجموعة 7: Provider Helpers

---

#### `spi_offload_trigger_get_priv`

```c
void *spi_offload_trigger_get_priv(struct spi_offload_trigger *trigger);
```

accessor بسيط بيرجع `trigger->priv`. الـ provider بيحتاجها جوه الـ ops callbacks لأن الـ `spi_offload_trigger` struct مش exposed كـ public header.

**Parameters:**
- `trigger` — الـ trigger instance اللي بيجي كـ argument في الـ callbacks

**Return:** الـ `priv` pointer اللي اتحط في `spi_offload_trigger_info.priv` وقت الـ registration.

**Key details:** مفيش locking — الـ priv بيتصفّر بس في `spi_offload_trigger_unregister` اللي بتتنادى بعد ما الـ ops تتصفّر. الـ callbacks نفسها مش هتتنادى بعد ما الـ ops تتصفّر، فالـ race مش ممكن في الـ normal flow.

---

### علاقة الـ kref بالـ ops/priv Nullification

الـ design pattern المستخدم هنا مهم: الـ trigger lifetime محمي بـ kref لكن الـ ops بتتصفّر بشكل منفصل لما الـ provider يروح:

```
Provider unregister:
  1. list_del (under global lock)         → new consumers won't find it
  2. ops = NULL, priv = NULL (under lock) → existing callers get -ENODEV
  3. kref_put                             → free if no consumers holding ref

Consumer put:
  ops->release if ops != NULL (under lock)
  kref_put → free if provider already unregistered
```

ده بيحمي من الـ use-after-free حتى لو الـ consumer والـ provider اتهدوا بترتيب عكسي.

---

### ASCII: Data Flow Architecture

```
  Consumer Driver (e.g., ADC peripheral)
        │
        ├─ devm_spi_offload_get()          ─→ SPI Controller (provider)
        │       └─ controller->get_offload()         │
        │                                            │
        ├─ devm_spi_offload_trigger_get()  ─→ Trigger Provider (e.g., PWM/CLK)
        │       └─ ops->match() + request()          │
        │                                            │
        ├─ spi_offload_trigger_validate()             │
        │       └─ ops->validate()                   │
        │                                            │
        ├─ spi_offload_trigger_enable()               │
        │       ├─ offload->ops->trigger_enable() ───┘
        │       └─ trigger->ops->enable()
        │
        ├─ devm_spi_offload_rx_stream_request_dma_chan()
        │       └─ offload->ops->rx_stream_request_dma_chan()
        │                   └─→ DMA Engine
        │
        └─ spi_offload_trigger_disable()
                ├─ offload->ops->trigger_disable()
                └─ trigger->ops->disable()
```
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. debugfs — المسارات المهمة

الـ SPI offload subsystem مش بيسجل entries خاصة بيه في debugfs بشكل مباشر، لكن الـ SPI controller اللي بيدعم offload غالباً بيسجل entries. نقدر نوصل للـ controller debugfs بالشكل ده:

```bash
# شوف كل entries الـ SPI في debugfs
ls /sys/kernel/debug/spi*/

# لو الـ controller بيستخدم DMA، اتفرج على الـ DMA engine debug
ls /sys/kernel/debug/dmaengine/
cat /sys/kernel/debug/dmaengine/summary

# شوف الـ clk اللي بيستخدمه الـ trigger provider (لو periodic)
ls /sys/kernel/debug/clk/
```

---

#### 2. sysfs — المسارات المهمة

```bash
# الـ SPI controller device
ls /sys/bus/spi/devices/

# الـ SPI peripheral device (consumer)
ls /sys/bus/spi/devices/spi0.0/

# DMA channels المرتبطة بالـ offload streams
ls /sys/class/dma/

# فحص الـ firmware node (Device Tree) المرتبطة بالـ trigger
ls /sys/firmware/devicetree/base/

# قراءة xfer_flags من sysfs لو الـ driver expose-ها
# (مش standard، بيعتمد على الـ provider implementation)
cat /sys/bus/spi/devices/spi0.0/of_node/compatible
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# تفعيل كل أحداث الـ SPI
echo 1 > /sys/kernel/debug/tracing/events/spi/enable

# تفعيل أحداث الـ DMA engine (مهمة لـ TX/RX streams)
echo 1 > /sys/kernel/debug/tracing/events/dma_fence/enable

# trace الـ function calls في spi-offload.c تحديداً
echo 'spi_offload_trigger_enable' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'spi_offload_trigger_disable' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'devm_spi_offload_get' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'devm_spi_offload_trigger_get' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'spi_offload_trigger_validate' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# قراءة الـ trace
cat /sys/kernel/debug/tracing/trace

# تعطيل بعد الفحص
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

**تتبع الـ trigger list لحظة بلحظة:**

```bash
# trace الـ lock/unlock على spi_offload_triggers_lock
echo 'mutex_lock mutex_unlock' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغل الـ consumer driver وشوف النتيجة
cat /sys/kernel/debug/tracing/trace_pipe
```

---

#### 4. printk والـ Dynamic Debug

```bash
# تفعيل dynamic debug لملف spi-offload.c كله
echo 'file spi-offload.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو لكل الـ SPI subsystem
echo 'file drivers/spi/* +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع معلومات إضافية (function name + line number)
echo 'file spi-offload.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ dynamic debug الفعّالة حالياً
grep spi-offload /sys/kernel/debug/dynamic_debug/control

# تغيير loglevel أثناء التشغيل لرؤية dev_dbg
dmesg -n 8
# أو
echo 8 > /proc/sys/kernel/printk
```

**لإضافة printk مؤقت في الكود أثناء التطوير:**

```c
/* في devm_spi_offload_get — لمعرفة لو get_offload بترجع إيه */
offload = spi->controller->get_offload(spi, config);
pr_debug("spi-offload: get_offload returned %ld\n", IS_ERR(offload) ? PTR_ERR(offload) : 0);

/* في spi_offload_trigger_get — تتبع الـ match loop */
list_for_each_entry(trigger, &spi_offload_triggers, list) {
    pr_debug("spi-offload: checking trigger fwnode=%pfw vs args fwnode=%pfw\n",
             trigger->fwnode, args->fwnode);
    ...
}
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_SPI_DEBUG` | يفعّل dev_dbg() في كل SPI subsystem |
| `CONFIG_DEBUG_KERNEL` | يفعّل kernel debug features عامة |
| `CONFIG_DEBUG_MUTEXES` | يكتشف deadlocks في `spi_offload_triggers_lock` و `trigger->lock` |
| `CONFIG_PROVE_LOCKING` | lockdep — يكتشف lock ordering violations |
| `CONFIG_DEBUG_OBJECTS_WORK` | يتحقق من lifetime الـ devm objects |
| `CONFIG_KASAN` | يكتشف memory bugs في alloc/free |
| `CONFIG_KFENCE` | lightweight memory safety detection |
| `CONFIG_DMA_API_DEBUG` | يفحص صحة الـ DMA operations في TX/RX streams |
| `CONFIG_DEBUG_SHIRQ` | يفحص shared IRQs لو الـ trigger بيستخدم interrupt |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug` / `dev_dbg` dynamically |
| `CONFIG_FTRACE` | أساسي لـ function tracing |
| `CONFIG_OF_DEBUG` | يساعد في debug الـ Device Tree parsing |

```bash
# تحقق من الـ configs الفعّالة في kernel الحالي
zcat /proc/config.gz | grep -E 'CONFIG_SPI|CONFIG_DMA_API_DEBUG|CONFIG_DEBUG_MUTEX'
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# فحص الـ SPI devices المتعرف عليها
ls -la /sys/bus/spi/devices/

# فحص الـ DMA channels المتاحة وحالتها
cat /sys/class/dma/*/in_use 2>/dev/null

# فحص الـ fwnode / Device Tree للـ trigger-sources property
cat /sys/firmware/devicetree/base/spi@xxx/compatible
# ابحث عن trigger-sources property
fdtdump /boot/dtb-$(uname -r) 2>/dev/null | grep -A5 trigger-sources

# استخدام dtc لتحليل الـ DTS
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A10 trigger

# فحص الـ module namespace
cat /sys/module/spi_offload/sections/.text 2>/dev/null
# أو
modinfo spi-offload 2>/dev/null

# فحص كل الـ kref counts المسرّبة (لو شكّك في memory leak)
# استخدم kmemleak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ / الكود | المعنى | الحل |
|---|---|---|
| `devm_spi_offload_get: -ENODEV` | الـ controller مش بيدعم `get_offload` أو مفيش offload مناسب | تأكد إن الـ SPI controller driver بيسجّل `get_offload` callback |
| `devm_spi_offload_get: -EINVAL` | الـ `spi` أو `config` pointer = NULL | تأكد من صحة المؤشرات قبل الاستدعاء |
| `devm_spi_offload_trigger_get: -ENOENT` أو `-EINVAL` | `trigger-sources` property مش موجودة في Device Tree | أضف property صح في الـ DTS |
| `devm_spi_offload_trigger_get: -EPROBE_DEFER` | الـ trigger provider لسه ما اتسجّلش | الـ consumer محتاج يتأجل (probe ordering) |
| `spi_offload_trigger_validate: -ENODEV` | الـ trigger provider اتنزع وهو بيتسخدم (hot-unplug) | تحقق من lifecycle الـ provider |
| `spi_offload_trigger_validate: -EOPNOTSUPP` | الـ trigger ops مش بتدعم `validate` | الـ provider محتاج يضيف `validate` callback |
| `spi_offload_trigger_enable: -ENODEV` | الـ `trigger->ops` بقت NULL (provider removed) | الـ provider اتمسح قبل ما consumer يخلص |
| `devm_spi_offload_tx_stream_request_dma_chan: -EOPNOTSUPP` | الـ offload ops مش بتدعم TX DMA stream | الـ hardware أو الـ driver مش بيدعم TX streaming |
| `devm_spi_offload_rx_stream_request_dma_chan: -EOPNOTSUPP` | نفس الكلام لـ RX | الـ hardware أو الـ driver مش بيدعم RX streaming |
| `devm_spi_offload_trigger_register: -EINVAL` | `fwnode` أو `ops` أو `ops->match` = NULL | الـ provider لازم يوفر كل الـ required fields |
| `devm_spi_offload_alloc: -ENOMEM` | فشل الـ allocation | مشكلة في memory pressure |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في spi_offload_trigger_get — لو trigger->ops اتمسحت بشكل غير متوقع */
guard(mutex)(&trigger->lock);
WARN_ON(!trigger->ops); /* ops لازم تكون موجودة بعد kref_get */

/* في spi_offload_trigger_enable — تحقق من الترتيب الصحيح */
WARN_ON(!offload);
WARN_ON(!trigger);

/* في spi_offload_trigger_unregister — تحقق إن الـ list مش فاضي */
WARN_ON(list_empty(&trigger->list));

/* في devm_spi_offload_alloc — تحقق من الـ dev */
WARN_ON(!dev);

/* dump_stack مفيد لتتبع من استدعى devm_spi_offload_get */
if (!spi->controller->get_offload) {
    dev_err(dev, "controller missing get_offload\n");
    dump_stack(); /* عشان نعرف من المستدعي */
    return ERR_PTR(-ENODEV);
}
```

---

### Hardware Level

---

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

```bash
# تحقق إن الـ SPI controller enabled في hardware
# (عبر clock gating / power domain)
cat /sys/kernel/debug/clk/spi_clk/enable_count
cat /sys/kernel/debug/clk/spi_clk/rate

# تحقق من الـ power domain
cat /sys/kernel/debug/pm_genpd/*/summary

# تحقق من الـ pinmux — هل الـ pins متوجّهة للـ SPI controller صح؟
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep spi

# تحقق من الـ trigger source hardware — لو periodic clock
cat /sys/kernel/debug/clk/trigger_clk/rate

# DMA channel state
cat /sys/class/dma/dma0chan0/in_use
```

---

#### 2. Register Dump Techniques

```bash
# قراءة registers الـ SPI controller بـ devmem2
# (استبدل 0xFF200000 بالـ base address الحقيقي من DTS)
devmem2 0xFF200000 w    # قراءة 32-bit register

# قراءة range كاملة بـ /dev/mem
# (محتاج CONFIG_STRICT_DEVMEM=n أو تستخدم /dev/mem مع io tool)
io -4 -r 0xFF200000 0x100   # اقرأ 256 byte من base address

# بديل بـ dd مع /dev/mem
dd if=/dev/mem bs=4 count=16 skip=$((0xFF200000/4)) 2>/dev/null | hexdump -C

# لو الـ platform بيدعم perf، استخدم perf mem لفحص accesses
perf mem record -a -- sleep 1
perf mem report
```

**مثال: قراءة SPI offload trigger register على SoC مبني على Analog Devices:**

```bash
# افرض إن الـ trigger control register على offset 0x40
BASE=0xFF200000
devmem2 $((BASE + 0x40)) w
# التفسير: bit[0] = trigger_enable, bit[1] = trigger_type
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**الأشياء اللي لازم تراقبها:**

```
SPI Bus Lines:
┌─────────────┬─────────────────────────────────────────────────────┐
│ SCLK        │ تحقق من frequency = المطلوب، وشكل الـ clock         │
│ MOSI        │ تحقق من البيانات لو TX stream شغّال                   │
│ MISO        │ تحقق من البيانات لو RX stream شغّال                   │
│ CS#         │ لازم يكون active أثناء الـ transfer فقط              │
│ TRIGGER_IN  │ الـ hardware trigger pulse — راقب timing              │
└─────────────┴─────────────────────────────────────────────────────┘
```

- **الـ trigger latency**: المسافة بين rising edge على TRIGGER_IN وأول SCLK edge.
- **الـ CS# glitch**: لو شايف CS# بيتحرك وسط الـ offload transfer → مشكلة في الـ arbitration.
- **الـ DMA underrun/overrun**: لو MOSI بيوقف وسط frame → الـ DMA ما وفّرش data بسرعة كافية.
- **الـ clock stretching**: لو الـ SPI clock بيتباطأ أو بيوقف → مشكلة في الـ clock source للـ trigger.

---

#### 4. المشاكل الشائعة في Hardware وأنماطها في kernel log

| المشكلة | Pattern في dmesg | الحل |
|---|---|---|
| الـ trigger provider ما اتسجّلش وقت probe الـ consumer | `spi_offload_trigger_get: -EPROBE_DEFER` | تأكد من ترتيب الـ probe أو استخدم `deferred_probe` |
| الـ trigger hardware مش بيتجاوب | لا SPI transfers بتحصل بعد enable | افحص الـ TRIGGER_IN signal بـ oscilloscope |
| الـ DMA TX underrun | بيانات SPI ناقصة أو zeros | زوّد الـ DMA buffer size أو قلّل الـ frequency |
| الـ DMA RX overflow | بيانات بتتضيع في الـ RX stream | زوّد buffer أو قلّل الـ trigger frequency |
| الـ CS# مش بيترفع بعد trigger disable | الـ bus محجوز رغم الـ disable | تحقق من `trigger_disable` callback implementation |
| fwnode mismatch | `spi_offload_trigger_get: -EPROBE_DEFER` (لو trigger موجود فعلاً) | تحقق من الـ phandle في DTS |

---

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DTS compile بشكل صح
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -B5 -A15 "trigger-sources"

# مثال على DTS صح للـ offload trigger:
# spi_offload_consumer: adc@0 {
#     compatible = "vendor,adc";
#     reg = <0>;
#     spi-max-frequency = <10000000>;
#     trigger-sources = <&timer0 0>;   /* phandle + args */
#     #trigger-source-cells = <0>;
# };
#
# timer0: timer@ff300000 {
#     compatible = "vendor,hw-timer";
#     #trigger-source-cells = <1>;     /* لازم تطابق nargs المتوقعة */
# };

# تحقق من وجود #trigger-source-cells
fdtget /sys/firmware/devicetree/base /timer@ff300000 '#trigger-source-cells'

# تحقق من الـ phandle الصح
fdtget /sys/firmware/devicetree/base /spi@ff200000/adc@0 trigger-sources

# لو fwnode_property_get_reference_args فشلت بـ -EINVAL أو -ENOENT:
# → الـ property اسمها غلط أو مش موجودة في node الـ provider_dev
dmesg | grep "spi.*offload\|trigger-sources\|fwnode"
```

**شجرة DT مبسطة للـ offload trigger setup:**

```
                    ┌─────────────────────┐
                    │  SPI Controller     │
                    │  (provider_dev)     │
                    │  trigger-sources = ─┼──────────────┐
                    └─────────────────────┘              │
                                                         ▼
                                              ┌─────────────────────┐
                                              │  Trigger Provider   │
                                              │  (timer / GPIO IRQ) │
                                              │  #trigger-source-   │
                                              │  cells = <1>        │
                                              └─────────────────────┘
```

---

### Practical Commands

---

#### سيناريو 1: فشل `devm_spi_offload_get`

```bash
# 1. تأكد إن الـ SPI controller driver loaded
lsmod | grep spi

# 2. شوف الـ kernel log للـ controller probe
dmesg | grep -i "spi\|offload" | tail -30

# 3. تحقق إن get_offload registered
# (لازم تشوف "spi offload" أو ما شابهه في dmesg أثناء probe)
dmesg | grep -i "get_offload\|offload.*register"

# 4. فعّل dynamic debug وجرب probe تاني
echo 'file drivers/spi/* +pflmt' > /sys/kernel/debug/dynamic_debug/control
modprobe -r my_spi_consumer_driver
modprobe my_spi_consumer_driver
dmesg | tail -50
```

**مثال output متوقع لو نجح الـ get:**
```
[   12.345678] spi-offload: allocated offload instance for spi0.0
[   12.345679] spi-offload: get_offload OK, caps=0x0f
```

---

#### سيناريو 2: الـ trigger مش بيتعرف (-EPROBE_DEFER)

```bash
# تحقق من الـ trigger providers المسجّلين
# الـ spi_offload_triggers list مش exposed مباشرة، لكن
# ممكن تشوف لو الـ provider probe نجح
dmesg | grep -i "trigger.*register\|offload.*trigger"

# تحقق من Device Tree
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null \
    | grep -A20 "trigger-sources"

# فحص الـ deferred probes
cat /sys/kernel/debug/devices_deferred

# تشغيل ftrace على spi_offload_trigger_get
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo 'spi_offload_trigger_get' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# trigger the probe
echo 1 > /sys/bus/platform/drivers/my_trigger_provider/bind 2>/dev/null
cat /sys/kernel/debug/tracing/trace
```

**مثال output:**
```
 0)               |  spi_offload_trigger_get() {
 0)               |    /* list empty → -EPROBE_DEFER */
 0)   0.234 us    |  }
```

---

#### سيناريو 3: فحص DMA stream channels

```bash
# تحقق من الـ DMA channels المطلوبة
cat /sys/kernel/debug/dmaengine/summary

# مثال output:
# dma0 (my-dma-controller): 4 channels
# Channel: dma0chan0 (spi-rx-stream): in use
# Channel: dma0chan1 (spi-tx-stream): in use

# فحص الـ DMA errors
dmesg | grep -i "dma.*error\|dma.*fail\|offload.*dma"

# تفعيل DMA engine tracing
echo 1 > /sys/kernel/debug/tracing/events/dma_fence/dma_fence_init/enable
echo 1 > /sys/kernel/debug/tracing/events/dma_fence/dma_fence_destroy/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغل الـ application
cat /sys/kernel/debug/tracing/trace_pipe &
# أوقف بعد فترة
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

---

#### سيناريو 4: Hot-unplug الـ trigger provider (ENODEV race)

```bash
# محاكاة الـ race condition بـ lockdep
# تأكد من CONFIG_PROVE_LOCKING=y في kernel config
zcat /proc/config.gz | grep CONFIG_PROVE_LOCKING

# راقب الـ lockdep warnings
dmesg -w | grep -i "lock\|WARNING\|BUG"

# الـ race يظهر كـ:
# WARNING: possible circular locking dependency detected
# أو
# spi-offload: ops is NULL, trigger provider removed while in use
```

---

#### سيناريو 5: فحص الـ kref leak

```bash
# تفعيل kmemleak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak | grep -i "spi\|offload\|trigger"

# لو لقيت leak في spi_offload_trigger:
# → غالباً devm cleanup ما اتنفّذش بالترتيب الصح
# → تحقق من devm_add_action_or_reset return values

# فحص الـ kobject refs
ls /sys/bus/spi/devices/ | wc -l
modprobe -r my_consumer_driver
ls /sys/bus/spi/devices/ | wc -l  # لازم يقل
```

---

#### ملخص سريع — أهم الأوامر

```bash
# 1. كل log الـ SPI offload
dmesg | grep -iE "spi.offload|trigger.source|offload.trigger"

# 2. dynamic debug لكل الـ subsystem
echo 'file drivers/spi/spi-offload.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# 3. ftrace على الـ functions الرئيسية
echo 'devm_spi_offload_get spi_offload_trigger_enable spi_offload_trigger_disable' \
    > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 4. DMA state
cat /sys/kernel/debug/dmaengine/summary

# 5. deferred probes
cat /sys/kernel/debug/devices_deferred | grep spi

# 6. Device Tree trigger property
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -B2 -A10 "trigger-sources"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على AM62x — الـ Trigger مش بيشتغل

#### العنوان
**SPI offload trigger** مش بيتفعّل على gateway صناعي بيستخدم AM62x

#### السياق
شركة بتبني industrial IoT gateway على Texas Instruments AM62x. الـ gateway بيقرأ بيانات من ADC خارجي (AD7768) عبر SPI بسرعة عالية — 1 MSPS. المطلوب إن الـ ADC يبعت إشارة `DRDY` (Data Ready) تُشغّل SPI transfer تلقائيًا من غير ما الـ CPU يتدخّل. المنتج وصل للـ production وبدأوا يلاقوا إن البيانات مش بتيجي.

#### المشكلة
الـ driver بيـreturn `ERR_PTR(-ENODEV)` من `devm_spi_offload_get()`. الـ log بيقول:

```
spi-adc ad7768: failed to get SPI offload: -19
```

#### التحليل
**الخطوة 1** — نشوف `devm_spi_offload_get()`:

```c
// spi-offload.c:117
if (!spi->controller->get_offload)
    return ERR_PTR(-ENODEV);  // <-- هنا بيوقع
```

الـ SPI controller driver على AM62x مش implement عمل `get_offload` callback. الـ offload subsystem موجود، بس الـ controller driver ما أضافش الـ hook.

**الخطوة 2** — نتأكد من الـ DT:

```dts
/* DT للـ consumer (AD7768 driver) */
&spi0 {
    ad7768: adc@0 {
        compatible = "adi,ad7768";
        reg = <0>;
        spi-max-frequency = <20000000>;
        /* مفيش trigger-sources هنا! */
    };
};
```

**الخطوة 3** — لو الـ controller عنده `get_offload`، الـ code بييجي هنا:

```c
offload = spi->controller->get_offload(spi, config);
if (IS_ERR(offload)) {
    kfree(resource);
    return offload;  // بيرجع الـ error للـ caller
}
```

المشكلة مش في `spi-offload.c` نفسه — الـ core صح، المشكلة في الـ controller driver.

#### الحل

**1. تعديل DT للـ trigger:**

```dts
&mcu_spi0 {
    /* الـ controller لازم يدعم SPI offload */
    #trigger-source-cells = <1>;

    ad7768: adc@0 {
        compatible = "adi,ad7768";
        reg = <0>;
        spi-max-frequency = <20000000>;
        trigger-sources = <&mcu_spi0 0>;  /* offload trigger index 0 */
        #trigger-source-cells = <0>;
    };
};
```

**2. تحقق من الـ controller driver إنه بيـexport الـ offload ops:**

```c
/* في controller driver */
static struct spi_offload *am62x_spi_get_offload(struct spi_device *spi,
                                                  const struct spi_offload_config *cfg)
{
    /* لازم تكون موجودة عشان devm_spi_offload_get ما يرجعش -ENODEV */
    ...
}
controller->get_offload = am62x_spi_get_offload;
```

**3. Debug command:**

```bash
# تحقق إن الـ offload trigger اتسجّل
cat /sys/kernel/debug/spi/offload/triggers

# تحقق من الـ fwnode matching
cat /proc/device-tree/mcu_spi0/ad7768/trigger-sources
```

#### الدرس المستفاد
الـ `devm_spi_offload_get()` بيرجع `-ENODEV` في حالتين: إما الـ controller مش support offload، أو الـ `get_offload` callback مش مسجّلة. الـ check الأولى في الـ code (`!spi->controller->get_offload`) بيكشف الحالة دي فورًا. لازم تتأكد إن الـ SoC controller driver نفسه بيدعم الـ offload feature ومش بس الـ subsystem موجود في الـ kernel.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — Race Condition في الـ Trigger Unregister

#### العنوان
**Kernel panic** لما بيتعمل module unload للـ SPI offload trigger provider على H616

#### السياق
Android TV box بيستخدم Allwinner H616 مع WiFi chip بيتكلم عبر SPI. الـ SPI offload controller driver كـ loadable module. أثناء OTA update، الـ system بيعمل `rmmod` للـ SPI controller module وهو شغال — بيحصل kernel panic.

#### المشكلة
```
BUG: kernel NULL pointer dereference
RIP: spi_offload_trigger_enable+0x2c/0x80
```

الـ consumer driver لسه بيحاول يـcall `spi_offload_trigger_enable()` بعد ما الـ provider اتسجّل ورا `trigger->ops` بقى NULL.

#### التحليل
**التسلسل المشكوك فيه:**

```
Thread A (consumer):              Thread B (rmmod):
spi_offload_trigger_enable()
  guard(mutex)(&trigger->lock)
  ← يمسك الـ lock
                                  spi_offload_trigger_unregister()
                                    scoped_guard(mutex, &trigger->lock)
                                    ← بينتظر Thread A يخلص
  if (!trigger->ops)  ← ops مش NULL لسه
    ...
  trigger->ops->enable(...)  ← بيشتغل
  ← يسيب الـ lock
                                    trigger->ops = NULL  ← هنا بيتضبط NULL
                                    trigger->priv = NULL

[لاحقًا، Thread A تاني:]
spi_offload_trigger_enable()
  guard(mutex)(&trigger->lock)
  if (!trigger->ops)
    return -ENODEV;  ← الحماية شغالة هنا
```

الكود صح فعلًا — لما بتمسك الـ `trigger->lock`، بتشيك `trigger->ops` قبل ما تستخدمها:

```c
// spi-offload.c:286-303
guard(mutex)(&trigger->lock);

if (!trigger->ops)
    return -ENODEV;   // حماية صح

if (offload->ops && offload->ops->trigger_enable) {
    ret = offload->ops->trigger_enable(offload);
    ...
}
```

المشكلة مش في `spi-offload.c` — المشكلة إن الـ consumer driver مش بيتعامل مع الـ `-ENODEV` صح، وبيكمل يستخدم الـ offload pointer بعد ما الـ provider راح.

#### الحل

**في الـ consumer driver، لازم تـhandle الـ `-ENODEV`:**

```c
ret = spi_offload_trigger_enable(offload, trigger, &config);
if (ret == -ENODEV) {
    /* الـ provider راح، وقف الـ streaming */
    wifi_spi_stop_streaming(priv);
    return;
}
if (ret) {
    dev_err(dev, "trigger enable failed: %d\n", ret);
    return;
}
```

**وكمان تأكد من الترتيب الصح في الـ cleanup:**

```c
/* في driver remove() */
spi_offload_trigger_disable(offload, trigger);  /* الأول */
/* devm بيعمل trigger put تلقائيًا بعدين */
```

**Debug:**

```bash
# شوف الـ kref count للـ trigger
echo 1 > /sys/kernel/debug/kmemleak/scan
cat /sys/kernel/debug/kmemleak

# تابع الـ module reference count
lsmod | grep spi_h616_offload
```

#### الدرس المستفاد
الـ `spi_offload_trigger_unregister()` بيـset الـ `ops` و `priv` لـ NULL تحت الـ lock — ده تصميم متعمد لحماية الـ hot-unplug. الـ consumer driver لازم يـhandle الـ `-ENODEV` من أي `trigger_enable` call عشان يتعامل مع السيناريو ده بدل ما يـcrash.

---

### السيناريو الثالث: IoT Sensor Hub على STM32MP1 — الـ DMA Channel مش بيتطلب

#### العنوان
**`-EOPNOTSUPP`** من `devm_spi_offload_rx_stream_request_dma_chan()` على STM32MP1

#### السياق
IoT sensor hub بيستخدم STM32MP1 لجمع بيانات من عدة sensors عبر SPI. المطلوب إن الـ RX data stream يروح مباشرة لـ DMA buffer من غير CPU overhead. الـ engineer كتب الـ consumer driver وبدأ الـ testing، لكن الـ DMA request بيفشل.

#### المشكلة
```
spi-sensor-hub: failed to get RX DMA channel: -95
/* -95 = EOPNOTSUPP */
```

#### التحليل
**الكود في `spi-offload.c:390-402`:**

```c
struct dma_chan
*devm_spi_offload_rx_stream_request_dma_chan(struct device *dev,
                                             struct spi_offload *offload)
{
    ...
    // الـ check الأول:
    if (!offload->ops || !offload->ops->rx_stream_request_dma_chan)
        return ERR_PTR(-EOPNOTSUPP);  // <-- هنا بيوقع

    chan = offload->ops->rx_stream_request_dma_chan(offload);
    ...
}
```

يعني الـ offload instance اتجاب صح من `devm_spi_offload_get()`، بس الـ `ops->rx_stream_request_dma_chan` callback مش معمولة في الـ provider.

**نتحقق من الـ `spi_offload_config` اللي اتبعتت:**

```c
/* في الـ consumer driver */
static const struct spi_offload_config sensor_offload_config = {
    .capability_flags = SPI_OFFLOAD_CAP_TRIGGER |
                        SPI_OFFLOAD_CAP_RX_STREAM_DMA,  /* طلبنا ده */
};
```

الـ `devm_spi_offload_get()` بيـpass الـ config للـ controller:

```c
offload = spi->controller->get_offload(spi, config);
```

الـ controller driver على STM32MP1 مش بيتحقق من الـ `capability_flags` صح، وبيرجع offload instance حتى لو مش supportاها.

#### الحل

**الـ provider driver لازم يرفض الـ config مش المدعومة:**

```c
/* في stm32_spi_get_offload() */
static struct spi_offload *stm32_spi_get_offload(struct spi_device *spi,
                                                  const struct spi_offload_config *config)
{
    struct stm32_spi *host = spi_controller_get_devdata(spi->controller);

    /* تحقق من الـ capabilities */
    if (config->capability_flags & SPI_OFFLOAD_CAP_RX_STREAM_DMA) {
        if (!host->dma_rx) {
            dev_err(&spi->dev, "RX DMA not available\n");
            return ERR_PTR(-ENODEV);
        }
    }

    offload->ops = &stm32_spi_offload_ops;
    return offload;
}

/* وكمان implement الـ callback */
static const struct spi_offload_ops stm32_spi_offload_ops = {
    .trigger_enable  = stm32_spi_trigger_enable,
    .trigger_disable = stm32_spi_trigger_disable,
    .rx_stream_request_dma_chan = stm32_spi_rx_dma_chan_get,  /* لازم تكون موجودة */
};
```

**DT تحقق:**

```dts
&spi2 {
    dmas = <&dmamux1 37 0x400 0x01>,   /* TX */
           <&dmamux1 38 0x400 0x01>;   /* RX */
    dma-names = "tx", "rx";
    /* بدون الـ dma-names، الـ rx_stream callback مش هيلاقي channel */
};
```

**Debug:**

```bash
# شوف الـ DMA channels المتاحة للـ SPI
cat /sys/bus/platform/devices/44009000.spi/dma*

# تحقق من الـ offload capabilities
cat /sys/kernel/debug/spi/spi2/offload_caps
```

#### الدرس المستفاد
الـ `devm_spi_offload_rx_stream_request_dma_chan()` بيـcheck إن الـ `ops->rx_stream_request_dma_chan` callback موجودة — لو مش موجودة، `-EOPNOTSUPP`. الـ provider لازم يـimplements الـ callback لو هو بيقول إنه بيدعم `SPI_OFFLOAD_CAP_RX_STREAM_DMA`. الـ capability flags في `get_offload()` لازم يتحققوا من الـ provider side.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — الـ Trigger Probe Defer Loop

#### العنوان
**`-EPROBE_DEFER`** infinite loop للـ SPI offload trigger consumer على i.MX8QM

#### السياق
Automotive ECU بيستخدم i.MX8QM مع radar sensor يتكلم عبر SPI. الـ radar driver بيحتاج trigger من hardware timer خارجي. الـ system بيبوت لكن الـ radar driver مش بيشتغل أبدًا — لوج بيظهر كل 30 ثانية يقول probe defer.

#### المشكلة
```
spi-radar imx8-radar: spi_offload_trigger_get deferred
spi-radar imx8-radar: spi_offload_trigger_get deferred
[repeated forever]
```

#### التحليل
**الكود في `spi_offload_trigger_get()` — السطر 181-182:**

```c
if (!match)
    return ERR_PTR(-EPROBE_DEFER);
```

الـ function دي بتدور على الـ trigger في الـ `spi_offload_triggers` global list:

```c
// spi-offload.c:172-179
list_for_each_entry(trigger, &spi_offload_triggers, list) {
    if (trigger->fwnode != args->fwnode)
        continue;

    match = trigger->ops->match(trigger, type, args->args, args->nargs);
    if (match)
        break;
}
```

الـ `-EPROBE_DEFER` معناه إن الـ trigger provider مش اتسجّل في الـ list لسه. لكن في حالتنا، الـ provider بيظهر في الـ log إنه probe صح...

**نتحقق من الـ `devm_spi_offload_trigger_get()` — السطر 214-218:**

```c
ret = fwnode_property_get_reference_args(dev_fwnode(offload->provider_dev),
                                         "trigger-sources",
                                         "#trigger-source-cells", 0, 0,
                                         &args);
if (ret)
    return ERR_PTR(ret);
```

المشكلة هنا! الـ `fwnode_property_get_reference_args()` بيدور على `trigger-sources` في الـ `provider_dev` node مش في الـ consumer node. الـ engineer حط الـ property في الـ radar device node بالغلط.

**الـ DT الغلط:**
```dts
radar@0 {
    compatible = "company,imx8-radar";
    reg = <0>;
    trigger-sources = <&timer1 0>;  /* غلط — ده لازم يكون في provider */
    #trigger-source-cells = <0>;
};
```

**الـ DT الصح** — الـ `trigger-sources` لازم تكون في الـ SPI controller node (الـ provider):

```dts
&lpspi0 {
    /* الـ provider بيعرّف مصدر الـ trigger بتاعه */
    trigger-sources = <&timer1 0>;
    #trigger-source-cells = <1>;

    radar@0 {
        compatible = "company,imx8-radar";
        reg = <0>;
        spi-max-frequency = <10000000>;
    };
};
```

#### الحل

**تصحيح الـ DT:**

```dts
/* imx8qm-ecu.dts */
&lpspi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_lpspi0>;
    trigger-sources = <&gpt1 0>;   /* GPT timer كـ trigger source */
    #trigger-source-cells = <1>;
    status = "okay";

    radar: spi-radar@0 {
        compatible = "company,imx8-radar-spi";
        reg = <0>;
        spi-max-frequency = <10000000>;
        spi-cpha;
    };
};
```

**تحقق من الـ fwnode matching:**

```bash
# شوف الـ trigger-sources property في الـ provider node
fdtget /boot/imx8qm-ecu.dtb /soc/bus@5a000000/spi@5a820000 trigger-sources

# تحقق من الـ registered triggers
cat /sys/kernel/debug/spi/offload/triggers 2>/dev/null || \
    dmesg | grep -i "offload.*trigger.*register"
```

**في الـ consumer driver:**

```c
/* offload->provider_dev هو الـ SPI controller، مش الـ radar device */
trigger = devm_spi_offload_trigger_get(dev, offload,
                                        SPI_OFFLOAD_TRIGGER_PERIODIC);
if (IS_ERR(trigger)) {
    /* -EPROBE_DEFER معناه الـ timer provider لسه ما probe-دش */
    return dev_err_probe(dev, PTR_ERR(trigger),
                         "failed to get offload trigger\n");
}
```

#### الدرس المستفاد
الـ `devm_spi_offload_trigger_get()` بيقرأ الـ `trigger-sources` من `offload->provider_dev` — وده هو الـ SPI **controller** node، مش الـ peripheral device node. الفرق ده مهم جدًا في الـ DT layout. لو الـ `fwnode` مش بيتطابق مع أي entry في الـ `spi_offload_triggers` list، الـ result هو `-EPROBE_DEFER` إلى الأبد.

---

### السيناريو الخامس: Custom Board Bring-up على RK3562 — الـ TX Static Data مش شغال مع Periodic Trigger

#### العنوان
تشغيل أول **SPI offload** من صفر على custom board بـRK3562 — الـ data خاطئة

#### السياق
startup بتبني custom board على RK3562 لتطبيق industrial control. الـ board بيستخدم DAC (AD5760) عبر SPI لإنتاج waveform دورية. المطلوب إن الـ SPI controller يبعت waveform data بشكل دوري تلقائي بـ`SPI_OFFLOAD_CAP_TX_STATIC_DATA` و `SPI_OFFLOAD_CAP_TRIGGER`. الـ bring-up الأول ظهرت فيه بيانات عشوائية على الـ DAC.

#### المشكلة
الـ trigger بيتفعّل، الـ DMA بيشتغل، لكن الـ DAC output عشوائي وغير متوقع. الـ oscilloscope بيظهر SPI transfers بتحصل بالتوقيت الصح لكن البيانات غلط.

#### التحليل
**خطوة 1 — نتحقق من الـ trigger enable sequence:**

```c
// spi-offload.c:280-307
int spi_offload_trigger_enable(struct spi_offload *offload,
                               struct spi_offload_trigger *trigger,
                               struct spi_offload_trigger_config *config)
{
    guard(mutex)(&trigger->lock);

    if (!trigger->ops)
        return -ENODEV;

    /* الأول: enable الـ offload في الـ controller */
    if (offload->ops && offload->ops->trigger_enable) {
        ret = offload->ops->trigger_enable(offload);
        if (ret)
            return ret;
    }

    /* التاني: enable الـ trigger source */
    if (trigger->ops->enable) {
        ret = trigger->ops->enable(trigger, config);
        if (ret) {
            /* لو الـ trigger فشل، disable الـ offload تاني */
            if (offload->ops && offload->ops->trigger_disable)
                offload->ops->trigger_disable(offload);
            return ret;
        }
    }

    return 0;
}
```

الترتيب صح: الـ offload controller بيتـenable الأول، بعدين الـ trigger. المشكلة مش هنا.

**خطوة 2 — نتحقق من الـ validate step:**

```c
// spi-offload.c:248-260
int spi_offload_trigger_validate(struct spi_offload_trigger *trigger,
                                 struct spi_offload_trigger_config *config)
{
    guard(mutex)(&trigger->lock);

    if (!trigger->ops)
        return -ENODEV;

    if (!trigger->ops->validate)
        return -EOPNOTSUPP;

    return trigger->ops->validate(trigger, config);
}
```

الـ engineer لقى إن `validate` بترجع `-EOPNOTSUPP` فـskip الـ validation خالص! لكن الـ `validate` callback من المفروض تضبط `config->periodic.frequency_hz` على القيمة الأقرب المدعومة من الـ hardware.

**الـ config بعد الـ validate:**

```c
struct spi_offload_trigger_config config = {
    .type = SPI_OFFLOAD_TRIGGER_PERIODIC,
    .periodic = {
        .frequency_hz = 100000,  /* طلبنا 100 kHz */
    },
};

ret = spi_offload_trigger_validate(trigger, &config);
/* لو -EOPNOTSUPP، config.periodic.frequency_hz لسه 100000 */
/* لكن الـ hardware ممكن يدور على 98304 Hz أو حاجة تانية */
/* النتيجة: الـ SPI clock مش متزامن مع الـ DAC update rate */
```

الـ timing مش دقيق، وبالتالي الـ DAC بيتحدّث في أوقات غير متوقعة مما يخلي الـ waveform خاطئة.

#### الحل

**خطوة 1 — implement الـ validate callback في الـ RK3562 controller driver:**

```c
static int rk3562_spi_offload_trigger_validate(struct spi_offload_trigger *trigger,
                                                struct spi_offload_trigger_config *config)
{
    struct rk3562_spi_offload *priv = spi_offload_trigger_get_priv(trigger);
    u64 actual_freq;

    if (config->type != SPI_OFFLOAD_TRIGGER_PERIODIC)
        return -EINVAL;

    /* اضبط الـ frequency على أقرب قيمة مدعومة */
    actual_freq = rk3562_clk_round_rate(priv->clk,
                                         config->periodic.frequency_hz);

    /* حدّث الـ config بالقيمة الفعلية */
    config->periodic.frequency_hz = actual_freq;

    return 0;
}
```

**خطوة 2 — في الـ consumer driver، تحقق من الـ adjusted frequency:**

```c
ret = spi_offload_trigger_validate(trigger, &config);
if (ret && ret != -EOPNOTSUPP) {
    dev_err(dev, "trigger validate failed: %d\n", ret);
    return ret;
}

/* تحقق إن الـ adjusted frequency مقبولة */
if (config.periodic.frequency_hz < min_freq ||
    config.periodic.frequency_hz > max_freq) {
    dev_err(dev, "adjusted freq %llu Hz out of range\n",
            config.periodic.frequency_hz);
    return -EINVAL;
}

dev_info(dev, "using trigger frequency: %llu Hz\n",
         config.periodic.frequency_hz);
```

**خطوة 3 — verify the waveform على الـ oscilloscope:**

```bash
# تحقق من الـ SPI clock على الـ scope
# وكمان راقب الـ actual trigger frequency
cat /sys/kernel/debug/clk/spi0_offload_clk/clk_rate

# شغّل الـ offload وراقب
echo 1 > /sys/class/dac/dac0/enable_offload
```

**خطوة 4 — التسلسل الصح للـ bring-up:**

```c
/* 1. اطلب الـ offload */
offload = devm_spi_offload_get(dev, spi, &offload_config);

/* 2. اطلب الـ trigger */
trigger = devm_spi_offload_trigger_get(dev, offload,
                                        SPI_OFFLOAD_TRIGGER_PERIODIC);

/* 3. validate الـ config أولًا */
ret = spi_offload_trigger_validate(trigger, &trigger_config);

/* 4. فعّل بالـ validated config */
ret = spi_offload_trigger_enable(offload, trigger, &trigger_config);

/* 5. لما تخلص */
spi_offload_trigger_disable(offload, trigger);
```

#### الدرس المستفاد
الـ `spi_offload_trigger_validate()` مش optional — ده الخطوة اللي بتخلي الـ hardware يقول لك "أنا هشتغل على التردد ده بالظبط". لو الـ provider مش implement الـ `validate` callback (وبترجع `-EOPNOTSUPP`)، لازم تعمله لأن غيابها معناه إن الـ `frequency_hz` اللي حطّيته ممكن ما يتنفّذش بدقة. في الـ bring-up الجديد، دايمًا implement الـ validate callback وابدأ بـlog الـ actual values قبل ما تـenable أي trigger.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [spi: axi-spi-engine: add offload support](https://lwn.net/Articles/998396/) | أهم مقال مباشر — يشرح أول تطبيق لـ SPI offload framework في kernel مع AXI SPI Engine و AD7944 ADC، ويوضح كيف يتم تسجيل SPI transactions وإعادة تشغيلها بـ hardware trigger لتحقيق 2 MSPS |
| [iio: ad7606: add SPI offload support](https://lwn.net/Articles/1016404/) | يغطي إضافة SPI offload support لعائلة ADC الـ ad7606، مثال عملي على استخدام الـ framework مع peripheral driver |
| [simple SPI framework](https://lwn.net/Articles/154425/) | المقال التاريخي الأصلي اللي عرّف الـ SPI subsystem في Linux kernel — أساس لفهم التطور |
| [Deterministic SPI latency with NXP DSPI driver](https://lwn.net/Articles/798467/) | يناقش تحسين الـ latency في SPI transfers — سياق مهم لفهم ليه offloading ضروري |

---

### التوثيق الرسمي في kernel

**الـ Documentation paths** المهمة:

```
Documentation/driver-api/spi.rst          # الوثيقة الرئيسية لـ SPI subsystem
Documentation/devicetree/bindings/spi/    # device tree bindings للـ SPI controllers
```

**الـ kernel docs online:**

- [Serial Peripheral Interface (SPI) — The Linux Kernel documentation](https://docs.kernel.org/driver-api/spi.html) — الوثيقة الرسمية الكاملة للـ SPI subsystem API
- [SPI userspace API](https://docs.kernel.org/spi/spidev.html) — واجهة الـ spidev لـ userspace

**الـ headers المهمة في الـ source tree:**

```
include/linux/spi/offload/types.h     # structs و flags الأساسية
include/linux/spi/offload/consumer.h  # API للـ consumer drivers
include/linux/spi/offload/provider.h  # API للـ provider (controller) drivers
drivers/spi/spi-offload.c             # التنفيذ الأساسي للـ framework
```

---

### مناقشات Mailing List وكلام المطورين

| الرابط | الوصف |
|--------|-------|
| [PATCH RFC v3 — spi: dt-bindings: add spi-offload properties](https://lore.kernel.org/lkml/20240722-dlech-mainline-spi-engine-offload-2-v3-1-7420e45df69b@baylibre.com/) | النسخة v3 من الـ patchset الأصلي (يوليو 2024) — David Lechner من BayLibre بالنيابة عن Analog Devices |
| [RFC v4 05/15 — spi: dt-bindings: add PWM SPI offload trigger](https://lore.kernel.org/lkml/9669117f-667e-4a2b-b815-c49bf0731eec@baylibre.com/) | النقاش حول trigger bindings في النسخة v4 (أكتوبر 2024) |
| [RFC v4 03/15 — spi: offload: add support for hardware triggers](https://patchwork-proxy.ozlabs.org/project/linux-pwm/patch/20241023-dlech-mainline-spi-engine-offload-2-v4-3-f8125b99f5a1@baylibre.com/) | التفاصيل التقنية لـ hardware trigger support في الـ framework |

**الـ LKML:** [`linux-kernel mailing list - 2024/10/05`](https://lists.openwall.net/linux-kernel/2024/10/05/) — أرشيف المناقشات المحيطة بالـ patchset

---

### Analog Devices Wiki

- [SPI Engine Offload FPGA Peripheral](https://wiki.analog.com/resources/fpga/peripherals/spi_engine/offload) — توثيق الـ AXI SPI Engine offload peripheral من Analog Devices، يشرح الـ hardware side: trigger input، TX/RX memory، و replay logic

---

### BayLibre — تقارير المساهمات

- [BayLibre contributions to Linux v6.15](https://baylibre.com/baylibre-contributions-to-linux-v6-15/) — تفصيل للـ SPI offload contributions في v6.15
- [BayLibre contributions to Linux v6.16](https://baylibre.com/baylibre-contributions-to-linux-v6-16/) — استمرار التطوير في v6.16

---

### kernelnewbies.org — تتبع تطور الـ subsystem

| الصفحة | الأهمية |
|--------|---------|
| [Linux_6.12](https://kernelnewbies.org/Linux_6.12) | الـ kernel version اللي شهدت إضافة SPI offload framework الأولية |
| [Linux_6.13](https://kernelnewbies.org/Linux_6.13) | متابعة التطوير مع إضافة controller و peripheral drivers |
| [Linux_6.18](https://kernelnewbies.org/Linux_6.18) | آخر تحديثات موثقة في الـ SPI subsystem |
| [Linux_2_6_24](https://kernelnewbies.org/Linux_2_6_24) | أول ظهور لـ SPI support في kernel — للتاريخ |

---

### elinux.org — موارد عملية

| الصفحة | الأهمية |
|--------|---------|
| [RPi SPI](https://elinux.org/RPi_SPI) | مثال عملي لاستخدام SPI على Raspberry Pi — مفيد لفهم الـ spidev interface |
| [SPI Memory Support in Linux and U-Boot (PDF)](https://elinux.org/images/7/73/Raynal-spi-memories.pdf) | ورقة بحثية عن SPI memory support — سياق أوسع للـ SPI ecosystem |
| [An Introduction to SPI-NOR Subsystem (PDF)](https://elinux.org/images/4/4e/An_Introduction_to_SPI-NOR_Subsystem_-_v3_0.pdf) | تعمق في SPI-NOR — فهم subsystem مرتبط يساعد على فهم التصميم العام |

---

### كتب مُوصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 14:** الـ Linux Device Model — لفهم `device`, `devm_*`, `kref`
- **الفصل 15:** Memory Mapping and DMA — لفهم `dma_chan` والـ streaming
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17:** Devices and Modules — فهم الـ device model و `EXPORT_SYMBOL_GPL`
- **الفصل 5:** System Calls — فهم الـ kernel/userspace boundary
- مفيد لفهم `mutex`, `kref`, `list_head` اللي الـ framework بيستخدمها بكثافة

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 15:** Embedded Linux Drivers — SPI driver development من منظور embedded
- يغطي real-world scenarios قريبة من use case الـ ADC/DAC اللي الـ SPI offload framework بيخدمها

#### Programming Embedded Systems — Michael Barr & Anthony Massa
- مرجع لفهم الـ hardware side: SPI protocol، timing، trigger signals

---

### search terms للبحث عن معلومات أكثر

```
# بحث عام في الـ kernel mailing list
site:lore.kernel.org "spi offload"
site:lore.kernel.org "spi_offload"

# بحث في LWN
site:lwn.net "SPI offload"
site:lwn.net "axi-spi-engine"

# بحث عن الـ drivers اللي بتستخدم الـ framework
site:lore.kernel.org "devm_spi_offload_get"
site:lore.kernel.org "spi_offload_trigger_enable"

# بحث عن real hardware implementations
"axi spi engine offload" linux kernel
"ad7944" OR "ad7380" SPI offload linux
"SPI_OFFLOAD_CAP_TRIGGER" kernel

# GitHub search
repo:torvalds/linux path:drivers/spi "offload"
repo:torvalds/linux path:include/linux/spi/offload

# documentation
docs.kernel.org SPI offload
kernel.org SPI subsystem documentation
```

---

### ملخص Commit المؤسس

الـ SPI offload framework اتقدم بواسطة **David Lechner** من **BayLibre** بالنيابة عن **Analog Devices Inc.** في النصف الثاني من 2024. الـ patchset مر بـ 4 نسخ RFC قبل الدمج في الـ mainline، وكان الهدف الأساسي هو تمكين الـ ADC من التسجيل بمعدل **2 MSPS** باستخدام AXI SPI Engine بدون أي تدخل من الـ CPU في الـ hot path.

الـ `drivers/spi/spi-offload.c` هو قلب الـ framework — بيوفر الـ devm-managed lifecycle لـ offload instances و triggers، وربطهم بـ DMA channels للـ streaming.
## Phase 8: Writing simple module

### الفكرة

**`devm_spi_offload_alloc`** هي أنسب function نعمل عليها kprobe — بتتحكم في تخصيص الـ offload instance اللي بيقدر الـ SPI controller يشتغل بدون CPU. هنراقب كل استدعاء ليها ونطبع حجم الـ private data المطلوب واسم الـ device.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * spi_offload_probe.c
 *
 * kprobe on devm_spi_offload_alloc() — prints device name and
 * requested private-data size every time an offload is allocated.
 */

#include <linux/module.h>      /* MODULE_* macros, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* kprobe API */
#include <linux/device.h>      /* struct device, dev_name() */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe على devm_spi_offload_alloc لرصد تخصيص SPI offload instances");

/* ------------------------------------------------------------------ */
/* kprobe handler — بيتنادى قبل ما devm_spi_offload_alloc تبدأ تشتغل */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * x86-64 calling convention:
     *   rdi = first arg  → struct device *dev
     *   rsi = second arg → size_t priv_size
     *
     * على ARM64 بيكون:
     *   x0 = dev, x1 = priv_size
     */
#ifdef CONFIG_X86_64
    struct device *dev   = (struct device *)regs->di;
    size_t         priv_size = (size_t)regs->si;
#elif defined(CONFIG_ARM64)
    struct device *dev   = (struct device *)regs->regs[0];
    size_t         priv_size = (size_t)regs->regs[1];
#else
    /* fallback — مش هنطبع اسم الـ device */
    struct device *dev   = NULL;
    size_t         priv_size = 0;
#endif

    pr_info("[spi_offload_probe] devm_spi_offload_alloc called: "
            "dev=\"%s\"  priv_size=%zu bytes\n",
            dev ? dev_name(dev) : "<unknown>",
            priv_size);

    return 0; /* 0 = استمر التنفيذ بشكل طبيعي */
}

/* ------------------------------------------------------------------ */
/* تعريف الـ kprobe struct                                             */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "devm_spi_offload_alloc", /* اسم الـ symbol المراد hook-ه */
    .pre_handler = handler_pre,              /* handler قبل التنفيذ          */
};

/* ------------------------------------------------------------------ */
/* module_init — بيسجّل الـ kprobe                                     */
/* ------------------------------------------------------------------ */
static int __init spi_offload_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[spi_offload_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[spi_offload_probe] planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit — بيلغي تسجيل الـ kprobe قبل ما الـ module يُرفع      */
/* ------------------------------------------------------------------ */
static void __exit spi_offload_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[spi_offload_probe] kprobe removed from %s\n", kp.symbol_name);
}

module_init(spi_offload_probe_init);
module_exit(spi_offload_probe_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيعرّف `struct kprobe`، `register_kprobe`، `unregister_kprobe` |
| `linux/device.h` | عشان `struct device` و`dev_name()` اللي بنستخدمهم في الـ handler |
| `linux/module.h` | الـ macros الأساسية زي `module_init` و`MODULE_LICENSE` |

الـ includes دي كافية لأننا مش بنستدعي أي SPI offload API — بس بنراقبه من بره.

---

#### الـ handler_pre

**الـ `pre_handler`** بيتنادى مباشرةً قبل تنفيذ `devm_spi_offload_alloc` — يعني إحنا شايفين الـ arguments قبل ما أي تخصيص يحصل. بنسحب `dev` و`priv_size` من الـ registers مباشرةً حسب الـ calling convention للـ architecture، وده أسلم من محاولة تحويل الـ stack يدوياً.

الـ `dev_name(dev)` بتدينا اسم الـ device كـ string جاهز للطباعة، فنعرف مين طلب الـ offload (مثلاً `spi0` أو `spi-pl022.0`).

---

#### الـ kprobe struct

```c
static struct kprobe kp = {
    .symbol_name = "devm_spi_offload_alloc",
    .pre_handler = handler_pre,
};
```

**الـ `symbol_name`** بيخلي الـ kernel يبحث عن عنوان الـ function في الـ kallsyms تلقائياً — مش محتاجين نحدد عنوان يدوي. لو الـ function مش exported أو مش موجودة، `register_kprobe` هترجع error.

---

#### module_init

`register_kprobe` بتزرع **breakpoint** (على x86: `INT3`) في أول byte من الـ function المستهدفة. لو فشلت (مثلاً الـ function مش قابلة للـ probe أو `CONFIG_KPROBES` معطّل)، بنرجع الـ error فوراً ومش بنكمل.

---

#### module_exit

**لازم** نعمل `unregister_kprobe` في الـ exit عشان الـ kernel يرجع الـ byte الأصلي (بدل الـ breakpoint) ويحرر الـ resources. لو متعملناش، أي call لـ `devm_spi_offload_alloc` بعد كده هيحاول ينادي handler في memory اتحررت → kernel panic.

---

### كيفية البناء والتشغيل

```bash
# Makefile بسيط في نفس المجلد
# obj-m += spi_offload_probe.o

make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
sudo insmod spi_offload_probe.ko

# شوف الـ output
sudo dmesg | grep spi_offload_probe

# إزالة الـ module
sudo rmmod spi_offload_probe
```

### مثال على الـ output المتوقع

```
[spi_offload_probe] planted kprobe at devm_spi_offload_alloc (ffffffffc0a12340)
[spi_offload_probe] devm_spi_offload_alloc called: dev="spi0"  priv_size=128 bytes
[spi_offload_probe] devm_spi_offload_alloc called: dev="spi1"  priv_size=64 bytes
[spi_offload_probe] kprobe removed from devm_spi_offload_alloc
```

> **ملاحظة:** الـ module ده محتاج `CONFIG_KPROBES=y` و`CONFIG_KALLSYMS=y` في الـ kernel config، وكمان `devm_spi_offload_alloc` لازم تكون في `kallsyms` — وده متوفر لأنها `EXPORT_SYMBOL_GPL`.
