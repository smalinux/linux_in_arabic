## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

**الـ `spi-mux.c`** ينتمي لـ subsystemين اتنين بيشتغلوا مع بعض:

| Subsystem | Maintainer | المسار |
|---|---|---|
| **SPI SUBSYSTEM** | Mark Brown | `drivers/spi/` + `include/linux/spi/` |
| **MULTIPLEXER SUBSYSTEM** | Peter Rosin | `drivers/mux/` + `include/linux/mux/` |

الملف نفسه جوه `drivers/spi/` — يعني هو driver SPI بيستخدم خدمات الـ MUX subsystem.

---

### المشكلة اللي بيحلها الملف — القصة الكاملة

تخيل عندك **SoC** (شريحة معالج) فيها controller SPI واحد بس معاه **4 chip selects** — يعني يقدر يتكلم مع 4 أجهزة بس في نفس الوقت (مش نفس الوقت الحرفي، بالتناوب).

بس انت محتاج تحط **8 أجهزة SPI** على نفس الـ board: flash, sensor, ADC, display, وغيرهم.

**الحل التقليدي في الهاردوير:** تحط **MUX chip** — دايرة صغيرة وظيفتها إنها "توزّع" الـ chip select الواحد على أكتر من جهاز. الـ SoC يقول للـ MUX "وصّلني بالجهاز رقم 3" — الـ MUX يقطع الاتصال بالباقيين ويوصّل الجهاز ده بس.

```
MOSI ─────────────────┬──────┬──────┬──────┐
MISO ─────────────────┼──────┼──────┼──────┤
SCK  ─────────────────┼──────┼──────┼──────┤
                       │      │      │      │
 ┌──────────┐        dev0   dev1   dev2   dev3
 │ SPI Ctrl │
 │          │ CS-X   ┌─────┐
 │  CS-X ───┼───────►│ MUX │──CS-0──► dev0
 │          │        │     │──CS-1──► dev1
 │  GPIO ───┼───────►│     │──CS-2──► dev2
 └──────────┘  sel   └─────┘──CS-3──► dev3
```

**المشكلة:** الـ Linux kernel شايف controller SPI واحد بس معاه CS واحد. مش عارف إن في MUX وورا المUX في 4 أجهزة.

**الحل اللي بيعمله `spi-mux.c`:** بيعمل **virtual SPI controller** جديد خالص في الـ kernel — الـ kernel يشوف controller تاني معاه 4 chip selects (عدد states الـ MUX). الأجهزة بتتسجل تحت الـ virtual controller ده، وبتتعامل مع الدراير بتاعها عادي جداً. الـ `spi-mux.c` هو اللي بيتوسط ويترجم: قبل ما يبعت أي message، يضبط الـ MUX على الـ channel الصح، يبعت الـ message، وبعدين يفك الـ MUX.

---

### الـ Big Picture بأسلوب ELI5

تخيل عندك **تليفون واحد** في البيت (SPI controller) وعندك 8 غرف (8 أجهزة). التليفون عنده خط واحد بس يتكلم عليه مع غرفة واحدة في وقت واحد.

الحل: تحط **سنترال صغير** (MUX chip). السنترال يقبل خط التليفون الواحد ده ويوزّعه على 8 غرف. بس الـ OS مش عارف بالسنترال ده.

**الـ `spi-mux.c`** هو "الأوفيس بوي" اللي بيقف قدام السنترال:
1. الـ OS بيقوله "كلّم غرفة 5"
2. هو يضبط السنترال على غرفة 5
3. يمرر المكالمة
4. لما تخلص، يقفل السنترال

الـ OS شايف إن في **تليفون تاني** معاه 8 خطوط — مش شايف السنترال خالص.

---

### ليه ده مفيد؟

- **توسيع عدد الأجهزة:** بتتخطى حد الـ chip selects الموجودة في الهاردوير.
- **لا تعديل في الـ device drivers:** الـ flash driver أو الـ sensor driver مش محتاج يعرف إن في MUX — بيشتغل عادي.
- **عام ومرن:** يشتغل مع أي نوع MUX (GPIO-based, register-based, إلخ) طالما فيه mux-control driver ليه.
- **حل hardware حقيقي بـ software شفاف:** الـ MUX chip موجود فعلاً في الـ board، الـ software بس بيديره بشكل transparent.

---

### ازاي يشتغل — الخطوات الأساسية

```
Probe time:
───────────
spi_mux_probe()
  ├── spi_alloc_host()          ← ينشئ virtual SPI controller
  ├── devm_mux_control_get()   ← يمسك handle على الـ MUX chip
  ├── ctlr->num_chipselect = mux_control_states()  ← عدد CS = عدد states الـ MUX
  └── devm_spi_register_controller()  ← يسجّل الـ virtual controller في الـ kernel

Transfer time:
──────────────
spi_mux_transfer_one_message()
  ├── spi_mux_select()
  │     ├── mux_control_select()   ← يضبط الـ MUX على الـ CS المطلوب
  │     └── spi_setup()            ← ينسخ إعدادات الجهاز (speed, mode, bits)
  ├── يحفظ الـ complete callback الأصلي
  ├── يحط callback بتاعه بدلاً منه
  └── spi_async()                  ← يبعت الـ message للـ parent controller

Completion:
───────────
spi_mux_complete_cb()
  ├── يرجّع الـ complete callback الأصلي للـ message
  ├── spi_finalize_current_message()
  └── mux_control_deselect()       ← يفك الـ MUX
```

---

### نقطة مهمة: الـ `current_cs` Cache

الـ driver بيحتفظ بـ `current_cs` — آخر chip select اتضبط. لو جاه request لنفس الـ CS، مش بيعمل `spi_setup()` تاني. ده optimization صغير بيوفر وقت.

```c
#define SPI_MUX_NO_CS  ((unsigned int)-1)  /* initial value — no CS selected yet */

if (priv->current_cs == spi_get_chipselect(spi, 0))
    return 0;  /* same CS, skip spi_setup() */
```

---

### الـ Lockdep Subclass — ليه مهم؟

```c
lockdep_set_subclass(&ctlr->io_mutex, 1);
lockdep_set_subclass(&ctlr->add_lock, 1);
```

الـ virtual controller بيشتغل **فوق** parent controller. الـ lock بتاع الـ virtual controller بيتاخد وهو الـ parent lock شايل نفسه. الـ lockdep بيشوف ده ويظن إن في **deadlock** — مع إنه مش كده. الـ subclass بيقول للـ lockdep "ده lock من نوع تاني، مش نفس اللي فوقه" فيسكت.

---

### الفرق بين `spi_setup()` و `spi_mux_setup()`

| | `spi_mux_setup()` | `spi_mux_select()` |
|---|---|---|
| امتى بيتكلم | عند registration الجهاز | قبل كل transfer |
| بيعمل إيه | بيعمل `spi_setup()` على الـ parent بإعدادات أي كانت | بينسخ إعدادات الجهاز الحالي وبعدين `spi_setup()` |
| ليه كده | لازم يعمل setup عشان الـ kernel يقبل الجهاز | الإعدادات الحقيقية بتتضبط وقت الـ transfer |

---

### الملفات المكوّنة للـ Subsystem

#### SPI Core
| الملف | الدور |
|---|---|
| `drivers/spi/spi.c` | قلب الـ SPI subsystem — الـ core |
| `include/linux/spi/spi.h` | تعريفات `spi_device`, `spi_controller`, `spi_message` |
| `drivers/spi/spi-mux.c` | **الملف الحالي** — virtual SPI controller فوق MUX |

#### MUX Subsystem
| الملف | الدور |
|---|---|
| `drivers/mux/core.c` | قلب الـ MUX subsystem |
| `include/linux/mux/consumer.h` | API للي بيستخدم الـ MUX (`mux_control_select`, `deselect`) |
| `drivers/mux/gpio.c` | MUX driver بيتحكم بـ GPIO pins |
| `drivers/mux/mmio.c` | MUX driver بيتحكم بـ memory-mapped registers |
| `drivers/mux/adg792a.c` | MUX driver لشريحة ADG792A |

#### Device Tree Binding
| الملف | الدور |
|---|---|
| `Documentation/devicetree/bindings/spi/spi-mux.yaml` | Schema الـ DT للـ spi-mux node |

#### ملفات لازم تعرفها
- **`drivers/spi/spi-bitbang.c`** — مثال على SPI controller driver بسيط للفهم
- **`drivers/i2c/muxes/i2c-mux.c`** — نفس الفكرة بالظبط لكن على I2C بدل SPI، تقدر تقارن
## Phase 2: شرح الـ SPI MUX Framework

### المشكلة اللي بيحلها الـ Driver ده

في الـ embedded systems، الـ SPI controller في الـ SoC بيجي بعدد محدود من الـ **chip select (CS)** lines — عادةً 2 أو 4. لو عندك board فيها 8 أجهزة SPI أو أكتر، هتلاقي نفسك في مشكلة: مفيش CS كفاية.

الحل الـ hardware هو إنك تحط **multiplexer** (زي 74HC4051 أو أي MUX chip) بين الـ SoC والأجهزة. الـ MUX بيأخد CS واحد من الـ SoC وبيختار منه واحد من عدة outputs. لكن الـ kernel لازم يعرف بذلك — لازم يتحكم في الـ MUX قبل ما يبعت أي transfer.

**الـ `spi-mux.c`** هو الحل الـ software لده: بيعمل **virtual SPI controller** فوق الـ real controller، وبيستخدم الـ **MUX subsystem** عشان يتحكم في انتخاب الـ CS.

---

### المشكلة بالتفصيل

```
بدون MUX driver:

  SoC SPI Controller
  (CS0, CS1 فقط)
       |
   +---+---+
   |       |
  Dev0    Dev1       ← فقط جهازين، مش كفاية
```

```
مع MUX hardware لكن بدون MUX driver:

  SoC SPI Controller
  (CS0 → MUX input)
  (GPIO0, GPIO1, GPIO2 → MUX select lines)
         |
       [MUX]
      / | \ \
  D0  D1  D2  D3    ← 4 أجهزة، لكن الـ kernel مش عارف يتعامل معاها كـ SPI devices
```

المشكلة إن الـ SPI subsystem بيتكلم مع `spi_device` مرتبط بـ `spi_controller`، ومفيش آلية تلقائية تقول للـ SPI core "أنا عايز أتكلم مع device رقم 3 اللي خلف الـ MUX".

---

### الحل: Virtual Controller Layer

**الـ `spi-mux` driver** بيعمل **`spi_controller` جديد** ظاهري. الـ devices اللي وراء الـ MUX بيتسجلوا تحت الـ virtual controller ده، وكل device بيعتقد إنه بيتكلم مع controller حقيقي. لما يجي transfer، الـ virtual controller بيعمل 3 خطوات:

1. بيختار الـ MUX state المناسب (عن طريق الـ MUX subsystem)
2. بيعمل setup للـ parent SPI device بإعدادات الـ child
3. بيعمل `spi_async` على الـ parent SPI controller الحقيقي

---

### Big Picture Architecture

```
  ┌──────────────────────────────────────────────────────┐
  │                  User/Driver Space                    │
  │   flash_driver   sensor_driver   adc_driver  ...      │
  └───────┬───────────────┬──────────────┬───────────────┘
          │               │              │
          ▼               ▼              ▼
  ┌──────────────────────────────────────────────────────┐
  │              SPI Core (kernel/drivers/spi/spi.c)      │
  │  spi_async() → queues spi_message → transfer_one_msg  │
  └──────────────────────┬───────────────────────────────┘
                         │
          ┌──────────────▼──────────────────┐
          │   Virtual SPI Controller         │
          │   (created by spi-mux.c)         │
          │                                  │
          │  .transfer_one_message =         │
          │      spi_mux_transfer_one_message│
          │  .setup = spi_mux_setup          │
          │  .num_chipselect = N (from MUX)  │
          └──────────────┬──────────────────┘
                         │
          ┌──────────────▼──────────────────┐
          │        spi_mux_priv              │
          │  ┌─────────────────────────┐     │
          │  │ *spi  → parent spi_dev  │     │
          │  │ *mux  → mux_control     │     │
          │  │ current_cs              │     │
          │  │ child_msg_complete (cb) │     │
          │  └─────────────────────────┘     │
          └──────┬───────────────┬───────────┘
                 │               │
                 ▼               ▼
  ┌──────────────────┐   ┌──────────────────────────┐
  │  MUX Subsystem   │   │  Parent SPI Controller   │
  │ (drivers/mux/)   │   │  (Real HW: SPI0 on SoC)  │
  │                  │   │                          │
  │ mux_control_     │   │  spi_async()             │
  │   select()       │   │  spi_setup()             │
  │ mux_control_     │   │  .transfer_one_message   │
  │   deselect()     │   │   (real HW driver)       │
  └────────┬─────────┘   └───────────────┬──────────┘
           │                             │
           ▼                             ▼
  ┌─────────────────┐          ┌─────────────────────┐
  │   MUX Hardware  │          │   SPI Bus (MOSI,    │
  │  (74HC4051 etc) │          │   MISO, SCK)        │
  │                 │          │                     │
  │  Select lines:  │          └──────────┬──────────┘
  │  A0, A1, A2 ───┼──────────►  [MUX output lines] │
  └─────────────────┘          │                     │
                                D0  D1  D2  D3  D4  D5
                               SPI devices behind MUX
```

---

### مثال حقيقي: Board فيها 6 SPI Flash Chips

```
  STM32MP1 SoC
  SPI1: CS0 → MUX input
  GPIO PA0, PA1, PA2 → MUX select lines (3 bits = 8 states)

  Device Tree:
  spi@... {                          ← parent SPI controller (real HW)
      spi-mux@0 {                    ← MUX device, هو نفسه spi_device على parent
          compatible = "spi-mux";
          mux-controls = <&mux0>;
          #address-cells = <1>;

          flash0@0 { ... };          ← virtual CS 0
          flash1@1 { ... };          ← virtual CS 1
          flash2@2 { ... };          ← virtual CS 2
          sensor@3 { ... };          ← virtual CS 3
      };
  };
```

الـ kernel يشوف `spi-mux@0` → يروح لـ `spi_mux_probe()` → بيعمل virtual `spi_controller` → الـ flash drivers بيشوفوا controller عادي وملهمش علم بالـ MUX.

---

### Analogy: موظف الاستقبال في شركة

تخيل شركة كبيرة (الـ SoC) عندها خط تليفون واحد للخارج (الـ SPI bus). عندها موظف استقبال (الـ parent SPI controller). في الشركة في 8 أقسام (SPI devices).

- **الـ MUX hardware**: زي لوحة تحويل المكالمات (PBX)
- **الـ `spi-mux` virtual controller**: زي مشرف الأقسام اللي بيبعت لك رقم داخلي
- **الـ `mux_control_select()`**: زي إنك تضغط رقم القسم على لوحة التحويل قبل ما تتكلم
- **الـ `spi_async(priv->spi, m)`**: زي إنك تعدي المكالمة للموظف الحقيقي (parent controller)
- **الـ `spi_mux_complete_cb()`**: لما الكلام ينتهي، الـ PBX بيفصل الخط ويرد للـ queue

لكن من وجهة نظر أي قسم: هو بيتكلم مع "خط تليفون مباشر" — ملوش علم بالـ PBX أو لوحة التحويل.

التحليل التفصيلي للـ analogy:

| عنصر في الـ analogy | المقابل في الكود |
|---|---|
| لوحة التحويل (PBX) | `struct mux_control *mux` |
| تحديد رقم القسم | `mux_control_select(priv->mux, spi_get_chipselect(spi, 0))` |
| تمرير المكالمة | `spi_async(priv->spi, m)` |
| إشعار انتهاء المكالمة | `spi_mux_complete_cb()` + `mux_control_deselect()` |
| خط تليفون وهمي للقسم | `spi_controller` الـ virtual المُعمل في `spi_mux_probe()` |
| المشرف اللي بيحفظ سياق المكالمة | `priv->child_msg_complete`, `priv->child_msg_context` |

---

### Core Abstraction: الـ Virtual SPI Controller

الفكرة المحورية في الـ driver هي **تجريد الـ MUX كـ SPI controller إضافي**:

```c
/* في spi_mux_probe() */

// 1. اخلق controller وهمي
ctlr = spi_alloc_host(&spi->dev, sizeof(*priv));

// 2. ربط الـ operations
ctlr->transfer_one_message = spi_mux_transfer_one_message;
ctlr->setup                = spi_mux_setup;

// 3. عدد الـ CS = عدد states في الـ MUX
ctlr->num_chipselect = mux_control_states(priv->mux);

// 4. سجّل الـ controller الوهمي في الـ SPI core
devm_spi_register_controller(&spi->dev, ctlr);
```

النتيجة: أي driver (flash, sensor, ADC) بيتكلم مع الـ virtual controller ده بنفس الـ API العادي `spi_sync()` / `spi_async()`، ومش محتاج يعرف أي حاجة عن الـ MUX.

---

### Data Structures وعلاقتها ببعض

```
  struct spi_mux_priv
  ┌─────────────────────────────────────────────┐
  │  *spi ──────────────────► struct spi_device  │
  │                           (parent device     │
  │                            على الـ real bus) │
  │                                              │
  │  *mux ──────────────────► struct mux_control │
  │                           (MUX subsystem)    │
  │                                              │
  │  current_cs               آخر CS متاخد       │
  │                                              │
  │  child_msg_complete ────► original callback  │
  │  child_msg_context  ────► original context   │
  │  child_msg_dev      ────► original spi_device│
  └─────────────────────────────────────────────┘
         ▲
         │  spi_controller_get_devdata(ctlr)
         │
  struct spi_controller   (virtual, created by spi-mux)
  ┌─────────────────────────────────────────────┐
  │  .transfer_one_message = spi_mux_transfer.. │
  │  .setup                = spi_mux_setup      │
  │  .num_chipselect       = N                  │
  │  .must_async           = true               │
  │  .defer_optimize_msg   = true               │
  │  .bus_num              = -1 (dynamic)       │
  └─────────────────────────────────────────────┘
         │
         │  يُرى من الـ child devices كـ:
         ▼
  struct spi_device (child: flash@0, sensor@1, ...)
  ┌─────────────────────────────────────────────┐
  │  .controller ──────────► virtual ctlr above │
  │  .chip_select[0]        = 0, 1, 2, ...      │
  │  .max_speed_hz          = device-specific   │
  │  .mode                  = device-specific   │
  └─────────────────────────────────────────────┘
```

---

### Transfer Flow: خطوة بخطوة

```
Child driver يعمل spi_async(child_spi_dev, msg)
         │
         ▼
SPI core يروح لـ virtual controller
         │
         ▼
spi_mux_transfer_one_message(ctlr, msg)
         │
         ├─► spi_mux_select(spi)
         │       │
         │       ├─► mux_control_select(priv->mux, cs)
         │       │   (يأخد lock على الـ MUX — exclusive access)
         │       │
         │       └─► لو cs اتغير:
         │               priv->spi->max_speed_hz = spi->max_speed_hz
         │               priv->spi->mode         = spi->mode
         │               priv->spi->bits_per_word= spi->bits_per_word
         │               spi_setup(priv->spi)    (يحدّث parent settings)
         │
         ├─► بيحتفظ بالـ callback الأصلي:
         │       priv->child_msg_complete = msg->complete
         │       priv->child_msg_context  = msg->context
         │       priv->child_msg_dev      = msg->spi
         │
         ├─► بيبدل الـ callback بتاعه:
         │       msg->complete = spi_mux_complete_cb
         │       msg->context  = priv
         │       msg->spi      = priv->spi    (parent device)
         │
         └─► spi_async(priv->spi, msg)
                 │
                 ▼
             Parent real HW controller يعمل التحويل
                 │
                 ▼
             spi_mux_complete_cb() يتنادى
                 │
                 ├─► يرجع الـ msg للـ child state
                 │   (complete, context, spi_device)
                 │
                 ├─► spi_finalize_current_message(ctlr)
                 │   (يخبر الـ virtual ctlr إن الـ msg خلص)
                 │
                 └─► mux_control_deselect(priv->mux)
                     (يحرر الـ MUX lock)
```

---

### Callback Hijacking Pattern

ده pattern مهم جداً في الكود ولازم تفهمه:

الـ `spi_message` فيه:
```c
struct spi_message {
    void (*complete)(void *context);  // callback لما الـ transfer يخلص
    void *context;                    // بيتبعت للـ complete()
    struct spi_device *spi;           // الجهاز المرتبط بالـ message
    ...
};
```

الـ virtual controller بيعمل **callback interception**:

```c
/* احفظ الـ original */
priv->child_msg_complete = m->complete;
priv->child_msg_context  = m->context;
priv->child_msg_dev      = m->spi;

/* حط callback بتاعك */
m->complete = spi_mux_complete_cb;
m->context  = priv;
m->spi      = priv->spi;   /* غيّر الـ spi_device للـ parent */
```

لما الـ parent controller يخلص، بيكال `spi_mux_complete_cb` اللي:
1. يرجع الـ msg لحالته الأصلية (child's complete, context, spi)
2. يكال `spi_finalize_current_message` على الـ virtual controller
3. يحرر الـ MUX

ده بيضمن إن الـ child driver يشوف الـ callback بتاعه زي ما هو، وكأن المعالجة اتعملت مباشرة.

---

### الـ MUX Subsystem

**الـ MUX subsystem** (`drivers/mux/`) هو framework مستقل بيدير أي hardware multiplexer. هو **مش جزء من SPI** — هو generic. بيستخدمه SPI مux وI2C mux وغيرهم.

الـ API الأساسي من `include/linux/mux/consumer.h`:

```c
// اختار state معينة في الـ MUX (بيأخد mutex داخلياً)
int mux_control_select(struct mux_control *mux, unsigned int state);

// حرر الـ MUX (يرجع الـ mutex)
int mux_control_deselect(struct mux_control *mux);

// عدد الـ states المتاحة (= عدد الـ outputs)
unsigned int mux_control_states(struct mux_control *mux);

// احصل على handle للـ MUX من الـ device tree
struct mux_control *devm_mux_control_get(struct device *dev, const char *name);
```

المهم: `mux_control_select()` بيأخد **mutex** — يعني لو في transfer شغال على الـ MUX، أي transfer تاني هيبان متأخر لحد ما الأول ينتهي. ده بيضمن الـ exclusive access للـ MUX.

---

### Lockdep Subclass

```c
lockdep_set_subclass(&ctlr->io_mutex, 1);
lockdep_set_subclass(&ctlr->add_lock, 1);
```

الـ virtual controller بيأخد `io_mutex` بينما الـ parent controller شايل `io_mutex` بتاعه بردو. لو كلهم من نفس الـ subclass، الـ lockdep هيصرخ "potential deadlock!" — لأنه بيشوف نفس النوع من الـ lock بيتأخد مرتين في نفس الـ thread.

الحل: الـ virtual controller's locks بتتحط في **subclass 1** بدل الـ default (0). ده بيقول للـ lockdep: "أنا عارف إن في nesting هنا، وده مقصود".

---

### ما بيديره الـ Driver vs. ما بيفوّضه

| الـ `spi-mux` driver بيتولاه | بيفوّضه لـ |
|---|---|
| إنشاء الـ virtual `spi_controller` | الـ SPI core: تسجيل الـ controller، إدارة الـ queue |
| اختيار الـ MUX state قبل الـ transfer | الـ MUX subsystem: التحكم الفعلي في الـ hardware |
| نسخ إعدادات الـ child (speed, mode, bpw) للـ parent | الـ parent SPI driver: الـ timing الفعلي على الـ bus |
| حفظ واستعادة الـ callback الأصلي | الـ SPI core: إدارة الـ `spi_message` lifecycle |
| تحرير الـ MUX بعد انتهاء الـ transfer | الـ MUX subsystem: mutex management |
| تحديد عدد الـ CS من عدد الـ MUX states | الـ SPI core: تخصيص الـ CS numbers للـ child devices |

---

### must_async و defer_optimize_message

```c
ctlr->must_async = true;
ctlr->defer_optimize_message = true;
```

**`must_async`**: الـ virtual controller مش بيدعم الـ fast synchronous path اللي موجود في الـ SPI core. الـ `spi_sync()` أحياناً بيتجاوز الـ queue لو الـ bus فاضي — ده مش آمن هنا لأن الـ MUX لازم يتأخد دايماً. فبيجبر الـ core يمشي عن طريق الـ async path دايماً.

**`defer_optimize_message`**: الـ SPI core بيقدر يعمل pre-optimization للـ `spi_message` (زي DMA mapping) قبل ما يبدأ الـ transfer. لكن الـ virtual controller ما بيعرفش الـ parent's constraints وقت الـ setup — لازم يأجل الـ optimization لوقت الـ transfer الفعلي.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Macros والـ Config Options المهمة

#### الـ Macro الوحيد في الملف

| Macro | القيمة | المعنى |
|-------|--------|--------|
| `SPI_MUX_NO_CS` | `(unsigned int)-1` | قيمة sentinel تعني "مفيش CS متحدد حاليًا في المux" — بتتحط في `current_cs` وقت الـ probe |

#### **الـ SPI Controller Flags** (من `spi.h` — بتتنسخ من الـ parent)

| Flag | Bit | المعنى |
|------|-----|--------|
| `SPI_CONTROLLER_HALF_DUPLEX` | BIT(0) | مش قادر يعمل full duplex |
| `SPI_CONTROLLER_NO_RX` | BIT(1) | مش قادر يقرأ |
| `SPI_CONTROLLER_NO_TX` | BIT(2) | مش قادر يكتب |
| `SPI_CONTROLLER_MUST_RX` | BIT(3) | لازم يكون فيه receive |
| `SPI_CONTROLLER_MUST_TX` | BIT(4) | لازم يكون فيه transmit |
| `SPI_CONTROLLER_GPIO_SS` | BIT(5) | الـ CS عن طريق GPIO |
| `SPI_CONTROLLER_SUSPENDED` | BIT(6) | الكونترولر suspended |
| `SPI_CONTROLLER_MULTI_CS` | BIT(7) | يقدر يفعّل أكتر من CS في نفس الوقت |

> الـ driver بييجي بـ `ctlr->flags = spi->controller->flags` — يعني بيورث كل القيود دي من الـ parent controller بالظبط.

#### **الـ SPI Mode Bits** المهمة (من `uapi/linux/spi/spi.h` — بتتنسخ هي كمان)

| Flag | المعنى |
|------|--------|
| `SPI_CPHA` | Clock Phase |
| `SPI_CPOL` | Clock Polarity |
| `SPI_CS_HIGH` | Chip Select active high |
| `SPI_LSB_FIRST` | LSB أول |
| `SPI_NO_TX` / `SPI_NO_RX` | بدون wire للـ TX أو RX |

#### **الـ Controller Options** اللي بيضبطها الـ driver صراحةً

| Field | القيمة | السبب |
|-------|--------|-------|
| `ctlr->bus_num` | `-1` | خلي الـ kernel يختار رقم bus تلقائي |
| `ctlr->must_async` | `true` | **إجباري** — الـ transfers لازم تكون async (عشان الـ parent bus شغال) |
| `ctlr->defer_optimize_message` | `true` | متعملش optimize للـ message قبل ما يتبعت فعلًا |
| `ctlr->num_chipselect` | `mux_control_states(priv->mux)` | عدد الـ CS = عدد states الـ mux |

---

### الـ Structs المهمة

#### 1. `struct spi_mux_priv` — قلب الـ driver

الـ struct الوحيد اللي الـ driver بيعرّفه. بيتحفظ في الـ driver data الخاصة بالـ child `spi_controller`.

```c
struct spi_mux_priv {
    struct spi_device    *spi;              // الـ SPI device على الـ parent bus
    unsigned int          current_cs;       // آخر CS اتعمله select في الـ mux

    void (*child_msg_complete)(void *context); // الـ callback الأصلي للـ child message
    void                 *child_msg_context;   // الـ context الأصلي للـ child message
    struct spi_device    *child_msg_dev;       // الـ spi_device الأصلي للـ child message
    struct mux_control   *mux;              // handle على الـ mux hardware
};
```

| Field | النوع | الدور |
|-------|-------|-------|
| `spi` | `struct spi_device *` | الـ "بوابة" على الـ parent controller — كل transfers بتمشي عبره |
| `current_cs` | `unsigned int` | cache للـ CS الحالي — لو نفس الـ CS، منعملش `spi_setup` تاني |
| `child_msg_complete` | function pointer | الـ driver بيسرق الـ callback من الـ message ويحفظه هنا |
| `child_msg_context` | `void *` | الـ context المرتبط بالـ callback المسروق |
| `child_msg_dev` | `struct spi_device *` | الـ spi_device المرتبط بالـ message الأصلي |
| `mux` | `struct mux_control *` | الـ handle على الـ mux — بيتجيب بـ `devm_mux_control_get` |

---

#### 2. `struct spi_device` — الـ Device على الـ Bus

بيمثل device واحد متوصل على SPI bus. في الـ driver ده بيتظهر في دورين:
- **`priv->spi`**: الـ device على الـ parent bus (اللي الـ mux driver نفسه متوصل عليه)
- **`m->spi` / `spi` parameter**: أي child device على الـ virtual bus الجديدة

| Field مهم | الدور في spi-mux |
|-----------|-----------------|
| `controller` | بيوصل الـ child device بالـ virtual controller |
| `max_speed_hz` | بتتنسخ من الـ child لـ `priv->spi` وقت كل select |
| `mode` | بتتنسخ كمان |
| `bits_per_word` | بتتنسخ كمان |
| `chip_select[0]` | رقم الـ CS — بيترجمه الـ mux لـ hardware line |

---

#### 3. `struct spi_controller` — الـ Controller (Parent وChild)

في الـ driver بيتعامل مع **اتنين** `spi_controller`:

| | الـ Parent Controller | الـ Child (Virtual) Controller |
|--|----------------------|-------------------------------|
| مين بيعمله | الـ hardware SPI driver | `spi_alloc_host()` في الـ probe |
| بيوصله مين | `priv->spi->controller` | `ctlr` في الـ probe |
| الـ `transfer_one_message` | بتاعة الـ hardware driver | `spi_mux_transfer_one_message` |
| الـ `setup` | بتاعة الـ hardware driver | `spi_mux_setup` |
| `cur_msg` | بيتستخدم في الـ complete callback | — |

الـ fields اللي الـ driver بيضبطها في الـ child controller:

```c
ctlr->mode_bits            = spi->controller->mode_bits;       // ورث من parent
ctlr->flags                = spi->controller->flags;           // ورث من parent
ctlr->bits_per_word_mask   = spi->controller->bits_per_word_mask; // ورث
ctlr->transfer_one_message = spi_mux_transfer_one_message;     // hook مهم
ctlr->setup                = spi_mux_setup;
ctlr->num_chipselect       = mux_control_states(priv->mux);
ctlr->bus_num              = -1;
ctlr->must_async           = true;
ctlr->defer_optimize_message = true;
```

---

#### 4. `struct spi_message` — الـ Transfer Request

بيمثل transaction كاملة (واحدة أو أكتر transfers). الـ driver بيعمل **callback hijacking** على الـ message:

| Field | الدور في spi-mux |
|-------|-----------------|
| `complete` | الـ driver بيستبدله بـ `spi_mux_complete_cb` — ويحفظ الأصلي في `priv->child_msg_complete` |
| `context` | الـ driver بيضع `priv` هنا بدل الـ context الأصلي |
| `spi` | الـ driver بيغيره من الـ child device لـ `priv->spi` (الـ parent device) |

---

#### 5. `struct mux_control` — الـ MUX Hardware

بيمثل channel واحد في الـ multiplexer. الـ API المستخدم:

| Function | المعنى |
|----------|--------|
| `devm_mux_control_get(&spi->dev, NULL)` | جيب الـ mux handle — مرتبط بعمر الـ device |
| `mux_control_states(priv->mux)` | عدد الـ states المتاحة = عدد الـ CS |
| `mux_control_select(priv->mux, cs)` | فعّل الـ state رقم `cs` في الـ mux (مع locking داخلي) |
| `mux_control_deselect(priv->mux)` | ألغِ التفعيل بعد انتهاء الـ transfer |

---

#### 6. `struct spi_driver` — تعريف الـ Driver نفسه

```c
static struct spi_driver spi_mux_driver = {
    .probe   = spi_mux_probe,
    .driver  = {
        .name           = "spi-mux",
        .of_match_table = spi_mux_of_match,
    },
    .id_table = spi_mux_id,
};
```

بيتسجل بـ `module_spi_driver()` اللي هي macro بتوسّع لـ `module_init` + `module_exit`.

---

### مخطط العلاقات بين الـ Structs

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    Hardware SPI Controller                      │
  │                    (struct spi_controller - parent)             │
  └────────────────────────┬────────────────────────────────────────┘
                           │
                           │  (bus)
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │   struct spi_device  "priv->spi"                                │
  │   (الـ spi-mux device على الـ parent bus)                      │
  │   .controller ──────────────────────────────► parent ctlr      │
  │   .max_speed_hz  ◄── copied from child on each select          │
  │   .mode          ◄── copied from child on each select          │
  └────────────────────────┬────────────────────────────────────────┘
                           │
                           │  spi_get_drvdata(priv->spi)
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │   struct spi_controller  "ctlr"  (Virtual / Child)             │
  │   .transfer_one_message = spi_mux_transfer_one_message         │
  │   .setup                = spi_mux_setup                        │
  │   .num_chipselect        = mux states count                    │
  │   .must_async            = true                                │
  │   [devdata] ────────────────────────────────────────────────►  │
  └──────────────────────────────┬──────────────────────────────────┘
                                 │  spi_controller_get_devdata(ctlr)
                                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │   struct spi_mux_priv                                           │
  │   .spi            ──────────────────────────► priv->spi above  │
  │   .current_cs     = last selected CS (cache)                   │
  │   .mux            ──────────────────────────────────────────►  │
  │   .child_msg_complete  (saved callback)                        │  ┌──────────────────┐
  │   .child_msg_context   (saved context)                         │  │ struct mux_control│
  │   .child_msg_dev       (saved spi_device*)                     │  │ (hardware MUX)   │
  └─────────────────────────────────────────────────────────────────┘  └──────────────────┘

  ┌─────────────────────────────────────────────────────────────────┐
  │   struct spi_device  "child devices"                            │
  │   (devices registered under the virtual controller)            │
  │   .controller ──────────────────────────────► virtual ctlr     │
  │   .chip_select[0]  = CS index → MUX state                      │
  └─────────────────────────────────────────────────────────────────┘
```

---

### دورة حياة الـ Driver — Lifecycle Diagram

```
  MODULE LOAD
      │
      ▼
  module_spi_driver(spi_mux_driver)
      │  يسجل الـ driver في الـ SPI bus
      ▼
  ┌─────────────────────────────────────────┐
  │  PROBE  — spi_mux_probe(spi)            │
  │                                         │
  │  1. spi_alloc_host()                    │
  │     └─► ينشئ struct spi_controller     │
  │         + spi_mux_priv مباشرة بعده     │
  │                                         │
  │  2. spi_set_drvdata(spi, ctlr)          │
  │     └─► يربط ctlr بالـ spi device      │
  │                                         │
  │  3. priv->spi = spi                     │
  │                                         │
  │  4. lockdep_set_subclass()              │
  │     └─► يحدد الـ lock class (depth=1)   │
  │                                         │
  │  5. devm_mux_control_get()              │
  │     └─► يجيب handle على الـ MUX HW     │
  │                                         │
  │  6. priv->current_cs = SPI_MUX_NO_CS   │
  │                                         │
  │  7. نسخ capabilities من parent         │
  │     mode_bits, flags, bpw_mask          │
  │                                         │
  │  8. ضبط ops:                           │
  │     transfer_one_message, setup         │
  │     num_chipselect, must_async, إلخ     │
  │                                         │
  │  9. devm_spi_register_controller()      │
  │     └─► ينشر الـ virtual bus           │
  │     └─► الـ DT children بيتعرفوا      │
  └─────────────────────────────────────────┘
      │
      ▼
  RUNNING — الـ driver جاهز لاستقبال transfers
      │
      ▼ (on each transfer request)
  ┌─────────────────────────────────────────┐
  │  TRANSFER CYCLE                         │
  │  (انظر Call Flow Diagram تحت)           │
  └─────────────────────────────────────────┘
      │
      ▼
  REMOVE (managed by devm — automatic)
      │
      ├─► devm_spi_register_controller: unregister virtual bus
      ├─► devm_mux_control_get: release mux handle
      └─► spi_controller_put: free ctlr + priv memory
```

---

### Call Flow Diagrams

#### 1. مسار الـ Setup

```
  child device calls spi_setup(child_spi)
      │
      ▼
  SPI core → ctlr->setup(child_spi)
      │        [= spi_mux_setup]
      ▼
  spi_mux_setup(spi):
      └─► spi_setup(priv->spi)
              │  [setup on parent bus with current settings]
              ▼
          parent controller->setup(priv->spi)
              └─► hardware register update
```

> ملاحظة: الـ `spi_mux_setup` بتعمل setup على الـ parent device بالـ settings الحالية — مش settings الـ child، لأن الـ child settings بتتطبق فقط وقت الـ transfer الفعلي.

---

#### 2. مسار الـ Transfer (المسار الأساسي)

```
  child device calls spi_async(child_spi, msg)
      │
      ▼
  SPI core queues message on virtual ctlr
      │
      ▼
  ctlr->transfer_one_message(ctlr, msg)
      │        [= spi_mux_transfer_one_message]
      │
      ├─► spi_mux_select(msg->spi)
      │       │
      │       ├─► mux_control_select(priv->mux, cs)
      │       │       └─► [MUX HW: activates the selected line]
      │       │           [blocks if another transfer is using mux]
      │       │
      │       └─► if cs != current_cs:
      │               copy child settings to priv->spi
      │               (max_speed_hz, mode, bits_per_word)
      │               spi_setup(priv->spi)
      │                   └─► parent controller->setup()
      │
      ├─► save originals:
      │   priv->child_msg_complete = msg->complete
      │   priv->child_msg_context  = msg->context
      │   priv->child_msg_dev      = msg->spi
      │
      ├─► hijack message:
      │   msg->complete = spi_mux_complete_cb
      │   msg->context  = priv
      │   msg->spi      = priv->spi   ← parent device!
      │
      └─► spi_async(priv->spi, msg)
              │
              ▼
          parent SPI controller processes the message
              │
              ▼
          transfer completes → calls msg->complete(msg->context)
              │         [= spi_mux_complete_cb(priv)]
              ▼
          spi_mux_complete_cb(priv):
              │
              ├─► restore originals in msg:
              │   msg->complete = priv->child_msg_complete
              │   msg->context  = priv->child_msg_context
              │   msg->spi      = priv->child_msg_dev
              │
              ├─► spi_finalize_current_message(ctlr)
              │       └─► يبلغ الـ virtual controller إن الـ message خلص
              │           → ينادي الـ callback الأصلي للـ child
              │
              └─► mux_control_deselect(priv->mux)
                      └─► [MUX HW: release the line]
                          [wakes up any waiting transfer]
```

---

#### 3. مسار الـ Callback Hijacking — الفكرة الجوهرية

```
  BEFORE hijack:                    AFTER hijack:
  ┌─────────────────┐               ┌─────────────────┐
  │  spi_message m  │               │  spi_message m  │
  │  .complete ─────┼──► child_cb   │  .complete ─────┼──► spi_mux_complete_cb
  │  .context  ─────┼──► child_ctx  │  .context  ─────┼──► priv
  │  .spi      ─────┼──► child_dev  │  .spi      ─────┼──► priv->spi
  └─────────────────┘               └─────────────────┘

  saved in priv:
  ┌─────────────────────────┐
  │  spi_mux_priv           │
  │  .child_msg_complete ───┼──► child_cb   (original)
  │  .child_msg_context  ───┼──► child_ctx  (original)
  │  .child_msg_dev      ───┼──► child_dev  (original)
  └─────────────────────────┘

  AFTER complete_cb restores:
  ┌─────────────────┐
  │  spi_message m  │   ← restored to original state
  │  .complete ─────┼──► child_cb   ✓
  │  .context  ─────┼──► child_ctx  ✓
  │  .spi      ─────┼──► child_dev  ✓
  └─────────────────┘
```

---

### استراتيجية الـ Locking

#### الـ Locks المستخدمة وترتيبها

| Lock | صاحبه | بيحمي إيه |
|------|--------|-----------|
| `parent_ctlr->io_mutex` | الـ parent SPI core | الـ physical bus access |
| `child_ctlr->io_mutex` | الـ virtual SPI core (subclass 1) | الـ virtual bus queue |
| `child_ctlr->add_lock` | الـ virtual SPI core (subclass 1) | إضافة devices للـ bus |
| `priv->mux` (internal mutex) | `mux_control` subsystem | الـ mux state — `mux_control_select` blocking |

#### مشكلة الـ Lock Ordering وحلها

لما الـ virtual controller بيشتغل، الـ parent controller بيكون ماسك الـ `io_mutex` بتاعه. لو الـ virtual controller حاول ياخد الـ `io_mutex` بتاعه وهو بنفس الـ class، الـ `lockdep` هيصرخ لأن ده يشبه deadlock من وجهة نظره.

الحل في الـ probe:

```c
/* الـ locks دي بتتاخد وهو الـ parent bus ماسك الـ io_mutex بتاعه */
lockdep_set_subclass(&ctlr->io_mutex, 1);
lockdep_set_subclass(&ctlr->add_lock, 1);
```

بيقول لـ `lockdep`: الـ locks دي من نوع مختلف (subclass 1) عن الـ parent locks (subclass 0)، فمش هيعتبرها نفس الـ lock ومش هيبلغ عن مشكلة.

#### الـ MUX Locking

الـ `mux_control_select()` بيعمل lock على الـ mux بشكل داخلي:
- لو اتنين transfers طلبوا نفس الـ mux، التاني بيتبلوك لحد ما الأول يخلص ويعمل `mux_control_deselect()`.
- الـ deselect بيتعمل في `spi_mux_complete_cb` — يعني في interrupt/softirq context — وده مقبول لأن `mux_control_deselect` مش بتنام.

#### الـ `must_async = true` — ليه ضروري

لو التransfer كانت sync، الـ thread الحالي هيستنى على `ctlr->cur_msg_completion`. الـ parent controller كمان ممكن يكون شغال في نفس الـ thread context. الـ `must_async = true` بيجبر كل الـ transfers تمشي عبر الـ kthread queue — فمش فيه deadlock بين الـ parent والـ child message pumps.
## Phase 4: شرح الـ Functions

---

### جدول ملخص الـ Functions والـ APIs

| Function | Category | Caller Context |
|---|---|---|
| `spi_mux_probe` | Registration | SPI core — عند detect الـ device |
| `spi_mux_setup` | Runtime | SPI core — `spi_setup()` call من child driver |
| `spi_mux_select` | Runtime / Helper | `spi_mux_transfer_one_message` فقط |
| `spi_mux_transfer_one_message` | Runtime | SPI core — kthread الـ message pump |
| `spi_mux_complete_cb` | Callback / Cleanup | Interrupt / softirq — parent controller callback |

**الـ External APIs المستخدمة:**

| API | من أين | الغرض |
|---|---|---|
| `spi_alloc_host` | `spi.h` | allocate `spi_controller` + private data |
| `spi_controller_get_devdata` | `spi.h` | استرجاع الـ `priv` من الـ controller |
| `spi_set_drvdata` / `spi_get_drvdata` | `spi.h` | ربط الـ ctlr بالـ `spi_device` |
| `spi_setup` | `spi.h` | تطبيق settings على الـ parent bus |
| `spi_get_chipselect` | `spi.h` | قراءة الـ CS index من الـ child device |
| `spi_async` | `spi.h` | إرسال message للـ parent controller بشكل async |
| `spi_finalize_current_message` | `spi.h` | إعلام الـ SPI core بانتهاء الـ message |
| `spi_controller_put` | `spi.h` | decrement refcount — cleanup path |
| `devm_spi_register_controller` | `spi.h` | تسجيل الـ virtual controller مع الـ core |
| `devm_mux_control_get` | `mux/consumer.h` | الحصول على الـ mux handle من الـ DT |
| `mux_control_select` | `mux/consumer.h` | تفعيل state معين في الـ mux (blocking) |
| `mux_control_deselect` | `mux/consumer.h` | تحرير الـ mux بعد انتهاء الـ transfer |
| `mux_control_states` | `mux/consumer.h` | عدد الـ states = عدد الـ chip selects |
| `lockdep_set_subclass` | `lockdep` | حل nested locking بين parent وchild bus |

---

### Group 1: Registration — إنشاء وتسجيل الـ Virtual Controller

هذه المجموعة مسؤولة عن بناء الـ virtual SPI controller الذي يمثل الـ mux، وربطه بالـ parent bus، وتسجيله في الـ SPI core. يتم استدعاؤها مرة واحدة فقط وقت الـ probe.

---

#### `spi_mux_probe`

```c
static int spi_mux_probe(struct spi_device *spi)
```

**ما تفعله:**
هي الـ entry point للدرايفر — تُنشئ `spi_controller` جديد يمثل الـ mux كـ virtual SPI bus، وتربطه بالـ parent `spi_device`. تقوم بجلب `mux_control` من الـ device tree، وتضبط خصائص الـ controller بحيث ترث capabilities الـ parent، ثم تسجله مع الـ SPI core عبر `devm_spi_register_controller`.

الـ `current_cs` يُضبط على `SPI_MUX_NO_CS` (أي `UINT_MAX`) للتأكد أن أول transfer ستفرض setup كاملة على الـ mux بغض النظر عن الـ CS المطلوب.

**Parameters:**
- `spi` — الـ `spi_device` للـ mux chip نفسه على الـ parent bus

**Return value:**
- `0` — نجاح
- `-ENOMEM` — فشل تخصيص الـ controller
- `PTR_ERR(priv->mux)` — فشل الحصول على الـ mux control (يستخدم `dev_err_probe` لتأجيل الـ error في حالة `-EPROBE_DEFER`)
- error من `devm_spi_register_controller`

**Key details:**

```
spi_alloc_host()
    └── allocs spi_controller + sizeof(spi_mux_priv) at end
            priv = spi_controller_get_devdata(ctlr)

lockdep_set_subclass(&ctlr->io_mutex, 1)
lockdep_set_subclass(&ctlr->add_lock, 1)
    └── parent bus holds class-0 locks
        child (mux) bus takes class-1 locks
        → avoids false lockdep circular dependency warning

devm_mux_control_get(&spi->dev, NULL)
    └── NULL = single unnamed mux in DT
        devm = auto-released on device removal

ctlr->mode_bits          = spi->controller->mode_bits   // inherit parent
ctlr->flags              = spi->controller->flags
ctlr->bits_per_word_mask = spi->controller->bits_per_word_mask
ctlr->num_chipselect     = mux_control_states(priv->mux)
ctlr->bus_num            = -1    // dynamic assignment
ctlr->must_async         = true  // enforce async-only transfers
ctlr->defer_optimize_message = true  // skip message optimization

devm_spi_register_controller()
    └── on failure → goto err_put_ctlr → spi_controller_put(ctlr)
```

**الـ `must_async = true` مهمة جداً:** لأن الدرايفر يستخدم `spi_async` على الـ parent، أي تنفيذ الـ transfer بشكل synchronous على الـ virtual controller سيؤدي لـ deadlock (parent bus lock مأخوذ بالفعل). الـ SPI core يرفض أي `spi_sync` call إذا كان `must_async` مضبوطاً.

**Who calls it:** SPI core عند match الـ device مع الـ driver (عبر `spi_bus_type.probe`).

---

### Group 2: Runtime — معالجة الـ Transfers

هذه المجموعة هي قلب الدرايفر — تُنفذ كل transfer عبر إعادة توجيهها للـ parent bus بعد ضبط الـ mux على الـ CS الصحيح.

---

#### `spi_mux_setup`

```c
static int spi_mux_setup(struct spi_device *spi)
```

**ما تفعله:**
الـ `setup` callback للـ virtual controller. يستدعيها الـ SPI core عندما يستدعي child driver دالة `spi_setup()`. في هذا الدرايفر، الـ setup الحقيقية لا تحدث هنا — بل تحدث عند كل transfer في `spi_mux_select`. السبب: لا يمكن معرفة أي child device سيكون النشط وقت الـ setup، والـ mux قد يكون متاحاً لعدة devices.

الوظيفة الوحيدة هنا هي استدعاء `spi_setup(priv->spi)` على الـ parent لضمان أن الـ parent bus initialized بشكل صحيح.

**Parameters:**
- `spi` — الـ child `spi_device` الذي طلب الـ setup

**Return value:**
- نتيجة `spi_setup(priv->spi)` — عادةً `0` أو error code

**Key details:**
- يمكن استدعاؤها عدة مرات دون مشكلة (idempotent من منظور الـ mux)
- لا تغير `priv->current_cs` — هذا متروك لـ `spi_mux_select`
- لا locking خاص — الـ SPI core يضمن serialization على مستوى الـ controller

**Who calls it:** SPI core عندما يستدعي child device driver الدالة `spi_setup()`.

---

#### `spi_mux_select`

```c
static int spi_mux_select(struct spi_device *spi)
```

**ما تفعله:**
تضبط الـ multiplexer hardware على الـ CS الخاص بالـ child device المطلوب. إذا كان الـ CS المطلوب هو نفس `current_cs`، تتجنب إعادة ضبط الـ parent device settings (optimization). في حالة تغيير الـ CS، تنسخ settings الـ child (speed, mode, bits_per_word) إلى الـ parent `spi_device`، ثم تستدعي `spi_setup` على الـ parent لتطبيقها.

**Parameters:**
- `spi` — الـ child `spi_device` الذي يريد التحدث (صاحب الـ message الحالي)

**Return value:**
- `0` — نجاح
- error من `mux_control_select` (مثلاً `-EBUSY` إذا الـ mux مأخوذ، أو error من hardware)
- error من `spi_setup` إذا فشل ضبط الـ parent

**Key details:**

```
mux_control_select(priv->mux, cs)
    ├── blocking call — ينتظر لو الـ mux مشغول
    ├── يرفع reference count على الـ mux
    └── يضبط الـ hardware line

if (priv->current_cs == cs) → return 0  // fast path: نفس الـ CS
else:
    copy child settings → parent spi_device
        priv->spi->max_speed_hz = spi->max_speed_hz
        priv->spi->mode         = spi->mode
        priv->spi->bits_per_word = spi->bits_per_word
    priv->current_cs = cs
    spi_setup(priv->spi)   // apply to parent controller hardware
```

**تحذير مهم:** الدالة تُستدعى فقط من `spi_mux_transfer_one_message` وهو يعمل في سياق الـ message pump kthread للـ virtual controller. يجب ألا تُستدعى أثناء transfer جارية على الـ parent. الـ comment في الكود يوضح هذا صراحةً.

**لماذا `mux_control_select` أولاً قبل فحص `current_cs`؟**
لأن الـ `mux_control_select` هي mutex/semaphore أيضاً — تضمن أن لا أحد آخر يستخدم الـ mux في نفس الوقت. الفحص على `current_cs` بعدها هو optimization فقط لتجنب `spi_setup` الزائدة.

**Who calls it:** `spi_mux_transfer_one_message` فقط — من kthread context.

---

#### `spi_mux_transfer_one_message`

```c
static int spi_mux_transfer_one_message(struct spi_controller *ctlr,
                                        struct spi_message *m)
```

**ما تفعله:**
هي الـ `transfer_one_message` callback — الـ hook الرئيسي الذي ينفذ أي `spi_message` وصل للـ virtual controller. تختار الـ CS الصحيح عبر `spi_mux_select`، ثم تعمل "callback hijacking" على الـ message: تحفظ الـ `complete` callback و`context` الأصليين، وتضع callback خاصها (`spi_mux_complete_cb`)، ثم ترسل الـ message للـ parent عبر `spi_async`.

**Parameters:**
- `ctlr` — الـ virtual `spi_controller` (مُدار من الدرايفر)
- `m` — الـ `spi_message` القادم من الـ child device driver

**Return value:**
- `0` — تم إرسال الـ message للـ parent بنجاح (async — الـ message لسه ما اتنفذش)
- error من `spi_mux_select` — المشكلة في الـ mux selection
- error من `spi_async` — المشكلة في قبول الـ parent للـ message

**Key details:**

```
Pseudocode:
─────────────────────────────────────────────
spi_mux_transfer_one_message(ctlr, m):

  spi = m->spi              // child device

  ret = spi_mux_select(spi)
  if ret → return ret       // mux failed, message not started

  // Save originals (stored in priv, not in message itself)
  priv->child_msg_complete = m->complete
  priv->child_msg_context  = m->context
  priv->child_msg_dev      = m->spi

  // Hijack the message
  m->complete = spi_mux_complete_cb
  m->context  = priv
  m->spi      = priv->spi   // redirect to parent spi_device

  return spi_async(priv->spi, m)
  //  ↑ non-blocking, returns immediately
  //    actual transfer happens in parent's kthread
─────────────────────────────────────────────
```

**لماذا الـ callback hijacking؟**
الـ `spi_async` لا يعطيك hook بعد انتهاء الـ transfer إلا عبر `m->complete`. الدرايفر يحتاج لـ:
1. إعادة الـ `m->spi` و`m->complete` لأصولهم قبل إبلاغ الـ child
2. استدعاء `mux_control_deselect` لتحرير الـ mux
3. استدعاء `spi_finalize_current_message` للـ virtual controller

كل هذا يتم في `spi_mux_complete_cb`.

**الـ `ctlr->must_async = true`:** يجبر الـ SPI core على عدم استدعاء هذه الدالة إلا في سياق async-safe، مما يمنع الـ deadlock.

**Who calls it:** SPI core message pump (kthread) للـ virtual controller.

---

### Group 3: Callback / Cleanup — إنهاء الـ Transfer وتحرير الـ Mux

---

#### `spi_mux_complete_cb`

```c
static void spi_mux_complete_cb(void *context)
```

**ما تفعله:**
هي الـ completion callback التي يضعها الدرايفر بدلاً من callback الـ child. تُستدعى من سياق الـ parent controller بعد انتهاء الـ transfer. تعيد الـ message لحالته الأصلية (تُعيد `complete`، `context`، و`spi` للـ child)، ثم تُبلغ الـ virtual controller بانتهاء الـ message عبر `spi_finalize_current_message`، وأخيراً تُحرر الـ mux عبر `mux_control_deselect`.

**Parameters:**
- `context` — يُمرر كـ `void *` ولكنه في الحقيقة `struct spi_mux_priv *`

**Return value:**
- `void` — لا ترجع شيئاً

**Key details:**

```
سياق الاستدعاء:
  Parent controller ISR أو tasklet أو workqueue
  → يستدعي m->complete(m->context)
  → يصل لـ spi_mux_complete_cb(priv)

داخل الدالة:
  ctlr = spi_get_drvdata(priv->spi)   // الـ virtual controller
  m    = ctlr->cur_msg                // الـ message الجاري

  // Restore original fields
  m->complete = priv->child_msg_complete
  m->context  = priv->child_msg_context
  m->spi      = priv->child_msg_dev

  spi_finalize_current_message(ctlr)
      ├── يُبلغ virtual controller أن الـ message انتهى
      ├── يستدعي m->complete(m->context) — الآن هو callback الـ child الأصلي
      └── يسمح بمعالجة الـ message التالي في الـ queue

  mux_control_deselect(priv->mux)
      └── يُحرر الـ mux (يخفض reference count)
          → mux متاح لأي device آخر الآن
```

**ترتيب العمليات مهم جداً:**
- `spi_finalize_current_message` قبل `mux_control_deselect` — لأن `finalize` قد تستدعي callback الـ child فوراً، والـ child قد يحتاج لمعرفة أن الـ transfer انتهت قبل أن يرى الـ mux متاحاً. هذا يحافظ على consistency.

**الـ `cur_msg` thread safety:**
الـ `ctlr->cur_msg` يكون valid في هذا السياق لأن الـ SPI core يضمن أن `cur_msg` لا يُصفَّر إلا بعد استدعاء `spi_finalize_current_message`. الدالة تعمل دائماً قبل هذه الاستدعاء.

**Who calls it:** Parent controller driver — من interrupt أو softirq أو completion context، بعد انتهاء الـ actual hardware transfer.

---

### ملاحظات على الـ Locking والـ Concurrency

```
Thread Model:
─────────────────────────────────────────────────────────────────
Virtual Controller kthread (class 1 locks):
  spi_mux_transfer_one_message()
      → spi_mux_select()
          → mux_control_select()    [blocking mutex]
          → spi_setup(parent)
      → spi_async(parent)           [releases kthread, non-blocking]

Parent Controller kthread (class 0 locks):
  (executes actual transfer)
      → on completion → spi_mux_complete_cb()
          → spi_finalize_current_message(virtual ctlr)
              → child driver callback
          → mux_control_deselect()
─────────────────────────────────────────────────────────────────
```

الـ `lockdep_set_subclass` في الـ probe يحل مشكلة أن الـ virtual controller يحمل `io_mutex` (subclass 1) بينما الـ parent controller يحمل `io_mutex` (subclass 0). بدون هذا، الـ lockdep سيرى "nested lock of same class" ويُطلق false positive على الـ circular dependency.

الـ `mux_control` يعمل كـ mutex ضمني — `mux_control_select` blocking يضمن أن transfer واحدة فقط تمر عبر الـ mux في أي وقت، حتى لو كان هناك عدة devices في قائمة الانتظار.
## Phase 5: دليل الـ Debugging الشامل

الـ driver ده هو `spi-mux` — بيعمل SPI controller افتراضي فوق mux hardware، كل state في الـ mux بيتعامل معاه كـ chip select منفصل. الـ debugging بتاعه بيمس ثلاث طبقات: الـ mux control subsystem، الـ SPI core، والـ virtual controller اللي بيتعمله.

---

### Software Level

#### 1. debugfs

الـ SPI subsystem بيعمل entries في `/sys/kernel/debug/spi/` — مفيش entry خاص بالـ spi-mux نفسه، لكن الـ mux control subsystem عنده entries.

```bash
# اتأكد إن debugfs mounted
mount | grep debugfs
# لو مش موجود:
mount -t debugfs none /sys/kernel/debug

# شوف كل الـ controllers الموجودة بما فيها الـ virtual spi-mux controller
ls /sys/kernel/debug/spi/

# شوف معلومات الـ mux control
ls /sys/kernel/debug/mux/
# بيدي حاجة زي:
# /sys/kernel/debug/mux/mux0  (أو اسم الـ mux provider)

# اقرأ state الـ mux الحالية
cat /sys/kernel/debug/mux/mux0
# المتوقع: رقم الـ state الحالية أو "locked" لو فيه transfer جاري
```

**تفسير الـ output:**
- رقم = الـ state الحالية للـ mux (يعني الـ chip select المختار)
- `-1` أو `idle` = مفيش حاجة مختارة (`SPI_MUX_NO_CS`)
- لو الـ mux locked وفيش transfer = مشكلة في `mux_control_deselect`

#### 2. sysfs

```bash
# افتكر إن الـ spi-mux بيعمل controller جديد رقمه dynamic (bus_num = -1)
# الأول اعرف رقمه
ls /sys/bus/spi/devices/ | grep spi

# افترض إن الـ parent device على spi0.0 والـ virtual controller هو spiN
# شوف الـ virtual controller
ls /sys/class/spi_master/

# إحصائيات الـ transfers على الـ parent device
cat /sys/class/spi_master/spi0/statistics/messages
cat /sys/class/spi_master/spi0/statistics/errors
cat /sys/class/spi_master/spi0/statistics/transfers

# إحصائيات على الـ virtual controller (اللي خلقه spi-mux)
# سيبولك N = رقم الـ controller الجديد
cat /sys/class/spi_master/spiN/statistics/messages
cat /sys/class/spi_master/spiN/statistics/errors

# device attributes للـ spi device اللي عليه الـ mux نفسه
cat /sys/bus/spi/devices/spi0.0/modalias
# المتوقع: spi:spi-mux

# شوف الـ child devices اللي تحت الـ virtual controller
ls /sys/bus/spi/devices/ | grep spiN
```

#### 3. ftrace

```bash
# enable الـ SPI tracepoints الأساسية
cd /sys/kernel/debug/tracing

# شوف الـ available events
grep spi available_events

# enable أهم الـ events للـ spi-mux
echo 1 > events/spi/spi_message_submit/enable
echo 1 > events/spi/spi_message_start/enable
echo 1 > events/spi/spi_message_done/enable
echo 1 > events/spi/spi_transfer_start/enable
echo 1 > events/spi/spi_transfer_stop/enable

# ابدأ الـ tracing
echo 1 > tracing_on

# شغّل الـ transfer من application
# بعدين
echo 0 > tracing_on
cat trace

# مثال output مفيد:
# spi_message_submit: spi0.0 len=4
# spi_transfer_start:  spi0.0 len=4 tx=[aa bb cc dd]
# spi_message_done:    spi0.0 len=4 status=0
#
# لو شايف spi0.0 بدل spiN.X فالـ message اتبعتت للـ parent مباشرة
# ده يعني إن spi_mux_transfer_one_message اشتغل صح ونقل الـ message

# لو عايز تتبع الـ mux control بالذات
echo 1 > events/mux/enable 2>/dev/null || echo "mux tracepoints not available"

# function tracer على spi_mux functions
echo function > current_tracer
echo 'spi_mux*' > set_ftrace_filter
echo 1 > tracing_on
```

#### 4. printk و dynamic debug

الـ driver بيستخدم `dev_dbg` في `spi_mux_select` — ده بيطلع بس لو dynamic debug enabled.

```bash
# enable الـ dynamic debug messages للـ spi-mux module
echo 'module spi_mux +p' > /sys/kernel/debug/dynamic_debug/control

# أو بالـ file name
echo 'file spi-mux.c +p' > /sys/kernel/debug/dynamic_debug/control

# enable مع line numbers وـ function names
echo 'file spi-mux.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# اتحقق إن الـ rule اتضاف
grep spi-mux /sys/kernel/debug/dynamic_debug/control

# المتوقع في dmesg بعد كده:
# [ 123.456] spi-mux spi0.0: setting up the mux for cs 2
# ده بيطلع كل مرة بيتغير الـ chip select — مهم لتتبع الـ mux switching
```

**تفعيل من kernel cmdline (للـ boot-time debugging):**
```
dyndbg="file spi-mux.c +p"
```

#### 5. Kernel Config Options

| Option | الغرض |
|--------|--------|
| `CONFIG_SPI_DEBUG` | يفعّل verbose logging في SPI core |
| `CONFIG_DEBUG_KERNEL` | أساسي لأي debugging |
| `CONFIG_LOCKDEP` | مهم جداً — الـ driver بيعمل `lockdep_set_subclass` للـ nested locks |
| `CONFIG_PROVE_LOCKING` | يكشف lockdep violations في الـ nested SPI mutexes |
| `CONFIG_DEBUG_MUTEXES` | يتابع الـ io_mutex و add_lock |
| `CONFIG_MUX_MMIO` | لو الـ mux hardware MMIO-based |
| `CONFIG_MUX_GPIO` | لو الـ mux hardware GPIO-based |
| `CONFIG_SPI_SPIDEV` | بيخلي تعمل test من userspace على الـ child devices |
| `CONFIG_FTRACE` + `CONFIG_TRACE_EVENTS_SPI` | للـ ftrace tracepoints |
| `CONFIG_DYNAMIC_DEBUG` | للـ dev_dbg activation |

```bash
# اتحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_SPI_DEBUG|CONFIG_LOCKDEP|CONFIG_MUX'
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# spi-tools: اختبر الـ child devices مباشرة
apt install spi-tools  # أو من المصدر

# اعرف كل الـ SPI devices
ls -la /dev/spidev*

# اعمل loopback test على أحد الـ child devices (MISO-MOSI connected)
spi-config -d /dev/spidevN.X -q
spi-pipe -d /dev/spidevN.X -s 1000000 < /dev/urandom | hexdump -C | head

# مع spidev_test (من kernel tools/spi)
spidev_test -D /dev/spidevN.0 -v -p "Hello"
spidev_test -D /dev/spidevN.1 -v -p "World"
# شغّلهم بالتناوب وراقب الـ mux switching في dmesg

# i2c-tools equivalent غير موجود للـ SPI، لكن تقدر تستخدم:
# /sys/bus/spi/devices/spiN.X/... للـ inspection

# mux state inspection عبر الـ mux provider driver
# (يعتمد على الـ mux type: gpio-mux, adg792a, إلخ)
# مثلاً للـ gpio-mux
cat /sys/kernel/debug/mux/*/state 2>/dev/null
```

#### 7. رسائل الأخطاء الشائعة

| رسالة الخطأ | المعنى | الإصلاح |
|-------------|---------|---------|
| `failed to get control-mux` | `devm_mux_control_get` فشلت — الـ mux provider مش موجود أو الـ DT مش صح | تحقق من `mux-controls` property في الـ DT + تأكد إن الـ mux driver loaded |
| `spi_alloc_host failed` (implicit -ENOMEM) | الـ memory allocation فشلت | قلل الـ memory pressure أو تحقق من kernel logs |
| `spi_setup failed` | الـ parent controller رفض الـ settings (speed/mode/bits) | تحقق إن `max_speed_hz` و `mode` متوافقين مع الـ parent controller |
| `spi_async returned -EINVAL` | الـ message أو الـ transfer configuration غلط | تحقق من `bits_per_word` و transfer lengths |
| `Transfer while locked` (lockdep) | نسيان `mux_control_deselect` أو race condition | راجع `spi_mux_complete_cb` — لازم يتعمل `deselect` بعد كل transfer |
| `mux_control_select: -EBUSY` | الـ mux محجوز من transfer تاني | الـ `must_async = true` المفروض يمنع ده، راجع الـ SPI core queuing |
| `SPI controller busy` | الـ parent controller مش فاضي | طبيعي في concurrency، راجع الـ timeout settings |
| lockdep warning: `possible recursive locking` | الـ subclass مش اتضبط صح | تحقق من `lockdep_set_subclass` calls في probe |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في spi_mux_select — تحقق من mux state consistency */
static int spi_mux_select(struct spi_device *spi)
{
    struct spi_mux_priv *priv = spi_controller_get_devdata(spi->controller);

    /* WARN لو بنحاول نعمل select وفيش mux */
    WARN_ON(!priv->mux);

    ret = mux_control_select(priv->mux, spi_get_chipselect(spi, 0));
    if (ret) {
        /* dump_stack هنا مهم لأنه بيوضح من فين جه الـ transfer */
        dev_err(&priv->spi->dev, "mux_control_select failed: %d\n", ret);
        dump_stack();
        return ret;
    }
    ...
}

/* في spi_mux_complete_cb — تحقق من state بعد الـ transfer */
static void spi_mux_complete_cb(void *context)
{
    struct spi_mux_priv *priv = (struct spi_mux_priv *)context;
    struct spi_controller *ctlr = spi_get_drvdata(priv->spi);
    struct spi_message *m = ctlr->cur_msg;

    /* WARN لو الـ message اتغيرت بطريقة غير متوقعة */
    WARN_ON(!m);
    WARN_ON(m->spi != priv->spi);  /* المفروض يكون parent spi */
    ...
    mux_control_deselect(priv->mux);
    /* WARN لو الـ current_cs اتغير خلال الـ transfer */
    WARN_ON(priv->current_cs == SPI_MUX_NO_CS);
}

/* في spi_mux_probe — تحقق من الـ states count معقول */
    WARN_ON(mux_control_states(priv->mux) == 0);
    WARN_ON(mux_control_states(priv->mux) > 256);  /* عدد غير معقول
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

الـ spi-mux driver بيحتفظ بـ `current_cs` في الـ `spi_mux_priv`. أي خلاف بين القيمة دي وبين الـ hardware state الفعلية بيسبب مشاكل صعبة.

```bash
# الخطوة 1: اعرف الـ mux provider driver ونوعه
# (gpio-mux, adg792a, nxp-74hc153، إلخ)
ls /sys/class/mux/

# الخطوة 2: لو gpio-mux، تحقق من الـ GPIO state مباشرة
cat /sys/class/gpio/gpio*/value
# قارن النتيجة بالـ chip select الحالي في الـ kernel

# الخطوة 3: قرّر manually إن الـ mux في الـ state الصح
# مثال: لو spi-mux بـ 4 channels على GPIO 10,11
# CS 0 = GPIO10=0, GPIO11=0
# CS 1 = GPIO10=1, GPIO11=0
# راقب القيم وقت الـ transfer

# الخطوة 4: تحقق من الـ spi device settings على الـ parent controller
# (speed, mode, bits_per_word) بعد كل mux switch
cat /sys/bus/spi/devices/spi0.0/max_speed_hz 2>/dev/null || \
    cat /sys/bus/spi/devices/spi0.0/of_node/spi-max-frequency 2>/dev/null
```

#### 2. Register Dump Techniques

الـ spi-mux نفسه مفيش registers خاصة بيه (software-only)، لكن الـ mux hardware provider عنده.

```bash
# لو الـ mux provider MMIO-based، اعرف العنوان من الـ DT
cat /proc/device-tree/spi-mux/mux-controls
# أو
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 mux-controls

# استخدم devmem2 لقراءة الـ mux control register
# مثال: mux control register على عنوان 0xFE200000
devmem2 0xFE200000 w  # قراءة 32-bit

# لو مش عندك devmem2
# من /dev/mem (بيحتاج CONFIG_DEVMEM=y وـ root)
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0xFE200000)
    val = struct.unpack('<I', m.read(4))[0]
    print(f'MUX_CTRL = 0x{val:08x}')
    m.close()
"

# لـ SPI controller registers (مثلاً BCM2835 SPI)
# اقرأ الـ CS register اللي بيتحكم في الـ chip select
devmem2 0x3F204000 w  # BCM2835 SPI0 CS register
# bit 0-1: CS field — المفروض يكون 0 دايماً لأن الـ mux هو اللي بيعمل الـ CS
```

#### 3. Logic Analyzer / Oscilloscope

**نقاط القياس الأساسية:**

```
              ┌─────────────────────────────────────────┐
              │         SPI Bus (Parent)                 │
              │  SCLK ──────────────────────────────►   │
              │  MOSI ──────────────────────────────►   │
              │  MISO ◄──────────────────────────────   │
              │  CS   ──────────────────────────────►   │
              └──────────────┬──────────────────────────┘
                             │
                      ┌──────┴──────┐
                      │  MUX CHIP   │
                      │  (e.g.      │
                      │  74HC4051)  │
                      └──┬──┬──┬───┘
                         │  │  │
                       CH0 CH1 CH2  (إلى الـ downstream devices)
```

**إيه اللي تراقبه:**

| الـ Signal | النقطة | إيه تدور عليه |
|-----------|--------|----------------|
| MUX SELECT pins | GPIO lines للـ mux | لازم يتغيروا قبل SCLK بـ setup time |
| CS (parent) | parent controller CS pin | المفروض يبقى active طول الـ transfer |
| SCLK | SPI clock | تحقق من الـ frequency بتطابق `max_speed_hz` |
| DATA lines على CHx | الـ channel المختار | تحقق إن الـ data وصلت للجهاز الصح |

**تايمنج مهم:**
- الـ mux switching time (setup + hold) لازم يتحقق قبل ما تبدأ الـ SCLK
- لو مفيش delay في `mux_control_select_delay`، الـ hardware المفروض يكون سريع بما يكفي
- قس الـ propagation delay للـ mux chip (من datasheet) وقارنه بالـ SPI clock period

```bash
# اعرف الـ delay المستخدم في الـ mux provider
grep -r 'idle-state\|settle-time\|setup-us' /proc/device-tree/ 2>/dev/null
```

#### 4. Hardware Issues → Kernel Log Patterns

| المشكلة | الـ Kernel Log Pattern | السبب المحتمل |
|---------|----------------------|----------------|
| الـ mux مش بيتحول | `setting up the mux for cs X` بيتكرر بس مفيش response | GPIO driver مش initialized أو الـ MUX chip مش مغذّى |
| data corruption | transfer succeeds بس data غلط | crosstalk بين channels، أو الـ MUX settling time أطول من المتوقع |
| transfer timeout | `SPI transfer timed out` | الـ device على الـ channel المختار مش موجود أو dead |
| MISO stuck high/low | data كلها 0xFF أو 0x00 | connection problem على الـ MUX output المختار |
| intermittent failures | errors بتظهر بشكل عشوائي | الـ MUX select غير stable — noise على GPIO lines |
| `-EBUSY` errors | `mux_control_select: -EBUSY` | إما race condition أو الـ mux مش deselected من قبل |

#### 5. Device Tree Debugging

```bash
# اعرض الـ DT node للـ spi-mux device
dtc -I fs -O dts /proc/device-tree 2>/dev/null | grep -A 30 'spi-mux'

# المتوقع لـ spi-mux node صح:
# &spi0 {
#     spi-mux@0 {
#         compatible = "spi-mux";
#         reg = <0>;                    /* CS على الـ parent */
#         spi-max-frequency = <1000000>;
#         mux-controls = <&mux1>;       /* ده المهم */
#         #address-cells = <1>;
#         #size-cells = <0>;
#
#         child0@0 { reg = <0>; ... };  /* CS 0 على الـ mux */
#         child1@1 { reg = <1>; ... };  /* CS 1 على الـ mux */
#     };
# };

# تحقق إن mux-controls property موجودة
cat /proc/device-tree/soc/spi@xxx/spi-mux@0/mux-controls 2>/dev/null | hexdump -C

# تحقق من عدد الـ states في الـ mux provider
# لازم يطابق عدد الـ child devices
cat /proc/device-tree/mux@xxx/mux-gpio-states 2>/dev/null | hexdump -C

# لو الـ mux-controls property مش موجودة = سبب فشل probe
# "failed to get control-mux"
dmesg | grep -E 'spi.mux|mux.control'

# تحقق من الـ phandle
grep -r 'mux-controls' /proc/device-tree/ 2>/dev/null

# اتحقق إن `#mux-control-cells` موجودة في الـ mux provider node
cat /proc/device-tree/.../mux-provider/\#mux-control-cells 2>/dev/null | hexdump
```

**مشاكل DT شائعة:**

| المشكلة | الأثر | الإصلاح |
|---------|-------|---------|
| `mux-controls` مش موجودة | `failed to get control-mux` | أضف الـ property مع الـ phandle الصح |
| عدد الـ child nodes > عدد الـ mux states | transfer على CS غير موجود | خلّي عدد الـ children = `mux_control_states()` |
| `spi-max-frequency` عالية جداً | data corruption | قلّلها أو تحقق من الـ mux propagation delay |
| `reg` مكرر في child nodes | device registration error | كل child لازم له `reg` مختلف |

---

### Practical Commands

#### أوامر جاهزة للنسخ

```bash
### 1. شوف إيه الـ spi-mux devices الموجودة
dmesg | grep -E 'spi.mux|spi-mux'
ls /sys/bus/spi/drivers/spi-mux/

### 2. تحقق إن الـ driver loaded
lsmod | grep spi_mux
modinfo spi_mux

### 3. enable كل الـ debug messages
echo 'module spi_mux +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'module mux +pflmt' > /sys/kernel/debug/dynamic_debug/control

### 4. راقب الـ log في real-time
dmesg -w | grep -E 'spi|mux'

### 5. enable كل الـ SPI tracing
cd /sys/kernel/debug/tracing
echo 1 > events/spi/enable
echo 1 > tracing_on
# شغّل الـ application
cat trace | head -50
echo 0 > tracing_on
echo > trace  # clear

### 6. إحصائيات الـ SPI controller
for ctlr in /sys/class/spi_master/*/statistics; do
    echo "=== $ctlr ==="
    for f in messages errors transfers bytes; do
        printf "  %-15s: " $f
        cat $ctlr/$f 2>/dev/null || echo "N/A"
    done
done

### 7. اختبر child device بـ spidev
# افترض الـ virtual controller هو spi1
ls /dev/spidev1.*
spidev_test -D /dev/spidev1.0 -v -l  # loopback test على CS0
spidev_test -D /dev/spidev1.1 -v -l  # loopback test على CS1

### 8. تحقق من الـ mux state الحالية
cat /sys/kernel/debug/mux/*/state 2>/dev/null || \
    cat /sys/kernel/debug/mux/* 2>/dev/null

### 9. تحقق من lockdep violations
dmesg | grep -E 'lockdep|circular|deadlock'

### 10. قرأة الـ DT للـ spi-mux
find /proc/device-tree -name "compatible" -exec grep -l 'spi-mux' {} \; 2>/dev/null | \
    while read f; do
        echo "=== $(dirname $f) ==="
        ls $(dirname $f)
    done
```

#### مثال على Output وتفسيره

**مثال 1: dynamic debug مع مشكلة في switching**
```
# dmesg output بعد تفعيل dynamic debug:
[  45.123] spi-mux spi0.0: setting up the mux for cs 0   <- أول request لـ CS0، طبيعي
[  45.234] spi-mux spi0.0: setting up the mux for cs 1   <- switch لـ CS1، طبيعي
[  45.235] spi-mux spi0.0: setting up the mux for cs 1   <- نفس الـ CS، مش المفروض
                                                            يطبع "setting up" تاني!
# التفسير: لو طبع "setting up" لنفس الـ CS مرتين، ده يعني
# إن priv->current_cs اتـ reset بطريقة غير متوقعة
# (مثلاً لو حصل remove/probe تاني، أو memory corruption)
```

**مثال 2: ftrace output طبيعي**
```
# cat /sys/kernel/debug/tracing/trace
          spidev-1234  [001]  45.100: spi_message_submit: spi1.0
          spidev-1234  [001]  45.101: spi_transfer_start: spi0.0 len=4  <- parent!
          spidev-1234  [001]  45.102: spi_transfer_stop:  spi0.0 len=4 status=0
          spidev-1234  [001]  45.103: spi_message_done:   spi1.0 status=0

# التفسير:
# - submit على spi1.0 = الـ child device على الـ virtual controller
# - transfer_start على spi0.0 = الـ parent controller (الـ mux نجح في الـ redirect)
# - ده proof إن spi_mux_transfer_one_message اشتغل صح
```

**مثال 3: إحصائيات errors**
```bash
cat /sys/class/spi_master/spi0/statistics/errors
# 0 = كل حاجة تمام
# > 0 = فيه مشاكل، راجع dmesg للتفاصيل

cat /sys/class/spi_master/spi1/statistics/errors
# لو spi0 errors = 0 و spi1 errors > 0
# = المشكلة في الـ mux switching أو الـ virtual controller level
# لو spi0 errors > 0 و spi1 errors > 0
# = المشكلة في الـ parent hardware أو الـ parent driver
```

**مثال 4: تحقق من الـ mux stuck**
```bash
# شغّل transfers متعاقبة على child devices مختلفة
for i in $(seq 1 10); do
    spidev_test -D /dev/spidev1.0 -p "A" &
    spidev_test -D /dev/spidev1.1 -p "B" &
    wait
done

# بعدين راقب:
dmesg | grep 'setting up the mux' | sort | uniq -c
# المتوقع: مرات متقاربة للـ cs 0 و cs 1
# لو cs واحد فقط = الـ mux عالق أو race condition موجود
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — أجهزة SPI أكتر من الـ CS المتاحة

#### العنوان
توسعة chip selects على SPI bus في gateway صناعي باستخدام spi-mux مع GPIO mux

#### السياق
فريق hardware بيبني industrial IoT gateway على **RK3562**. الـ SoC بيوفر SPI0 بـ 2 chip selects بس، لكن الـ BOM محتاج 4 أجهزة على نفس الـ bus:
- Flash NOR (W25Q128)
- ADC (ADS8688)
- DAC (DAC8568)
- Ethernet controller (ENC28J60)

الـ hardware engineer حط **74HC4052** كـ analog mux يتحكم فيه GPIO، والـ software engineer محتاج يدخل كل ده في الـ Device Tree.

#### المشكلة
بعد تشغيل الـ board، الـ SPI devices مش بتظهر في `/sys/bus/spi/devices/`، والـ dmesg بيقول:

```
spi-mux spi0.0: failed to get control-mux: -ENOENT
```

#### التحليل
الـ error بييجي من السطر ده في `spi_mux_probe()`:

```c
priv->mux = devm_mux_control_get(&spi->dev, NULL);
if (IS_ERR(priv->mux)) {
    ret = dev_err_probe(&spi->dev, PTR_ERR(priv->mux),
                "failed to get control-mux\n");
    goto err_put_ctlr;
}
```

**الـ `NULL`** في `devm_mux_control_get` معناها إن الـ driver بيدور على mux بدون اسم محدد — بيستخدم الـ default mux المرتبط بالـ device عبر `mux-controls` property في الـ DT. لو الـ property دي مش موجودة أو الـ mux provider مش registered، بترجع `-ENOENT`.

الـ DT كان ناقص:

```dts
/* خطأ — مفيش mux-controls */
&spi0 {
    spi-mux@0 {
        compatible = "spi-mux";
        reg = <0>;
        spi-max-frequency = <50000000>;
    };
};
```

#### الحل
**أولاً**: إضافة GPIO mux provider في الـ DT:

```dts
mux: mux-controller {
    compatible = "gpio-mux";
    #mux-control-cells = <0>;
    mux-gpios = <&gpio2 RK_PA0 GPIO_ACTIVE_HIGH>,
                <&gpio2 RK_PA1 GPIO_ACTIVE_HIGH>;
    /* 2 GPIOs = 4 states = 4 chip selects */
};

&spi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&spi0_pins>;
    status = "okay";

    spi_mux: spi-mux@0 {
        compatible = "spi-mux";
        reg = <0>;
        spi-max-frequency = <50000000>;
        /* ربط الـ mux provider */
        mux-controls = <&mux>;

        #address-cells = <1>;
        #size-cells = <0>;

        flash@0 {
            compatible = "winbond,w25q128";
            reg = <0>; /* mux state 0 */
            spi-max-frequency = <40000000>;
        };

        adc@1 {
            compatible = "ti,ads8688";
            reg = <1>; /* mux state 1 */
            spi-max-frequency = <10000000>;
        };

        dac@2 {
            compatible = "ti,dac8568";
            reg = <2>; /* mux state 2 */
            spi-max-frequency = <25000000>;
        };

        eth@3 {
            compatible = "microchip,enc28j60";
            reg = <3>; /* mux state 3 */
            spi-max-frequency = <20000000>;
        };
    };
};
```

**ثانياً**: التحقق من الـ `num_chipselect` اللي بيتحسب أوتوماتيك:

```c
/* في spi_mux_probe() */
ctlr->num_chipselect = mux_control_states(priv->mux);
/* لو الـ GPIO mux عنده 2 GPIOs → 4 states → 4 chip selects */
```

التحقق بعد الإصلاح:
```bash
ls /sys/bus/spi/devices/
# spi0.0  spi1.0  spi1.1  spi1.2  spi1.3
```

#### الدرس المستفاد
الـ `spi-mux` driver محتاج صراحةً الـ `mux-controls` property في الـ DT تاعت الـ spi-mux node نفسها — مش الـ parent controller. الـ `devm_mux_control_get(&spi->dev, NULL)` بيدور على الـ phandle في الـ node المرتبطة بالـ `spi_device` اللي هو الـ spi-mux نفسه، مش الـ SPI controller.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — Lockdep Warning في الـ Production Kernel

#### العنوان
`possible circular locking dependency` في dmesg عند تشغيل SPI mux على H616

#### السياق
شركة بتصنع Android TV box على **Allwinner H616**. الـ design بيستخدم SPI mux للوصول لـ NAND Flash وـ Wi-Fi module (عبر SPI) على نفس الـ bus. بعد تشغيل kernel 6.6 مع `CONFIG_LOCKDEP=y` للـ testing، ظهر warning مزعج:

```
======================================================
WARNING: possible circular locking dependency detected
------------------------------------------------------
spi1/317 is trying to acquire lock:
  (&ctlr->io_mutex){+.+.}-{3:3}
but task is already holding lock:
  (&ctlr->io_mutex){+.+.}-{3:3}
```

#### المشكلة
الـ lockdep بيشوف إن `io_mutex` اتأخد مرتين على نفس الـ class — مرة من الـ parent SPI controller، ومرة من الـ child controller اللي أنشأه `spi-mux`.

#### التحليل
في `spi_mux_probe()`، الـ driver بيعمل ده صراحةً:

```c
/*
 * Increase lockdep class as these lock are taken while the parent bus
 * already holds their instance's lock.
 */
lockdep_set_subclass(&ctlr->io_mutex, 1);
lockdep_set_subclass(&ctlr->add_lock, 1);
```

الكود ده موجود ومقصود — بيخلي الـ lockdep يفهم إن `io_mutex` بتاع الـ child controller هو **subclass** مختلف عن بتاع الـ parent. لكن الـ warning ظهر لأن الـ kernel version اللي بنوا بيه كان قديم (5.15 backport) وما كانش فيه هذه الأسطر.

**الـ flow اللي بيحصل**:
1. الـ parent SPI controller بياخد `io_mutex` (class 0)
2. بيكال `spi_mux_transfer_one_message()`
3. اللي بتكال `spi_async(priv->spi, m)` — ده بيروح للـ parent controller تاني
4. الـ parent بيحاول ياخد `io_mutex` تاني

بدون الـ subclass، الـ lockdep بيشوفها deadlock محتمل. مع الـ subclass، بيعرف إن الـ class=1 ممكن يتأخد جوا class=0.

#### الحل
التأكد إن patch السطور دي موجودة في الـ kernel المستخدم:

```c
lockdep_set_subclass(&ctlr->io_mutex, 1);
lockdep_set_subclass(&ctlr->add_lock, 1);
```

لو الـ kernel قديم، backport الـ patch ده:

```bash
# جيب الـ commit اللي أضاف lockdep_set_subclass
git log --oneline drivers/spi/spi-mux.c | head -20

# ابحث عن الـ patch المناسب
git log --all --grep="lockdep" -- drivers/spi/spi-mux.c
```

**التحقق** إن الـ warning اختفى:
```bash
dmesg | grep -i "circular\|lockdep\|possible"
# لازم يطلع فاضي بعد الإصلاح
```

#### الدرس المستفاد
الـ `spi-mux` بيخلق nested locking scenario حقيقي: الـ parent SPI controller بياخد lock، وجوا نفس الـ context بيكال child controller اللي بياخد lock تاني. الـ `lockdep_set_subclass` مش مجرد workaround — ده الحل الصح الموثق في الـ kernel لأي driver بينشئ child controller على parent bus محمي بـ mutex.

---

### السيناريو الثالث: Custom Board على STM32MP1 — بيانات corrupt في SPI transfer بعد switch

#### العنوان
بيانات غلط تيجي من ADC بعد تبديل الـ mux من Flash لـ ADC على STM32MP1

#### السياق
فريق embedded بيبني **industrial sensor node** على **STM32MP157**. عندهم SPI bus فيه:
- QSPI Flash (mode 0, 50MHz, 8-bit)
- 16-bit ADC: ADS131M04 (mode 1, 8MHz, 16-bit)

ربطوهم بـ spi-mux عبر analog switch (TS5A23159). بعد قراءة من الـ Flash وسويتش للـ ADC، القراءات غلط وrandom.

#### المشكلة
الـ ADC بيرجع قيم عشوائية بعد أول transfer من الـ Flash.

#### التحليل
في `spi_mux_select()`:

```c
static int spi_mux_select(struct spi_device *spi)
{
    struct spi_mux_priv *priv = spi_controller_get_devdata(spi->controller);
    int ret;

    ret = mux_control_select(priv->mux, spi_get_chipselect(spi, 0));
    if (ret)
        return ret;

    /* لو نفس الـ CS، ما يعملش إعادة setup */
    if (priv->current_cs == spi_get_chipselect(spi, 0))
        return 0;

    /* copy child device settings */
    priv->spi->max_speed_hz = spi->max_speed_hz;
    priv->spi->mode = spi->mode;
    priv->spi->bits_per_word = spi->bits_per_word;

    priv->current_cs = spi_get_chipselect(spi, 0);

    return spi_setup(priv->spi);
}
```

الـ logic صح: لما بيسويتش من Flash (CS=0) لـ ADC (CS=1)، `current_cs` بيتغير فبيعمل `spi_setup()` بالـ settings الجديدة (mode 1, 8MHz, 16-bit).

**لكن المشكلة**: الـ analog switch (TS5A23159) محتاج **propagation delay** بعد تغيير الـ GPIO. الـ `mux_control_select()` بيتكال مع delay=0 (راجع الـ `mux_control_select` macro في consumer.h):

```c
static inline int __must_check mux_control_select(struct mux_control *mux,
                          unsigned int state)
{
    return mux_control_select_delay(mux, state, 0); /* delay = 0! */
}
```

الـ switch بيتعمل لكن الـ SPI transfer بيبدأ قبل ما الـ analog switch يستقر.

#### الحل
في الـ DT، استخدام `idle-state` وإضافة الـ settling time عبر `mux-idle-state` أو استخدام `mux_control_select_delay` بدل `mux_control_select`. لكن بما إن الـ driver بيستخدم `mux_control_select` اللي delay=0، الحل الصحيح من الـ DT side:

```dts
mux: mux-controller {
    compatible = "gpio-mux";
    #mux-control-cells = <0>;
    mux-gpios = <&gpiof 12 GPIO_ACTIVE_HIGH>;
    /* إضافة settling time للـ analog switch */
    settling-time-us = <1>; /* 1µs كافية لـ TS5A23159 */
};
```

أو التحقق من الـ GPIO mux driver إنه بيدعم `settling-time-us` ويطبقه في `mux_control_select_delay`.

**التحقق**:
```bash
# قراءة ADC قبل وبعد الإصلاح
cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw
# بعد الإصلاح: قيمة ثابتة ومنطقية
```

#### الدرس المستفاد
الـ `spi-mux` driver بيفترض إن الـ mux switching فوري. الـ analog switches الحقيقية عندها propagation delay (عادةً 0.5–5µs). لازم يتحدد `settling-time-us` في الـ DT تاعت الـ mux provider، مش في الـ spi-mux node نفسها.

---

### السيناريو الرابع: i.MX8M Plus — SPI mux بيـ hang في async transfer بعد فترة

#### العنوان
الـ system بيـ hang بعد ساعات من التشغيل في automotive ECU على i.MX8M Plus

#### السياق
**Automotive ECU** على **i.MX8M Plus** فيه SPI mux يوصل:
- CAN-FD transceiver (MCP2518FD) — transfers متكررة كل 1ms
- EEPROM (AT25256) — writes نادرة

بعد 6–8 ساعات من التشغيل، الـ system بيـ hang كامل. الـ watchdog مش بيشتغل لأن الـ task المسؤول blocked.

#### المشكلة
الـ `spi_mux_transfer_one_message()` بيعمل `spi_async()` لكن الـ complete callback مش بيتكال في حالات معينة، فالـ message queue بتـ block.

#### التحليل
في `spi_mux_transfer_one_message()`:

```c
static int spi_mux_transfer_one_message(struct spi_controller *ctlr,
                        struct spi_message *m)
{
    struct spi_mux_priv *priv = spi_controller_get_devdata(ctlr);
    struct spi_device *spi = m->spi;
    int ret;

    ret = spi_mux_select(spi);
    if (ret)
        return ret;

    /* استبدال الـ complete callback */
    priv->child_msg_complete = m->complete;
    priv->child_msg_context = m->context;
    priv->child_msg_dev = m->spi;

    m->complete = spi_mux_complete_cb;
    m->context = priv;
    m->spi = priv->spi;

    return spi_async(priv->spi, m);
}
```

والـ complete callback:

```c
static void spi_mux_complete_cb(void *context)
{
    struct spi_mux_priv *priv = (struct spi_mux_priv *)context;
    struct spi_controller *ctlr = spi_get_drvdata(priv->spi);
    struct spi_message *m = ctlr->cur_msg;

    m->complete = priv->child_msg_complete;
    m->context = priv->child_msg_context;
    m->spi = priv->child_msg_dev;
    spi_finalize_current_message(ctlr);
    mux_control_deselect(priv->mux);
}
```

**المشكلة الحقيقية**: `ctlr->cur_msg` — ده pointer للـ message الحالية في الـ parent controller. لو في race condition بين الـ complete callback والـ parent controller pumping، الـ `cur_msg` ممكن يكون اتغير. ده بيسبب كوبتة في الـ message fields وممكن يـ corrupt الـ queue.

**التحقق من الـ must_async flag**:

```c
/* في spi_mux_probe() */
ctlr->must_async = true;
```

ده مهم — بيجبر كل transfers تعدي الـ async path. لو مش موجود في kernel قديم، ممكن سي يحصل spi_sync من context غلط.

#### الحل
**أولاً**: تأكد إن `must_async = true` موجودة في الـ probe (موجودة في الكود الحالي).

**ثانياً**: تأكد إن `defer_optimize_message = true` موجودة:

```c
ctlr->defer_optimize_message = true;
```

**ثالثاً**: للـ debugging، استخدم `ftrace`:

```bash
# تتبع الـ SPI callbacks
echo 1 > /sys/kernel/debug/tracing/events/spi/enable
cat /sys/kernel/debug/tracing/trace_pipe | grep spi_mux

# مراقبة الـ message queue
cat /sys/kernel/debug/spi/spi0/statistics
```

**رابعاً**: لو الـ hang متكرر، خفض تردد الـ polling:
```dts
can@1 {
    compatible = "microchip,mcp2518fd";
    reg = <1>;
    spi-max-frequency = <18000000>; /* مش 25MHz */
    interrupts-extended = <&gpio3 5 IRQ_TYPE_LEVEL_LOW>;
    /* استخدم interrupt بدل polling لتقليل load */
};
```

#### الدرس المستفاد
الـ `spi_mux_complete_cb` بيلمس `ctlr->cur_msg` مباشرةً — ده sensitive جداً للـ timing. الـ `must_async = true` و`defer_optimize_message = true` في الـ probe مش اختيارية — هما ضمانات ضرورية لسلامة الـ async path. أي kernel version قديم بدونهم ممكن يعمل hangs في production.

---

### السيناريو الخامس: AM62x — Bring-up جديد، الـ child SPI devices مش بتـ probe

#### العنوان
Custom board bring-up على AM62x: الـ spi-mux بيـ probe لكن child devices مش بتظهر

#### السياق
فريق بيعمل **board bring-up** على **TI AM625** لـ IoT sensor hub. الـ design عنده:
- SPI2 bus
- Resistor-based mux (74CBTLV3253) متحكم فيه بـ GPIO

الـ dmesg بيظهر:
```
spi-mux spi2.0: loaded successfully
```

لكن لما بيتعملوا:
```bash
ls /sys/bus/spi/devices/
```
بيلاقوا بس `spi2.0` — مفيش child devices.

#### المشكلة
الـ DT عنده child nodes لكنهم مش بيـ register كـ SPI devices.

#### التحليل
في `spi_mux_probe()`:

```c
ctlr->num_chipselect = mux_control_states(priv->mux);
ctlr->bus_num = -1; /* kernel يختار رقم أوتوماتيك */
```

الـ `num_chipselect` بيتحدد من `mux_control_states()`. لو الـ GPIO mux provider مش configured صح، الـ states ممكن ترجع 0 أو 1 بدل 4.

**فحص الـ DT**:

```dts
/* مشكلة هنا */
mux: mux-controller {
    compatible = "gpio-mux";
    #mux-control-cells = <0>;
    mux-gpios = <&main_gpio0 42 GPIO_ACTIVE_HIGH>; /* GPIO واحد = 2 states بس */
};
```

عندهم GPIO واحد → 2 states بس، لكن محتاجين 4 devices. الـ `num_chipselect = 2`، وعندهم child nodes بـ `reg = <2>` و`reg = <3>` اللي أكبر من `num_chipselect - 1`.

الـ SPI core بيـ reject أي device عنده `reg` أكبر من أو يساوي `num_chipselect`.

**التحقق**:
```bash
dmesg | grep -i "spi\|chipselect\|cs"
# هتلاقي:
# spi spi1.2: no driver for spi1.2 (or invalid cs)
```

#### الحل
**أولاً**: إصلاح الـ DT بإضافة GPIO تاني:

```dts
mux: mux-controller {
    compatible = "gpio-mux";
    #mux-control-cells = <0>;
    /* غيّر لـ 2 GPIOs = 4 states */
    mux-gpios = <&main_gpio0 42 GPIO_ACTIVE_HIGH>,
                <&main_gpio0 43 GPIO_ACTIVE_HIGH>;
};

&spi2 {
    status = "okay";
    cs-gpios = <&main_gpio0 38 GPIO_ACTIVE_LOW>;

    spi_mux: spi-mux@0 {
        compatible = "spi-mux";
        reg = <0>;
        spi-max-frequency = <25000000>;
        mux-controls = <&mux>;

        #address-cells = <1>;
        #size-cells = <0>;

        temp_sensor@0 {
            compatible = "ti,tmp125";
            reg = <0>; /* mux state 0: GPIOs = 00 */
            spi-max-frequency = <5000000>;
        };

        pressure@1 {
            compatible = "bosch,bmp388";
            reg = <1>; /* mux state 1: GPIOs = 01 */
            spi-max-frequency = <10000000>;
        };

        accel@2 {
            compatible = "adi,adxl345";
            reg = <2>; /* mux state 2: GPIOs = 10 */
            spi-max-frequency = <5000000>;
        };

        gyro@3 {
            compatible = "invensense,mpu6000";
            reg = <3>; /* mux state 3: GPIOs = 11 */
            spi-max-frequency = <8000000>;
        };
    };
};
```

**ثانياً**: التحقق بعد الإصلاح:

```bash
# كم state عند الـ mux؟
cat /sys/kernel/debug/mux/mux0/state

# هل الـ child controller اتعمل بـ 4 chipselects؟
grep -r "num_chipselect" /sys/bus/spi/devices/spi*/

# هل الـ child devices اتعملوا probe؟
ls /sys/bus/spi/devices/
# المفروض: spi2.0  spi3.0  spi3.1  spi3.2  spi3.3

# فحص كل device
for d in /sys/bus/spi/devices/spi3.*; do
    echo "$d: $(cat $d/modalias 2>/dev/null)"
done
```

**ثالثاً**: لو الـ device ظهر لكن الـ driver ما اتحملش:

```bash
# تحميل الـ driver يدوياً للاختبار
modprobe spi-bmp388
dmesg | tail -5
```

#### الدرس المستفاد
عند الـ bring-up، أول خطوة هي التحقق من `mux_control_states()` — عدد الـ states اللي بيرجعها الـ mux provider بيحدد `num_chipselect`. أي child node بـ `reg` أكبر من أو يساوي القيمة دي بيتجاهله الـ SPI core. الصيغة: N GPIOs في GPIO mux = 2^N states. التحقق من ده أسهل من ساعات debugging.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

الـ LWN.net أهم مصدر لمتابعة تطور subsystems في الـ kernel، وفيما يخص الـ SPI mux:

| المقال | الوصف |
|--------|-------|
| [spi: SPI bus multiplexer](https://lwn.net/Articles/785816/) | أول patch رسمي لإضافة generic SPI multiplexer بيستخدم الـ mux-control subsystem |
| [SPI bus multiplexing](https://lwn.net/Articles/811209/) | مراجعة للمقاربة النهائية اللي اتبنت في الـ kernel — بتستخدم `mux_control` بدل GPIO مباشر |
| [simple SPI framework](https://lwn.net/Articles/154425/) | المقال التاريخي اللي عرّف الـ SPI subsystem الأصلي في الـ kernel |
| [Serial Peripheral Interface (SPI) — kernel docs](https://static.lwn.net/kerneldoc/driver-api/spi.html) | الـ driver API الرسمي للـ SPI من موقع LWN |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة في شجرة الـ kernel مباشرةً:

```
Documentation/spi/spi-summary.rst          # نظرة عامة على الـ SPI subsystem
Documentation/spi/spi.rst                  # API كامل للـ SPI
Documentation/devicetree/bindings/spi/spi-mux.yaml   # DT binding للـ spi-mux driver
Documentation/devicetree/bindings/mux/mux-controller.txt  # كيفية تعريف mux-controller في DT
```

الـ **DT binding** الرسمي للـ driver:
- [`spi-mux.yaml`](https://www.kernel.org/doc/Documentation/devicetree/bindings/spi/spi-mux.yaml) — بيوثق كيفية استخدام `compatible = "spi-mux"` مع `mux-controls` في الـ device tree

الـ **mux controller binding**:
- [`mux-controller.txt`](https://www.kernel.org/doc/Documentation/devicetree/bindings/mux/mux-controller.txt) — بيشرح كيفية تعريف `mux-controller` والـ states المتاحة

---

### Commits المهمة في الـ Kernel

#### الـ Commit الأساسي — إضافة `spi-mux.c`

الـ driver اتضاف في **Linux 5.7** بواسطة **Chris Packham** من Allied Telesis:

- **Patchwork (v2 — الـ patch النهائي):** [spi: Add generic SPI multiplexer](https://patchwork.kernel.org/project/spi-devel-general/patch/20200114233857.25933-3-chris.packham@alliedtelesis.co.nz/)
- **DT bindings patch:** [dt-bindings: spi: Document binding for generic SPI multiplexer](https://patchwork.ozlabs.org/project/devicetree-bindings/patch/20200204032838.20739-2-chris.packham@alliedtelesis.co.nz/)
- **مناقشة LKML النهائية:** [Re: PATCH v5 2/2 — spi: Add generic SPI multiplexer](https://lkml.org/lkml/2020/11/16/1537)

#### محاولات سابقة (لم تُدمج)

- **GPIO-based SPI mux (2013):** [V4 — spi: Add SPI mux core and GPIO-based mux driver](https://patchwork.kernel.org/project/spi-devel-general/patch/1370981760-897-1-git-send-email-dries.vanpuymbroeck@dekimo.com/) — بيستخدم GPIO مباشر بدل الـ mux-control abstraction
- **V3 من نفس الفكرة:** [V3 — Add SPI mux core and GPIO-based mux driver](https://patchwork.kernel.org/patch/2654611/)

#### الـ Sparx5 SPI mux — مثال على استخدام حقيقي

- [mux: sparx5: Add Sparx5 SPI mux driver](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20200702101331.26375-5-lars.povlsen@microchip.com/) — بيوضح كيف vendor عمل custom mux driver بيشتغل مع `spi-mux.c`

---

### نقاشات Mailing List

| الرابط | الموضوع |
|--------|---------|
| [linux-spi narkive archive](https://linux-spi.vger.kernel.narkive.com/cnNyJmfw/patch-spi-driver-for-gpio-controlled-spi-multiplexer) | أول نقاش على قائمة linux-spi عن GPIO-controlled SPI multiplexer |
| [LKML — mux subsystem patch v9](https://lore.kernel.org/lkml/1486568517-6997-4-git-send-email-peda@axentia.se/) | الـ patch الأصلي للـ mux subsystem نفسه (Peter Rosin) — الـ foundation اللي `spi-mux.c` بيبني عليه |
| [spinics.net — mux EXPERT select](https://www.spinics.net/lists/kernel/msg4966435.html) | نقاش عن إمكانية enable الـ MULTIPLEXER config بدون EXPERT |

---

### Kernelnewbies.org

الصفحات دي على kernelnewbies.org بتتبع التغييرات في الـ SPI subsystem عبر إصدارات الـ kernel:

| الإصدار | الرابط |
|---------|--------|
| Linux 6.5 | [kernelnewbies.org/Linux_6.5](https://kernelnewbies.org/Linux_6.5) |
| Linux 6.12 | [kernelnewbies.org/Linux_6.12](https://kernelnewbies.org/Linux_6.12) |
| Linux 4.18 | [kernelnewbies.org/Linux_4.18](https://kernelnewbies.org/Linux_4.18) |
| Linux 3.15 | [kernelnewbies.org/Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) |

ابحث في كل صفحة عن كلمة "SPI" عشان تلاقي التغييرات المرتبطة بالـ release ده.

---

### Elinux.org

| الصفحة | الوصف |
|--------|-------|
| [RPi SPI](https://elinux.org/RPi_SPI) | شرح عملي لاستخدام SPI على Raspberry Pi مع Linux — مفيد لفهم الـ chip select وإعداد الـ device tree |
| [BeagleBoard/SPI](https://elinux.org/BeagleBoard/SPI) | SPI على BeagleBoard بالـ mcspi driver — مثال board-level حقيقي |
| [Tests:MSIOF-SPI-CS-GPIO](https://elinux.org/Tests:MSIOF-SPI-CS-GPIO) | اختبارات SPI مع GPIO chip selects على Renesas — قريب جداً من use-case الـ spi-mux |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 1 و2**: مقدمة البنية التحتية للـ drivers
- **الفصل 14 (The Linux Device Model)**: بيشرح `platform_device`, `spi_device`, و `bus_type` — أساس فهم كيف `spi_mux_probe` بيشتغل
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 14 (The Block I/O Layer)** و **الفصل 17 (Devices and Modules)**: بيغطي `module_spi_driver`, `probe/remove`, و `spi_register_controller`
- مهم لفهم الـ driver lifecycle في `spi_mux_probe` وإزاي `devm_*` بيبسّط الـ cleanup

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 15 (Device Drivers)**: بيتكلم عن SPI drivers في بيئة embedded بشكل عملي
- **الفصل 16 (Kernel Debugging Techniques)**: مفيد لفهم `dev_dbg` اللي بيستخدمها الـ driver

---

### مصدر الـ Kernel على GitHub

```
https://github.com/torvalds/linux/blob/master/drivers/spi/spi-mux.c
https://github.com/torvalds/linux/tree/master/drivers/mux
https://github.com/torvalds/linux/blob/master/include/linux/mux/consumer.h
https://github.com/torvalds/linux/blob/master/include/linux/spi/spi.h
```

---

### Search Terms للبحث عن معلومات أكثر

استخدم الـ keywords دي للبحث في lore.kernel.org, LWN, أو Google:

```
"spi-mux" linux kernel driver
"mux_control_select" SPI
"spi_alloc_host" "spi_register_controller"
"spi_async" mux multiplexer
"lockdep_set_subclass" SPI bus
"must_async" spi_controller kernel
"defer_optimize_message" spi
site:lore.kernel.org spi-mux
site:patchwork.kernel.org spi mux
```
## Phase 8: Writing simple module

### الهدف

هنعمل kernel module بيستخدم **kprobe** عشان يعمل hook على الـ `spi_mux_transfer_one_message` — الـ function الأساسية في الـ driver اللي بتتولى كل transfer عبر الـ SPI mux. كل ما جه transfer جديد، الـ kprobe handler بيطبع اسم الـ SPI device والـ chip select المستخدم والـ speed.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_spi_mux.c
 *
 * Hooks spi_mux_transfer_one_message() to log every SPI transfer
 * that passes through an SPI mux controller.
 *
 * Build with a Makefile that sets obj-m := kprobe_spi_mux.o
 */

#include <linux/kernel.h>       /* pr_info, pr_err */
#include <linux/module.h>       /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/spi/spi.h>      /* struct spi_controller, struct spi_message, spi_get_chipselect */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("SPI Mux Observer");
MODULE_DESCRIPTION("kprobe on spi_mux_transfer_one_message to log SPI mux transfers");

/* ------------------------------------------------------------------ */
/*  pre-handler: called just before spi_mux_transfer_one_message runs  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * spi_mux_transfer_one_message(struct spi_controller *ctlr,
     *                               struct spi_message  *m)
     *
     * On x86-64:  arg0 → rdi (ctlr),  arg1 → rsi (m)
     * On arm64:   arg0 → x0  (ctlr),  arg1 → x1  (m)
     *
     * We read both args via the architecture-neutral helpers below.
     */
#if defined(CONFIG_X86_64)
    struct spi_controller *ctlr = (struct spi_controller *)regs->di;
    struct spi_message    *m    = (struct spi_message    *)regs->si;
#elif defined(CONFIG_ARM64)
    struct spi_controller *ctlr = (struct spi_controller *)regs->regs[0];
    struct spi_message    *m    = (struct spi_message    *)regs->regs[1];
#else
    /* unsupported arch — skip safely */
    return 0;
#endif

    /* Basic sanity: both pointers must be non-NULL */
    if (!ctlr || !m || !m->spi)
        return 0;

    pr_info("spi-mux-probe: transfer on controller %s  "
            "device %s  cs=%u  speed=%u Hz\n",
            dev_name(&ctlr->dev),           /* e.g. "spi2.0"  */
            dev_name(&m->spi->dev),         /* e.g. "spi2.1"  */
            spi_get_chipselect(m->spi, 0),  /* logical chip-select index */
            m->spi->max_speed_hz);          /* max negotiated speed */

    return 0; /* 0 → let the original function continue normally */
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "spi_mux_transfer_one_message", /* function to hook */
    .pre_handler = handler_pre,                    /* our callback     */
};

/* ------------------------------------------------------------------ */
/*  module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init kprobe_spi_mux_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("spi-mux-probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("spi-mux-probe: planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit kprobe_spi_mux_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("spi-mux-probe: kprobe removed from %s\n", kp.symbol_name);
}

module_init(kprobe_spi_mux_init);
module_exit(kprobe_spi_mux_exit);
```

---

### شرح كل جزء

#### الـ `#include`s

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل الـ API الخاصة بالـ kprobes |
| `linux/spi/spi.h` | بيعرّف `struct spi_controller`، `struct spi_message`، و `spi_get_chipselect()` |
| `linux/module.h` | الـ macros الأساسية لأي kernel module |

---

#### الـ `handler_pre`

**الـ `struct pt_regs *regs`** بيحمل snapshot من الـ CPU registers اللحظة ما الـ kprobe اتفعّل. بنقرأ منه الـ function arguments حسب الـ calling convention للـ architecture (x86-64: `rdi/rsi`، arm64: `x0/x1`).

**الـ `spi_get_chipselect(m->spi, 0)`** بيرجع الـ logical chip-select index للـ device المطلوب، وده المعلومة المفيدة عشان تعرف أي فرع من الـ mux اتحدد.

الـ `return 0` في نهاية الـ pre-handler معناه "كمّل تنفيذ الـ original function بشكل طبيعي" — مش interrupt الـ flow.

---

#### الـ `struct kprobe kp`

**الـ `symbol_name`** بيخلي الـ kernel يحسب عنوان الـ function تلقائيًا من الـ kallsyms بدل ما نحتاج نحدد عنوان hardcoded. الـ `pre_handler` بيشتغل قبل ما الـ original function تتنفذ — مناسب للـ logging لأننا بنقرأ الـ arguments قبل ما تتغير.

---

#### الـ `module_init` / `module_exit`

**`register_kprobe`** بيحط breakpoint في الـ kernel memory عند عنوان الـ function، وبيربط الـ handler بيها. لو فشل (مثلاً الـ symbol مش exported أو `CONFIG_KPROBES` مش enabled) بيرجع error code سالب.

**`unregister_kprobe`** في الـ exit مهم جداً: لو ما عملناهوش، الـ kernel هيكمّل يستدعي الـ handler بعد ما الـ module اتـunload، وده بيسبب **use-after-free** وـcrash مضمون.

---

### Makefile للبناء

```makefile
obj-m := kprobe_spi_mux.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
make
sudo insmod kprobe_spi_mux.ko
# trigger any SPI transfer through the mux
sudo dmesg | grep spi-mux-probe
sudo rmmod kprobe_spi_mux
```

---

### متطلبات الـ kernel config

| Option | القيمة المطلوبة |
|--------|----------------|
| `CONFIG_KPROBES` | `y` |
| `CONFIG_KALLSYMS` | `y` |
| `CONFIG_SPI_MUX` | `m` أو `y` (عشان الـ symbol يكون موجود) |
