## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبع له الملف

الـ `include/linux/mux/consumer.h` بيتبع لـ **MULTIPLEXER SUBSYSTEM** في Linux kernel، المسؤول عنه Peter Rosin من Axentia Technologies. الـ subsystem ده موجود في:
- `drivers/mux/`
- `include/linux/mux/`
- `Documentation/devicetree/bindings/mux/`

---

### الصورة الكبيرة — تخيل الموضوع بالحياة الواقعية

#### القصة: مفتاح الكهرباء الذكي

تخيل إن عندك **حنفية مياه واحدة** بس في المطبخ، وعندك 4 خراطيم — واحد لحديقة يمين، وواحد لحديقة شمال، وواحد للسطح، وواحد لأحواض الزهور. مش ممكن تفتح الـ 4 مع بعض لأن الضغط هيقل ومش هيحصل أي حاجة. الحل؟ **موزع (manifold) بصنبور دوار** تختار منه خرطوم واحد بس في كل وقت.

ده بالظبط هو الـ **Multiplexer** في الـ hardware.

في العالم الحقيقي للـ embedded systems:
- عندك **ADC واحد** (محول إشارة تناظرية لرقمية) وعشر حساسات.
- عندك **I2C bus واحد** وعليه أجهزة بنفس العنوان.
- عندك **SPI bus واحد** وعشر chips.

المشكلة: مش ممكن تتكلم مع الكل في نفس الوقت. الحل: **MUX chip** — بتختار مين يتوصل بالـ bus في كل لحظة.

---

### الـ Multiplexer Subsystem — ليه موجود في الـ kernel؟

قبل الـ subsystem ده، كل driver كان يتعامل مع الـ MUX بطريقته الخاصة — كل واحد بـ GPIO مختلف، أو register مختلف، بدون أي تنسيق. الناتج: chaos، وكل ما جاء driver جديد يحتاج MUX لازم يعيد اختراع العجلة.

الـ **MULTIPLEXER SUBSYSTEM** اتعمل عشان:
1. **يوحد الواجهة** بين أي consumer (driver يريد يستخدم MUX) وأي hardware MUX.
2. **يمنع التعارض** — لو driver A فتح قناة 3، driver B ينتظر لحد ما A يخلص.
3. **يدعم أي نوع MUX** سواء GPIO-based أو MMIO-based أو I2C-controlled.

---

### دور الملف `consumer.h` تحديداً

الـ subsystem عنده **طرفان**:

| الطرف | الملف | الدور |
|-------|-------|-------|
| **Driver جانب** (اللي يصنع الـ MUX) | `include/linux/mux/driver.h` | ازاي تسجل MUX chip في الـ kernel |
| **Consumer جانب** (اللي يستخدم الـ MUX) | `include/linux/mux/consumer.h` | ازاي تطلب MUX وتشغله |

`consumer.h` هو **بطاقة التحكم** اللي بيمسكها أي driver يريد يستخدم MUX. بيعرف:

- **`mux_control_get` / `devm_mux_control_get`** — احجز لك handle على MUX معين.
- **`mux_control_select` / `mux_control_select_delay`** — اختار state معينة (قناة معينة) مع lock تلقائي.
- **`mux_control_try_select`** — نفس الكلام لكن non-blocking (لو مش متاح يرجع error فوراً).
- **`mux_control_deselect`** — خلص من الـ MUX وارجعه لـ idle state وافتح الـ lock للتاني.
- **`mux_state_*`** — نسخة pre-configured، الـ consumer عنده state محفوظ مسبقاً من الـ device tree.

#### الفرق بين `mux_control` و `mux_state`

- الـ **`mux_control`** هو الـ MUX نفسه — عندك N state تختار منهم وقت ما تريد.
- الـ **`mux_state`** هو MUX + state محددة مسبقاً من الـ DT — يعني الـ consumer مش محتاج يعرف رقم الـ state، الـ kernel جابها من الـ device tree له.

---

### السيناريو الكامل — من البداية للنهاية

```
Hardware:
  ADC chip ← MUX chip (8 channels) ← 8 sensors

Device Tree:
  sensor-driver {
      muxes = <&my_mux 3>;   /* أنا عايز قناة رقم 3 */
  }

Driver Code (consumer):
  mux = devm_mux_control_get(dev, "sensor-mux");
  mux_control_select(mux, 3);   /* اختار قناة 3، lock تلقائي */
  /* اقرأ من الـ ADC */
  mux_control_deselect(mux);    /* ارجع لـ idle، حرر الـ lock */
```

لو في نفس الوقت driver تاني عمل `select` على نفس الـ MUX — **ينتظر** (semaphore) لحد ما الأول يعمل `deselect`.

---

### الـ Files المكونة للـ Subsystem

#### Core & Headers

| الملف | الدور |
|-------|-------|
| `include/linux/mux/consumer.h` | واجهة الـ consumer — موضوع شرحنا |
| `include/linux/mux/driver.h` | واجهة الـ driver — `struct mux_chip`, `struct mux_control`, ops |
| `drivers/mux/core.c` | القلب — تسجيل الـ chip، الـ select/deselect logic، الـ locking |

#### Hardware Drivers

| الملف | الـ Hardware |
|-------|-------------|
| `drivers/mux/gpio.c` | MUX يتحكم فيه عن طريق GPIO pins |
| `drivers/mux/mmio.c` | MUX يتحكم فيه عن طريق memory-mapped register |
| `drivers/mux/adg792a.c` | Analog Devices ADG792A — MUX chip محدد |
| `drivers/mux/adgs1408.c` | Analog Devices ADGS1408 — MUX chip محدد |

#### Documentation & Bindings

| الملف | المحتوى |
|-------|---------|
| `Documentation/devicetree/bindings/mux/` | ازاي تعرّف الـ MUX في device tree |
| `Documentation/ABI/testing/sysfs-class-mux*` | الـ sysfs interface للـ MUX |

---

### ملفات مهمة يعرفها القارئ

- **`include/linux/mux/driver.h`** — لازم تقراه جنب `consumer.h` عشان تفهم `struct mux_control` الموجود في كل الـ function signatures.
- **`drivers/mux/core.c`** — فيه الـ implementation الفعلي لكل functions الـ `consumer.h`، وفيه تعريف `struct mux_state` اللي متخفي عن الـ consumers.
- **`drivers/mux/gpio.c`** — أبسط مثال على driver بيستخدم `driver.h` لتسجيل MUX chip.
- **`include/dt-bindings/mux/mux.h`** — بيعرف constants زي `MUX_IDLE_AS_IS` و `MUX_IDLE_DISCONNECT`.
## Phase 2: شرح الـ MUX (Multiplexer) Framework

---

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في الـ embedded systems، كتير جداً تلاقي hardware فيها **signal multiplexer** — يعني قطعة بتقدر توجّه signal من أكتر من مصدر لوجهة واحدة (أو العكس). المشكلة مش في الـ hardware نفسه، المشكلة في الـ software:

- كل driver كان بيتعامل مع الـ mux بطريقته الخاصة — مفيش abstraction موحدة.
- لو driver A و driver B الاتنين محتاجين نفس الـ mux chip، مين بيعمل locking؟ مين بيتأكد إن الاتنين مش بيتعاركوا على نفس الـ resource؟
- لو الـ mux اتغيّر state وانقطع التيار في النص، مين بيرجع الـ mux للـ idle state؟

قبل الـ MUX subsystem (أُضيف في kernel 4.13)، الـ drivers كانت بتعمل كل ده يدوياً، وكانت النتيجة code مكرر وbugs في الـ concurrency.

---

### الحل — الـ Kernel بيتعامل ازاي؟

الـ kernel قدّم **MUX framework** — layer of abstraction موحدة بتعمل التالي:

1. **فصل الـ provider عن الـ consumer**: الـ driver اللي بيتحكم في الـ hardware مش نفسه اللي بيستخدمه.
2. **Mutual exclusion تلقائي**: الـ framework بيستخدم **semaphore** لحماية كل `mux_control` — consumer واحد بس يقدر يـselect في نفس الوقت.
3. **Idle state management**: لما consumer يخلص من الـ mux، الـ framework بيرجعه لـ **idle state** تلقائياً لو محدد.
4. **Device Tree integration**: الـ consumers بيطلبوا الـ mux بالاسم من الـ DT، مش hard-coded addresses.

---

### تشبيه من الواقع — السكة الحديد

تخيّل محطة قطارات كبيرة فيها **switch track** — القضبان اللي بتوجّه القطر من خط لخط تاني.

| مفهوم في السكة الحديد | مقابله في MUX Framework |
|---|---|
| الـ switch track (التحويلة) | `struct mux_control` — الـ mux controller نفسه |
| لوحة التحكم في المحطة | `struct mux_chip` — الـ chip اللي فيها المتحكمات |
| القطر اللي عايز يعدي | الـ consumer driver |
| مأمور الإشارات | الـ MUX core — بيدير الـ locking والـ state |
| رقم الخط المطلوب | الـ `state` (رقم من 0 لـ N-1) |
| الخط الافتراضي لما مفيش قطر | `idle_state` |
| التذكرة (حجز الخط) | `mux_control_select()` — بتاخد الـ semaphore |
| تسليم الخط بعد ما القطر يعدي | `mux_control_deselect()` — بترجع الـ semaphore |
| خطان مستقلين في نفس المحطة | مصفوفة `mux_chip->mux[]` — أكتر من `mux_control` في نفس الـ chip |
| قطر تاني استنى يعدي | `down_killable(&mux->lock)` — blocking wait على الـ semaphore |
| قطر مش مستني — يكمل في خط بديل | `mux_control_try_select()` — بترجع `-EBUSY` لو مش متاح |

الربط الأعمق: زي ما مأمور الإشارات مش محتاج يعرف إيه اللي شايل القطر — بضاعة ولا ركاب — الـ MUX core مش محتاج يعرف إيه الـ consumer بيعمله بعد ما يـselect الـ state. هو بس بيضمن إن ما فيش اتنين بيستخدموا نفس التحويلة في نفس الوقت.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Consumer Drivers                            │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  IIO driver  │  │  SPI driver  │  │  PHY / ADC driver    │  │
│  │  (ADC mux)   │  │  (chip sel)  │  │  (signal routing)    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                 │                       │              │
│         └─────────────────┴───────────────────────┘             │
│                           │                                      │
│              mux_control_get() / devm_mux_control_get()         │
│              mux_state_get()   / devm_mux_state_get()           │
└───────────────────────────┼──────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MUX Core (drivers/mux/core.c)                │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Consumer API                                             │   │
│  │  mux_control_select_delay()  mux_control_try_select()    │   │
│  │  mux_control_deselect()      mux_control_states()        │   │
│  │  mux_state_select()          mux_state_deselect()        │   │
│  └───────────────────────────┬──────────────────────────────┘   │
│                              │                                    │
│  ┌───────────────────────────▼──────────────────────────────┐   │
│  │  Internal Logic                                           │   │
│  │  - semaphore locking per mux_control                     │   │
│  │  - cached_state (avoid redundant hardware writes)        │   │
│  │  - idle_state restoration on deselect                    │   │
│  │  - settle time delay (delay_us after last_change)        │   │
│  │  - OF/DT phandle resolution                              │   │
│  └───────────────────────────┬──────────────────────────────┘   │
│                              │                                    │
│  ┌───────────────────────────▼──────────────────────────────┐   │
│  │  Driver API (provider side)                               │   │
│  │  mux_chip_alloc()  mux_chip_register()                   │   │
│  │  mux_chip_free()   mux_chip_unregister()                 │   │
│  │  devm_mux_chip_alloc()  devm_mux_chip_register()         │   │
│  └───────────────────────────┬──────────────────────────────┘   │
└───────────────────────────────┼─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MUX Drivers (Providers)                      │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐ │
│  │  gpio mux    │  │  mmio mux    │  │  ADG792A / ADGS1408   │ │
│  │  (GPIOs set  │  │  (register   │  │  (SPI-controlled mux) │ │
│  │   state)     │  │   write)     │  │                       │ │
│  └──────────────┘  └──────────────┘  └───────────────────────┘ │
│                                                                  │
│    كل driver بينفذ ops->set(mux, state) بس — مش أكتر           │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Real Hardware                             │
│                                                                  │
│   GPIO pins / MMIO registers / SPI registers                    │
│   → select which physical signal path is active                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ MUX framework بيقدم abstraction مبنية على **اتنين structs أساسيين**:

#### 1. `struct mux_chip` — الـ Container

```c
struct mux_chip {
    unsigned int controllers;       /* عدد الـ mux controllers في الـ chip */
    struct device dev;              /* embedded device — مش pointer، embedded */
    int id;                         /* رقم unique من IDA */
    const struct mux_control_ops *ops; /* vtable — ops->set() بس */
    struct mux_control mux[] __counted_by(controllers); /* flexible array */
};
```

الـ `mux_chip` هو الـ **chip المادي** — ممكن يكون IC واحد فيه 4 أو 8 mux controllers مستقلين. الـ `mux[]` هو **flexible array member** — الـ kernel بيعمل `kzalloc` للـ chip والـ controllers في block واحد في الـ memory.

#### 2. `struct mux_control` — الـ Core Unit

```c
struct mux_control {
    struct semaphore lock;  /* mutual exclusion — واحد بس في نفس الوقت */
    struct mux_chip *chip;  /* back-pointer للـ chip */
    int cached_state;       /* آخر state معروف، أو MUX_CACHE_UNKNOWN */
    unsigned int states;    /* عدد الـ states الصالحة (من 0 لـ states-1) */
    int idle_state;         /* state لما مش في استخدام */
    ktime_t last_change;    /* وقت آخر تغيير — للـ settle time */
};
```

الـ **semaphore** هنا مش mutex — اختيار مقصود. الـ mutex في Linux بيقول إن اللي اخده هو بس اللي يقدر يرجعه. الـ semaphore مش عنده القيد ده — أي consumer يقدر يعمل `up()`. بس عملياً الـ consumer بيعمل `down()` في `select` و `up()` في `deselect`، فالـ usage pattern هو mutual exclusion.

#### 3. `struct mux_state` — الـ Pre-configured Handle

```c
/* defined in core.c — مش exposed في driver.h */
struct mux_state {
    struct mux_control *mux;   /* pointer للـ mux_control */
    unsigned int state;        /* الـ state المحدد مسبقاً في الـ DT */
};
```

الـ `mux_state` هو **convenience wrapper** — بدل ما الـ consumer يحتفظ بـ pointer للـ `mux_control` ورقم الـ state منفصلين، الـ `mux_state` بيجمعهم. لو driver دايماً بيستخدم نفس الـ state (مثلاً ADC input رقم 3)، يـget `mux_state` بدل `mux_control`.

---

### الـ Struct Relationships

```
┌─────────────────────────────────────────────┐
│              struct mux_chip                 │
│  ┌──────────────────────────────────────┐   │
│  │  struct device dev (embedded)        │   │
│  │    .class = &mux_class               │   │
│  │    .parent = gpio/i2c/spi device     │   │
│  └──────────────────────────────────────┘   │
│  controllers = 2                             │
│  ops ──────────────────────────────────┐    │
│                                         │    │
│  mux[0] ┌────────────────────────────┐ │    │
│         │ struct mux_control         │ │    │
│         │   lock (semaphore)         │ │    │
│         │   chip ──► (back to chip)  │ │    │
│         │   cached_state = 1         │ │    │
│         │   states = 4               │ │    │
│         │   idle_state = 0           │ │    │
│         │   last_change = ktime      │ │    │
│         └────────────────────────────┘ │    │
│  mux[1] ┌────────────────────────────┐ │    │
│         │ struct mux_control         │ │    │
│         │   ...                      │ │    │
│         └────────────────────────────┘ │    │
│                                         │    │
│  [priv data — sizeof_priv bytes]        │    │
└─────────────────────────────────────────┘    │
                                               │
                                               ▼
                              ┌─────────────────────────────┐
                              │   struct mux_control_ops    │
                              │   int (*set)(mux, state)    │
                              └─────────────────────────────┘

      ┌──────────────────────────────────────────┐
      │           struct mux_state               │
      │   mux ──────────────────► mux_control    │
      │   state = 3  (pre-baked state number)    │
      └──────────────────────────────────────────┘
```

---

### الـ State Machine لـ mux_control_select / deselect

```
        ┌─────────────────────────────────────────────────────┐
        │                   IDLE / FREE                        │
        │  cached_state = idle_state (أو MUX_CACHE_UNKNOWN)  │
        │  semaphore count = 1                                 │
        └──────────────────┬──────────────────────────────────┘
                           │
              mux_control_select(mux, state)
                           │
              down_killable(&mux->lock)  ←── blocks if busy
                           │
                           ▼
        ┌─────────────────────────────────────────────────────┐
        │              LOCKED / IN USE                         │
        │  cached_state = requested state                      │
        │  semaphore count = 0                                 │
        │  hardware physically set to state                    │
        └──────────────────┬──────────────────────────────────┘
                           │
              mux_control_deselect(mux)
                           │
              if idle_state != MUX_IDLE_AS_IS:
                  ops->set(mux, idle_state)
                           │
              up(&mux->lock)
                           │
                           ▼
                        IDLE / FREE
```

---

### الـ Settle Time — تفصيلة مهمة

بعض الـ multiplexers بتحتاج وقت بعد التغيير عشان الـ signal يستقر (**settling time**). الـ framework بيتعامل مع ده بذكاء:

```c
static void mux_control_delay(struct mux_control *mux, unsigned int delay_us)
{
    ktime_t delayend;
    s64 remaining;

    if (!delay_us)
        return;

    /* هنحسب الوقت المتبقي من آخر تغيير */
    delayend = ktime_add_us(mux->last_change, delay_us);
    remaining = ktime_us_delta(delayend, ktime_get());
    if (remaining > 0)
        fsleep(remaining);  /* sleep بس اللي فضل فعلاً */
}
```

يعني لو consumer طلب delay 100µs، والـ mux اتغيّر من 80µs، الـ kernel بينام بس 20µs — مش 100µs كاملة. ده **تحسين في الأداء** مش واضح لأول وهلة.

---

### الـ Idle State — ثلاثة احتمالات

| القيمة | المعنى |
|---|---|
| `MUX_IDLE_AS_IS` (=-1) | متغيرش الـ state لما ما فيش consumer |
| `MUX_IDLE_DISCONNECT` (=-2) | افصل الـ mux تماماً (لو الـ hardware يدعم ده) |
| رقم موجب (0, 1, 2, ...) | ارجع لـ state محدد كـ idle |

الـ driver بيحدد ده في `mux_chip->mux[i].idle_state` قبل الـ register. الـ default هو `MUX_IDLE_AS_IS`.

---

### الـ DT Integration — ازاي الـ consumer بيطلب الـ mux؟

الـ framework مبني على الـ Device Tree. الـ consumer بيحط في الـ DT node بتاعه:

```dts
/* الـ consumer */
adc@0 {
    compatible = "ti,ads1015";
    mux-controls = <&mux1 2>;       /* chip mux1، controller index 2 */
    mux-control-names = "input-mux";
};

/* أو لو عايز pre-baked state */
sensor@1 {
    mux-states = <&mux1 0 3>;       /* chip mux1، controller 0، state 3 */
    mux-state-names = "my-channel";
};
```

الـ `mux_get()` في الـ core بيعمل:
1. يدور على `mux-control-names` / `mux-state-names` عشان يلاقي الـ index
2. يـparse الـ phandle: `of_parse_phandle_with_args()`
3. يدور على الـ `mux_chip` المسجل بالـ `of_node`: `of_find_mux_chip_by_node()`
4. لو مش موجود لسه → يرجع `-EPROBE_DEFER` (الـ kernel يحاول تاني بعدين)

---

### الـ Framework بيملك إيه vs بيفوّض إيه؟

| المسؤولية | الـ MUX Core يملكها | الـ Driver بيعملها |
|---|---|---|
| Mutual exclusion (locking) | ✅ semaphore per mux_control | ❌ |
| State caching | ✅ cached_state | ❌ |
| Idle state restoration | ✅ في deselect() | ❌ |
| Settle time management | ✅ last_change + delay | ❌ |
| DT phandle resolution | ✅ mux_get() | ❌ |
| Device lifecycle (devres) | ✅ devm_* variants | ❌ |
| Hardware write | ❌ | ✅ `ops->set(mux, state)` |
| عدد الـ states | ❌ | ✅ `mux->states = N` |
| الـ idle_state القيمة | ❌ | ✅ `mux->idle_state = X` |
| الـ chip private data | ❌ | ✅ `mux_chip_priv(chip)` |

الـ driver Interface بسيط جداً — الـ driver بينفّذ **function واحدة فقط**:

```c
struct mux_control_ops {
    int (*set)(struct mux_control *mux, int state);
};
```

كل الـ complexity التانية — locking، caching، DT، devres — الـ framework بيتكفل بيها.

---

### Consumer API — الفرق بين الـ variants

| الدالة | الـ Blocking؟ | الـ Delay؟ | متى تستخدمها؟ |
|---|---|---|---|
| `mux_control_select()` | ✅ ينتظر | ❌ | الحالة العادية |
| `mux_control_select_delay()` | ✅ ينتظر | ✅ | لو الـ hardware محتاج settle time |
| `mux_control_try_select()` | ❌ `-EBUSY` | ❌ | real-time contexts أو fallback logic |
| `mux_control_try_select_delay()` | ❌ `-EBUSY` | ✅ | try + settle time |
| `mux_state_select()` | ✅ ينتظر | ❌ | pre-configured state handle |
| `mux_control_deselect()` | — | — | **إلزامي** بعد أي select ناجح |

**ملاحظة مهمة**: الـ `__must_check` على كل الـ select functions — لو الـ driver تجاهل الـ return value، الـ compiler بيطلع warning. ده لأن لو `select()` فشلت، **ما تعملش** `deselect()` — المتحكم مش locked، وعمل `deselect()` هيعمل `up()` على semaphore مش مأخود.

---

### مثال كامل — GPIO MUX Driver

الـ driver في `drivers/mux/gpio.c` هو أبسط مثال:

```c
/* كل اللي الـ driver بيعمله هو set() */
static int mux_gpio_set(struct mux_control *mux, int state)
{
    struct mux_gpio *mux_gpio = mux_chip_priv(mux->chip);
    int i;

    /* كل bit في الـ state بيتحول لـ GPIO value */
    for (i = 0; i < mux_gpio->ngpios; i++) {
        gpiod_set_value_cansleep(mux_gpio->gpios[i], state & 1);
        state >>= 1;
    }
    return 0;
}

static const struct mux_control_ops mux_gpio_ops = {
    .set = mux_gpio_set,   /* ده بس — الـ framework يتكفل بالباقي */
};
```

يعني مثلاً 3 GPIOs → 8 states (000 لـ 111 ثنائي). الـ framework يحدد إن `states = 8`، والـ driver بيكتب الـ bits على الـ GPIOs.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Constants والـ Flags والـ Special Values

#### الـ Idle State Constants (من `dt-bindings/mux/mux.h`)

| Constant | القيمة | المعنى |
|---|---|---|
| `MUX_IDLE_AS_IS` | `-1` | ابق على الـ state الحالي لما المux مش مستخدم |
| `MUX_IDLE_DISCONNECT` | `-2` | افصل الـ input/output تمامًا (high impedance) |
| `MUX_CACHE_UNKNOWN` | `-1` (= `MUX_IDLE_AS_IS`) | الـ cached state مش معروف (internal marker) |

#### الـ Return Values من الـ Consumer API

| الحالة | القيمة | السبب |
|---|---|---|
| نجاح | `0` | الـ state اتضبط وال lock اتأخد |
| فشل عام | errno سالب | خطأ في الـ hardware أو باراميتر غلط |
| `-EBUSY` | `-16` | الـ mux مشغول (`try_select` بس) |
| `-EINVAL` | `-22` | state خارج النطاق أو باراميتر غلط |
| `-EPROBE_DEFER` | `-517` | الـ mux chip لسه ما اتسجلش |

#### الـ DT Properties المستخدمة في `mux_get()`

| Property | نوعها | الاستخدام |
|---|---|---|
| `mux-controls` | phandle array | يربط consumer بـ mux controller |
| `mux-control-names` | string array | أسماء الـ controllers للـ lookup بالاسم |
| `mux-states` | phandle array | يربط consumer بـ mux controller + state محددين |
| `mux-state-names` | string array | أسماء الـ states للـ lookup بالاسم |
| `#mux-control-cells` | integer | عدد الخلايا في الـ phandle (0 أو 1) |
| `#mux-state-cells` | integer | عدد الخلايا في الـ phandle (1 أو 2) |

---

### 1. الـ Structs المهمة

#### `struct mux_control_ops`
**الغرض:** جدول عمليات الـ hardware — الـ vtable الوحيد في النظام كله.

| Field | النوع | الوصف |
|---|---|---|
| `set` | `int (*)(struct mux_control *, int state)` | اضبط الـ hardware على state معين، يرجع 0 أو errno |

الـ driver بيحط pointer على struct من النوع ده في `mux_chip->ops`. في وقت الـ select، الـ core بيعمل `mux->chip->ops->set(mux, state)`.

---

#### `struct mux_control`
**الغرض:** تمثّل controller واحد جوه الـ chip — ممكن يكون فيها أكتر من واحد.

| Field | النوع | من يكتبه | الوصف |
|---|---|---|---|
| `lock` | `struct semaphore` | core فقط | يحمي الـ state ويضمن exclusive access |
| `chip` | `struct mux_chip *` | core فقط | back-pointer للـ chip الأم |
| `cached_state` | `int` | core فقط | الـ state الحالي في الـ hardware، أو `MUX_CACHE_UNKNOWN` |
| `states` | `unsigned int` | driver (قبل register) | عدد الـ states الصالحة (0 .. states-1) |
| `idle_state` | `int` | driver (قبل register) | الـ state لما المux مش مستخدم |
| `last_change` | `ktime_t` | core فقط | وقت آخر تغيير للـ state (لحساب الـ settling delay) |

**العلاقات:**
- `mux_control->chip` → `mux_chip` (الأم)
- الـ `mux_control` نفسها جزء من array جوه `mux_chip->mux[]`

---

#### `struct mux_chip`
**الغرض:** تمثّل الـ physical chip الكاملة — ممكن تحتوي على أكتر من controller.

| Field | النوع | الوصف |
|---|---|---|
| `controllers` | `unsigned int` | عدد الـ controllers جوه الـ chip |
| `dev` | `struct device` | الـ kernel device object (class = "mux") |
| `id` | `int` | ID فريد من `mux_ida` (يتسمى `muxchipN`) |
| `ops` | `const struct mux_control_ops *` | الـ vtable — الـ driver يحطها قبل الـ register |
| `mux[]` | `struct mux_control[]` | flexible array من الـ controllers (+ private data بعدهم) |

**ملاحظة مهمة:** الـ `mux_chip_alloc()` بتخصص memory للـ `mux[]` + `sizeof_priv` bytes إضافية في نفس الـ allocation. الـ `mux_chip_priv()` بترجع pointer على الجزء الإضافي ده.

---

#### `struct mux_state`
**الغرض:** wrapper بسيط يربط `mux_control` بـ state محدد — مريح للـ consumer اللي عنده state ثابت.

| Field | النوع | الوصف |
|---|---|---|
| `mux` | `struct mux_control *` | pointer على الـ controller |
| `state` | `unsigned int` | الـ state المحدد لهذا الـ consumer |

الـ `mux_state` بيتعمل بـ `kzalloc` في `mux_state_get()` وبيتحرر في `mux_state_put()`. مش جزء من الـ `mux_chip` في الـ memory.

---

### 2. مخطط علاقات الـ Structs

```
  ┌─────────────────────────────────────────────────────┐
  │                   struct mux_chip                   │
  │                                                     │
  │  controllers = N                                    │
  │  dev  ──────────────────────────► struct device     │
  │  id   = 3  (muxchip3)                               │
  │  ops ──────────────────────────► struct mux_control_ops
  │                                    └─ set()         │
  │  mux[0] ┐                                           │
  │  mux[1] ├── struct mux_control (embedded array)     │
  │  mux[2] ┘                                           │
  │  [priv data]  ◄── mux_chip_priv()                   │
  └─────────────────────────────────────────────────────┘
           │
           │  mux[i].chip  (back-pointer)
           ▼
  ┌──────────────────────────────┐
  │     struct mux_control       │
  │                              │
  │  lock  (semaphore, count=1)  │
  │  chip ───────────────────────┼──► mux_chip (أعلاه)
  │  cached_state = 2            │
  │  states = 4                  │
  │  idle_state = MUX_IDLE_AS_IS │
  │  last_change = ktime         │
  └──────────────────────────────┘
           ▲
           │  mstate->mux
  ┌──────────────────────────────┐
  │      struct mux_state        │
  │                              │
  │  mux   ──────────────────────┘
  │  state = 2                   │
  └──────────────────────────────┘
           ▲
           │  يستخدمها الـ consumer
  ┌──────────────────────────────┐
  │    Consumer Driver           │
  │  (e.g. i2c-mux, iio, etc.)  │
  └──────────────────────────────┘
```

---

### 3. مخططات الـ Lifecycle

#### Driver Side — دورة حياة الـ mux_chip

```
  [Driver Probe]
       │
       ▼
  mux_chip_alloc(dev, N, sizeof_priv)
       │  ← kzalloc للـ chip + N controllers + priv
       │  ← ida_alloc → chip->id
       │  ← سما_init لكل mux->lock
       │  ← mux->cached_state = MUX_CACHE_UNKNOWN
       │  ← device_initialize()
       ▼
  [Driver يضبط: chip->ops, mux[i].states, mux[i].idle_state]
       │
       ▼
  mux_chip_register(mux_chip)
       │  ← يحط كل controller على idle_state لو مش unknown
       │  ← device_add() → chip يظهر في sysfs
       ▼
  [Chip جاهز للاستخدام من الـ consumers]
       │
       │  ... حياة طبيعية ...
       │
       ▼
  mux_chip_unregister(mux_chip)
       │  ← device_del()
       ▼
  mux_chip_free(mux_chip)
       │  ← put_device() → مع آخر reference
       │       → mux_chip_release()
       │             → ida_free(chip->id)
       │             → kfree(chip)
       ▼
  [انتهى]
```

#### Consumer Side — دورة حياة الـ mux_control

```
  [Consumer Probe]
       │
       ▼
  devm_mux_control_get(dev, "name")
       │  ← يقرأ DT: mux-controls / mux-control-names
       │  ← of_find_mux_chip_by_node() → get_device() على الـ chip
       │  ← يرجع &chip->mux[controller_index]
       ▼
  [Consumer عنده pointer على mux_control]
       │
       ▼
  mux_control_select(mux, state)   ← blocking
  أو
  mux_control_try_select(mux, state)  ← non-blocking
       │
       ▼
  [استخدام الـ hardware]
       │
       ▼
  mux_control_deselect(mux)
       │
       ▼
  [Consumer Remove]  ← devm يعمل mux_control_put() تلقائيًا
       │  ← put_device() على الـ chip
       ▼
  [انتهى]
```

#### Consumer Side — دورة حياة الـ mux_state

```
  [Consumer Probe]
       │
       ▼
  devm_mux_state_get(dev, "name")
       │  ← kzalloc لـ struct mux_state
       │  ← mux_get(dev, name, &state) → يملا mstate->mux و mstate->state
       ▼
  [Consumer عنده mux_state جاهز]
       │
       ▼
  mux_state_select(mstate)   ←  = mux_control_select(mstate->mux, mstate->state)
       │
       ▼
  [استخدام الـ hardware]
       │
       ▼
  mux_state_deselect(mstate)
       │
       ▼
  [Consumer Remove]  ← devm يعمل mux_state_put() تلقائيًا
       │  ← mux_control_put(mstate->mux)
       │  ← kfree(mstate)
       ▼
  [انتهى]
```

---

### 4. مخططات تدفق الـ Calls

#### تدفق `mux_control_select_delay()`

```
consumer calls mux_control_select_delay(mux, state, delay_us)
  │
  ├─► down_killable(&mux->lock)
  │     │  ← blocking semaphore down
  │     │  ← يرجع -EINTR لو العملية اتقتلت
  │
  ├─► __mux_control_select(mux, state)
  │     │
  │     ├─► WARN_ON(state < 0 || state >= mux->states)
  │     │
  │     ├─► if (mux->cached_state == state) → return 0  [no-op]
  │     │
  │     └─► mux_control_set(mux, state)
  │               │
  │               └─► mux->chip->ops->set(mux, state)
  │                         │  ← الـ hardware driver بيضبط الـ physical mux
  │                         │
  │                   ← يحدّث mux->cached_state
  │                   ← يحدّث mux->last_change = ktime_get()
  │
  ├─► [لو نجح] mux_control_delay(mux, delay_us)
  │     │  ← يحسب الوقت المتبقي من last_change
  │     │  ← fsleep(remaining) لو في وقت متبقي
  │
  ├─► [لو فشل] up(&mux->lock)  ← يفك الـ lock لو الـ select فشل
  │
  └─► return 0 أو errno
```

#### تدفق `mux_control_try_select_delay()`

```
consumer calls mux_control_try_select_delay(mux, state, delay_us)
  │
  ├─► down_trylock(&mux->lock)
  │     ├─► لو المux مشغول → return -EBUSY فورًا (non-blocking)
  │     └─► لو free → يأخد الـ lock ويكمل
  │
  └─► [نفس تدفق __mux_control_select + delay أعلاه]
```

#### تدفق `mux_control_deselect()`

```
consumer calls mux_control_deselect(mux)
  │
  ├─► if (idle_state != MUX_IDLE_AS_IS && idle_state != cached_state)
  │     └─► mux_control_set(mux, idle_state)
  │               └─► ops->set(mux, idle_state)  ← يرجع المux لحالة الـ idle
  │
  └─► up(&mux->lock)  ← يفك الـ lock دايمًا حتى لو الـ set فشل
```

#### تدفق `mux_get()` — الـ DT Lookup

```
mux_get(dev, mux_name, state_ptr)
  │
  ├─► لو mux_name != NULL
  │     └─► of_property_match_string(np, "mux-[control|state]-names", mux_name)
  │               → index في الـ array
  │
  ├─► of_parse_phandle_with_args(np, "mux-[controls|states]", ...)
  │     → args.np = الـ mux chip node
  │     → args.args[] = [controller_index, state_value]
  │
  ├─► of_find_mux_chip_by_node(args.np)
  │     └─► class_find_device_by_of_node(&mux_class, np)
  │               → get_device() على الـ chip (يزود الـ refcount)
  │
  ├─► تحليل args.args_count لاستخراج controller index و state
  │
  └─► return &mux_chip->mux[controller]
```

---

### 5. استراتيجية الـ Locking

#### الـ Lock الوحيد: `mux_control->lock` (semaphore)

الـ subsystem كله يعتمد على **semaphore واحد** لكل `mux_control`.

```
  ┌────────────────────────────────────────────────────────┐
  │              mux_control->lock  (semaphore, init=1)    │
  │                                                        │
  │  يحمي:                                                 │
  │    - mux->cached_state  (قراءة + كتابة)               │
  │    - العملية الكاملة: select → استخدام → deselect     │
  │    - استدعاء ops->set()                                │
  └────────────────────────────────────────────────────────┘
```

| السيناريو | الـ Lock Behavior |
|---|---|
| `mux_control_select()` | `down_killable()` — blocking، قابل للـ interrupt بـ signal |
| `mux_control_try_select()` | `down_trylock()` — non-blocking، يرجع `-EBUSY` فورًا |
| `mux_control_deselect()` | `up()` دايمًا — حتى لو الـ idle state set فشل |
| فشل الـ `__mux_control_select` | `up()` فورًا — مش من حق الـ caller يعمل deselect |

#### قواعد الـ Locking

1. **الـ lock يتأخد في `select` ويتحرر في `deselect` فقط** — مش جزئي.
2. **لو `select` فشل → لا تعمل `deselect`** — الـ lock اتحرر تلقائيًا.
3. **لو `select` نجح → لازم تعمل `deselect`** — حتى لو العملية نفسها فشلت.
4. **الـ `last_change` و `cached_state`** بيتكتبوا بس وقت الـ lock ممسوك.
5. **مفيش lock عالـ `mux_chip`** — الـ `struct device` بيستخدم الـ reference counting (`get_device` / `put_device`) للـ lifetime management.

#### ترتيب الـ Locks (Lock Ordering)

بما إن كل `mux_control` عنده lock مستقل، ومفيش كود بياخد أكتر من lock في نفس الوقت، **مفيش خطر deadlock من داخل الـ subsystem**.

لو الـ consumer driver عنده lock خاص بيه:

```
  consumer_lock  →  mux_control->lock   ✓ (الترتيب الصح)
  mux_control->lock  →  consumer_lock   ✗ (ممنوع — يسبب deadlock)
```

أي كود بياخد `mux_control->lock` (عن طريق `select`) يجب ألا يحاول يأخد lock تاني أعلى في التسلسل الهرمي.
## Phase 4: شرح الـ Functions

---

### ملخص الـ API — Cheatsheet

#### Consumer API (الـ header المعني)

| Function | Signature Summary | Purpose |
|---|---|---|
| `mux_control_states` | `(mux) → uint` | عدد الـ states المتاحة |
| `mux_control_select_delay` | `(mux, state, delay_us) → int` | Select blocking مع delay |
| `mux_control_select` | `(mux, state) → int` | Select blocking بدون delay |
| `mux_control_try_select_delay` | `(mux, state, delay_us) → int` | Select non-blocking مع delay |
| `mux_control_try_select` | `(mux, state) → int` | Select non-blocking بدون delay |
| `mux_control_deselect` | `(mux) → int` | تحرير الـ mux بعد الاستخدام |
| `mux_state_select_delay` | `(mstate, delay_us) → int` | Select state object blocking |
| `mux_state_select` | `(mstate) → int` | Select state object بدون delay |
| `mux_state_try_select_delay` | `(mstate, delay_us) → int` | Select state object non-blocking |
| `mux_state_try_select` | `(mstate) → int` | Select state object non-blocking بدون delay |
| `mux_state_deselect` | `(mstate) → int` | تحرير الـ mux عبر state object |
| `mux_control_get` | `(dev, name) → mux_control*` | الحصول على handle |
| `mux_control_put` | `(mux)` | تحرير الـ handle |
| `devm_mux_control_get` | `(dev, name) → mux_control*` | managed get |
| `devm_mux_state_get` | `(dev, name) → mux_state*` | managed state get |

#### Internal / Core Helpers

| Function | Visibility | Purpose |
|---|---|---|
| `mux_control_set` | static | Low-level driver `ops->set` wrapper |
| `__mux_control_select` | static | Core select logic (lock يجب يكون مأخوذ) |
| `mux_control_delay` | static | تطبيق الـ settling delay |
| `mux_get` | static | DT parsing → mux_control pointer |
| `mux_state_get` | static | Allocate + fill mux_state |
| `mux_state_put` | static | Free mux_state |
| `of_find_mux_chip_by_node` | static | DT node → mux_chip lookup |

---

### Group 1: Query API

#### الغرض من المجموعة
الـ consumer محتاج يعرف إيه الـ states المتاحة قبل ما يختار. دي الـ API الوحيدة اللي مش بتعمل lock.

---

#### `mux_control_states`

```c
unsigned int mux_control_states(struct mux_control *mux);
```

بترجع عدد الـ states المتاحة في الـ mux controller. الـ states صح هي `0` لحد `states - 1`. الـ driver بيضبط `mux->states` وقت الـ allocation.

| Parameter | النوع | الوصف |
|---|---|---|
| `mux` | `struct mux_control *` | الـ mux controller المطلوب |

**Return:** عدد الـ states — دايمًا قيمة موجبة.

**Key details:** مفيش locking هنا — `mux->states` بيتضبط مرة واحدة وقت الـ registration وما بيتغيرش. مناسب للاستدعاء من أي context.

---

### Group 2: Selection API (Blocking)

#### الغرض من المجموعة
الـ consumer بيطلب state معين ويستنى لو الـ mux مشغول بـ consumer تاني. الـ lock هنا هو `semaphore` (`mux->lock`) يتضمن إن consumer واحد بس بيستخدم الـ mux في وقت واحد.

---

#### `mux_control_select_delay`

```c
int __must_check mux_control_select_delay(struct mux_control *mux,
                                          unsigned int state,
                                          unsigned int delay_us);
```

بتعمل `down_killable` على الـ semaphore فبتستنى لو الـ mux مأخوذ، لكن قابلة للـ kill signal. لو نجحت، بتستدعي `__mux_control_select` اللي بتكتب الـ state الجديد عبر `ops->set`. بعدين بتطبق الـ `delay_us` كـ settling time عشان الـ signal يستقر.

| Parameter | الوصف |
|---|---|
| `mux` | الـ mux controller |
| `state` | الـ state المطلوب (0 ≤ state < mux->states) |
| `delay_us` | وقت الانتظار بعد التغيير بالـ microseconds |

**Return:** `0` عند النجاح، errno سالب عند الخطأ (`-EINTR` لو اتـ kill أثناء الانتظار، `-EINVAL` لو state خارج النطاق، errno من `ops->set` لو فشل الـ hardware).

**Key details:**
- لو الـ select فشل، الـ semaphore بيتـ `up` تلقائيًا — الـ caller **لا يستدعي** `mux_control_deselect` في حالة الفشل.
- لو نجح، الـ lock بيبقى ممسوك لحد ما `mux_control_deselect` يتستدعى.
- الـ `delay_us` بيحسب من `mux->last_change` مش من وقت الاستدعاء — لو الـ state ما اتغيرش (cached) الـ delay مش بيتطبق.

**Pseudocode:**
```
mux_control_select_delay(mux, state, delay_us):
    ret = down_killable(&mux->lock)   // block until free, or killed
    if ret < 0: return ret

    ret = __mux_control_select(mux, state)
    if ret >= 0:
        mux_control_delay(mux, delay_us)  // wait for signal settling
    else:
        up(&mux->lock)                    // release on failure

    return ret
```

---

#### `mux_control_select`

```c
static inline int __must_check mux_control_select(struct mux_control *mux,
                                                  unsigned int state);
```

**الـ inline wrapper** اللي بيستدعي `mux_control_select_delay` بـ `delay_us = 0`. الاستخدام الأشيع لما الـ hardware مش محتاج settling time.

---

#### `mux_state_select_delay`

```c
int __must_check mux_state_select_delay(struct mux_state *mstate,
                                        unsigned int delay_us);
```

بتستدعي `mux_control_select_delay(mstate->mux, mstate->state, delay_us)` مباشرة. الـ `mux_state` بيكپسل الـ (mux + state) كـ pre-configured pair — الـ consumer مش محتاج يمرر الـ state صراحةً.

| Parameter | الوصف |
|---|---|
| `mstate` | الـ mux state object اللي اتجاب عبر `devm_mux_state_get` |
| `delay_us` | settling time |

**Return:** نفس `mux_control_select_delay`.

---

#### `mux_state_select`

```c
static inline int __must_check mux_state_select(struct mux_state *mstate);
```

Wrapper بـ `delay_us = 0` على `mux_state_select_delay`.

---

### Group 3: Selection API (Non-Blocking / Try)

#### الغرض من المجموعة
نفس مجموعة الـ blocking، بس الـ caller مش عايز يستنى — لو الـ mux مشغول بيرجع `-EBUSY` على طول. مفيد في الـ interrupt context أو لما الـ driver عنده fallback path.

---

#### `mux_control_try_select_delay`

```c
int __must_check mux_control_try_select_delay(struct mux_control *mux,
                                              unsigned int state,
                                              unsigned int delay_us);
```

بتستخدم `down_trylock` بدل `down_killable` — لو الـ semaphore مشغول بترجع `-EBUSY` فورًا من غير ما تستنى. باقي السلوك متطابق مع `mux_control_select_delay`.

| Parameter | الوصف |
|---|---|
| `mux` | الـ mux controller |
| `state` | الـ state المطلوب |
| `delay_us` | settling time |

**Return:** `0` عند النجاح، `-EBUSY` لو الـ mux مشغول، errno سالب تاني في حالات الخطأ.

**Key details:** آمن للاستدعاء من contexts اللي ممنوع فيها النوم، **لكن** لو الـ `__mux_control_select` رجع صح وضبط الـ state، الـ `mux_control_delay` بيعمل `fsleep` — يعني الـ delay نفسه بينوّم. لو الـ caller في atomic context، يجب إن `delay_us = 0`.

---

#### `mux_control_try_select`

```c
static inline int __must_check mux_control_try_select(struct mux_control *mux,
                                                      unsigned int state);
```

Wrapper بـ `delay_us = 0`.

---

#### `mux_state_try_select_delay`

```c
int __must_check mux_state_try_select_delay(struct mux_state *mstate,
                                            unsigned int delay_us);
```

Delegating wrapper — بيمرر الـ `mstate->mux` و `mstate->state` لـ `mux_control_try_select_delay`.

---

#### `mux_state_try_select`

```c
static inline int __must_check mux_state_try_select(struct mux_state *mstate);
```

Wrapper بـ `delay_us = 0`.

---

### Group 4: Deselect / Release API

#### الغرض من المجموعة
**لازم** يتستدعى بعد كل `select` ناجح. بيرجع الـ mux لـ idle state (لو محدد) وبيحرر الـ semaphore. حتى لو حصل error في الرجوع للـ idle state، الـ lock بيتحرر.

---

#### `mux_control_deselect`

```c
int mux_control_deselect(struct mux_control *mux);
```

لو الـ `mux->idle_state` مش `MUX_IDLE_AS_IS` والـ cached state مختلف عنه، بيستدعي `mux_control_set(mux, idle_state)` عشان يرجع الـ hardware لوضع الـ idle. بعدين بيعمل `up(&mux->lock)` دايمًا — حتى لو فشل الـ set.

| Parameter | الوصف |
|---|---|
| `mux` | نفس الـ mux اللي اتعمله select ناجح |

**Return:** `0` عند النجاح. errno سالب لو الرجوع للـ idle state فشل — بس الـ lock بيتحرر في الحالتين.

**Key details:**
- `up(&mux->lock)` غير مشروطة — الـ caller مش محتاج يعمل cleanup تاني.
- الـ **contract**: كل `select` ناجح → `deselect` واحد بالظبط. ولا `select` فاشل → مفيش `deselect`.

```
mux_control_deselect(mux):
    if idle_state != MUX_IDLE_AS_IS AND idle_state != cached_state:
        ret = mux_control_set(mux, idle_state)   // try to restore idle
    up(&mux->lock)                                // always release
    return ret
```

---

#### `mux_state_deselect`

```c
int mux_state_deselect(struct mux_state *mstate);
```

Thin wrapper — بيستدعي `mux_control_deselect(mstate->mux)`. نفس السلوك تمامًا.

---

### Group 5: Acquisition API (Get/Put)

#### الغرض من المجموعة
الـ consumer قبل ما يقدر يعمل select، محتاج يجيب handle على الـ `mux_control`. الـ get بيعمل DT lookup ويزوّد reference على الـ `mux_chip` device. الـ put بيرجعها.

---

#### `mux_control_get`

```c
struct mux_control *mux_control_get(struct device *dev, const char *mux_name);
```

بتعمل parse لـ DT properties `mux-controls` و `mux-control-names` لتحديد الـ mux المطلوب. بتستدعي `mux_get(dev, mux_name, NULL)` — الـ `NULL` للـ state يدل إنها تشتغل بـ `mux-controls` مش `mux-states`.

| Parameter | الوصف |
|---|---|
| `dev` | الـ consumer device (لازم يكون ليه `of_node`) |
| `mux_name` | اسم الـ mux في `mux-control-names`، أو `NULL` للـ index 0 |

**Return:** Pointer صح لـ `mux_control`، أو `ERR_PTR(-EPROBE_DEFER)` لو الـ mux chip لسه مش registered، أو errno تاني.

**Key details:**
- `EPROBE_DEFER` شايع — الـ mux driver ممكن يتحمّل بعد الـ consumer driver.
- الـ function بتزوّد reference على `mux_chip->dev` عبر `get_device` داخل `of_find_mux_chip_by_node`.

**DT Lookup Flow:**
```
mux_control_get(dev, name):
    index = of_property_match_string(np, "mux-control-names", name)
    args  = of_parse_phandle_with_args(np, "mux-controls", ...)
    chip  = of_find_mux_chip_by_node(args.np)   // get_device internally
    controller = args.args[0] if present else 0
    return &chip->mux[controller]
```

---

#### `mux_control_put`

```c
void mux_control_put(struct mux_control *mux);
```

بتعمل `put_device(&mux->chip->dev)` — بترجع الـ reference اللي أخدتها `mux_control_get`. لو مفيش references تانية، الـ chip ممكن يتحرر.

| Parameter | الوصف |
|---|---|
| `mux` | الـ handle اللي اتجاب بـ `mux_control_get` |

**Return:** void.

**Key details:** الـ caller يجب إنه يتأكد إن الـ mux مش في حالة selected وقت الـ put — لأن الـ semaphore مش بيتحرر هنا.

---

#### `devm_mux_control_get`

```c
struct mux_control *devm_mux_control_get(struct device *dev,
                                         const char *mux_name);
```

الـ resource-managed version. بتستدعي `mux_control_get` وبتسجل `devm_mux_control_release` كـ devres action. لما `dev` يتـ detach أو يتـ destroy، الـ kernel بياخد على عاتقه استدعاء `mux_control_put` تلقائيًا.

| Parameter | الوصف |
|---|---|
| `dev` | الـ consumer device اللي بيتـ managed |
| `mux_name` | اسم الـ mux |

**Return:** نفس `mux_control_get`.

**Key details:**
- الـ `devres_alloc` بيخصص `sizeof(struct mux_control *)` — بيخزن pointer للـ handle.
- في حالة الفشل، بيعمل `devres_free` ويرجع الـ error.
- الـ consumer drivers الحديثة الكل بيستخدم الـ `devm_` version وبتختفي الـ manual cleanup.

---

#### `devm_mux_state_get`

```c
struct mux_state *devm_mux_state_get(struct device *dev,
                                     const char *mux_name);
```

مثل `devm_mux_control_get` بس للـ `mux_state`. بتستدعي `mux_state_get` اللي بتعمل:
1. `kzalloc` لـ `struct mux_state`.
2. `mux_get(dev, mux_name, &mstate->state)` — الـ `state` pointer غير NULL يخلي الـ parse يشتغل على `mux-states` و `#mux-state-cells`.
3. بترجع الـ `mstate` اللي فيه الـ `mux` pointer والـ `state` المحددة من الـ DT.

| Parameter | الوصف |
|---|---|
| `dev` | الـ consumer device |
| `mux_name` | اسم الـ mux state في `mux-state-names` |

**Return:** `struct mux_state *` أو `ERR_PTR`.

**Key details:**
- الـ `mux_state` هو abstraction فوق `mux_control` — بيخزن الـ state المطلوب معه. مفيد لما الـ consumer دايمًا عايز نفس الـ state.
- الـ cleanup عبر `devm_mux_state_release` → `mux_state_put` → `mux_control_put` + `kfree(mstate)`.

---

### Group 6: Internal Helpers (Core Internals)

#### الغرض من المجموعة
الـ functions دي مش exported — بتشكل الـ engine الداخلي للـ consumer subsystem.

---

#### `mux_control_set` (static)

```c
static int mux_control_set(struct mux_control *mux, int state)
```

الـ low-level wrapper الوحيد اللي بيكلم الـ driver مباشرة. بيستدعي `mux->chip->ops->set(mux, state)`. لو نجح، بيحدث `mux->cached_state` و `mux->last_change`. لو فشل، الـ cached state بيبقى `MUX_CACHE_UNKNOWN`.

**Key details:** الـ `last_change` timestamp بيتحدث هنا — ده اللي بيعتمد عليه `mux_control_delay` لحساب الـ remaining settling time.

---

#### `__mux_control_select` (static)

```c
static int __mux_control_select(struct mux_control *mux, int state)
```

**شرط:** الـ `mux->lock` لازم يكون مأخوذ من قبل الـ caller.

بتعمل:
1. Validate إن الـ state في النطاق.
2. لو `cached_state == state` → ترجع 0 بدون ما تكلم الـ hardware (تحسين مهم).
3. استدعاء `mux_control_set`.
4. لو فشل وعنده idle state، بتحاول ترجع لـ idle عبر `mux_control_set(mux, idle_state)`.

```
__mux_control_select(mux, state):
    if state out of range: return -EINVAL
    if cached_state == state: return 0        // no-op, optimized path
    ret = mux_control_set(mux, state)
    if ret >= 0: return 0
    if idle_state != MUX_IDLE_AS_IS:
        mux_control_set(mux, idle_state)      // best-effort revert
    return ret
```

---

#### `mux_control_delay` (static)

```c
static void mux_control_delay(struct mux_control *mux, unsigned int delay_us)
```

بتحسب الوقت اللي فات من آخر تغيير (`mux->last_change`) وبتنام الفرق المتبقي عبر `fsleep`. لو `delay_us == 0` بترجع على طول. لو الوقت عدى بالفعل، مفيش نوم.

**Key details:** التصميم ده ذكي — لو الـ hardware خلص إنه استقر من لحتو، الـ overhead صفر. الـ `fsleep` بتختار أنسب sleep mechanism حسب المدة.

---

#### `mux_get` (static)

```c
static struct mux_control *mux_get(struct device *dev, const char *mux_name,
                                   unsigned int *state)
```

الـ core DT parser. الـ `state` pointer بيحدد الـ mode:
- `NULL` → parse من `mux-controls` / `#mux-control-cells`
- non-NULL → parse من `mux-states` / `#mux-state-cells` ويملأ `*state`

بتستدعي `of_find_mux_chip_by_node` اللي بتعمل `class_find_device_by_of_node` — لو الـ chip مش موجود → `-EPROBE_DEFER`.

---

### الـ Locking Model

```
Consumer Thread A          Consumer Thread B
       │                          │
mux_control_select()        mux_control_select()
       │                          │
  down_killable() ◄──── semaphore (count=1) ────► down_killable()
       │ (acquired)               │ (blocked)
   ops->set()                     │ waiting...
       │                          │
mux_control_deselect()            │
       │                          │
    up() ──────────────────► unblocked
                            ops->set()
                                  │
                         mux_control_deselect()
```

الـ semaphore بيضمن **mutual exclusion** على مستوى كل `mux_control` منفصل — multiplexers مختلفة على نفس الـ chip مستقلين عن بعض.

---

### الـ Usage Pattern الصحيح

```c
/* Acquisition — مرة واحدة في probe */
mux = devm_mux_control_get(dev, "my-mux");
if (IS_ERR(mux))
    return PTR_ERR(mux);  /* -EPROBE_DEFER possible */

/* Runtime — لكل transaction */
ret = mux_control_select(mux, MY_STATE);
if (ret)
    return ret;           /* NO deselect on failure */

/* ... use the hardware ... */

ret = mux_control_deselect(mux);  /* always call on success */
```

---

### جدول الـ Error Codes

| Error | السبب |
|---|---|
| `-EPROBE_DEFER` | الـ mux chip لسه مش registered في الـ sysfs |
| `-EINVAL` | state خارج النطاق، أو DT cells غلط |
| `-EBUSY` | الـ mux مشغول (try_select فقط) |
| `-EINTR` | signal وصل أثناء انتظار الـ semaphore |
| `-ENOMEM` | فشل الـ allocation في devm variants |
| errno من `ops->set` | hardware error من الـ driver |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ sysfs المتعلقة بالـ MUX Subsystem

الـ **mux subsystem** بيسجّل كل chip كـ device تحت class اسمه `mux`، فممكن تلاقيه في:

```bash
# اعرض كل mux chips المسجّلة
ls /sys/class/mux/

# مثال على output:
# muxchip0  muxchip1

# اعرض معلومات الـ device
ls /sys/class/mux/muxchip0/
# device  power  subsystem  uevent

# اعرض الـ parent device
cat /sys/class/mux/muxchip0/device/uevent

# اعرض الـ device name
cat /sys/class/mux/muxchip0/uevent
# DEVTYPE=mux-chip
# SEQNUM=...
```

> **ملاحظة:** الـ mux subsystem مش بيعرض `cached_state` أو `idle_state` مباشرةً في sysfs — المعلومات دي في kernel internal فقط. لازم تستخدم ftrace أو dynamic debug علشان تشوفها.

#### 2. مدخلات الـ debugfs

الـ mux core نفسه مش بيعمل debugfs entries، لكن لو الـ underlying driver (زي GPIO أو I2C) بيعمل، تقدر تعمل:

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# اعرض كل devices من class mux
ls /sys/kernel/debug/

# لو شغّال GPIO mux، تقدر تشوف حالة الـ GPIOs:
ls /sys/kernel/debug/gpio
cat /sys/kernel/debug/gpio | grep -A5 "mux"
# GPIOs 100-103, platform/gpio-mux.0, mux:
#  gpio-100 (mux0              ) out lo
#  gpio-101 (mux1              ) out hi
```

#### 3. استخدام الـ ftrace — Tracepoints والـ Events

الـ **ftrace** هو الأداة الأقوى لمتابعة state transitions في الـ mux:

```bash
# ابدأ بـ function tracing لكل functions في mux core
cd /sys/kernel/debug/tracing

# trace mux_control_select_delay و mux_control_deselect
echo 'mux_control_select_delay' >> set_ftrace_filter
echo 'mux_control_deselect' >> set_ftrace_filter
echo '__mux_control_select' >> set_ftrace_filter
echo 'mux_control_set' >> set_ftrace_filter
echo 'mux_control_try_select_delay' >> set_ftrace_filter

echo function > current_tracer
echo 1 > tracing_on

# شغّل الكود اللي بيستخدم الـ mux
# ...

echo 0 > tracing_on
cat trace
```

**مثال على output:**

```
# TASK-PID   CPU#  IRQS-OFF  NEED-RESCHED  HARDIRQ/SOFTIRQ  PREEMPT-DEPTH  TIMESTAMP  FUNCTION
mydriver-123  [001]  ....  1234.567890: mux_control_select_delay <-mydriver_read
mydriver-123  [001]  ....  1234.567891: __mux_control_select <-mux_control_select_delay
mydriver-123  [001]  ....  1234.567892: mux_control_set <-__mux_control_select
mydriver-123  [001]  ....  1234.567950: mux_control_deselect <-mydriver_read
```

**تفسير الـ output:** الـ timestamp بين `select` و `deselect` = وقت الاحتجاز، لو طويل = أحد لازم ينتظر.

```bash
# trace مع الـ arguments (function_graph أوضح)
echo function_graph > current_tracer
echo 'mux_control_select_delay' > set_graph_function
echo 1 > tracing_on
cat trace | head -50
```

#### 4. تفعيل الـ printk / Dynamic Debug

```bash
# تفعيل dynamic debug لكل messages في mux-core
echo 'module mux_core +p' > /sys/kernel/debug/dynamic_debug/control

# أو بالـ file name
echo 'file drivers/mux/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p = print, f = function name, l = line number, m = module name, t = thread id

# تفعيل debug لـ gpio-mux driver
echo 'file drivers/mux/gpio.c +p' > /sys/kernel/debug/dynamic_debug/control

# تأكد إن التفعيل اشتغل
grep mux /sys/kernel/debug/dynamic_debug/control | grep '=p'

# الرسائل هتظهر في dmesg
dmesg -w | grep -i mux
```

**الـ pr_fmt** في core.c هو `"mux-core: "` — فكل رسائله بتبدأ بيه:

```bash
dmesg | grep "mux-core:"
# [  5.123] mux-core: muxchipX failed to get a device id
```

#### 5. خيارات الـ Kernel Config للـ Debugging

| Config Option | الفايدة |
|---|---|
| `CONFIG_MULTIPLEXER=y` | الـ mux core نفسه |
| `CONFIG_MUX_GPIO=y` | GPIO-controlled mux driver |
| `CONFIG_MUX_MMIO=y` | MMIO/Regmap mux driver |
| `CONFIG_MUX_ADG792A=y` | Analog Devices ADG792A driver |
| `CONFIG_MUX_ADGS1408=y` | Analog Devices ADGS1408 driver |
| `CONFIG_DEBUG_SEMAPHORE` | debug الـ semaphore اللي بيحمي `mux->lock` |
| `CONFIG_LOCK_STAT` | إحصائيات lock contention |
| `CONFIG_PROVE_LOCKING` | lockdep للـ semaphore في `mux_control` |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug بشكل عام |
| `CONFIG_FTRACE` | function tracing |
| `CONFIG_FUNCTION_TRACER` | trace function calls |
| `CONFIG_FUNCTION_GRAPH_TRACER` | graph trace أوضح |
| `CONFIG_OF` | Device Tree support (مطلوب لـ mux lookup) |
| `CONFIG_REGMAP_MMIO` | مطلوب لـ `MUX_MMIO` |

```bash
# تحقق من الـ config في kernel الشغّال
zcat /proc/config.gz | grep -E 'MULTIPLEXER|MUX_|DEBUG_SEMAPHORE|LOCK_STAT'
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# اعرض كل mux devices المسجّلة عن طريق sysfs
find /sys -name "uevent" -exec grep -l "mux-chip" {} \;

# اعرض الـ device tree node المرتبط بكل mux chip
for chip in /sys/class/mux/muxchip*; do
    echo "=== $chip ==="
    cat $chip/uevent
    # اعرض الـ of_node لو موجود
    ls -la $chip/device/of_node 2>/dev/null && \
        cat $chip/device/of_node/compatible 2>/dev/null
done

# اعرض الـ consumers اللي بيستخدموا mux معين
# (عن طريق devres — مش مباشر، بس ممكن تستنتج منه)
cat /sys/kernel/debug/devices_deferred  # لو في EPROBE_DEFER

# لو شغّال I2C mux (ADG792A):
i2cdetect -l
i2cdetect -y 1   # scan bus 1
i2cdump -y 1 0x4c  # dump registers
```

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `mux controller 'name' not found` | الاسم في `mux-control-names` مش موجود أو مش متطابق | تحقق من DT property `mux-control-names` في consumer node |
| `failed to get mux-control name(i)` | `of_parse_phandle_with_args` فشل | تحقق من `mux-controls` phandle في DT |
| `wrong #mux-control-cells for /node` | عدد args في DT مش صح | `#mux-control-cells` لازم يتطابق مع عدد args |
| `wrong #mux-state-cells for /node` | نفس المشكلة لكن لـ mux-states | تحقق من `#mux-state-cells` |
| `bad mux controller N specified` | رقم الـ controller أكبر من عدد الـ controllers المتاحة | تحقق من قيمة controller index في DT |
| `muxchipX failed to get a device id` | IDA allocation فشل — نادر جداً | مشكلة في الـ memory |
| `unable to set idle state` | `mux->ops->set()` فشل وقت الـ registration | مشكلة في الـ hardware أو الـ driver |
| `device_add failed in mux_chip_register: -N` | فشل تسجيل الـ device | تحقق من الـ parent device والـ sysfs |
| `failed to get gpios` | GPIO unavailable — غالباً deferred probe | تأكد إن GPIO controller جاهز |
| `invalid idle-state N` | قيمة `idle-state` في DT أكبر من عدد الـ states | صحّح الـ DT |
| `-EBUSY from mux_control_try_select` | الـ mux محجوز من consumer تاني | استخدم `mux_control_select` (blocking) أو انتظر |
| `-EPROBE_DEFER from mux_control_get` | الـ mux chip لسه مش registered | ترتيب الـ probe — عادي ويتحل تلقائياً |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

الكود الموجود فعلاً بيستخدم:

```c
/* في mux_chip_alloc() */
if (WARN_ON(!dev || !controllers))   /* يكشف برمجة غلط */
    return ERR_PTR(-EINVAL);

/* في __mux_control_select() */
if (WARN_ON(state < 0 || state >= mux->states))  /* state خارج النطاق */
    return -EINVAL;

/* في gpio.c */
WARN_ON(pins != mux_gpio->gpios->ndescs);  /* عدد pins مش متطابق */
```

**نقاط إضافية مقترحة للـ debugging:**

```c
/* في mux_control_set() — تحقق إن الـ ops موجودة */
WARN_ON(!mux->chip->ops || !mux->chip->ops->set);

/* لو `cached_state` بقى MUX_CACHE_UNKNOWN بعد set ناجح */
WARN_ON(mux->cached_state == MUX_CACHE_UNKNOWN && ret >= 0);

/* في mux_control_deselect() — تحقق إن الـ semaphore مش 0 قبل up() */
/* (الـ semaphore count بعد deselect لازم يرجع 1) */
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware متطابقة مع الـ Kernel

```bash
# --- GPIO MUX ---
# اعرض قيم الـ GPIO pins اللي بتتحكم في الـ mux
cat /sys/kernel/debug/gpio

# مثال: mux بـ 3 pins → state = بيناري قيمة الـ pins
# gpio-10 = 1, gpio-11 = 0, gpio-12 = 1 → state = 0b101 = 5

# قارن بـ cached_state اللي في الـ kernel عن طريق ftrace
echo 'mux_control_set' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# --- I2C MUX (ADG792A) ---
# اقرأ الـ register الحالي من الـ chip
i2cget -y 1 0x4c 0x00   # قيم الـ register = state الفعلي

# --- MMIO MUX ---
# اقرأ الـ bitfield من الـ register مباشرةً (شوف القسم التالي)
```

#### 2. تقنيات الـ Register Dump

```bash
# --- devmem2 (أداة userspace) ---
# لازم تعرف الـ physical address من الـ DT أو datasheet

# تثبيت الأداة
apt-get install devmem2   # Debian/Ubuntu
# أو: build من source

# اقرأ register بحجم 32-bit من عنوان 0xFE000000
devmem2 0xFE000000 w

# اكتب قيمة جديدة (احترس!)
devmem2 0xFE000000 w 0x00000002

# --- /dev/mem مباشرةً ---
# اقرأ register في Python
python3 -c "
import mmap, os, struct
fd = os.open('/dev/mem', os.O_RDWR | os.O_SYNC)
BASE = 0xFE000000
SIZE = 0x1000
with mmap.mmap(fd, SIZE, mmap.MAP_SHARED, mmap.PROT_READ|mmap.PROT_WRITE, offset=BASE) as m:
    val = struct.unpack('<I', m.read(4))[0]
    print(f'Register value: 0x{val:08x}')
os.close(fd)
"

# --- io utility (busybox/util-linux) ---
io -4 -r 0xFE000000    # read 32-bit
io -4 -w 0xFE000000 2  # write 2 to register

# --- regmap debugfs (لـ MMIO MUX) ---
ls /sys/kernel/debug/regmap/
# platform-FE000000.syscon/
cat /sys/kernel/debug/regmap/platform-FE000000.syscon/registers
# 00: 00000002   ← الـ bitfield اللي بيتحكم في الـ mux
```

#### 3. نصائح Logic Analyzer / Oscilloscope

**لـ GPIO MUX:**

```
الـ GPIO lines اللي بتتحكم في الـ mux
GPIO_A ─────┐
GPIO_B ─────┤──► [MUX CHIP] ──► Selected Channel
GPIO_C ─────┘

على الـ Logic Analyzer:
- سجّل كل GPIO pins مع الـ SPI/I2C/trigger signal على نفس الـ channel
- ابدأ الـ trigger على rising edge أول GPIO يتغير
- ابحث عن الـ settling time بعد آخر GPIO تغيّر
  (الـ mux_control_delay() بتضمن الانتظار ده)
```

**لـ I2C MUX (ADG792A):**

```
SCL ─────────────────────────────────
SDA ──[START]─[ADDR+W]─[REG]─[DATA]─[STOP]──

على الـ Logic Analyzer:
- فعّل I2C decoder
- ابحث عن transactions لعنوان 0x4C (أو ما يتطابق مع A0,A1,A2 pins)
- كل write transaction = state change
- تأكد من ACK على كل byte
```

**للتحقق من الـ settling time:**

```
Channel output ──────────────────────────────
                                    ↑ يبدأ يستقر
GPIO trigger  ──┐             ┌─────
                └─────────────┘
                |←── delay ──→|
                   (delay_us في mux_control_select_delay)

على الـ Oscilloscope:
- قس الوقت من آخر GPIO change لحد ما الـ signal يستقر
- قارنه بالـ delay_us المبرمج
- لو الـ settling أطول من الـ delay → زوّد الـ delay
```

#### 4. مشاكل Hardware شائعة وأنماطها في الـ Kernel Log

| مشكلة الـ Hardware | النمط في الـ Kernel Log | السبب المحتمل |
|---|---|---|
| الـ MUX مش بيستجاوش | `unable to set idle state` وقت boot | تغذية كهربية مش مستقرة أو I2C fault |
| GPIO مش بيتحرك | `failed to get gpios` أو `gpiod_multi_set_value` بيفشل | GPIO controller مش initialized |
| State غلط بعد select | الـ consumer بيشتكي من بيانات خاطئة | الـ settling time قصير — زوّد `delay_us` |
| Lock إلى الأبد | الـ kernel هانج، لا response | mux selected ومفيش `deselect` (bug في consumer) |
| I2C timeout | `i2c i2c-1: timeout` أو `NACK` | الـ VCC للـ ADG792A مش موجود أو short circuit |
| MMIO write فاشل | `BUG: unable to handle kernel paging request` | عنوان MMIO غلط في DT |
| Contention عالي | kernel log بطيء، latency عالية | multiple consumers بيتقاتلوا على نفس الـ mux |

#### 5. تفعيل Device Tree Debugging والتحقق

```bash
# اعرض الـ DT node المرتبط بالـ mux chip
# المسار بيكون symlink من sysfs للـ DT node
ls -la /sys/class/mux/muxchip0/device/of_node

# اقرأ properties من الـ DT node مباشرةً
# compatible
cat /sys/class/mux/muxchip0/device/of_node/compatible
# مثال: gpio-mux

# اقرأ idle-state
hexdump -C /sys/class/mux/muxchip0/device/of_node/idle-state
# 00000000  00 00 00 ff                        ← MUX_IDLE_AS_IS = -1

# اعرض الـ DT الكامل
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A20 "gpio-mux"

# تحقق من الـ consumer node
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -B2 -A10 "mux-controls"

# مثال على DT صح لـ GPIO mux:
# mux0: mux-controller {
#     compatible = "gpio-mux";
#     #mux-control-cells = <0>;
#     mux-gpios = <&gpio1 10 GPIO_ACTIVE_HIGH>,
#                 <&gpio1 11 GPIO_ACTIVE_HIGH>;
#     idle-state = <0>;
# };

# consumer:
# adc@0 {
#     mux-controls = <&mux0>;
#     mux-control-names = "adc-mux";
# };

# لو في error في الـ DT parsing:
dmesg | grep -E "mux|phandle|of_parse"

# تحقق من #mux-control-cells
hexdump -C /sys/firmware/devicetree/base/path/to/mux/\#mux-control-cells
# 00000000  00 00 00 00                        ← 0 cells (single controller)
# 00000000  00 00 00 01                        ← 1 cell (controller index)
```

---

### Practical Commands

#### 1. أوامر جاهزة للـ Copy

```bash
#!/bin/bash
# === MUX Subsystem Debug Script ===

echo "=== 1. Registered MUX Chips ==="
ls /sys/class/mux/ 2>/dev/null || echo "No mux chips found"

echo ""
echo "=== 2. MUX Device Details ==="
for chip in /sys/class/mux/muxchip*; do
    [ -d "$chip" ] || continue
    echo "--- $chip ---"
    cat "$chip/uevent" 2>/dev/null
    readlink -f "$chip/device" 2>/dev/null
done

echo ""
echo "=== 3. GPIO Pins Used by MUX ==="
cat /sys/kernel/debug/gpio 2>/dev/null | grep -i mux

echo ""
echo "=== 4. Kernel Messages About MUX ==="
dmesg | grep -iE "mux|muxchip" | tail -30

echo ""
echo "=== 5. Deferred Probe (EPROBE_DEFER) ==="
cat /sys/kernel/debug/devices_deferred 2>/dev/null | grep -i mux

echo ""
echo "=== 6. Enable Dynamic Debug for MUX ==="
echo 'module mux_core +p' > /sys/kernel/debug/dynamic_debug/control 2>/dev/null && \
    echo "Dynamic debug enabled for mux_core" || \
    echo "Failed to enable (maybe not compiled with CONFIG_DYNAMIC_DEBUG)"

echo ""
echo "=== 7. Start FTrace for MUX Functions ==="
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > set_ftrace_filter
for fn in mux_control_select_delay mux_control_deselect \
          __mux_control_select mux_control_set \
          mux_control_try_select_delay mux_state_select_delay; do
    echo $fn >> set_ftrace_filter 2>/dev/null
done
echo function > current_tracer
echo 1 > tracing_on
echo "FTrace running — press Enter after reproducing the issue..."
read
echo 0 > tracing_on
echo "=== FTrace Results ==="
cat trace | grep -v "^#" | head -50
```

#### 2. اختبار Lock Contention

```bash
# شوف لو في lock waiting على الـ mux semaphore
# بإنك تـ trace down_killable (اللي بيستخدمها mux_control_select_delay)

cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo 'down_killable' > set_ftrace_filter
echo 'up' >> set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on

# شغّل الـ workload
sleep 5

echo 0 > tracing_on
# ابحث عن down_killable بدون up مباشراً بعده (= waiting)
cat trace | grep -E "down_killable|^.*up " | head -40
```

#### 3. فحص حالة الـ Semaphore عن طريق /proc

```bash
# اعرض كل threads منتظرة على semaphores
cat /proc/*/status | grep -A3 "Name:" | grep -B1 "State.*S "
# أي thread في state "S (sleeping)" ممكن يكون منتظر مux lock

# أو استخدم perf
perf record -g -e sched:sched_stat_sleep -- sleep 5
perf report --stdio | grep mux
```

#### 4. تفسير نتائج الـ FTrace

```
# مثال على trace يكشف delay صح:
 mydriver-456   [000] .... 1000.000100: mux_control_select_delay <-spi_transfer
 mydriver-456   [000] .... 1000.000105: __mux_control_select <-mux_control_select_delay
 mydriver-456   [000] .... 1000.000106: mux_control_set <-__mux_control_select
 mydriver-456   [000] .... 1000.000200: mux_control_deselect <-spi_transfer
#                                        ↑                     ↑
#                               select @ 100µs          deselect @ 200µs
#                               مدة الاحتجاز = 100µs — معقول

# مثال على مشكلة contention:
 consumer1-100  [000] .... 2000.000100: mux_control_select_delay <-...
 consumer2-200  [001] .... 2000.000150: mux_control_select_delay <-...  ← دخل ينتظر
 consumer1-100  [000] .... 2000.050000: mux_control_deselect <-...      ← بعد 50ms
 consumer2-200  [001] .... 2000.050001: __mux_control_select <-...      ← consumer2 اشتغل
# الـ consumer2 انتظر 50ms! ده ممكن يكون مشكلة لو الـ timing حساس
```

#### 5. فحص MMIO Register للـ MUX

```bash
# خطوات:
# 1. احصل على الـ physical address من DT
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    grep -B5 -A20 "bitfield-mux\|mmio-mux\|syscon-mux" | \
    grep "reg ="

# 2. اقرأ الـ register (مثال: address 0x1C000000، offset 0x08)
devmem2 $((0x1C000000 + 0x08)) w
# Output: /dev/mem opened. Value at address 0x1C000008: 0x00000004
# ← لو الـ mux bitfield هو bits [3:2] → value = 1

# 3. احسب الـ state من الـ bitfield
# shift = 2, width = 2 → state = (0x04 >> 2) & 0x3 = 1
python3 -c "val=0x04; shift=2; mask=0x3; print('state =', (val >> shift) & mask)"
# state = 1
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — تعارض الـ SPI MUX بين درايفرين

#### العنوان
**Deadlock** في الـ SPI bus بسبب استخدام `mux_control_select` بدل `mux_control_try_select`

#### السياق
شركة صناعية بتبني **industrial gateway** على SoC **RK3562**. الجهاز عنده SPI bus واحد متصل بـ **analog MUX (74HC4051)**، بيوزع الـ bus على 3 sensors مختلفين. في درايفر الـ sensor الأول (`temp_sensor.c`) وفي درايفر الـ sensor التاني (`pressure_sensor.c`)، الاتنين بيطلبوا الـ MUX من threads مختلفين.

#### المشكلة
الـ system بيتفرز (freeze) بشكل عشوائي تحت load عالي. الـ `dmesg` مش بيظهر crash، بس الـ I/O بيوقف. الـ watchdog بيعمل reset بعد 30 ثانية.

#### التحليل
الكود في `temp_sensor.c`:
```c
/* Thread A: temperature sensor read */
ret = mux_control_select(mux, 0); /* blocking — waits for mutex */
if (ret)
    return ret;
spi_read(spi, &temp_buf, sizeof(temp_buf));
/* BUG: forgot mux_control_deselect — mutex never released */
return 0;
```

الـ `mux_control_select` بتستدعي `mux_control_select_delay(mux, state, 0)` — اللي بتاخد **mutex** داخلي على الـ `mux_control`. لو الـ caller ما استدعاش `mux_control_deselect`، الـ mutex بيفضل محجوز.

الـ `pressure_sensor.c` Thread B بيستدعي:
```c
ret = mux_control_select(mux, 1); /* blocks forever — mutex held by Thread A */
```

الـ Thread B بيتعلق لأن `mux_control_select` blocking بطبيعتها — بتستنى على الـ mutex مش بترجع error.

#### الحل
**الحل الفوري** — إصلاح الـ `temp_sensor.c`:
```c
ret = mux_control_select(mux, 0);
if (ret)
    return ret;

ret = spi_read(spi, &temp_buf, sizeof(temp_buf));

mux_control_deselect(mux); /* MUST always call this */
return ret;
```

**الحل الأفضل** — استخدام `mux_control_try_select` في الـ time-sensitive paths:
```c
ret = mux_control_try_select(mux, 1); /* non-blocking — returns -EBUSY if taken */
if (ret == -EBUSY) {
    dev_dbg(dev, "mux busy, retry later\n");
    return -EAGAIN;
}
```

**الفرق الجوهري من الهيدر:**
- `mux_control_select` ← blocking، بتستنى لحد ما الـ MUX يتحرر
- `mux_control_try_select` ← non-blocking، بترجع `-EBUSY` فوراً لو محجوز

#### الدرس المستفاد
كل `mux_control_select` **لازم** يقابلها `mux_control_deselect` في كل code paths — حتى في الـ error paths. استخدم `goto cleanup` pattern وفكر في `mux_control_try_select` لأي درايفر بيشتغل من interrupt context أو RT thread.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ USB MUX مش بيتبدل

#### العنوان
الـ USB-C port مش بيتحول بين الـ USB3 والـ DisplayPort بسبب غلطة في الـ DT `mux-controls`

#### السياق
منتج **Android TV box** على **Allwinner H616** بيستخدم USB-C port واحد يدعم USB3 SuperSpeed وDisplayPort Alt Mode. في الـ SoC في **USBMUX** hardware بيتحكم في الـ SS lanes. الـ bring-up engineer كتب الـ DT وربط الـ consumer driver، بس الـ DisplayPort مش بيشتغل.

#### المشكلة
الـ `devm_mux_control_get` بترجع `-ENODEV` وبتظهر رسالة:
```
mux-consumer: can't get mux 'usb-dp-mux'
```

#### التحليل
الـ `devm_mux_control_get(dev, "usb-dp-mux")` بتدور على الـ MUX بالاسم `"usb-dp-mux"` في الـ `mux-controls` property في الـ DT node بتاع الـ consumer.

الـ DT الغلط:
```dts
/* WRONG — مش موجود mux-control-names */
&usbdp_phy {
    mux-controls = <&usbmux 0>;
    /* missing: mux-control-names = "usb-dp-mux"; */
};
```

لما الكود بيستدعي `devm_mux_control_get(dev, "usb-dp-mux")`، الـ framework بيدور على اسم `"usb-dp-mux"` في `mux-control-names` property. لو الـ property مش موجودة، بيـfallback لـ index 0 **بس لو الاسم `NULL`**. لما بيبعت اسم explicit، بيرجع `-ENODEV`.

#### الحل
إصلاح الـ DT:
```dts
&usbdp_phy {
    mux-controls = <&usbmux 0>;
    mux-control-names = "usb-dp-mux"; /* matches the string in devm_mux_control_get */
};
```

أو لو الـ driver بيستخدم index مش اسم، غير الـ driver call:
```c
/* استخدام NULL كـ name بياخد أول entry في mux-controls */
mux = devm_mux_control_get(dev, NULL);
```

**للتحقق:**
```bash
cat /sys/kernel/debug/mux/usbmux/state  # تشوف الـ current state
```

#### الدرس المستفاد
الـ `mux-control-names` في الـ DT **لازم تتطابق بالظبط** مع الـ string اللي بتتبعت لـ `devm_mux_control_get`. دايما check الـ DT binding documentation للـ consumer driver قبل الـ bring-up.

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — الـ I2C MUX بياخد delay غلط

#### العنوان
بيانات غلط من الـ I2C sensors بسبب تجاهل الـ settling time في `mux_control_select_delay`

#### السياق
**IoT environmental sensor** بيستخدم **STM32MP1** مع **PCA9548A I2C MUX** بيوزع I2C bus على 8 sensors (temperature, humidity, CO2, إلخ). الـ driver بيقرأ كل الـ sensors بالتسلسل. في بعض القراءات بتطلع قيم عشوائية — خصوصاً أول قراءة بعد تبديل الـ MUX channel.

#### المشكلة
أول قراءة بعد كل `mux_control_select` بترجع جاربيج. الـ oscilloscope بيوري إن الـ I2C transaction بتبدأ قبل ما الـ MUX يخلص switching.

#### التحليل
الكود الحالي:
```c
/* selects channel with zero delay — BUG for slow analog MUX */
ret = mux_control_select(mux, channel);  /* calls select_delay(mux, channel, 0) */
if (ret)
    return ret;
i2c_smbus_read_byte_data(client, REG_DATA);
```

الـ `mux_control_select` هي wrapper بتستدعي `mux_control_select_delay(mux, state, 0)` — بيوضح الهيدر:
```c
static inline int __must_check mux_control_select(struct mux_control *mux,
                                                  unsigned int state)
{
    return mux_control_select_delay(mux, state, 0);
}
```

الـ `delay_us = 0` معناها إن مفيش انتظار بعد تبديل الـ MUX. الـ **PCA9548A** datasheet بيحدد switching time حتى **2.5µs**. الـ I2C transaction بتبدأ فوراً وبيحصل glitch.

#### الحل
استخدام `mux_control_select_delay` مباشرة بدل الـ wrapper:
```c
#define PCA9548A_SWITCH_DELAY_US  5  /* 2.5µs typ + margin */

ret = mux_control_select_delay(mux, channel, PCA9548A_SWITCH_DELAY_US);
if (ret)
    return ret;
i2c_smbus_read_byte_data(client, REG_DATA);
```

لو بتستخدم `mux_state` (اللي بتـencapsulate المعلومات دي):
```c
/* devm_mux_state_get بتجيب mux_state جاهزة بالـ channel والـ delay */
mstate = devm_mux_state_get(dev, "co2-sensor");
ret = mux_state_select_delay(mstate, PCA9548A_SWITCH_DELAY_US);
```

#### الدرس المستفاد
الـ `mux_control_select` (zero delay) مناسبة للـ digital/GPIO MUX السريعة بس. للـ analog MUX أو الـ I2C MUX switches، **دايماً استخدم `mux_control_select_delay`** مع قيمة من الـ datasheet + margin.

---

### السيناريو 4: Automotive ECU على i.MX8 — memory leak من عدم استخدام `devm_`

#### العنوان
Memory leak في الـ UART MUX driver بعد كل module reload بسبب استخدام `mux_control_get` بدل `devm_mux_control_get`

#### السياق
**Automotive ECU** على **NXP i.MX8** بيستخدم hardware MUX لتوزيع 3 UART interfaces على 6 external connectors. الـ driver بيتعمله reload كل مرة الـ ECU بيتبدل mode (diagnostic vs operational). الـ system بيبدأ يبطأ تدريجياً بعد كذا cycle.

#### المشكلة
الـ `/proc/meminfo` بيوري إن `Slab` memory بتكبر مع كل module reload. الـ `kmemleak` بيظهر:
```
unreferenced object 0xffff888... (size 256):
  comm "modprobe", pid 1234
  [<...>] mux_control_get+0x...
```

#### التحليل
الـ driver بيستخدم:
```c
static int uart_mux_probe(struct platform_device *pdev)
{
    struct uart_mux_priv *priv;
    priv->mux = mux_control_get(&pdev->dev, "uart-mux"); /* manual get */
    /* ... */
    return 0;
}

static int uart_mux_remove(struct platform_device *pdev)
{
    /* BUG: mux_control_put never called */
    return 0;
}
```

الفرق من الهيدر واضح:
- `mux_control_get` + `mux_control_put` ← **يدوي**، لازم تتعامل معاهم في كل paths
- `devm_mux_control_get` ← **automatic**، بيتـrelease لما الـ device بيتـdetach

لما الـ `remove` callback ما بيستدعيش `mux_control_put`، الـ `mux_control` object بيفضل allocated في الـ memory.

#### الحل
**الإصلاح السريع** — إضافة `mux_control_put` في الـ remove:
```c
static int uart_mux_remove(struct platform_device *pdev)
{
    struct uart_mux_priv *priv = platform_get_drvdata(pdev);
    mux_control_put(priv->mux);
    return 0;
}
```

**الإصلاح الصح** — استخدام `devm_` من الأساس:
```c
static int uart_mux_probe(struct platform_device *pdev)
{
    struct uart_mux_priv *priv;
    /* auto-released when device is removed */
    priv->mux = devm_mux_control_get(&pdev->dev, "uart-mux");
    if (IS_ERR(priv->mux))
        return PTR_ERR(priv->mux);
    return 0;
}
/* remove callback doesn't need to touch the mux at all */
```

**للتشخيص:**
```bash
# تشغيل kmemleak scan
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

#### الدرس المستفاد
في kernel drivers، **دايماً فضل `devm_mux_control_get`** على `mux_control_get` إلا لو عندك سبب وجيه للـ manual lifetime management. الـ `devm_` APIs موجودة عشان تمنع exactly هذا النوع من الـ bugs.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ `__must_check` بيكشف bug خفي في الـ HDMI MUX

#### العنوان
الـ HDMI display مش بتظهر حاجة بسبب تجاهل return value من `mux_control_select`

#### السياق
فريق bring-up بيشتغل على **custom display board** مبنية على **TI AM62x**. الـ board عندها **video MUX** بيختار بين HDMI وeDP outputs. الـ driver اتكتب بسرعة وبعد compile بدون warnings، بس الـ HDMI شاشة فاضية.

#### المشكلة
الـ HDMI output مش بيشتغل. الـ `dmesg` مفيش فيه errors. الـ developer واثق إن الـ driver صح لأن الـ build نظيف.

#### التحليل
الكود الأصلي:
```c
static int hdmi_mux_enable(struct hdmi_priv *priv)
{
    /* BUG: return value ignored — compiler SHOULD warn but developer missed it */
    mux_control_select(priv->mux, HDMI_OUTPUT);

    hdmi_phy_init(priv);
    return 0;
}
```

الـ `mux_control_select` معرفة في الهيدر كـ:
```c
static inline int __must_check mux_control_select(struct mux_control *mux,
                                                  unsigned int state)
```

الـ `__must_check` attribute بيخلي الـ compiler يطلع warning لو الـ return value اتتجاهل. بس الـ developer compile بـ `-W` مش `-Wall`، فالـ warning اتضاع.

لو `mux_control_select` رجعت error (مثلاً الـ MUX controller مش probe اتمه بعد — deferred probe scenario)، الـ `hdmi_phy_init` بتـrun على MUX في state غلط.

#### كيف اكتشفوا الـ bug
لما عملوا build بـ `-Wall`:
```
warning: ignoring return value of 'mux_control_select',
         declared with attribute 'warn_unused_result'
```

بعدين اكتشفوا إن الـ MUX controller بياخد وقت في الـ probe (I2C MUX على slow bus)، فالـ `mux_control_select` كانت بترجع `-EPROBE_DEFER` وهو متجاهل.

#### الحل
```c
static int hdmi_mux_enable(struct hdmi_priv *priv)
{
    int ret;

    ret = mux_control_select(priv->mux, HDMI_OUTPUT);
    if (ret) {
        dev_err(priv->dev, "failed to select HDMI mux: %d\n", ret);
        return ret; /* propagates -EPROBE_DEFER correctly */
    }

    hdmi_mux_enable(priv);
    return 0;
}
```

**علشان الـ `-EPROBE_DEFER` يشتغل صح**, الـ probe function لازم ترجع الـ error للـ framework:
```c
static int hdmi_probe(struct platform_device *pdev)
{
    /* ... */
    ret = hdmi_mux_enable(priv);
    if (ret)
        return ret; /* kernel will retry probe later */
}
```

**إجبار الـ warnings من Makefile:**
```makefile
ccflags-y += -Wall -Werror
```

#### الدرس المستفاد
الـ `__must_check` في الهيدر مش decoration — دي **contract**. كل function في `mux/consumer.h` تقريباً معلمة `__must_check` لأن فشلها معناه إن الـ MUX في state مجهول. **دايماً build بـ `-Wall`** في الـ development، وعامل الـ `-EPROBE_DEFER` كـ first-class case في أي driver بيستخدم `devm_mux_control_get`.
## Phase 7: مصادر ومراجع

### مصادر رسمية — Kernel Documentation

| المصدر | الرابط |
|--------|--------|
| **Device Tree Binding** لـ mux-controller | [Documentation/devicetree/bindings/mux/mux-controller.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/mux/mux-controller.txt) |
| **SPI mux binding** (YAML) | [Documentation/devicetree/bindings/spi/spi-mux.yaml](https://www.kernel.org/doc/Documentation/devicetree/bindings/spi/spi-mux.yaml) |
| **Intel PMC North Mux-Agent** | [docs.kernel.org — intel-pmc-mux](https://docs.kernel.org/firmware-guide/acpi/intel-pmc-mux.html) |
| **I2C muxes and complex topologies** | [docs.kernel.org — i2c-topology](https://docs.kernel.org/i2c/i2c-topology.html) |
| **I2C mux GPIO driver** | [docs.kernel.org — i2c-mux-gpio](https://www.kernel.org/doc/html/latest/i2c/muxes/i2c-mux-gpio.html) |
| **PINCTRL subsystem** (مرتبط بـ pin muxing) | [kernel.org — pinctl](https://www.kernel.org/doc/html/v4.13/driver-api/pinctl.html) |
| **drivers/mux** على GitHub | [torvalds/linux — drivers/mux](https://github.com/torvalds/linux/tree/master/drivers/mux) |

---

### مقالات LWN.net

الـ **LWN.net** هو أهم مرجع لمتابعة تطور الـ kernel — الروابط دي جات من نتائج البحث الفعلية:

#### 1. mux controller abstraction and iio/i2c muxes
- **الرابط:** [lwn.net/Articles/716704](https://lwn.net/Articles/716704/)
- **الأهمية:** المقال الرئيسي اللي غطى اقتراح Peter Rosin لإضافة subsystem موحد لـ mux controllers، وشرح كيف يُحل مشكلة تعقيد الـ device tree عند استخدام I2C muxes وغيرها.

#### 2. usb: add support for Intel dual role port mux
- **الرابط:** [lwn.net/Articles/689134](https://lwn.net/Articles/689134/)
- **الأهمية:** مثال عملي لاستخدام mux في سياق USB dual-role.

#### 3. USB Type-C device-connection, mux and switch support
- **الرابط:** [lwn.net/Articles/749740](https://lwn.net/Articles/749740/)
- **الأهمية:** بيوضح كيف اتربط الـ `drivers/mux` framework مع USB Type-C Port Manager.

#### 4. The kernel connection multiplexer
- **الرابط:** [lwn.net/Articles/657999](https://lwn.net/Articles/657999/)
- **الأهمية:** سياق أوسع للـ multiplexing في الـ kernel بشكل عام.

---

### نقاشات Mailing List

الـ patches الأصلية بتاعة الـ subsystem اتناقشت على LKML:

| الموضوع | الرابط |
|---------|--------|
| **[PATCH v9 03/10] mux: minimal mux subsystem and gpio-based mux controller** — Peter Rosin | [lore.kernel.org](https://lore.kernel.org/lkml/1486568517-6997-4-git-send-email-peda@axentia.se/) |
| **[PATCH v15 13/13] mux: mmio-based syscon mux controller** | [mail-archive.com](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1395896.html) |
| **RFC: mux enable state from DT** — Peter Rosin | [lore.kernel.org](https://lore.kernel.org/linux-devicetree/69f73f64-6424-4e3f-9068-195e959b9762@axentia.se/) |

> الـ subsystem اتقدم في **v9** من الـ patch series في فبراير 2017، وتم merge-ه في **Linux 4.12**. الـ author هو **Peter Rosin** من Axentia Technologies.

---

### Phoronix — أخبار الإضافة للـ Kernel

| المصدر | الرابط |
|--------|--------|
| Mux Controller Subsystem Proposed For Linux 4.12 | [phoronix.com](https://www.phoronix.com/news/Mux-Proposed-Linux-4.12) |

---

### KernelNewbies.org — تتبع الـ Kernel Versions

الـ mux-related changes موثقة في صفحات الـ kernel versions دي:

| الـ Kernel | ما يخص الـ mux | الرابط |
|------------|----------------|--------|
| **Linux 4.10** | إضافة Mellanox CPLD mux driver | [kernelnewbies.org/Linux_4.10](https://kernelnewbies.org/Linux_4.10) |
| **Linux 4.17** | USB Type-C device-connection, mux and switch support | [kernelnewbies.org/Linux_4.17](https://kernelnewbies.org/Linux_4.17) |
| **Linux 4.19** | Clock mux drivers (at91 I2S, tegra sdmmc…) | [kernelnewbies.org/Linux_4.19](https://kernelnewbies.org/Linux_4.19) |
| **Linux 5.12** | MLX CPLD mux driver extensions | [kernelnewbies.org/Linux_5.12](https://kernelnewbies.org/Linux_5.12) |
| **Linux 6.1** | Hardware GPU MUX support | [kernelnewbies.org/Linux_6.1](https://kernelnewbies.org/Linux_6.1) |
| **Linux 6.13** | `simple-mux` idle-state configuration | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) |

---

### eLinux.org — Pin Muxing في Embedded Linux

| المصدر | الرابط |
|--------|--------|
| Introduction to pin muxing and GPIO control under Linux (ELC-2021 slides) | [elinux.org — PDF](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) |
| BeagleBoard — Cape Compatibility / MUX_MODE | [elinux.org](https://elinux.org/BeagleBoard/GSoC/2020_Projects/Cape_Compatibility) |

---

### Analog Devices Wiki — مثال Driver عملي

| المصدر | الرابط |
|--------|--------|
| ADGS1408/ADGS1409 8:1/Dual 4:1 Muxes Linux Driver | [wiki.analog.com](https://wiki.analog.com/resources/tools-software/linux-drivers/mux/adgs1408) |

> مثال ممتاز على driver حقيقي بيستخدم الـ `mux_control` consumer API.

---

### كتب موصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول ذات الصلة:**
  - Chapter 14: The Linux Device Model — فاهم resource management وـ `devm_*` pattern
  - Chapter 3: Char Drivers — فهم أساسيات الـ driver API
- **رابط مجاني:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصول ذات الصلة:**
  - Chapter 17: Devices and Modules — lifecycle الـ driver وـ bus/device model
  - Chapter 13: The Virtual Filesystem — فهم abstraction layers في الـ kernel
- **ISBN:** 978-0672329463

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصول ذات الصلة:**
  - Chapter 11: BusyBox و Kernel Drivers — تشغيل drivers على hardware حقيقي
  - Chapter 15: Debugging Embedded Linux — trace وـ debug لـ mux issues
- **ISBN:** 978-0137017836

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الأهمية:** تغطية عميقة لـ device model وـ sysfs وـ reference counting — كلها أساس لفهم `mux_control_get/put`.

---

### مصادر الكود المباشرة

```
include/linux/mux/consumer.h   ← الـ consumer API (الملف الحالي)
include/linux/mux/driver.h     ← الـ provider/driver API
drivers/mux/core.c             ← الـ core implementation
drivers/mux/gpio.c             ← GPIO-based mux driver
drivers/mux/mmio.c             ← MMIO/syscon-based mux driver
```

---

### مصطلحات البحث — Search Terms

للبحث عن معلومات إضافية استخدم الـ terms دي:

```
"mux_control_select" linux kernel
"struct mux_control" driver implementation
"devm_mux_control_get" device tree example
linux kernel multiplexer subsystem consumer provider
"drivers/mux" linux kernel 4.12
Peter Rosin mux controller axentia
linux iio mux channel
i2c mux topology linux kernel
```
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على `mux_control_select_delay` — أهم function في الـ consumer API، بتتحكم في تحويل الـ mux لـ state معين. كل مرة أي driver يحاول يغير الـ state، الـ kprobe بتعترض الاستدعاء وتطبع الـ mux pointer والـ state المطلوب والـ delay.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_mux_select.c
 *
 * Intercepts every call to mux_control_select_delay() and logs
 * the mux pointer, requested state, and delay value.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/mux/consumer.h>   /* struct mux_control definition        */

/* ------------------------------------------------------------------ */
/* pre-handler: runs just BEFORE mux_control_select_delay executes      */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Calling convention (x86-64 System V ABI):
     *   rdi = arg0 -> struct mux_control *mux
     *   rsi = arg1 -> unsigned int state
     *   rdx = arg2 -> unsigned int delay_us
     */
    struct mux_control *mux  = (struct mux_control *)regs->di;
    unsigned int        state = (unsigned int)regs->si;
    unsigned int        delay = (unsigned int)regs->dx;

    pr_info("mux_select_probe: mux=%p  state=%u  delay_us=%u\n",
            mux, state, delay);

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/* post-handler: runs just AFTER mux_control_select_delay returns       */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* ax holds the return value on x86-64 */
    long retval = (long)regs->ax;

    pr_info("mux_select_probe: returned %ld (%s)\n",
            retval, retval == 0 ? "ok" : "error");
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                    */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "mux_control_select_delay",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* module init / exit                                                   */
/* ------------------------------------------------------------------ */
static int __init mux_probe_init(void)
{
    int ret = register_kprobe(&kp);

    if (ret < 0) {
        pr_err("mux_select_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("mux_select_probe: planted at %p\n", kp.addr);
    return 0;
}

static void __exit mux_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("mux_select_probe: removed\n");
}

module_init(mux_probe_init);
module_exit(mux_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on mux_control_select_delay — logs state switches");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل الـ API بتاعته |
| `linux/mux/consumer.h` | عشان نقدر نستخدم `struct mux_control` بالاسم الصح في الـ cast |

---

#### الـ pre-handler

الـ `pt_regs` بيحتوي على قيم الـ registers اللي كانت موجودة لما الـ function اتنادت. على x86-64، الـ arguments بتتمرر في `rdi`, `rsi`, `rdx` بالترتيب — فبنـcast كل واحد للنوع المناسب ونطبعه في `pr_info`. الـ return value صفر يقول للـ kernel "أكمل التنفيذ الطبيعي".

---

#### الـ post-handler

بيتشغل بعد ما `mux_control_select_delay` ترجع. بنقرأ `regs->ax` اللي فيه الـ return value، وبنطبعه عشان نعرف إذا الـ state switch نجح أو فيه error. ده مفيد جداً في debugging حالات الـ contention على الـ mux lock.

---

#### `struct kprobe kp`

بنحدد اسم الـ symbol بالـ string، والـ kernel بيحل العنوان وقت الـ `register_kprobe`. كتابة الاسم كـ string بدل pointer بتخلي الكود portable بدون الحاجة لـ `kallsyms` يدوياً.

---

#### `module_init` — `mux_probe_init`

`register_kprobe` بتزرع الـ breakpoint وسط الـ kernel code بشكل آمن. لو فشلت — مثلاً لو الـ function مش موجودة أو الـ kprobes مش مفعّلة في الـ kernel config — بنرجع الـ error code فوراً بدل ما نسيب الـ module في حالة نص تسجيل.

---

#### `module_exit` — `mux_probe_exit`

`unregister_kprobe` ضرورية لأن الـ kprobe بتشاور على callback function جوه الـ module. لو فضل الـ kprobe مسجّل بعد ما الـ module اتـunload، أي استدعاء لـ `mux_control_select_delay` هيـjump لكود اتشال من الذاكرة وده kernel panic مضمون.

---

### Makefile

```makefile
obj-m += kprobe_mux_select.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ module

```bash
# بناء وتحميل
make
sudo insmod kprobe_mux_select.ko

# شوف الـ output في الـ kernel log
sudo dmesg -w | grep mux_select_probe

# إزالة الـ module
sudo rmmod kprobe_mux_select
```

**مثال على الـ output المتوقع:**

```
[  142.331200] mux_select_probe: planted at ffffffffc08a1340
[  145.002100] mux_select_probe: mux=ffff888003b2c000  state=2  delay_us=0
[  145.002115] mux_select_probe: returned 0 (ok)
[  145.880300] mux_select_probe: mux=ffff888003b2c000  state=0  delay_us=500
[  145.880340] mux_select_probe: returned 0 (ok)
```

كل سطر بيكشف مين طلب تحويل الـ mux، لأي state، وبكام microsecond delay — معلومة كافية لـ debug أي مشكلة في الـ mux subsystem بدون أي تعديل في الـ source code الأصلي.
