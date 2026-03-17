## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبعه الملف

الملف `include/linux/mux/driver.h` جزء من **MULTIPLEXER SUBSYSTEM** في الـ Linux kernel، المسؤول عنه Peter Rosin من Axentia Technologies. الـ subsystem ده موجود في `drivers/mux/` و `include/linux/mux/`.

---

### الفكرة الكبيرة — تخيل كده

تخيل عندك تلاجة فيها باب واحد بس، وعندك 4 أوض محتاجين يفتحوا نفس الباب ده في أوقات مختلفة. محدش يقدر يفتح الباب غير واحد في نفس الوقت، ولازم أي حد يستنى التاني يخلص.

ده بالظبط اللي بيعمله الـ **Multiplexer (MUX)** في الهاردوير. الـ MUX chip هو دايرة إلكترونية بتوصّل **مدخل واحد** بـ **مخرج واحد من عدة مخارج** في نفس الوقت. زي كشري: واجهة واحدة بتوصّل لأكتر من جهاز.

---

### المشكلة اللي بيحلها الـ subsystem

في الأنظمة المدمجة (embedded systems)، الـ CPU غالباً بيكون عنده عدد محدود من الـ pins أو الـ buses. مثلاً:
- عندك bus واحد للـ I2C، بس عندك 8 sensors.
- عندك قناة ADC واحدة بس عندك 4 مصادر إشارة.
- عندك SPI واحدة بس عندك 6 أجهزة.

الحل؟ تحط **MUX chip** في النص يختار كل مرة "مين اللي متوصل دلوقتي".

الـ kernel بيحتاج framework موحّد عشان:
1. أي driver يقدر يطلب "عايز أكلم الجهاز رقم 3".
2. الـ framework يضمن إن محدش تاني بيستخدم نفس الـ MUX في نفس الوقت.
3. لما الـ driver يخلص، الـ MUX يرجع لحالة الـ idle.

---

### دور الملف `mux/driver.h` تحديداً

الملف ده هو **واجهة كاتب الـ driver** — يعني اللي بيكتب driver لشريحة MUX محددة (زي `adgs1408.c` أو `gpio.c`) بيستخدم التعريفات دي.

فيه ثلاث قطع أساسية:

#### 1. `struct mux_control_ops`
الـ **operations table** — بتقول للـ framework "إزاي تحرك الـ MUX". فيها function pointer واحدة بس:
- **`set`**: غيّر حالة الـ MUX لرقم معين.

#### 2. `struct mux_control`
بيمثّل **controller واحد داخل الـ chip**. الـ chip الواحدة ممكن يكون فيها أكتر من controller (أكتر من MUX مستقل). فيه:
- **`lock`**: semaphore بيمنع أكتر من driver يستخدم نفس الـ controller في نفس الوقت.
- **`cached_state`**: آخر state اتضبط، عشان متعملش write للهاردوير لو مش محتاج.
- **`states`**: عدد الحالات المتاحة (كام wire ممكن يتوصل).
- **`idle_state`**: لما محدش شاغل الـ MUX، يروح لأنهي حالة؟
- **`last_change`**: timestamp آخر تغيير، مفيد للـ timing requirements.

#### 3. `struct mux_chip`
بيمثّل الـ **chip الفيزيائية كلها** — اللي ممكن تحتوي على عدة controllers. فيه:
- **`dev`**: ربطها بـ device model عادي.
- **`ops`**: الـ operations اللي كاتب الـ driver حددها.
- **`mux[]`**: array من الـ controllers — آخر تريكة في C: flexible array member.

#### الـ API للـ driver writers
```c
// إنشاء chip جديدة وتسجيلها
struct mux_chip *mux_chip_alloc(struct device *dev, unsigned int controllers, size_t sizeof_priv);
int mux_chip_register(struct mux_chip *mux_chip);

// نسخة devm — kernel بيتكفل بالـ cleanup
struct mux_chip *devm_mux_chip_alloc(...);
int devm_mux_chip_register(...);

// تحرير يدوي
void mux_chip_unregister(struct mux_chip *mux_chip);
void mux_chip_free(struct mux_chip *mux_chip);
```

---

### القصة الكاملة — إزاي الأجزاء بتتكلم مع بعض

```
┌─────────────────────────────────────────────────────┐
│                  Consumer Driver                    │
│   (e.g., IIO driver محتاج يقرأ من sensor معين)     │
│         mux_control_select(mux, state)              │
└────────────────────┬────────────────────────────────┘
                     │  consumer.h API
                     ▼
┌─────────────────────────────────────────────────────┐
│               MUX Core (core.c)                     │
│  - يأخذ الـ lock                                    │
│  - يشوف لو الـ state اتغيرت فعلاً                  │
│  - يستنى الـ settling time                          │
└────────────────────┬────────────────────────────────┘
                     │  driver.h API → ops->set()
                     ▼
┌─────────────────────────────────────────────────────┐
│            MUX Chip Driver (e.g., gpio.c)           │
│  - يكتب على الـ GPIO pins أو I2C registers         │
│  - يغير الـ hardware state فعلاً                   │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
              ┌──────────────┐
              │  MUX Hardware│
              │  (الشريحة)  │
              └──────┬───────┘
          ┌──────────┼──────────┐
        CH0        CH1        CH2  ...
```

**الـ `driver.h`** هو الحد الفاصل بين الـ core وبين الـ chip drivers.

---

### مثال حقيقي — IIO ADC مع MUX

تخيل عندك `STM32` فيه ADC channel واحدة بس، وعندك 8 temperature sensors. بتحط `ADGS1408` MUX chip (8-to-1 analog switch):

1. الـ IIO driver يطلب `mux_control_select(mux, 3)` عشان يقرأ sensor رقم 3.
2. الـ core يأخذ الـ lock — لو حد تاني شاغل بينتظر.
3. الـ core يشوف لو الـ cached_state مش 3، يكلم `ops->set(mux, 3)`.
4. الـ `adgs1408.c` driver يكتب الـ value على الـ SPI.
5. الـ ADC يقرأ.
6. الـ driver يكلم `mux_control_deselect(mux)` — الـ MUX يرجع لـ idle state.

---

### الفرق بين `driver.h` و `consumer.h`

| الملف | لمين؟ | بيعمل إيه؟ |
|---|---|---|
| `mux/driver.h` | كاتب الـ MUX chip driver | يعرّف الـ chip ويسجلها في الـ kernel |
| `mux/consumer.h` | أي driver محتاج يستخدم MUX | يطلب state ويحرر الـ MUX |

---

### الملفات المهمة في الـ Subsystem

| الملف | الدور |
|---|---|
| `include/linux/mux/driver.h` | **الملف ده** — واجهة كاتب الـ driver |
| `include/linux/mux/consumer.h` | واجهة مستخدم الـ MUX |
| `drivers/mux/core.c` | قلب الـ subsystem — الـ locking والـ caching والـ sysfs |
| `drivers/mux/gpio.c` | driver للـ MUX اللي بيتحكم فيه GPIO pins |
| `drivers/mux/mmio.c` | driver للـ MUX في الـ memory-mapped registers |
| `drivers/mux/adgs1408.c` | driver لشريحة ADGS1408 من Analog Devices |
| `drivers/mux/adg792a.c` | driver لشريحة ADG792A من Analog Devices |
| `include/dt-bindings/mux/mux.h` | constants زي `MUX_IDLE_AS_IS` و `MUX_IDLE_DISCONNECT` |
| `Documentation/devicetree/bindings/mux/` | كيفية وصف الـ MUX في الـ device tree |
| `Documentation/ABI/testing/sysfs-class-mux*` | الـ sysfs interface للـ MUX |

---

### نقاط دقيقة تستاهل انتباه

- **`__counted_by(controllers)`** في `mux[]`: annotation حديثة للـ compiler عشان يتحقق من حدود الـ array — حماية من الـ buffer overflow.
- **`mux_chip_priv()`**: بعد آخر عنصر في الـ array مباشرة في الـ memory في extra space للـ driver عشان يخزن بياناته الخاصة — تريكة شائعة في الـ kernel بدل الـ malloc منفصل.
- **`cached_state = -1`** (= `MUX_CACHE_UNKNOWN`): عند البداية الـ kernel مش عارف الـ hardware state، فبيجبر أول `set()` تحصل حتى لو الـ state نفسها.
## Phase 2: شرح الـ MUX Framework

### المشكلة — ليه الـ subsystem ده موجود؟

في الأنظمة المدمجة، الـ hardware بيحتوي دايمًا على **multiplexer** — سواء كان:
- **analog mux** زي `ADG704` بيوصّل ناتج ADC واحد لـ 4 sensors مختلفة
- **I2C mux** زي `PCA9548` بيوصّل master واحد على 8 buses
- **GPIO-controlled mux** بيختار بين SPI flash و SD card على نفس الـ bus

المشكلة: كل driver كان بيتعامل مع الـ mux بشكل خاص بيه — يكتب على GPIO مباشرةً أو يكلم الـ mux chip بدون أي تنسيق مع باقي الـ drivers. النتيجة؟

- **Race condition**: درايفرين يحاولوا يغيروا الـ mux في نفس الوقت
- **Code duplication**: كل driver بيعيد اختراع عجلة التحكم في الـ mux
- **No idle management**: الـ mux بيفضل على channel غلط بعد ما الـ driver يخلص

---

### الحل — مدخل الـ Kernel

الـ kernel حل المشكلة دي بـ **MUX subsystem** — framework مركزي بيعمل:

1. **Abstraction layer** موحّد بين consumers (اللي محتاجين الـ mux) و providers (الـ mux chip drivers)
2. **Mutual exclusion** تلقائي عبر semaphore لكل controller
3. **Idle state management** — لما consumer يخلص، الـ mux يرجع لحالة محددة
4. **Device Tree integration** — ربط الـ consumer بالـ mux عبر DT properties

---

### التشبيه الواقعي — كابينة الكهرباء

تخيل كابينة كهرباء في عمارة:

| مفهوم الكابينة | مقابله في الـ MUX Framework |
|---|---|
| الكابينة نفسها | `struct mux_chip` |
| كل قاطع (circuit breaker) | `struct mux_control` |
| حالة القاطع (on/off/pos1/pos2) | `state` (رقم من 0 → states-1) |
| الكهربائي اللي بيشغّل | consumer driver |
| شركة الكهرباء اللي عملت الكابينة | mux chip driver |
| قفل الكابينة (منع دخول اتنين) | `struct semaphore lock` |
| وضع الطوارئ (قطع كامل) | `MUX_IDLE_DISCONNECT` |
| "سيبها زي ما هي" | `MUX_IDLE_AS_IS` |
| رقم كل قاطع في الكابينة | `mux_control_get_index()` |
| شركة تانية تعمل كابينة بنفس المواصفات | driver تاني يعمل `mux_chip` تاني |

الكابينة (mux_chip) فيها N قاطع (mux_control)، كل قاطع عنده عدد حالات محدد (states). الكهربائي (consumer) لازم يطلب إذن (semaphore) قبل ما يلمس أي قاطع — ومش ممكن اتنين يعملوا ده في نفس الوقت.

---

### البنية الكبيرة — أين يقع الـ MUX في الـ Kernel؟

```
┌─────────────────────────────────────────────────────────────┐
│                     Consumer Drivers                        │
│   (iio/adc driver, i2c-mux driver, phy driver, ...)        │
│                                                             │
│   mux_control_get()  →  mux_control_select()               │
│                          mux_control_deselect()             │
└────────────────────────┬────────────────────────────────────┘
                         │  consumer API  (consumer.h)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    MUX Core (mux/core.c)                    │
│                                                             │
│  ┌─────────────────┐    ┌──────────────────────────────┐   │
│  │  mux_chip_alloc │    │  mux_control_select_delay()  │   │
│  │  mux_chip_reg   │    │  → semaphore acquire         │   │
│  │  mux_chip_free  │    │  → compare cached_state      │   │
│  └─────────────────┘    │  → call ops->set()           │   │
│                         │  → update cached_state       │   │
│                         │  → record last_change        │   │
│                         └──────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────┘
                         │  driver API  (driver.h)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   MUX Chip Drivers                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ gpio-mux.c   │  │ mmio-mux.c   │  │ adg792a.c (spi)  │  │
│  │ (GPIO lines) │  │ (MMIO regs)  │  │ (analog switch)  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                             │
│  كل واحد بيعمل:  mux_chip_alloc() + set ops->set          │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                     Hardware                                │
│   GPIO pins / I2C registers / SPI commands / MMIO regs     │
└─────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ MUX framework بيقدم abstraction واحدة بسيطة جداً:

> **الـ mux هو جهاز عنده N حالة رقمية (0 → N-1)، ومحتاج exclusive access.**

كل التعقيد الـ hardware-specific اتخبى وراء callback واحد بس:

```c
struct mux_control_ops {
    int (*set)(struct mux_control *mux, int state);
};
```

الـ core بيأخد الـ state (رقم)، يحوله لـ hardware action عبر `ops->set`. الـ driver مش محتاج يفكر في locking أو caching أو idle management — كل ده الـ core بيعمله.

---

### تفاصيل الـ Structs وعلاقتها ببعض

```
struct mux_chip
├── controllers: 3            ← عدد الـ mux controllers في الـ chip
├── dev: struct device        ← الـ chip نفسه كـ kernel device
├── id: int                   ← internal ID للتعرف عليه
├── ops: *mux_control_ops     ← pointer للـ ops table (shared بين كل controllers)
│       └── set(mux, state)   ← الـ callback الوحيد المطلوب من الـ driver
│
└── mux[]: struct mux_control[]   ← flexible array، حجمه = controllers
    │
    ├── mux[0]
    │   ├── lock: semaphore    ← mutex لهذا الـ controller بالتحديد
    │   ├── chip: *mux_chip    ← back-pointer للـ chip الأب
    │   ├── cached_state: int  ← آخر state تم تطبيقه (-1 = unknown)
    │   ├── states: uint       ← عدد الحالات الممكنة (driver يحدده وقت alloc)
    │   ├── idle_state: int    ← MUX_IDLE_AS_IS(-1) أو MUX_IDLE_DISCONNECT(-2) أو رقم
    │   └── last_change: ktime_t ← timestamp آخر تغيير
    │
    ├── mux[1]  ...
    └── mux[2]  ...
```

#### الـ flexible array trick

```c
struct mux_chip {
    ...
    struct mux_control mux[] __counted_by(controllers);
};
```

الـ `mux[]` هو **flexible array member** — الـ array مش بيتخزن منفصل في الـ heap، هو جزء من نفس الـ allocation block مباشرةً بعد الـ struct. ده بيوفر:
- allocation واحدة بدل اتنين
- cache locality أفضل
- الـ `mux_chip_priv()` بيستغل ده بذكاء: الـ private data بتاع الـ driver بتتحط مباشرةً بعد آخر `mux_control` في نفس الـ block

```
Memory Layout بعد mux_chip_alloc():
┌──────────────────┬────────────────┬────────────────┬────────────────┬──────────────┐
│   struct         │  mux_control   │  mux_control   │  mux_control   │ driver priv  │
│   mux_chip       │    [0]         │    [1]         │    [2]         │    data      │
│   (header)       │                │                │                │              │
└──────────────────┴────────────────┴────────────────┴────────────────┴──────────────┘
▲                                                                      ▲
mux_chip                                                         mux_chip_priv()
```

---

### الـ cached_state — تحسين الأداء

الـ `cached_state` بيمنع الكتابة غير الضرورية على الـ hardware. لو الـ consumer طلب state 2 والـ mux أصلاً على state 2، الـ core مش بيكلم الـ driver خالص — بيرجع 0 مباشرةً.

- القيمة `-1` معناها "مش عارف" — اتعمل reset أو أول مرة، يبقى لازم نكتب حتى لو الـ state هو 0.
- **الـ driver ممنوع منعاً باتاً يكتب في `cached_state`** — ده internal للـ core بس.

---

### الـ idle_state — إدارة حالة الخمول

لما consumer ينهي شغله ويعمل `mux_control_deselect()`، الـ core بيشوف `idle_state`:

| قيمة `idle_state` | السلوك |
|---|---|
| `MUX_IDLE_AS_IS` (-1) | ابقى على نفس الـ state — لا تغيير |
| `MUX_IDLE_DISCONNECT` (-2) | افصل الـ mux تماماً (high-Z) لو الـ hardware يدعم |
| `0` أو أي رقم موجب | ارجع لهذا الـ state تحديداً |

مثال عملي: ADC مشترك بين 4 sensors. لما sensor A خلص قراءته، عايز الـ mux يرجع لـ channel 0 (الـ safe default) — يبقى `idle_state = 0`.

---

### الـ Semaphore — ليس Mutex وده مقصود

الـ lock هو `struct semaphore` مش `struct mutex`. الفرق المهم:

- الـ **mutex** في الـ Linux kernel مرتبط بالـ task اللي عمل lock عليه — نفس الـ task هو بس اللي يعمل unlock
- الـ **semaphore** مش بيه ownership — أي task تاني ممكن يعمل up()

ده مهم هنا لأن `mux_control_select` و `mux_control_deselect` ممكن يتنفذوا من **contexts مختلفة** في بعض الـ use-cases (مثلاً interrupt handler يعمل deselect بعد DMA transfer خلص).

> **ملاحظة**: ده يحتاج فهم مبدأي لـ **Linux locking primitives** — الـ mutex, semaphore, spinlock وأمتى تستخدم كل واحد.

---

### الـ last_change — Settling Time Awareness

```c
ktime_t last_change;
```

بعض الـ analog multiplexers محتاجين وقت بعد التغيير عشان الـ signal يستقر (settling time). الـ `mux_control_select_delay()` بيستخدم `last_change` لحساب لو في delay ناقص لسه من آخر تغيير — لو الوقت الكافي عدى مش بيضيف delay إضافي.

---

### الـ Framework يملك مقابل ما يفوضه للـ Driver

| المسؤولية | المالك |
|---|---|
| Mutual exclusion (locking/unlocking) | MUX Core |
| Caching الـ current state | MUX Core |
| تطبيق الـ idle_state عند deselect | MUX Core |
| إدارة الـ settling delay | MUX Core |
| تسجيل الـ device في kernel device model | MUX Core |
| Device Tree lookups (consumer → mux) | MUX Core |
| الكتابة الفعلية على الـ hardware registers | Chip Driver (ops->set) |
| تحديد عدد الـ states لكل controller | Chip Driver (وقت alloc/register) |
| تحديد الـ idle_state لكل controller | Chip Driver (وقت alloc/register) |
| الـ hardware initialization | Chip Driver |

---

### دورة حياة الـ Mux Chip Driver

```
probe()
  │
  ├── devm_mux_chip_alloc(dev, N, sizeof_priv)
  │       └── allocates: mux_chip + N×mux_control + priv_data (single alloc)
  │
  ├── chip->ops = &my_mux_ops         ← ربط الـ ops
  │
  ├── for each controller i:
  │       chip->mux[i].states = M     ← عدد الحالات
  │       chip->mux[i].idle_state = X ← سلوك الخمول
  │
  └── devm_mux_chip_register(dev, chip)
          └── يسجل الـ chip كـ device ويعلن عن توفر الـ controllers

remove() / driver unbind:
  └── devm cleanup → mux_chip_unregister() + mux_chip_free() تلقائياً
```

---

### مثال عملي — ADC مع 4 Channels

```c
/* chip driver: gpio-controlled 4:1 analog mux */
static int my_mux_set(struct mux_control *mux, int state)
{
    struct my_priv *priv = mux_chip_priv(mux->chip);
    /* write 2-bit state to GPIO lines A0, A1 */
    gpiod_set_value(priv->gpio_a0, state & 1);
    gpiod_set_value(priv->gpio_a1, (state >> 1) & 1);
    return 0;
}

static const struct mux_control_ops my_ops = {
    .set = my_mux_set,
};

static int my_mux_probe(struct platform_device *pdev)
{
    struct mux_chip *chip;
    struct my_priv *priv;

    /* 1 controller, 4 states, priv data = my_priv */
    chip = devm_mux_chip_alloc(&pdev->dev, 1, sizeof(*priv));
    priv = mux_chip_priv(chip);           /* priv sits right after mux[] */

    priv->gpio_a0 = devm_gpiod_get(&pdev->dev, "a0", GPIOD_OUT_LOW);
    priv->gpio_a1 = devm_gpiod_get(&pdev->dev, "a1", GPIOD_OUT_LOW);

    chip->ops = &my_ops;
    chip->mux[0].states = 4;              /* 4 possible states: 0,1,2,3 */
    chip->mux[0].idle_state = 0;          /* default to channel 0 when idle */

    return devm_mux_chip_register(&pdev->dev, chip);
}
```

```c
/* consumer driver: ADC driver using the mux */
static int read_sensor(struct iio_dev *indio_dev, int channel)
{
    struct my_adc *adc = iio_priv(indio_dev);
    int ret;

    /* acquire exclusive access + switch to channel */
    ret = mux_control_select(adc->mux, channel);
    if (ret)
        return ret;

    ret = do_adc_conversion(adc);         /* actual hardware read */

    mux_control_deselect(adc->mux);       /* release + apply idle_state */
    return ret;
}
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Constants المهمة

الـ `idle_state` بيتحدد إما بـ قيمة state حقيقية أو بـ constant خاص من `dt-bindings/mux/mux.h`:

| Constant | القيمة | المعنى |
|---|---|---|
| `MUX_IDLE_AS_IS` | `-1` | خلي الـ mux على حاله لما متحدش بيستخدمه |
| `MUX_IDLE_DISCONNECT` | `-2` | افصله تماماً عند الـ idle |
| `cached_state = -1` | `-1` | معناه "مش عارف الـ state الحالي" (uninitialized) |

**قيم `states`:** بتحدد عدد الـ states الصحيحة من `0` لـ `states - 1`. الـ driver بيضبطها قبل الـ registration.

---

### الـ Structs المهمة

#### 1. `struct mux_control_ops`

**الغرض:** الـ vtable — بيعرّف العمليات اللي الـ hardware driver بينفذها.

| Field | النوع | الوصف |
|---|---|---|
| `set` | `int (*)(struct mux_control *, int state)` | بيطلب من الـ hardware يغير الـ mux لـ state معين |

- الـ `set` callback بترجع `0` للنجاح أو error code سالب.
- الـ core بيستدعيها بعد ما يعمل validation للـ `state` المطلوب.
- مفيش `get` لأن الـ core بيكاش الـ state في `cached_state`.

---

#### 2. `struct mux_control`

**الغرض:** بيمثل **مفتاح mux واحد** — يعني قناة تحكم واحدة داخل الـ chip.

| Field | النوع | الوصف |
|---|---|---|
| `lock` | `struct semaphore` | بيحمي الـ state — binary semaphore، يسمح لـ consumer واحد بس في نفس الوقت |
| `chip` | `struct mux_chip *` | pointer للـ parent chip |
| `cached_state` | `int` | الـ state الحالي أو `-1` لو مش معروف |
| `states` | `unsigned int` | عدد الـ states المتاحة (الـ driver بيضبطها) |
| `idle_state` | `int` | الـ state عند الـ idle أو `MUX_IDLE_AS_IS` / `MUX_IDLE_DISCONNECT` |
| `last_change` | `ktime_t` | timestamp آخر تغيير (nanosecond resolution) |

**ملاحظات مهمة:**
- الـ driver يقدر يكتب في `states` و `idle_state` **بس بين الـ alloc والـ register**.
- الـ `cached_state` ملكية الـ core خالص — الـ driver ما يلمسهاش أبداً.
- الـ `lock` هو **semaphore مش mutex** — يعني الـ `up()` ممكن يتعمل من thread تاني أو من interrupt context.

---

#### 3. `struct mux_chip`

**الغرض:** بيمثل **الـ chip الفيزيائي** اللي بيحوي واحد أو أكتر من الـ mux controllers.

| Field | النوع | الوصف |
|---|---|---|
| `controllers` | `unsigned int` | عدد الـ mux controllers في الـ chip |
| `dev` | `struct device` | الـ kernel device — بيربطه بالـ device model |
| `id` | `int` | ID داخلي للتعريف بالـ device |
| `ops` | `const struct mux_control_ops *` | pointer للـ vtable بتاع الـ driver |
| `mux[]` | `struct mux_control[]` | flexible array — الـ controllers نفسهم محجوزين inline في آخر الـ struct |

**`__counted_by(controllers)`:** annotation للـ compiler بيقوله إن حجم الـ flexible array متحكم فيه بـ `controllers` — بيساعد في الـ bounds checking.

**الـ Private Memory:** بعد آخر عنصر في `mux[]` مباشرةً، الـ `mux_chip_alloc` بيحجز `sizeof_priv` bytes إضافية — الـ `mux_chip_priv()` بيرجعلك pointer ليها.

```c
/* layout في الـ memory بعد mux_chip_alloc() */
[ struct mux_chip ][ mux[0] ][ mux[1] ]...[ mux[N-1] ][ private data ]
                    ^--- mux[] flexible array ---^      ^--- priv ----^
```

---

### علاقات الـ Structs

```
┌─────────────────────────────────────────────────────────┐
│                    struct mux_chip                       │
│                                                         │
│  controllers = N                                        │
│  dev ──────────────────────────────► struct device      │
│  id                                                     │
│  ops ──────────────────────────────► struct mux_control_ops
│  mux[0] ◄──────────────────────────┐  │  .set()        │
│  mux[1]                            │  └────────────────┘
│  ...                               │
│  mux[N-1]                          │
│  [private data]                    │
└─────────────────────────────────────┘
         │
         │  mux[i] هو struct mux_control
         ▼
┌──────────────────────────────────┐
│       struct mux_control         │
│                                  │
│  lock (semaphore)                │
│  chip ──────────────────────────►│ back-pointer للـ parent
│  cached_state                    │
│  states                          │
│  idle_state                      │
│  last_change (ktime_t)           │
└──────────────────────────────────┘
```

**الـ macro `to_mux_chip(x)`:** بيعمل `container_of` من `struct device *` للـ `struct mux_chip` عبر الـ `dev` field — الطريقة الكلاسيكية في الـ kernel للوصول للـ parent struct.

**الـ `mux_control_get_index()`:** بيحسب index الـ controller بطرح pointers:
```c
return mux - mux->chip->mux;  /* pointer arithmetic */
```

---

### دورة حياة الـ mux_chip

```
[Driver Probe]
     │
     ▼
mux_chip_alloc(dev, N, sizeof_priv)
  ├── يحجز mux_chip + N * mux_control + sizeof_priv
  ├── يعمل device_initialize(&chip->dev)
  └── يرجع struct mux_chip *
     │
     ▼
[Driver يضبط الـ fields]
  ├── chip->ops = &my_ops
  ├── for each mux[i]:
  │     mux[i].states = M
  │     mux[i].idle_state = MUX_IDLE_AS_IS
  └── (mش يلمس cached_state)
     │
     ▼
mux_chip_register(chip)
  ├── يعمل device_add(&chip->dev)
  ├── يسجل في sysfs
  └── يبدأ يقبل requests من consumers
     │
     ▼
[Consumers يستخدموا mux_control_get() / devm_mux_control_get()]
     │
     ▼
[Driver Remove]
     │
     ▼
mux_chip_unregister(chip)
  └── device_del(&chip->dev)
     │
     ▼
mux_chip_free(chip)
  └── put_device() → يحرر الـ memory
```

**الـ devm variants:**
- `devm_mux_chip_alloc()` + `devm_mux_chip_register()` بيربطوا الـ lifetime بالـ device — الـ cleanup بيتعمل تلقائي عند الـ unbind.

---

### Call Flow Diagrams

#### تغيير الـ state (من الـ consumer API)

```
consumer calls mux_control_select(mux, state)
  │
  ├── يتحقق إن state < mux->states
  │
  ├── down(&mux->lock)          ← ياخد الـ semaphore (blocking)
  │     └── لو حد تاني شايله → ينتظر
  │
  ├── لو state == mux->cached_state → skip hardware call
  │
  ├── chip->ops->set(mux, state)   ← يكلم الـ hardware driver
  │     └── [Driver Code]
  │           └── يكتب في الـ hardware registers
  │
  ├── mux->cached_state = state
  ├── mux->last_change = ktime_get()
  └── يرجع 0 أو error
```

#### تحرير الـ state

```
consumer calls mux_control_deselect(mux)
  │
  ├── لو idle_state != MUX_IDLE_AS_IS:
  │     └── chip->ops->set(mux, idle_state)
  │           └── يرجع الـ mux للـ idle position
  │
  └── up(&mux->lock)            ← يحرر الـ semaphore
```

---

### استراتيجية الـ Locking

| Lock | النوع | اللي بيحميه | من بيمسكه |
|---|---|---|---|
| `mux_control.lock` | `struct semaphore` | الـ state الكامل للـ controller: الاستخدام الحصري + `cached_state` | الـ consumer عبر `mux_control_select()` |

**التفاصيل:**

- **Binary Semaphore (count=1):** كل `mux_control` عنده semaphore منفصل — يعني ممكن يستخدموا controllers مختلفة في نفس الوقت بدون تعارض.
- **مش mutex:** الـ semaphore ما عندوش owner — يعني thread A ممكن يعمل `down()` وthread B يعمل `up()` — ده بيسمح بـ patterns زي الـ async completion.
- **الـ `cached_state` محمي بالـ lock:** الـ core ما بيقراش ولا بيكتبش فيها إلا وهو شايل الـ lock.
- **لا lock على مستوى الـ chip:** الـ `mux_chip` نفسه مالوش lock — الـ registration/unregistration بتتم عبر الـ kernel device model locks.
- **ترتيب الـ locks:** مفيش nested locking في الـ API ده — كل controller مستقل.

**مثال عملي — I2C multiplexer:**

```
Thread A: mux_control_select(mux[0], 2)   /* يحجز mux[0] */
Thread B: mux_control_select(mux[1], 0)   /* يحجز mux[1] بالتوازي — OK */
Thread C: mux_control_select(mux[0], 1)   /* ينتظر Thread A يحرر mux[0] */
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ Structs والـ Types الأساسية

| النوع | الوصف |
|-------|--------|
| `struct mux_control_ops` | vtable بتاع الـ driver — فيها pointer للـ `set` callback |
| `struct mux_control` | يمثل controller واحد جوه الـ chip — بيتحكم في حالة الـ mux |
| `struct mux_chip` | يمثل الـ chip الكاملة اللي بتضم array من الـ `mux_control` |
| `ktime_t last_change` | timestamp بـ nanosecond resolution لآخر تغيير في الحالة |

#### جدول الـ Functions

| الـ Function | النوع | الغرض |
|---|---|---|
| `mux_chip_alloc()` | Allocation | بتخصص `mux_chip` + private data في heap |
| `mux_chip_register()` | Registration | بتسجل الـ chip في kernel device model |
| `mux_chip_unregister()` | Cleanup | بتفك تسجيل الـ chip |
| `mux_chip_free()` | Cleanup | بتحرر الـ memory المخصصة |
| `devm_mux_chip_alloc()` | devres Allocation | نسخة managed من `mux_chip_alloc()` |
| `devm_mux_chip_register()` | devres Registration | نسخة managed من `mux_chip_register()` |
| `mux_chip_priv()` | Helper / Accessor | بترجع pointer للـ private driver data |
| `mux_control_get_index()` | Helper / Accessor | بترجع index الـ controller جوه الـ chip |

---

### المجموعة 1: Allocation Functions

الغرض من المجموعة دي إنها بتعمل allocate للـ `mux_chip` struct في الـ kernel heap، مع الأخذ في الاعتبار إن الـ chip ممكن تحتاج extra private data خاص بالـ driver. الـ core بيحجز مكان الـ `mux_control` array مباشرةً بعد الـ `mux_chip` في نفس الـ allocation block، وبعديها ممكن الـ driver يطلب مساحة إضافية private data تيجي بعديهم.

```
Memory Layout بعد mux_chip_alloc():
+------------------+-----------------------------+-------------------+
|   mux_chip       |  mux_control[controllers]   |  priv data        |
| (fixed size)     |  (controllers * sizeof ctrl) |  (sizeof_priv)    |
+------------------+-----------------------------+-------------------+
        ^                                                ^
        |                                                |
   mux_chip ptr                                 mux_chip_priv() يرجع هنا
```

---

#### `mux_chip_alloc()`

```c
struct mux_chip *mux_chip_alloc(struct device *dev,
                                unsigned int controllers,
                                size_t sizeof_priv);
```

بتخصص `mux_chip` جديدة في الـ heap وبتربطها بالـ `dev` اللي بتتسجل جواه. بتحجز مكان لـ `controllers` عدد من الـ `mux_control` وأي private data إضافية محتاجها الـ driver. الـ function بتعمل initialize للـ `device` struct الداخلية وبتسند `id` فريد للـ chip.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ parent device — الـ chip هتتسجل تحتيه في الـ device tree |
| `controllers` | `unsigned int` | عدد الـ mux controllers اللي الـ chip دي بتتحكم فيهم |
| `sizeof_priv` | `size_t` | حجم الـ private data اللي الـ driver محتاجها، `0` لو مش محتاج |

**Return value:** pointer لـ `mux_chip` جديدة في حالة النجاح، أو `ERR_PTR()` في حالة الفشل (عادةً `-ENOMEM`).

**Key details:**
- الـ allocation بتحصل بـ `kzalloc()` — كل الـ memory بتتعمل zero-init.
- الـ `mux_control` array موجودة كـ flexible array member (`mux[]`) في آخر الـ `mux_chip` struct — وده بيخلي الـ private data تيجي بعديها مباشرةً.
- الـ `__counted_by(controllers)` annotation موجودة على الـ flexible array لـ bounds checking في الـ Clang/GCC sanitizers.
- بعد الـ allocation، الـ driver لازم يعمل set لـ `mux->states` و`mux->idle_state` لكل controller **قبل** ما يعمل register.
- الـ `cached_state` بيتبدأ بـ `-1` (يعني unknown/not set).

**Caller context:** يتنادى من أي process context — مش مسموح من interrupt context لأن الـ allocation بتاخد sleeping allocator.

---

#### `devm_mux_chip_alloc()`

```c
struct mux_chip *devm_mux_chip_alloc(struct device *dev,
                                     unsigned int controllers,
                                     size_t sizeof_priv);
```

نسخة **devres-managed** من `mux_chip_alloc()`. بتعمل نفس الـ allocation بس بتسجل cleanup action تلقائي على الـ `dev` — لما الـ device تتفصل أو الـ driver يعمل `remove`، الـ chip بتتحرر أوتوماتيك من غير ما الـ driver يحتاج يعمل `mux_chip_free()` بنفسه.

**Parameters:** نفس `mux_chip_alloc()` بالظبط.

**Return value:** نفس `mux_chip_alloc()`.

**Key details:**
- بتستخدم `devm_add_action()` أو `devres` mechanism داخلياً لتسجيل الـ cleanup.
- الـ driver مش محتاج يعمل `mux_chip_free()` في الـ error path أو في الـ `remove` callback.
- مناسبة جداً للـ platform drivers والـ I2C/SPI drivers اللي بتعمل probe/remove.

**Caller context:** Process context فقط.

---

### المجموعة 2: Registration Functions

المجموعة دي مسؤولة عن تسجيل الـ `mux_chip` في الـ kernel device model وجعلها متاحة للـ consumers اللي بيستخدموا الـ `mux/consumer.h` API. التسجيل بيحصل بعد ما الـ driver يعمل set لكل الـ state information المطلوبة.

---

#### `mux_chip_register()`

```c
int mux_chip_register(struct mux_chip *mux_chip);
```

بتسجل الـ `mux_chip` في الـ kernel device model عن طريق استدعاء `device_register()` على الـ `mux_chip->dev`. بعد الـ registration، الـ chip بتبقى visible للـ consumers اللي بيطلبوا الـ mux controllers عن طريق `mux_control_get()`.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `mux_chip` | `struct mux_chip *` | الـ chip اللي اتخصصت بـ `mux_chip_alloc()` وجاهزة للتسجيل |

**Return value:** `0` في حالة النجاح، أو error code سالب في حالة الفشل.

**Key details:**
- لازم يتنادى **بعد** ما الـ driver يعمل set لـ `mux->states` و`mux->idle_state` و`mux_chip->ops` لكل controller.
- بعد الـ registration، الـ semaphore lock في كل `mux_control` بيبقى جاهز للـ consumers.
- لو `idle_state` مش `MUX_IDLE_AS_IS`، الـ core ممكن يستدعي الـ `ops->set` عشان يحط الـ mux في الـ idle state.
- الـ `mux_chip->dev` بيتسجل كـ child للـ `dev` اللي اتدت في `mux_chip_alloc()`.

**Caller context:** Process context فقط.

---

#### `devm_mux_chip_register()`

```c
int devm_mux_chip_register(struct device *dev, struct mux_chip *mux_chip);
```

نسخة **devres-managed** من `mux_chip_register()`. بتسجل cleanup action تلقائي يعمل `mux_chip_unregister()` لما الـ `dev` تتفصل.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ device اللي بيتسجل عليها الـ cleanup action |
| `mux_chip` | `struct mux_chip *` | الـ chip المراد تسجيلها |

**Return value:** `0` أو error code سالب.

**Key details:**
- مناسبة جداً لما تيجي مع `devm_mux_chip_alloc()` — الترتيب الصح هو alloc ثم register، وعند الـ unwind بيحصل unregister ثم free أوتوماتيك.
- الـ cleanup actions في devres بتنفذ بترتيب عكسي (LIFO).

---

### المجموعة 3: Cleanup / Teardown Functions

---

#### `mux_chip_unregister()`

```c
void mux_chip_unregister(struct mux_chip *mux_chip);
```

بتفك تسجيل الـ `mux_chip` من الـ kernel device model. بعديها الـ consumers مش هيقدروا يعملوا `mux_control_get()` جديد على الـ chip دي. الـ consumers الموجودين اللي عندهم reference محتاجين يعملوا `mux_control_put()` أولاً.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `mux_chip` | `struct mux_chip *` | الـ chip المراد فك تسجيلها |

**Return value:** `void`

**Key details:**
- بتستدعي `device_unregister()` على الـ `mux_chip->dev`.
- لو في consumers لسه شايلين reference على أي `mux_control` جوه الـ chip، الـ device reference counting هيمنع الـ free الفعلي لحد ما يعملوا `put`.
- في الغالب بيتنادى من `remove` callback أو من الـ devm cleanup.

---

#### `mux_chip_free()`

```c
void mux_chip_free(struct mux_chip *mux_chip);
```

بتحرر الـ memory المخصصة للـ `mux_chip` وكل الـ private data المرتبطة بيها. يتنادى بعد `mux_chip_unregister()` في الـ non-devm path.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `mux_chip` | `struct mux_chip *` | الـ chip المراد تحريرها |

**Return value:** `void`

**Key details:**
- مش لازم تنادي عليها لو استخدمت `devm_mux_chip_alloc()` — الـ devres framework هيعملها أوتوماتيك.
- لو عملت `mux_chip_alloc()` يدوياً، لازم تعمل `mux_chip_free()` يدوياً في كل error path وفي الـ `remove` callback.
- الـ `id` الخاص بالـ chip بيتحرر من الـ ida.

---

### المجموعة 4: Inline Helper Functions

الـ helpers دي بسيطة لكن مهمة — بتوفر clean accessor API للـ driver بدل ما يعمل pointer arithmetic يدوي.

---

#### `mux_chip_priv()`

```c
static inline void *mux_chip_priv(struct mux_chip *mux_chip)
{
    return &mux_chip->mux[mux_chip->controllers];
}
```

بترجع pointer للـ private driver data اللي تم حجزها في `mux_chip_alloc()` عبر الـ `sizeof_priv` parameter. الـ private data موجودة مباشرةً بعد آخر عنصر في الـ `mux[]` flexible array.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `mux_chip` | `struct mux_chip *` | الـ chip اللي عايز تجيب منها الـ private data |

**Return value:** `void *` pointer للـ private data — الـ driver بيعمله cast للـ type المناسب.

**Key details:**
- الـ implementation بتعتمد على إن الـ flexible array `mux[]` هي آخر field في الـ struct — فـ `&mux[controllers]` بيوصل لأول byte بعد الـ array.
- لو الـ `sizeof_priv` كانت `0` في وقت الـ allocation، الـ returned pointer صحيح syntactically لكن بلاش تقرأ/تكتب منه.
- الـ typical usage:
  ```c
  struct my_priv *priv = mux_chip_priv(chip);
  priv->some_field = value;
  ```

**Caller context:** يتنادى من أي context — inline computation بحتة، مفيش locking.

---

#### `mux_control_get_index()`

```c
static inline unsigned int mux_control_get_index(struct mux_control *mux)
{
    return mux - mux->chip->mux;
}
```

بترجع الـ index بتاع الـ `mux_control` جوه الـ parent chip. الـ calculation بتاعتها pointer subtraction بسيطة — لأن الـ `mux[]` array عناصرها contiguous في الـ memory.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `mux` | `struct mux_control *` | الـ controller اللي عايز تعرف رقمه |

**Return value:** `unsigned int` — الـ zero-based index للـ controller جوه الـ chip.

**Key details:**
- مفيدة جداً في الـ `ops->set` callback لما الـ driver محتاج يعرف أنهو controller اتطلب منه يتغير، لأن كل الـ controllers في الـ chip بيشتركوا في نفس الـ `ops`.
- الـ typical usage في الـ `set` callback:
  ```c
  static int my_mux_set(struct mux_control *mux, int state)
  {
      struct mux_chip *chip = mux->chip;
      struct my_priv *priv = mux_chip_priv(chip);
      unsigned int idx = mux_control_get_index(mux); /* أنهو controller؟ */
      /* استخدم idx عشان تعمل set للـ hardware register الصح */
      return regmap_write(priv->regmap, REG_MUX(idx), state);
  }
  ```

**Caller context:** يتنادى من أي context.

---

### المجموعة 5: Ops Struct و vtable

---

#### `struct mux_control_ops` — الـ vtable

```c
struct mux_control_ops {
    int (*set)(struct mux_control *mux, int state);
};
```

دي الـ vtable الوحيدة في الـ subsystem — الـ driver لازم يوفر implementation للـ `set` callback. الـ mux core بينادي عليها لما consumer يطلب تغيير حالة الـ mux.

**الـ `set` callback:**

```c
int (*set)(struct mux_control *mux, int state);
```

| Parameter | الوصف |
|---|---|
| `mux` | الـ controller المطلوب تغييره — استخدم `mux_control_get_index()` لمعرفة رقمه و`mux->chip` للوصول للـ chip |
| `state` | الحالة الجديدة المطلوبة — integer من `0` لـ `mux->states - 1`، أو `MUX_IDLE_DISCONNECT` |

**Return value:** `0` في حالة النجاح، error code سالب في حالة الفشل (مثلاً `-EIO` لو الـ hardware ما ردش).

**Key details:**
- الـ callback بيتنادى وهو شايل الـ `mux->lock` semaphore — مش مسموح بـ sleep طويل.
- الـ core بيعمل update لـ `mux->cached_state` **بعد** ما الـ `set` callback يرجع بنجاح.
- الـ `last_change` timestamp بيتحدث في الـ core بعد الـ `set` الناجح.
- الـ driver مسؤول عن الـ hardware access فقط — الـ state tracking والـ locking هما مسؤولية الـ core.

---

### تدفق الحياة الكاملة لـ mux driver

```
Driver probe():
    1. mux_chip_alloc(dev, N, sizeof_priv)
         └─> kzalloc(mux_chip + N*mux_control + sizeof_priv)
         └─> device_initialize(&chip->dev)

    2. priv = mux_chip_priv(chip)  --> اعمل setup للـ private state

    3. for each controller i:
         chip->mux[i].states    = NUM_STATES
         chip->mux[i].idle_state = MUX_IDLE_AS_IS (or specific state)

    4. chip->ops = &my_mux_ops    --> اربط الـ vtable

    5. mux_chip_register(chip)
         └─> device_add(&chip->dev)
         └─> chip visible to consumers via mux_control_get()

Driver remove():
    1. mux_chip_unregister(chip)
         └─> device_del(&chip->dev)
         └─> consumers can no longer get new references

    2. mux_chip_free(chip)
         └─> kfree(chip)

Consumer usage (مش جزء من driver.h):
    mux_control_get() --> down(&mux->lock)
    مux_control_select(state) --> ops->set(mux, state)
    mux_control_deselect() --> ops->set(mux, idle_state); up(&mux->lock)
```

---

### الـ Locking Model

الـ `mux_control` بيستخدم **binary semaphore** (`struct semaphore lock`) — مش mutex — عشان يسمح للـ consumer بأنه يعمل `up()` من thread مختلف عن اللي عمل `down()`. الـ consumers بيشتغلوا على النموذج ده:

```
Consumer A                     Consumer B
    |                              |
down(&mux->lock)              down(&mux->lock)  [blocked]
ops->set(mux, state_A)            |
... use the mux ...               |
ops->set(mux, idle_state)         |
up(&mux->lock)             -----> | [unblocked]
                              ops->set(mux, state_B)
```

الـ `cached_state` مش محمي بـ spinlock منفصل — الـ semaphore هو الحماية الوحيدة. الـ driver **ممنوع** يكتب في `cached_state` مباشرةً.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs entries

الـ **mux subsystem** مش بيسجّل entries في debugfs بشكل مباشر، لكن ممكن توصل للمعلومات عن طريق الـ sysfs والـ device model.

```bash
# تأكد إن debugfs متوقّعة وموجودة
mount | grep debugfs
# لو مش موجودة
mount -t debugfs none /sys/kernel/debug

# استعراض كل الـ entries المتعلقة بالـ mux
ls /sys/kernel/debug/
```

لو بتكتب driver جديد وعاوز تضيف debugfs:

```c
#include <linux/debugfs.h>

/* في probe() الخاصة بالـ driver */
struct dentry *dbg_dir = debugfs_create_dir("my_mux", NULL);
debugfs_create_u32("cached_state", 0444, dbg_dir, &mux->cached_state);
debugfs_create_u32("states",       0444, dbg_dir, &mux->states);
```

---

#### 2. sysfs entries

الـ **mux class** بتسجّل الـ chip كـ device تحت `/sys/class/mux/`.

```bash
# اعرض كل الـ mux chips المسجّلة
ls /sys/class/mux/

# مثال: muxchip0
ls /sys/class/mux/muxchip0/

# اعرض الـ parent device
cat /sys/class/mux/muxchip0/../uevent

# اعرض اسم الـ device
cat /sys/class/mux/muxchip0/uevent

# trace الـ device hierarchy
udevadm info -a /sys/class/mux/muxchip0
```

الـ fields المهمة اللي ممكن تشوفها:

| المسار | المحتوى |
|---|---|
| `/sys/class/mux/muxchipN/uevent` | DEVTYPE, MAJOR, MINOR |
| `/sys/class/mux/muxchipN/../of_node` | symlink للـ Device Tree node |
| `/sys/class/mux/muxchipN/power/` | runtime PM state |

---

#### 3. ftrace — tracepoints وـ events

الـ mux subsystem مش عنده tracepoints رسمية، لكن ممكن تستخدم **function tracer** على دوال الـ core.

```bash
# شوف الدوال المتاحة في الـ mux subsystem
grep mux /sys/kernel/debug/tracing/available_filter_functions | grep -i "mux_control\|mux_chip"

# فعّل tracing على دوال الـ mux core
echo 'mux_control_select_delay' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'mux_control_deselect'    >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'mux_control_set'         >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'mux_chip_register'       >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل workload
cat /sys/kernel/debug/tracing/trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

لو عاوز تعرف timing دقيق بين select وdeselect:

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'mux_control_select_delay' > /sys/kernel/debug/tracing/set_graph_function
cat /sys/kernel/debug/tracing/trace
```

---

#### 4. printk وـ dynamic debug

الـ `core.c` بيستخدم `pr_fmt` اللي بيضيف prefix `"mux-core: "` لكل رسائل الـ `pr_*`.

```bash
# فعّل dynamic debug للـ mux-core module
echo 'module mux-core +p' > /sys/kernel/debug/dynamic_debug/control

# أو لو بتشتغل مع driver معين (مثلاً gpio-mux)
echo 'module gpio-mux +p'   > /sys/kernel/debug/dynamic_debug/control
echo 'module mux-mmio +p'   > /sys/kernel/debug/dynamic_debug/control

# فعّل كل رسائل الـ mux بالـ file
echo 'file drivers/mux/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mux/gpio.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# تابع الـ kernel log
dmesg -w | grep -i "mux"
```

الـ flags المتاحة في dynamic_debug:
| Flag | المعنى |
|---|---|
| `p` | طباع الرسالة |
| `f` | أضف اسم الـ function |
| `l` | أضف رقم السطر |
| `m` | أضف اسم الـ module |
| `t` | أضف الـ thread ID |

---

#### 5. Kernel config options للـ debugging

```
CONFIG_MUX_CORE=m              # الـ core كـ module (أسهل للـ reload)
CONFIG_MUX_GPIO=m              # driver GPIO-based mux
CONFIG_MUX_MMIO=m              # driver MMIO-based mux

# عام للـ driver debugging
CONFIG_DEBUG_KERNEL=y
CONFIG_DEBUG_DRIVER=y          # dev_dbg() يطبع
CONFIG_DYNAMIC_DEBUG=y         # dynamic debug framework
CONFIG_FTRACE=y
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_GRAPH_TRACER=y

# للـ semaphore/locking debugging (الـ mux->lock هو semaphore)
CONFIG_DEBUG_ATOMIC_SLEEP=y    # اكتشف down() في atomic context
CONFIG_PROVE_LOCKING=y         # lockdep
CONFIG_LOCK_STAT=y             # لقياس lock contention
CONFIG_DEBUG_LOCK_ALLOC=y

# memory
CONFIG_KASAN=y                 # اكتشف out-of-bounds في mux_chip_alloc
CONFIG_DEBUG_SLAB=y

# device model
CONFIG_DEBUG_DEVRES=y          # لتتبع devm_mux_chip_alloc/register
```

---

#### 6. أدوات خاصة بالـ subsystem

```bash
# استخدم udevadm لتتبع إضافة/حذف الـ mux device
udevadm monitor --kernel --udev &
# ثم load/unload الـ module
modprobe gpio-mux
modprobe -r gpio-mux

# استعراض كل الـ devices من class mux
for d in /sys/class/mux/*/; do
    echo "=== $d ==="
    cat "$d/uevent" 2>/dev/null
    ls "$d"
done

# تحقق من الـ IDA (ID allocator) لمعرفة كام chip مسجّل
cat /proc/iomem | grep -i mux   # لو MMIO-based
ls /sys/bus/platform/devices/ | grep mux
```

---

#### 7. جدول رسائل الـ error الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---|---|---|
| `mux-core: muxchipX failed to get a device id` | `ida_alloc()` فشل — نفد الـ IDs أو memory | تحقق من `CONFIG_IDR_DEBUG`، راجع leaks في `mux_chip_free()` |
| `unable to set idle state` | الـ `.set()` callback رجع error وقت الـ register | تحقق من الـ hardware وتأكد إن `mux->idle_state` في النطاق |
| `device_add failed in mux_chip_register: -N` | فشل إضافة الـ device للـ sysfs | تحقق من تعارض الأسماء أو missing parent device |
| `WARN_ON(!dev || !controllers)` | استدعاء `mux_chip_alloc()` بـ NULL أو صفر | تحقق من الـ driver اللي بيستدعي الـ alloc |
| `WARN_ON(state < 0 \|\| state >= mux->states)` | طلب state خارج النطاق | تأكد إن الـ consumer بيستخدم `mux_control_states()` |
| `mux-core: ... -EINTR` | `down_killable()` اتقطع بـ signal | طبيعي لو الـ process اتقتل، لكن لو متكرر = deadlock |
| `mux-core: ... -ENODEV` | الـ mux_control مش موجود أو اتمسح | تحقق من الـ lifetime وإن الـ consumer مش بيستخدم مؤشر stale |

---

#### 8. أماكن استراتيجية لـ `dump_stack()` وـ `WARN_ON()`

```c
/* في الـ .set() callback بتاعك لو الـ state مش منطقي */
int my_mux_set(struct mux_control *mux, int state)
{
    /* تحقق إن الـ chip لسه شغال */
    if (WARN_ON(!my_priv->initialized)) {
        dump_stack();
        return -EIO;
    }

    /* تحقق من نطاق الـ state قبل الكتابة للـ hardware */
    if (WARN_ON(state < 0 || state >= mux->states))
        return -EINVAL;

    /* بعد الكتابة، تحقق من الـ readback */
    readback = read_hw_state(my_priv);
    if (WARN_ON(readback != state)) {
        dev_err(&mux->chip->dev,
                "state mismatch: wrote %d, read %d\n", state, readback);
    }
    return 0;
}

/* في الـ consumer، قبل ما تعمل deselect */
void my_consumer_done(struct mux_control *mux)
{
    /* تحقق إن الـ semaphore مش released مرتين */
    WARN_ON(mux->lock.count > 0); /* يعني مش locked */
    mux_control_deselect(mux);
}
```

أهم نقاط وضع الـ `WARN_ON`:
- بعد `mux_chip_alloc()` للتحقق من القيم الابتدائية
- في `mux_control_set()` لو `cached_state` اتغير خارج الـ lock
- في الـ `.set()` callback لو الـ hardware ما استجابش

---

### Hardware Level

---

#### 1. التحقق إن hardware state يطابق kernel state

الـ `cached_state` في `struct mux_control` هو ما يظنه الـ kernel إن الـ hardware عليه. لازم تتحقق يدوياً:

```bash
# اقرأ الـ state الحالي من الـ sysfs (لو الـ driver عمل expose)
# مثلاً لـ gpio-mux
cat /sys/kernel/debug/gpio   # شوف قيم الـ GPIO اللي بتتحكم في الـ mux

# لـ mmio-mux، اقرأ الـ register مباشرة (شوف القسم التالي)

# قارن مع الـ cached_state عبر طباعته بـ dynamic debug أو بـ printk مؤقت
```

---

#### 2. Register dump — devmem2 وـ /dev/mem

لـ **MMIO-based mux** (مثل `mux-mmio.c` اللي بيستخدم `regmap`):

```bash
# أولاً، اعرف الـ base address من الـ Device Tree أو iomem
cat /proc/iomem | grep -i mux

# استخدم devmem2 لقراءة الـ register
# مثال: base = 0xFE200000, offset الـ mux register = 0x10
devmem2 0xFE200010 w   # قراءة 32-bit word

# لو devmem2 مش متاح، استخدم /dev/mem
python3 -c "
import mmap, os, struct
fd = os.open('/dev/mem', os.O_RDONLY)
m = mmap.mmap(fd, 4096, offset=0xFE200000)
val = struct.unpack('<I', m[0x10:0x14])[0]
print(hex(val))
m.close(); os.close(fd)
"

# أو استخدم busybox devmem
busybox devmem 0xFE200010
```

لـ **GPIO-based mux**:

```bash
# اعرف أرقام الـ GPIO lines
cat /sys/kernel/debug/gpio

# تحقق من قيمة كل GPIO يدوياً
gpioget gpiochip0 5   # مثلاً GPIO 5 هو bit 0 للـ mux
gpioget gpiochip0 6   # bit 1

# الـ state = (bit1 << 1) | bit0
```

---

#### 3. Logic Analyzer / Oscilloscope

للتحقق من تسلسل الأحداث على الـ hardware:

- **نقاط القياس**: شبّك على خطوط الـ GPIO أو الـ SPI/I2C/MMIO اللي بيتحكم فيها الـ mux driver.
- **التريغر**: اضبط الـ trigger على أول edge للـ select signal.
- **ما تدور عليه**:
  - زمن الـ settling بعد تغيير الـ state — قارنه مع `delay_us` اللي بتحدده في `mux_control_select_delay()`.
  - الـ glitch على خطوط الـ output أثناء تغيير الـ state.
  - تأكد إن الـ idle state بيتطبق فعلاً بعد `mux_control_deselect()`.

```
Timeline مثال لـ GPIO mux بـ 2 pins:

  GPIO_A: ─────┐        ┌─────
               └────────┘
  GPIO_B: ──────────┐   ┌─────
                    └───┘
               ↑         ↑
           select(2)  deselect→idle
```

- **Oscilloscope**: مهم لقياس الـ settling time الحقيقي وتوثيقه ضد `delay_us`.

---

#### 4. مشاكل hardware شائعة وـ kernel log patterns

| المشكلة | الـ kernel log pattern | التشخيص |
|---|---|---|
| الـ mux مش بيتغير فعلياً | لا يوجد error، لكن الـ consumer بيفشل | readback من الـ register != written value |
| power-on state غلط | `unable to set idle state` | الـ hardware في state غير متوقع وقت الـ boot |
| GPIO مش configured صح | `gpiod_set_value: invalid gpio` | تحقق من `pinctrl` وـ DT |
| تدخل كهربي (glitch) | بيان متقطع من الـ consumer | ابحث عن capacitor على خطوط الـ select |
| الـ I2C mux خسر الـ address | `i2c i2c-X: ... timeout` بعد mux select | تحقق من الـ pull-ups وـ bus speed |
| regmap write فشل | `-EIO` من `mux_control_set` | تحقق من الـ clocks والـ power domains |

---

#### 5. Device Tree debugging

الـ mux subsystem بيقرأ الـ DT لتحديد الـ `idle-state` والـ number of controllers.

```bash
# تحقق إن الـ DT node للـ mux موجود وصح
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "mux"

# أو
cat /sys/firmware/devicetree/base/soc/mux@FE200000/compatible 2>/dev/null
cat /sys/firmware/devicetree/base/soc/mux@FE200000/mux-states 2>/dev/null | xxd

# تحقق من الـ idle-state
cat /sys/firmware/devicetree/base/soc/mux@FE200000/idle-state 2>/dev/null | xxd
# القيمة: FFFFFFFE = MUX_IDLE_AS_IS(-2 as u32)، FFFFFFFD = MUX_IDLE_DISCONNECT

# تحقق من الـ #mux-control-cells
cat /sys/firmware/devicetree/base/soc/mux@FE200000/'#mux-control-cells' | xxd
```

مثال DT node صحيح:

```dts
mux: mux-controller@0 {
    compatible = "gpio-mux";
    #mux-control-cells = <0>;
    mux-gpios = <&gpio0 5 GPIO_ACTIVE_HIGH>,
                <&gpio0 6 GPIO_ACTIVE_HIGH>;
    idle-state = <MUX_IDLE_AS_IS>;  /* -1 */
};
```

```bash
# تأكد إن الـ consumer بيشير للـ mux صح
grep -r "mux-controls" /sys/firmware/devicetree/base/ 2>/dev/null
```

---

### Practical Commands

---

#### سكريبت تشخيص شامل

```bash
#!/bin/bash
# mux-debug.sh — سكريبت تشخيص الـ mux subsystem

echo "=== MUX Chips ==="
ls /sys/class/mux/ 2>/dev/null || echo "No mux chips found"

echo ""
echo "=== MUX Device Info ==="
for chip in /sys/class/mux/*/; do
    echo "--- $chip ---"
    udevadm info "$chip" 2>/dev/null | grep -E "DEVPATH|DRIVER|OF_"
done

echo ""
echo "=== GPIO State (for GPIO mux) ==="
cat /sys/kernel/debug/gpio 2>/dev/null | grep -i mux

echo ""
echo "=== Kernel Messages ==="
dmesg | grep -i "mux" | tail -30

echo ""
echo "=== Dynamic Debug: enable mux-core ==="
echo 'file drivers/mux/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control 2>/dev/null \
    && echo "Dynamic debug enabled" || echo "Failed (need root or CONFIG_DYNAMIC_DEBUG)"
```

---

#### تفعيل ftrace لمتابعة كل mux state changes

```bash
#!/bin/bash
# ftrace-mux.sh

TDIR=/sys/kernel/debug/tracing

# صفّر
echo 0 > $TDIR/tracing_on
echo > $TDIR/trace

# اختر الـ function tracer
echo function > $TDIR/current_tracer

# فلتر على دوال الـ mux فقط
echo 'mux_control_select_delay
mux_control_deselect
mux_state_select_delay
mux_state_deselect' > $TDIR/set_ftrace_filter

# شغّل
echo 1 > $TDIR/tracing_on
echo "Tracing started. Run your workload, then press Enter..."
read

echo 0 > $TDIR/tracing_on
echo "=== Trace Output ==="
cat $TDIR/trace
```

**مثال output مع تفسيره:**

```
# TASK-PID   CPU#  ||||  TIMESTAMP  FUNCTION
# |          |     ||||  |          |
 app-1234    [001] ....  12.345678: mux_control_select_delay <-i2c_mux_select
 app-1234    [001] ....  12.346001: mux_control_deselect     <-i2c_mux_deselect
```

- الـ `<-i2c_mux_select` يقول مين استدعى الـ select.
- الفرق بين timestamps = وقت استخدام الـ mux (~323µs في المثال).

---

#### قراءة الـ semaphore count لكشف الـ deadlock

```bash
# لو عندك kernel مع CONFIG_LOCK_STAT
cat /proc/lock_stat | grep -A 5 "mux"

# أو استخدم crash/gdb مع vmlinux لو عندك kernel dump
# crash> struct mux_control <addr>
# تحقق من lock.count — لو 0 يعني locked، 1 يعني free
```

---

#### التحقق من الـ idle state بعد الـ unregister

```bash
# فعّل dynamic debug قبل rmmod
echo 'file drivers/mux/core.c +p' > /sys/kernel/debug/dynamic_debug/control

# راقب dmesg
dmesg -w &
DMESG_PID=$!

# unload الـ module
rmmod my_mux_driver

# شوف إيه الـ state الأخير اللي اتكتب
kill $DMESG_PID
dmesg | grep "mux" | tail -10
```

---

#### جدول سريع: أداة لكل مشكلة

| المشكلة | الأداة | الأمر |
|---|---|---|
| مش شايف الـ mux chip | sysfs | `ls /sys/class/mux/` |
| state مش بيتغير | ftrace | `echo function > .../current_tracer` |
| deadlock في الـ lock | lockdep | `dmesg \| grep "possible deadlock"` |
| register value غلط | devmem2 | `devmem2 <addr> w` |
| DT mismatch | dtc | `dtc -I fs /sys/firmware/devicetree/base` |
| idle state مش بيتطبق | dynamic debug | `echo 'file drivers/mux/core.c +p'` |
| GPIO mux قيمة غلط | gpioget | `gpioget gpiochipN <line>` |
| timing مشكلة | oscilloscope + delay_us | راجع `mux_control_select_delay()` |
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — SPI MUX مش بيتبدل صح

#### العنوان
**SPI bus mux عالق في state واحدة في gateway صناعي**

#### السياق
شركة بتبني industrial IoT gateway على **TI AM62x**. الـ board عندها SPI bus واحدة بتخدم 3 أجهزة: flash memory، temperature sensor، و pressure transducer. المهندس استخدم GPIO-based mux chip (74HC4051) عشان يوزع الـ SPI على الـ 3 أجهزة. الـ driver اتكتب باستخدام **`mux/driver.h`** API.

#### المشكلة
بعد boot، الـ temperature sensor شغال تمام، لكن الـ pressure transducer مش بيرد خالص. الـ flash بيشتغل برضو. المهندس يفتح oscilloscope يلاقي الـ CS line بتاعت الـ pressure transducer مش بتتحرك.

#### التحليل
الـ driver بيستخدم `mux_control_ops.set()` عشان يغير state الـ mux. المشكلة في الـ `cached_state`:

```c
struct mux_control {
    struct semaphore lock;
    struct mux_chip *chip;
    int cached_state;   /* <-- هنا المشكلة */
    unsigned int states;
    int idle_state;
    ktime_t last_change;
};
```

الـ `cached_state` بيتعمله initialize بـ `-1` (يعني "مش محدد"). أول ما الـ temperature sensor اتقرأ، الـ mux core استدعى `ops->set()` وخزّن الـ state الجديدة في `cached_state`. المرة الجاية لما الكود طلب نفس الـ state تاني (لو في retry أو multiple reads للـ temperature sensor)، الـ core اكتشف إن `cached_state` مطابق فـ skip الـ `ops->set()` call — ده تحسين للـ performance عشان ميستدعيش الـ GPIO unnecessarily.

المشكلة: الـ pressure transducer driver كان بيطلب `state 2`، لكن قبل ما يوصله، حصل hardware glitch رجّع الـ mux للـ state 0 فعلياً على الـ GPIO. الـ `cached_state` في الـ kernel فضل بيقول `2`، فالـ core اعتقد إن مفيش داعي يستدعي `ops->set()` تاني — والنتيجة: الـ hardware والـ software out of sync.

#### الحل

**1. إجبار re-sync عند كل acquire:**

الحل الفوري: ضبط `idle_state` على `MUX_IDLE_DISCONNECT` في الـ DT:

```dts
mux: mux-controller {
    compatible = "gpio-mux";
    mux-gpios = <&gpio0 10 GPIO_ACTIVE_HIGH>,
                <&gpio0 11 GPIO_ACTIVE_HIGH>;
    idle-state = <MUX_IDLE_DISCONNECT>;  /* force re-set on each use */
    #mux-control-cells = <0>;
};
```

**2. Debug عشان تتأكد:**

```bash
# شوف الـ mux state الحالية
cat /sys/kernel/debug/mux/*/state 2>/dev/null

# أو عن طريق dynamic debug
echo "module gpio-mux +p" > /sys/kernel/debug/dynamic_debug/control
dmesg | grep mux
```

**3. في الـ driver، استخدم `mux_control_get_index()` للـ logging:**

```c
dev_dbg(&chip->dev, "setting mux[%u] to state %d",
        mux_control_get_index(mux), state);
```

#### الدرس المستفاد
الـ `cached_state` ده optimization داخلي في الـ mux core — **لا تعتمد على إن الـ hardware دايماً في sync معاه**. لو عندك hardware يمكن يتغير من برا (power glitch, EMI)، استخدم `MUX_IDLE_DISCONNECT` عشان تجبر الـ core يبعت الـ state في كل مرة.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI/CVBS Video MUX

#### العنوان
**Video output mux بيسبب kernel panic عند hot-plug**

#### السياق
منتج Android TV box على **Allwinner H616**. الـ board عندها مدخل video output واحد بيتوزع بين HDMI و CVBS عن طريق analog mux. الـ driver بيستخدم `devm_mux_chip_alloc()` و `devm_mux_chip_register()`.

#### المشكلة
لما المستخدم بيعمل hot-unplug للـ HDMI cable وهو في إيده Android شغّال، الـ kernel بيعمل panic بـ NULL pointer dereference في الـ mux driver.

#### التحليل
الـ panic بيحصل في الـ `set` callback:

```c
struct mux_chip {
    unsigned int controllers;
    struct device dev;        /* <-- الـ device embedded مش pointer */
    int id;
    const struct mux_control_ops *ops;
    struct mux_control mux[] __counted_by(controllers);
};
```

الـ مهندس كان بيستخدم `to_mux_chip()`:

```c
#define to_mux_chip(x) container_of((x), struct mux_chip, dev)
```

المشكلة: الـ HDMI hot-plug callback كان بيحاول يوصل لـ `mux_chip` عن طريق الـ `dev` pointer بعد ما الـ `devm` cleanup بدأت. الـ `devm_mux_chip_alloc` بيربط الـ lifetime بالـ `device`، لكن الـ HDMI subsystem كان عنده reference قديمة على الـ `mux_chip` مش محمية.

الـ `struct mux_control` بيحتوي على `struct semaphore lock` — الـ lock اتحذف قبل ما الـ HDMI callback يخلص.

#### الحل

**1. استخدم `get_device()` / `put_device()` صح:**

```c
/* في الـ probe */
chip = devm_mux_chip_alloc(dev, 1, 0);
/* احتفظ بـ reference زيادة للـ HDMI callback */
get_device(&chip->dev);
platform_set_drvdata(pdev, chip);

/* في الـ HDMI disconnect callback */
struct mux_chip *chip = platform_get_drvdata(pdev);
if (chip) {
    mux_control_deselect(mux);
    put_device(&chip->dev);  /* release the extra ref */
}
```

**2. تأكد إن `devm_mux_chip_register` اتنادى في الآخر:**

```c
/* الترتيب مهم جداً — register بعد ما كل setup خلص */
ret = devm_mux_chip_register(dev, chip);
if (ret)
    return ret;
```

**3. Debug:**

```bash
# تابع reference counting
echo 1 > /sys/bus/platform/drivers/h616-video-mux/unbind 2>&1
dmesg | tail -30
```

#### الدرس المستفاد
الـ `struct device dev` **embedded** جوا `mux_chip` مش pointer — ده معناه إن الـ `container_of` آمن، لكن لازم تحافظ على الـ reference count بشكل صريح لو في subsystems تانية بتمسك pointer على الـ chip.

---

### السيناريو 3: STM32MP1 Industrial Controller — I2C MUX لـ Multiple Sensor Boards

#### العنوان
**I2C address conflict على STM32MP1 بعد إضافة sensor board جديدة**

#### السياق
controller صناعي على **STM32MP157** بيتحكم في production line. الـ board عندها I2C bus واحدة بتخدم 4 sensor boards مختلفة، كل واحدة عليها نفس الـ I2C address (0x48 — TMP117 temperature sensor). الحل: استخدام **PCA9548A** I2C mux chip مع driver مبني على `mux/driver.h`.

#### المشكلة
بعد إضافة sensor board خامسة، الـ system بدأ يقرأ قيم خاطئة من sensors تانية. المهندس لاحظ إن بعض القراءات بتجي من الـ board الغلط.

#### التحليل
النظر في `mux_chip_alloc()`:

```c
struct mux_chip *mux_chip_alloc(struct device *dev,
                                unsigned int controllers,
                                size_t sizeof_priv);
```

وفي الـ `mux_chip`:

```c
struct mux_chip {
    unsigned int controllers;  /* عدد الـ mux controllers */
    struct device dev;
    int id;
    const struct mux_control_ops *ops;
    struct mux_control mux[] __counted_by(controllers);
};
```

المهندس اكتشف إن الـ PCA9548A driver اتعمله alloc بـ `controllers = 4` (الـ channels الأصلية)، ولما أضاف الـ channel الخامسة، عمل `controllers = 5` لكن نسي يعمل `mux_chip_unregister` و `mux_chip_free` للـ chip القديمة الأول. الـ old chip فضلت registered مع 4 controllers.

الأدهى: الـ `mux_chip_priv()`:

```c
static inline void *mux_chip_priv(struct mux_chip *mux_chip)
{
    return &mux_chip->mux[mux_chip->controllers];
    /* الـ private data بيبدأ بعد آخر mux controller */
}
```

الـ private data بتاع الـ old chip (الـ i2c_adapter pointers) اتكتب فوقها لأن الـ new chip allocated مع wrong size.

#### الحل

**1. الـ DT الصح للـ PCA9548A مع 5 channels:**

```dts
i2c-mux@70 {
    compatible = "nxp,pca9548";
    reg = <0x70>;
    #address-cells = <1>;
    #size-cells = <0>;

    i2c@0 { reg = <0>; /* sensor board 1 */ };
    i2c@1 { reg = <1>; /* sensor board 2 */ };
    i2c@2 { reg = <2>; /* sensor board 3 */ };
    i2c@3 { reg = <3>; /* sensor board 4 */ };
    i2c@4 { reg = <4>; /* sensor board 5 — الجديدة */ };
};
```

**2. في الـ driver، تحقق من الـ controllers count:**

```c
/* alloc بالعدد الصح من البداية */
chip = devm_mux_chip_alloc(dev, num_channels,
                           num_channels * sizeof(struct i2c_adapter *));
if (IS_ERR(chip))
    return PTR_ERR(chip);

/* الـ private area بعد mux[] array — آمن بس حسب sizeof_priv */
adapters = mux_chip_priv(chip);
```

**3. Debug:**

```bash
# شوف كل الـ registered mux chips
ls /sys/bus/mux/devices/
# شوف controllers لكل chip
cat /sys/bus/mux/devices/*/controllers 2>/dev/null
```

#### الدرس المستفاد
الـ `mux_chip_priv()` بترجع pointer على المنطقة بعد آخر `mux_control` في الـ flexible array — لو غلطت في `controllers` عند الـ alloc، الـ private data هيبقى في مكان غلط. **دايماً اعمل free للـ chip القديمة قبل ما تعمل alloc جديدة.**

---

### السيناريو 4: i.MX8M Mini Automotive ECU — USB MUX لـ Dual-Role Port

#### العنوان
**USB OTG port مش بيتبدل بين host وdevice mode في ECU سياراتي**

#### السياق
ECU سياراتي على **NXP i.MX8M Mini** — الـ board عندها USB port واحدة بتشتغل كـ device (للـ flashing/diagnostic) أو host (لتوصيل USB storage). الـ switching بيتم عبر GPIO-controlled USB mux، driver مبني على `mux/driver.h`.

#### المشكلة
في production, بعض الـ ECUs رافضة تتحول من device mode لـ host mode. الـ مشكلة intermittent — بتظهر في ~5% من الوحدات.

#### التحليل
الـ `mux_control` struct:

```c
struct mux_control {
    struct semaphore lock;   /* binary semaphore */
    struct mux_chip *chip;
    int cached_state;
    unsigned int states;
    int idle_state;
    ktime_t last_change;     /* <-- timestamp مهم هنا */
};
```

الـ `last_change` بيسجل آخر مرة اتغير فيها الـ mux. الـ USB mux chip (FSUSB42) محتاج minimum settling time بعد كل switch — الـ datasheet بيقول 100µs.

الـ مشكلة: الـ driver كان بيحسب الـ settling time غلط، كان بيقرأ `last_change` لكن بيحسب الـ elapsed time بشكل خاطئ على boards اللي عندها clock drift صغير.

التحقيق:

```bash
# على الـ ECU المتأثرة
cat /sys/kernel/debug/mux/*/last_change_ns 2>/dev/null
# أو لو مفيش debugfs entry
dmesg | grep -i "mux\|usb.*switch"
```

#### الحل

**1. في الـ `set` callback، احترم الـ settling time صح:**

```c
static int usb_mux_set(struct mux_control *mux, int state)
{
    struct usb_mux_priv *priv = mux_chip_priv(mux->chip);
    ktime_t now = ktime_get();
    s64 elapsed_us;

    /* استخدم last_change للـ rate limiting */
    elapsed_us = ktime_to_us(ktime_sub(now, mux->last_change));
    if (elapsed_us < USB_MUX_SETTLE_US)
        udelay(USB_MUX_SETTLE_US - elapsed_us);

    /* set GPIO */
    gpiod_set_value_cansleep(priv->sel_gpio, state);
    return 0;
}
```

**2. الـ DT للـ USB mux:**

```dts
usb-mux {
    compatible = "gpio-mux";
    mux-gpios = <&gpio3 5 GPIO_ACTIVE_HIGH>;
    idle-state = <0>;  /* default: device mode */
    #mux-control-cells = <0>;
};
```

**3. تحقق من الـ semaphore:**
الـ `struct semaphore lock` في `mux_control` بيحمي الـ state — تأكد إن الـ USB host driver والـ USB device driver مش بيعملوا concurrent access بدون `mux_control_select()` و `mux_control_deselect()`.

#### الدرس المستفاد
الـ `last_change` timestamp في `mux_control` ده **resource قيّم** — استخدمه لتطبيق hardware settling times بدل ما تحط `msleep()` ثابتة. ده بيخلي الـ driver أدق وأسرع.

---

### السيناريو 5: RK3562 Custom Board Bring-up — Multi-Channel ADC MUX

#### العنوان
**ADC mux driver مش بيتعرف على الـ channels الصح في bring-up جديد**

#### السياق
مهندس بيعمل bring-up لـ custom board على **Rockchip RK3562** — الـ board عندها ADC واحد (12-bit، 8 channels hardware) لكن محتاج يقيس 16 sensor. الحل: external CD74HC4067 analog mux بـ 16 channel. Driver جديد بيتكتب من الصفر باستخدام `mux/driver.h`.

#### المشكلة
أول ما شغّل الـ driver، الـ kernel بيطبع:

```
mux-gpio: invalid number of states (0)
mux-gpio: probe failed: -EINVAL
```

وبعض الـ channels بتقرأ نفس القيمة رغم إنها موصّلة لـ sensors مختلفة.

#### التحليل
المشكلة في تهيئة `mux_control`:

```c
struct mux_control {
    struct semaphore lock;
    struct mux_chip *chip;
    int cached_state;
    unsigned int states;    /* <-- لازم > 0 وصح */
    int idle_state;
    ...
};
```

الـ `states` لازم تتضبط **قبل** `mux_chip_register()`. الـ kernel بيتحقق منها في الـ register:

```c
/* من المشكلة الأولى — مهندس نسي يضبط states */
chip = devm_mux_chip_alloc(dev, 1, 0);
/* خطأ: نسي السطر ده */
chip->mux[0].states = 16;  /* CD74HC4067 has 16 states */
/* ثم */
devm_mux_chip_register(dev, chip);  /* فشل بـ -EINVAL */
```

المشكلة الثانية (نفس القيم): بعد ما صلّح الـ states، لاحظ إن channels 8-15 بتقرأ زي channels 0-7. السبب: نسي يضبط الـ `idle_state`:

```c
chip->mux[0].idle_state = MUX_IDLE_AS_IS;
/* أو */
chip->mux[0].idle_state = 0; /* channel 0 عند الـ idle */
```

لكن المشكلة الحقيقية: الـ GPIO bits كانت 3 فقط (لـ 8 states)، لكن الـ CD74HC4067 محتاج 4 GPIO bits (لـ 16 states).

#### الحل

**1. الـ DT الصح:**

```dts
adc-mux: mux-controller {
    compatible = "gpio-mux";
    /* 4 GPIO bits لـ 16 channels: 2^4 = 16 */
    mux-gpios = <&gpio1 0 GPIO_ACTIVE_HIGH>,
                <&gpio1 1 GPIO_ACTIVE_HIGH>,
                <&gpio1 2 GPIO_ACTIVE_HIGH>,
                <&gpio1 3 GPIO_ACTIVE_HIGH>;
    idle-state = <0>;
    #mux-control-cells = <0>;
};
```

**2. الـ driver الصح:**

```c
static int cd74hc4067_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct mux_chip *chip;

    /* alloc: controller واحد، مفيش private data */
    chip = devm_mux_chip_alloc(dev, 1, 0);
    if (IS_ERR(chip))
        return PTR_ERR(chip);

    chip->ops = &cd74hc4067_ops;

    /* ضبط states قبل register — هذا إلزامي */
    chip->mux[0].states = 16;
    chip->mux[0].idle_state = MUX_IDLE_AS_IS;

    return devm_mux_chip_register(dev, chip);
}
```

**3. Debug بعد الـ fix:**

```bash
# تأكد من الـ registration
ls /sys/bus/mux/devices/
# شوف الـ states
cat /sys/bus/mux/devices/mux-gpio.0/mux0/states
# expected: 16
```

**4. استخدم `mux_control_get_index()` في الـ ADC consumer driver:**

```c
/* للـ logging أثناء bring-up */
unsigned int idx = mux_control_get_index(mux);
dev_dbg(dev, "selecting ADC channel via mux[%u]", idx);
ret = mux_control_select(mux, channel);
```

#### الدرس المستفاد
الـ API واضح: الترتيب هو **alloc → set ops → set states → set idle_state → register**. أي خطوة ناقصة قبل `register` هتخلي الـ kernel يرفض الـ driver. الـ `states` بيحدد كام state صحيح — وعدد الـ GPIO bits في الـ DT لازم يكون `ceil(log2(states))` بالظبط.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ** LWN.net هو المرجع الأساسي لتتبع تطور الـ Linux kernel — الروابط دي حقيقية ومباشرة من نتائج البحث:

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| mux controller abstraction and iio/i2c muxes (النقاش الأصلي) | [lwn.net/Articles/716704](https://lwn.net/Articles/716704/) | النقاش الأول للـ subsystem قبل الإضافة |
| mux controller abstraction and iio/i2c muxes (patch series) | [lwn.net/Articles/722702](https://lwn.net/Articles/722702/) | الـ patch series النهائي قبل الـ merge |
| USB Type-C device-connection, mux and switch support | [lwn.net/Articles/749063](https://lwn.net/Articles/749063/) | استخدام الـ mux framework في USB Type-C |
| typec: mux: Introduce support for multiple USB TypeC muxes | [lwn.net/Articles/892421](https://lwn.net/Articles/892421/) | توسيع الـ framework لدعم multiple muxes |
| add gpio-line-mux | [lwn.net/Articles/1045004](https://lwn.net/Articles/1045004/) | إضافة gpio-line-mux حديثة |

---

### توثيق الـ Kernel الرسمي

**الـ** kernel documentation الرسمي موجود في `Documentation/` وعلى `docs.kernel.org`:

- **توثيق الـ mux controller bindings في Device Tree:**
  `Documentation/devicetree/bindings/mux/mux-controller.txt`
  → [kernel.org/doc/Documentation/devicetree/bindings/mux/mux-controller.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/mux/mux-controller.txt)

- **توثيق Intel PMC Mux (مثال تطبيقي):**
  `Documentation/firmware-guide/acpi/intel-pmc-mux.rst`
  → [docs.kernel.org/firmware-guide/acpi/intel-pmc-mux.html](https://docs.kernel.org/firmware-guide/acpi/intel-pmc-mux.html)

- **توثيق I2C topology (استخدام الـ mux مع I2C):**
  `Documentation/i2c/i2c-topology.rst`
  → [docs.kernel.org/i2c/i2c-topology.html](https://docs.kernel.org/i2c/i2c-topology.html)

- **الكود المصدري للـ drivers:**
  → [github.com/torvalds/linux/tree/master/drivers/mux](https://github.com/torvalds/linux/tree/master/drivers/mux)

- **الـ header الرئيسي للـ consumer API:**
  `include/linux/mux/consumer.h`

- **الـ header الرئيسي للـ driver API (الملف الحالي):**
  `include/linux/mux/driver.h`

---

### Commits المهمة في تاريخ الـ Subsystem

**الـ** subsystem اتضاف في Linux 4.12 بواسطة **Peter Rosin** من Axentia Technologies:

- **الـ patch series الأصلي (v9)** — أول إضافة للـ mux subsystem:
  [lore.kernel.org — PATCH v9 03/10: mux: minimal mux subsystem](https://lore.kernel.org/lkml/1486568517-6997-4-git-send-email-peda@axentia.se/)

- **الـ PULL REQUEST لـ 4.12:**
  [lkml.iu.edu — PULL REQUEST mux for 4.12](https://lkml.iu.edu/hypermail/linux/kernel/1703.3/01529.html)

- **تفاصيل الـ merge في Phoronix:**
  [phoronix.com — Mux Controller Subsystem Proposed For Linux 4.12](https://www.phoronix.com/news/Mux-Proposed-Linux-4.12)

---

### نقاشات الـ Mailing List

**الـ** mailing list هو المكان اللي اتناقش فيه الـ design قبل الـ merge:

- **النقاش الأصلي على LKML (lore.kernel.org):**
  [lore.kernel.org — mux subsystem patch series](https://lore.kernel.org/lkml/1486568517-6997-4-git-send-email-peda@axentia.se/)

- **نقاش إضافة DT enable state:**
  [lore.kernel.org — Add support for reading mux enable state from DT](https://lore.kernel.org/linux-devicetree/69f73f64-6424-4e3f-9068-195e959b9762@axentia.se/)

- **أرشيف LKML العام:**
  [lkml.org](https://lkml.org/) — ابحث عن `mux controller Peter Rosin`

---

### kernelnewbies.org — تغييرات الـ Kernel حسب الإصدار

**الـ** kernelnewbies.org بيوثق التغييرات في كل kernel version — الإصدارات دي فيها تغييرات مرتبطة بالـ mux subsystem:

| الإصدار | الرابط | التغيير المرتبط |
|---------|--------|----------------|
| Linux 4.12 | [kernelnewbies.org/Linux_4.12](https://kernelnewbies.org/Linux_4.12) | إضافة الـ mux controller subsystem نفسه |
| Linux 4.17 | [kernelnewbies.org/Linux_4.17](https://kernelnewbies.org/Linux_4.17) | USB Type-C mux and switch support |
| Linux 4.19 | [kernelnewbies.org/Linux_4.19](https://kernelnewbies.org/Linux_4.19) | I2S clock mux driver |
| Linux 5.10 | [kernelnewbies.org/Linux_5.10](https://kernelnewbies.org/Linux_5.10) | intel_pmc_mux device role support |
| Linux 6.13 | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) | simple-mux idle-state support |

---

### elinux.org — موارد الـ Embedded Linux

**الـ** elinux.org بيغطي الـ pin mux في سياق الـ embedded boards:

- **Device Trees والـ pin mux على BeagleBone:**
  [elinux.org/EBC_Device_Trees](https://elinux.org/EBC_Device_Trees)
  → بيشرح إزاي كل pin خارجي بيتوصل بـ multiplexer يقدر يتوجّه لـ 8 internal pins مختلفة

- **BeagleBoard Zippy — pin mux للـ expansion boards:**
  [elinux.org/BeagleBoard_Zippy](https://elinux.org/BeagleBoard_Zippy)

---

### الكتب الموصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول ذات الصلة:**
  - Chapter 14: The Linux Device Model — بيشرح `struct device` اللي بيورث منها `mux_chip`
  - Chapter 3: Char Drivers — لفهم الـ device registration pattern
- **رابط مجاني:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول ذات الصلة:**
  - Chapter 17: Devices and Modules — الـ device model والـ bus/driver/device relationship
  - Chapter 5: System Calls — فهم الـ kernel/userspace boundary
- **مناسب لـ:** فهم الـ semaphore والـ locking اللي بيستخدمه `mux_control`

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصول ذات الصلة:**
  - Chapter 16: Kernel Debugging Techniques
  - Chapter 6: System Initialization — بيشرح الـ Device Tree وعلاقته بالـ mux bindings
- **مناسب لـ:** فهم الـ `idle_state` وإزاي الـ DT bindings بتشتغل مع الـ mux subsystem

---

### مصطلحات البحث الموصى بيها

لو عايز تلاقي معلومات أكتر، استخدم المصطلحات دي:

```
# على LKML / lore.kernel.org
"mux controller" "Peter Rosin" linux kernel
"struct mux_chip" linux kernel driver
"mux_chip_alloc" OR "mux_chip_register" linux kernel

# على Google / DuckDuckGo
linux kernel "drivers/mux" subsystem
linux "include/linux/mux/driver.h" explanation
linux mux controller consumer driver howto

# على GitHub (في kernel source)
path:drivers/mux language:C
path:include/linux/mux language:C

# على docs.kernel.org
mux controller kernel documentation
```

---

### ملخص سريع للمصادر الأهم

| الأولوية | المصدر | ليه مهم |
|----------|--------|---------|
| ⭐⭐⭐ | [LWN 716704](https://lwn.net/Articles/716704/) | النقاش الأصلي للتصميم |
| ⭐⭐⭐ | [lore.kernel.org patch v9](https://lore.kernel.org/lkml/1486568517-6997-4-git-send-email-peda@axentia.se/) | الكود الأصلي والـ rationale |
| ⭐⭐⭐ | [kernel DT binding docs](https://www.kernel.org/doc/Documentation/devicetree/bindings/mux/mux-controller.txt) | كيفية استخدام الـ DT |
| ⭐⭐ | [github drivers/mux](https://github.com/torvalds/linux/tree/master/drivers/mux) | الكود الحالي للـ drivers |
| ⭐⭐ | [Phoronix 4.12 news](https://www.phoronix.com/news/Mux-Proposed-Linux-4.12) | سياق تاريخي مبسط |
| ⭐ | [elinux EBC Device Trees](https://elinux.org/EBC_Device_Trees) | أمثلة عملية على embedded |
## Phase 8: Writing simple module

### الهدف

هنعمل kernel module بيعمل **kprobe** على الدالة `mux_chip_register` — اللي بتتسجل كل مرة driver بيضيف mux chip جديد للنظام. ده بيخلينا نشوف مين بيسجل chips، وكام controller فيها، وإيه الـ device name.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * mux_register_probe.c
 * Hooks mux_chip_register() via kprobe to log every mux chip registration.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* kprobe API */
#include <linux/mux/driver.h>  /* struct mux_chip */

/* --- kprobe callback: fires just before mux_chip_register() executes --- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument lands in rdi.
     * On arm64 it's x0. We use regs_get_kernel_argument() for portability.
     */
    struct mux_chip *chip = (struct mux_chip *)regs_get_kernel_argument(regs, 0);

    if (!chip)
        return 0;

    pr_info("mux_probe: registering chip id=%d controllers=%u device=%s\n",
            chip->id,
            chip->controllers,
            dev_name(&chip->dev));   /* device name set before register */

    return 0; /* non-zero would abort the probed function — we never do that */
}

/* --- post handler: fires after mux_chip_register() returns --- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* regs->ax (x86) / regs->regs[0] (arm64) = return value */
    long ret = regs_return_value(regs);

    pr_info("mux_probe: mux_chip_register returned %ld\n", ret);
}

/* --- kprobe descriptor --- */
static struct kprobe kp = {
    .symbol_name = "mux_chip_register", /* function to hook by name */
    .pre_handler  = handler_pre,        /* called before the function */
    .post_handler = handler_post,       /* called after the function */
};

/* --- init: register the kprobe --- */
static int __init mux_probe_init(void)
{
    int ret = register_kprobe(&kp);

    if (ret < 0) {
        pr_err("mux_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("mux_probe: planted kprobe on %s at %p\n",
            kp.symbol_name, kp.addr); /* addr filled in by kernel after register */
    return 0;
}

/* --- exit: unregister the kprobe before the module is removed --- */
static void __exit mux_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("mux_probe: kprobe removed\n");
}

module_init(mux_probe_init);
module_exit(mux_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on mux_chip_register to log mux chip registrations");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | لازم لأي kernel module — بيعرّف `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kernel.h` | بيجيب `pr_info` / `pr_err` للـ logging |
| `linux/kprobes.h` | الـ API الأساسية: `struct kprobe`, `register_kprobe`, `regs_get_kernel_argument` |
| `linux/mux/driver.h` | عشان نعرف `struct mux_chip` ونقدر نقرأ `id`, `controllers`, `dev` |

---

#### الـ `handler_pre`

ده الـ callback اللي بيتنفذ **قبل** ما `mux_chip_register` تشتغل. الـ kernel بيديه `struct pt_regs` — ده snapshot من الـ CPU registers لحظة الاستدعاء. بنستخدم `regs_get_kernel_argument(regs, 0)` عشان نجيب أول argument (الـ `mux_chip *`) بطريقة portable بين x86 و arm64.

بنطبع الـ `id` و `controllers` و اسم الـ device عشان نعرف إيه الـ chip اللي اتسجلت.

---

#### الـ `handler_post`

بيتنفذ **بعد** ما الدالة تكمل وترجع. بنستخدم `regs_return_value(regs)` عشان نجيب الـ return value (صفر = نجاح، سالب = error). ده مفيد عشان نعرف لو الـ registration فشلت ليه.

---

#### الـ `struct kprobe`

الحقل `symbol_name` بيخلي الـ kernel يبحث عن الدالة في الـ kallsyms ويحط breakpoint تلقائي. الـ `addr` بيتملى بعد `register_kprobe` — لو الـ symbol مش موجود أو mux driver مش مكمبايل في الـ kernel هيرجع error.

---

#### الـ `mux_probe_init`

بيسجل الـ kprobe. لو فشل (مثلاً الـ symbol مش exported أو CONFIG_KPROBES=n) بيطبع error ويرجع الـ error code عشان `insmod` يفشل بـ message واضحة.

---

#### الـ `mux_probe_exit`

**لازم** نعمل `unregister_kprobe` قبل ما يتشال الـ module — لو متعملتش، الـ kernel هيحاول يستدعي callback موجودة في memory اتحررت وده kernel panic مضمون.

---

### Makefile

```makefile
obj-m += mux_register_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ module

```bash
# بناء الـ module
make

# تحميله
sudo insmod mux_register_probe.ko

# شوف اللوج — أي mux driver بيتحمل هيظهر هنا
sudo dmesg | grep mux_probe

# تفريغه
sudo rmmod mux_register_probe

# لو عايز تتحقق إن الـ kprobe اشتغل
sudo cat /sys/kernel/debug/kprobes/list
```

---

### مثال على الـ output

```
[  142.831204] mux_probe: planted kprobe on mux_chip_register at ffffffffc08a1230
[  143.002917] mux_probe: registering chip id=0 controllers=4 device=gpio-mux.0
[  143.002931] mux_probe: mux_chip_register returned 0
```

الـ `id=0` يعني أول chip، `controllers=4` يعني 4 mux lines على نفس الـ chip، واسم الـ device جاي من الـ platform bus.
