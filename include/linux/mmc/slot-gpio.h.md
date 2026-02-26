## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ subsystem ده؟

الملف ده جزء من **MMC/SD subsystem** في Linux kernel — وهو الـ subsystem المسؤول عن كل بطاقات الذاكرة: SD cards, eMMC, SDIO.

---

### الصورة الكبيرة — تشبيه بسيط

تخيل عندك **قارئ SD card** في laptop. الجهاز محتاج يعرف حاجتين:

1. **هل في كارت اتحط؟** — card detect (CD)
2. **الكارت ده read-only ولا لأ؟** — write protect (RO)

طريقتين لمعرفة الحاجتين دول:
- **Hardware dedicated circuits** — بعض الـ controllers عندهم دوائر خاصة.
- **GPIO pins** — معظم الـ embedded boards بتستخدم GPIO عادي: لما الكارت يتحط، البـ pin بيتغير حالته.

الملف `slot-gpio.h` هو الـ **public API** اللي بيوفر abstraction layer فوق الـ GPIO لاستخدامه في كشف وجود الكارت وحالة الحماية من الكتابة. بدل ما كل host driver يكتب كود GPIO من الأول، يستخدم الـ functions دي الجاهزة.

---

### ليه الملف ده مهم؟

```
[SD Card Slot]
      |
   [GPIO Pin] ←─── يتغير لما الكارت يتحط أو يتشال
      |
[slot-gpio.h API]
      |
[MMC Core] ─→ يعمل card detection / block I/O
```

- **`mmc_gpio_get_cd()`** — اقرأ حالة الكارت (موجود / غايب).
- **`mmc_gpio_get_ro()`** — اقرأ حالة الحماية من الكتابة.
- **`mmc_gpiod_request_cd()`** — سجّل GPIO كـ card-detect pin.
- **`mmc_gpiod_request_ro()`** — سجّل GPIO كـ write-protect pin.
- **`mmc_gpiod_request_cd_irq()`** — فعّل interrupt على الـ GPIO لما الكارت يتحط أو يتشال (بدل polling).
- **`mmc_gpio_set_cd_wake()`** — خلي الـ card-detect GPIO يصحّي الجهاز من الـ sleep لما يتحط كارت.

---

### القصة الكاملة — كيف بيشتغل؟

**1. التهيئة (probe time):**
الـ host driver بيعمل probe، بيلاقي GPIO مخصص لـ card-detect في الـ device tree أو platform data، بيستدعي `mmc_gpiod_request_cd()` لتسجيل الـ GPIO مع تحديد debounce time (عادة 200ms عشان الـ mechanical bounce).

**2. الـ Interrupt:**
`mmc_gpiod_request_cd_irq()` بتسجل interrupt handler على الـ GPIO. أي تغيير في الـ pin (rising أو falling edge) بيوقّع interrupt، اللي بيستدعي `mmc_detect_change()` في الـ MMC core.

**3. الـ MMC Core:**
الـ core بيشوف التغيير، بيقرأ الحالة عن طريق `mmc_gpio_get_cd()`، لو في كارت جديد يبدأ initialization، لو الكارت اتشال يعمل cleanup.

**4. الحماية من الكتابة:**
لما الـ filesystem يحاول يكتب، الـ driver بيسأل `mmc_gpio_get_ro()` — لو الـ GPIO بيقول read-only، يرجع error للـ userspace.

---

### ملفات تانية المفروض تعرفها

| الملف | الدور |
|-------|-------|
| `drivers/mmc/core/slot-gpio.c` | الـ implementation الفعلي لكل الـ functions |
| `drivers/mmc/core/slot-gpio.h` | internal header (struct `mmc_gpio` المخفية) |
| `include/linux/mmc/host.h` | تعريف `struct mmc_host` — الـ host controller الرئيسي |
| `drivers/mmc/core/core.c` | الـ MMC core — `mmc_detect_change()` وغيرها |
| `drivers/mmc/core/host.c` | إنشاء وتدمير الـ host |
| `include/linux/gpio/consumer.h` | الـ GPIO descriptor API اللي slot-gpio بتبني عليه |

---

### ملفات الـ subsystem الأساسية

**Core:**
```
drivers/mmc/core/
├── core.c          # قلب الـ MMC subsystem
├── host.c          # إدارة الـ host controllers
├── slot-gpio.c     # تنفيذ الـ GPIO card-detect API ← نحن هنا
├── sd.c            # SD card specific logic
├── mmc.c           # eMMC specific logic
├── sdio.c          # SDIO specific logic
└── block.c         # block device interface
```

**Headers:**
```
include/linux/mmc/
├── slot-gpio.h     # ← الملف ده (public GPIO API)
├── host.h          # struct mmc_host
├── core.h          # types أساسية
└── card.h          # struct mmc_card
```

**Host Drivers (أمثلة):**
```
drivers/mmc/host/
├── sdhci.c         # Standard Host Controller Interface
├── dw_mmc.c        # Synopsys DesignWare MMC
├── mtk-sd.c        # MediaTek
└── sdhci-msm.c     # Qualcomm Snapdragon
```

كل host driver من دول ممكن يستخدم `slot-gpio.h` لو الـ board بتستخدم GPIO لكشف الكارت.
## Phase 2: شرح الـ MMC Slot GPIO Framework

---

### 1. المشكلة — ليه الـ framework ده موجود أصلاً؟

كل **MMC/SD host controller** محتاج يعرف حاجتين:
- هل فيه كارت موجود في الـ slot؟ (**card-detect / CD**)
- هل الكارت ده write-protected؟ (**read-only / RO**)

بعض الـ controllers عندهم دوائر hardware جاهزة لده. لكن كتير من الـ controllers — خصوصاً في الـ embedded boards — بتوصل الـ CD والـ RO على **GPIO pins** عادية من الـ SoC.

**المشكلة:** كل driver بيعمل نفس الكود من الأول — يطلب GPIO، يسجّل IRQ، يعمل debounce، يلغيه في الـ suspend وهكذا. كود متكرر في كل driver بيزيد bugs ويصعّب الصيانة.

---

### 2. الحل — الـ kernel بيعمل إيه؟

الـ kernel عمل **generic helper layer** اسمه `mmc slot-gpio framework` في:
- `drivers/mmc/core/slot-gpio.c`
- `include/linux/mmc/slot-gpio.h`

الفكرة: الـ host driver بيستدعي function واحدة عشان يسجّل الـ GPIO، والـ framework بيتكفل بكل التفاصيل:
- طلب الـ `gpio_desc` من الـ **GPIO descriptor API (gpiod)**
- ضبط الـ debounce (hardware أو software fallback)
- تسجيل الـ threaded IRQ على rising + falling edges
- ربط الـ IRQ بـ `mmc_detect_change()` تلقائياً
- دعم الـ wake-from-suspend على الـ CD IRQ

---

### 3. Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                     MMC Core (mmc/core/)                    │
  │                                                             │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │              slot-gpio.c  (generic helper)           │   │
  │  │                                                      │   │
  │  │  struct mmc_gpio {                                   │   │
  │  │      gpio_desc  *cd_gpio  ──────────────────────┐    │   │
  │  │      gpio_desc  *ro_gpio  ────────────────────┐  │    │   │
  │  │      irq_handler_t cd_gpio_isr               │  │    │   │
  │  │      cd_debounce_delay_ms                    │  │    │   │
  │  │      cd_irq                                  │  │    │   │
  │  │  }                                           │  │    │   │
  │  │                                              │  │    │   │
  │  │  mmc_gpiod_request_cd() ──► devm_gpiod_get  │  │    │   │
  │  │  mmc_gpiod_request_ro() ──► devm_gpiod_get     │    │   │
  │  │  mmc_gpiod_request_cd_irq() ► request_threaded_irq  │   │
  │  │  mmc_gpio_get_cd()      ──► gpiod_get_value  │       │   │
  │  │  mmc_gpio_get_ro()      ──► gpiod_get_value     │    │   │
  │  │  mmc_gpio_set_cd_wake() ──► enable_irq_wake  │       │   │
  │  └──────────────────────────────────────────────┼──┼────┘   │
  │                   ▲                             │  │        │
  │                   │ calls                       ▼  ▼        │
  │  ┌────────────────┴─────┐            ┌──────────────────┐   │
  │  │  MMC Host Driver     │            │  GPIO Subsystem  │   │
  │  │  (e.g. sdhci-of-*)   │            │  (gpio/consumer) │   │
  │  │                      │            │                  │   │
  │  │  probe():            │            │  gpio_desc       │   │
  │  │   mmc_gpiod_request_cd()          │  gpiod_get_value │   │
  │  │   mmc_gpiod_request_ro()          │  gpiod_to_irq    │   │
  │  │   mmc_add_host()     │            │  gpiod_set_debounce   │
  │  └──────────────────────┘            └──────────────────┘   │
  │                                                             │
  │        ┌────────────────────────────────────┐              │
  │        │         struct mmc_host             │              │
  │        │   .slot.handler_priv ──► mmc_gpio  │              │
  │        │   .slot.cd_irq                     │              │
  │        │   .slot.cd_wake_enabled            │              │
  │        └────────────────────────────────────┘              │
  └─────────────────────────────────────────────────────────────┘
                              │
                    IRQ fires on CD pin
                              │
                              ▼
                   mmc_gpio_cd_irqt()
                              │
                              ▼
                   mmc_detect_change()  ──► card rescan
```

---

### 4. Analogy واقعي

تخيّل **موظف استقبال في شركة** (الـ framework) بدل ما كل قسم يعيّن حارس بوابة لوحده.

| الـ Analogy | الـ Kernel Concept |
|---|---|
| الموظف نفسه | `slot-gpio.c` — الكود المشترك |
| البوابة الإلكترونية | GPIO pin (CD أو RO) |
| جرس التنبيه | IRQ على الـ GPIO |
| تأخير الرد بعد الجرس | `cd_debounce_delay_ms` |
| إبلاغ الإدارة | `mmc_detect_change()` |
| كل قسم (HR، Eng...) | كل host driver (sdhci، dw_mmc...) |
| بدل ما كل قسم يعيّن حارسه | بدل ما كل driver يكتب GPIO handling بتاعته |

---

### 5. الـ Core Abstraction

الـ abstraction الأساسي هو **`struct mmc_gpio`** — وهو private context بيتخزن في `host->slot.handler_priv`:

```c
/* drivers/mmc/core/slot-gpio.c — private, مش exposed في الـ header */
struct mmc_gpio {
    struct gpio_desc *ro_gpio;          /* descriptor للـ write-protect pin */
    struct gpio_desc *cd_gpio;          /* descriptor للـ card-detect pin */
    irq_handler_t    cd_gpio_isr;       /* custom ISR أو default */
    char             *ro_label;         /* اسم للـ debugfs/sysfs */
    char             *cd_label;
    u32              cd_debounce_delay_ms; /* fallback software debounce */
    int              cd_irq;            /* IRQ number مخصص (اختياري) */
};
```

والـ `struct mmc_slot` (في `mmc/host.h`) هو نقطة الوصل بين الـ framework والـ `mmc_host`:

```c
struct mmc_slot {
    int  cd_irq;              /* IRQ number المسجّل النهائي أو -EINVAL */
    bool cd_wake_enabled;     /* wake-from-suspend مفعّل؟ */
    void *handler_priv;       /* يشاور على struct mmc_gpio */
};
```

---

### 6. الـ Framework بيمتلك إيه vs. بيفوّض إيه؟

| الجانب | يمتلكه الـ Framework | يفوّضه للـ Driver |
|---|---|---|
| طلب GPIO descriptor | `mmc_gpiod_request_cd/ro()` | اختيار الـ `con_id` والـ `idx` من DT |
| ضبط debounce | `gpiod_set_debounce()` + software fallback | تحديد قيمة الـ debounce بـ microseconds |
| تسجيل IRQ | `devm_request_threaded_irq()` | تحديد IRQ مخصص (اختياري) عبر `mmc_gpio_set_cd_irq()` |
| قراءة حالة CD/RO | `mmc_gpio_get_cd/ro()` | استدعاؤهم من `get_cd()`/`get_ro()` callbacks |
| wake-from-suspend | `enable/disable_irq_wake()` | طلب التفعيل عبر `mmc_gpio_set_cd_wake()` |
| polarity (active-low/high) | ضبط `gpiod_toggle_active_low()` تلقائياً | إعلان الـ caps (`MMC_CAP2_CD_ACTIVE_HIGH`) |
| تسجيل الـ IRQ handler نفسه | `mmc_gpio_cd_irqt()` → `mmc_detect_change()` | يقدر يعيّن custom ISR لو محتاج |
| إدارة lifetime الـ resources | كلها `devm_*` — تُحرَّر مع الـ device | لا شيء، الـ cleanup تلقائي |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Caps المهمة — Cheatsheet

#### `caps` flags المتعلقة بـ slot-gpio

| Flag | Value | المعنى |
|------|-------|--------|
| `MMC_CAP_NEEDS_POLL` | `(1 << 5)` | لو مفيش IRQ، يبقى polling على card-detect |
| `MMC_CAP_NONREMOVABLE` | `(1 << 8)` | الكارت مش قابل للإزالة (eMMC) — مش محتاج CD GPIO |
| `MMC_CAP_CD_WAKE` | `(1 << 28)` | تفعيل الـ wakeup من sleep لما الكارت يتحرك |

#### `caps2` flags المتعلقة بـ slot-gpio

| Flag | Value | المعنى |
|------|-------|--------|
| `MMC_CAP2_CD_ACTIVE_HIGH` | `(1 << 10)` | سيجنال card-detect active-high (عكس الـ default) |
| `MMC_CAP2_RO_ACTIVE_HIGH` | `(1 << 11)` | سيجنال write-protect active-high |
| `MMC_CAP2_NO_WRITE_PROTECT` | `(1 << 18)` | مفيش pin فيزيائي لـ write-protect |

#### IRQ Flags المستخدمة في `mmc_gpiod_request_cd_irq`

| Flag | المعنى |
|------|--------|
| `IRQF_TRIGGER_RISING` | اشتغل لما السيجنال يطلع |
| `IRQF_TRIGGER_FALLING` | اشتغل لما السيجنال ينزل |
| `IRQF_ONESHOT` | الـ IRQ thread يتشغل مرة واحدة بدون re-enable قبل ما يخلص |

#### Return Values للـ API functions

| Function | 0 | 1 | `-ENOSYS` | negative errno |
|----------|---|---|-----------|----------------|
| `mmc_gpio_get_ro()` | الكارت R/W | الكارت Read-Only | GPIO مش configured | error |
| `mmc_gpio_get_cd()` | الكارت غايب | الكارت موجود | GPIO مش configured | error |
| `mmc_gpio_set_cd_wake()` | success | — | — | error من `enable_irq_wake` |

---

### 1. الـ Structs المهمة

#### `struct mmc_gpio` (في `drivers/mmc/core/slot-gpio.c` — internal)

الـ struct الأساسي اللي بيخزن كل حاجة خاصة بالـ GPIO slot. مش exposed في الـ header العام — private implementation detail.

| Field | Type | الغرض |
|-------|------|--------|
| `ro_gpio` | `struct gpio_desc *` | descriptor للـ GPIO الخاص بـ write-protect |
| `cd_gpio` | `struct gpio_desc *` | descriptor للـ GPIO الخاص بـ card-detect |
| `cd_gpio_isr` | `irq_handler_t` | الـ ISR handler للـ CD interrupt (default: `mmc_gpio_cd_irqt`) |
| `ro_label` | `char *` | اسم وصفي للـ RO GPIO ("`<devname> ro`") |
| `cd_label` | `char *` | اسم وصفي للـ CD GPIO ("`<devname> cd`") |
| `cd_debounce_delay_ms` | `u32` | وقت الـ debounce بالـ ms (default 200ms) |
| `cd_irq` | `int` | IRQ number خاص بالـ CD لو الـ platform حدده manually (غير الـ GPIO IRQ) |

---

#### `struct mmc_slot` (في `include/linux/mmc/host.h`)

الـ hook اللي بيربط الـ `mmc_host` بـ slot-gpio subsystem.

| Field | Type | الغرض |
|-------|------|--------|
| `cd_irq` | `int` | رقم الـ IRQ المستخدم فعلاً للـ card-detect (`-EINVAL` لو مش موجود) |
| `cd_wake_enabled` | `bool` | هل الـ wake-on-CD مفعّل دلوقتي؟ |
| `handler_priv` | `void *` | pointer لـ `struct mmc_gpio` (private context) |

---

#### `struct mmc_host` (الـ fields ذات الصلة بـ slot-gpio)

| Field | Type | الغرض |
|-------|------|--------|
| `slot` | `struct mmc_slot` | يحتوي على الـ CD IRQ والـ `mmc_gpio` context |
| `caps` | `u32` | bitfield يحتوي على `MMC_CAP_NEEDS_POLL` و `MMC_CAP_CD_WAKE` |
| `caps2` | `u32` | bitfield يحتوي على `MMC_CAP2_CD_ACTIVE_HIGH` و `MMC_CAP2_RO_ACTIVE_HIGH` |
| `trigger_card_event` | `bool` | يتفعّل من الـ ISR عشان يعلم الـ core إن في event |
| `parent` | `struct device *` | الـ device الأب — بيتستخدم لـ `devm_*` allocations |

---

### 2. رسم علاقات الـ Structs

```
┌─────────────────────────────────────────────────┐
│                  struct mmc_host                │
│                                                 │
│  ┌──────────────────────────────┐               │
│  │       struct mmc_slot        │               │
│  │  cd_irq: int                 │               │
│  │  cd_wake_enabled: bool       │               │
│  │  handler_priv: void* ────────┼──────┐        │
│  └──────────────────────────────┘      │        │
│                                        │        │
│  caps: u32  (MMC_CAP_NEEDS_POLL,       │        │
│              MMC_CAP_CD_WAKE)          │        │
│  caps2: u32 (MMC_CAP2_CD_ACTIVE_HIGH, │        │
│              MMC_CAP2_RO_ACTIVE_HIGH)  │        │
│  trigger_card_event: bool              │        │
│  parent: struct device* ───┐           │        │
└────────────────────────────┼───────────┼────────┘
                             │           │
                             │           ▼
                             │   ┌─────────────────────────┐
                             │   │     struct mmc_gpio      │
                             │   │  (private, slot-gpio.c)  │
                             │   │                          │
                             │   │  ro_gpio: gpio_desc*─────┼──► GPIO RO pin
                             │   │  cd_gpio: gpio_desc*─────┼──► GPIO CD pin
                             │   │  cd_gpio_isr: fn*        │
                             │   │  ro_label: char*         │
                             │   │  cd_label: char*         │
                             │   │  cd_debounce_delay_ms    │
                             │   │  cd_irq: int             │
                             │   └─────────────────────────┘
                             │
                    devm_kzalloc() ──► مرتبط بعمر الـ device
                    devm_kasprintf()   ويتحرر تلقائياً
```

---

### 3. Lifecycle Diagrams

#### دورة حياة الـ CD GPIO (Card-Detect)

```
Host Driver Probe
      │
      ▼
mmc_alloc_host() ── ينشئ mmc_host
      │
      ▼
mmc_gpio_alloc()
  ├── devm_kzalloc(mmc_gpio)
  ├── ctx->cd_debounce_delay_ms = 200
  ├── ctx->cd_irq = -EINVAL
  ├── slot->cd_irq = -EINVAL
  └── slot->handler_priv = ctx
      │
      ▼
mmc_gpiod_request_cd()          [optional: mmc_gpio_set_cd_irq() لو platform عنده IRQ خاص]
  ├── devm_gpiod_get_index()     ── يجيب gpio_desc
  ├── gpiod_set_debounce()       ── hardware debounce لو متاح
  ├── gpiod_toggle_active_low()  ── لو override_active_level أو MMC_CAP2_CD_ACTIVE_HIGH
  └── ctx->cd_gpio = desc
      │
      ▼
mmc_add_host()
  └── يستدعي mmc_gpiod_request_cd_irq() داخلياً
        ├── لو MMC_CAP_NEEDS_POLL → skip IRQ
        ├── لو ctx->cd_irq >= 0 → استخدم الـ IRQ المحدد من platform
        ├── غير كده → gpiod_to_irq(ctx->cd_gpio)
        └── devm_request_threaded_irq()
              handler: mmc_gpio_cd_irqt
              flags: RISING | FALLING | ONESHOT
      │
      ▼
[RUNTIME: Card Insert/Remove]
      │
      ▼
mmc_gpio_cd_irqt() [IRQ fires]
  ├── host->trigger_card_event = true
  └── mmc_detect_change(host, debounce_jiffies)
      │
      ▼
[SUSPEND]
mmc_gpio_set_cd_wake(host, true)
  ├── check MMC_CAP_CD_WAKE
  ├── enable_irq_wake(slot->cd_irq)
  └── slot->cd_wake_enabled = true
      │
      ▼
[RESUME]
mmc_gpio_set_cd_wake(host, false)
  ├── disable_irq_wake(slot->cd_irq)
  └── slot->cd_wake_enabled = false
      │
      ▼
Host Driver Remove
  └── devm cleanup تلقائي:
        ├── free_irq
        ├── gpiod_put
        ├── kfree(cd_label, ro_label)
        └── kfree(mmc_gpio)
```

#### دورة حياة الـ RO GPIO (Write-Protect)

```
mmc_gpiod_request_ro()
  ├── devm_gpiod_get_index()
  ├── gpiod_set_debounce()
  ├── gpiod_toggle_active_low() ── لو MMC_CAP2_RO_ACTIVE_HIGH
  └── ctx->ro_gpio = desc

[RUNTIME: Read query]
mmc_gpio_get_ro()
  ├── gpiod_cansleep(ro_gpio)?
  │     YES → gpiod_get_value_cansleep()
  │     NO  → gpiod_get_value()
  └── return 0 (R/W) or 1 (Read-Only)
```

---

### 4. Call Flow Diagrams

#### تدفق `mmc_gpiod_request_cd_irq()`

```
mmc_gpiod_request_cd_irq(host)
      │
      ├── slot->cd_irq >= 0 ?  ──YES──► return (already set)
      │
      ├── !ctx || !ctx->cd_gpio ?  ──YES──► return
      │
      ├── ctx->cd_irq >= 0 ?
      │     YES → irq = ctx->cd_irq        (platform-specified IRQ)
      │     NO  →
      │       host->caps & MMC_CAP_NEEDS_POLL ?
      │         YES → irq = -EINVAL         (polling mode, skip IRQ)
      │         NO  → irq = gpiod_to_irq()  (get IRQ from GPIO)
      │
      ├── irq >= 0 ?
      │     YES →
      │       ctx->cd_gpio_isr = mmc_gpio_cd_irqt  (if not custom)
      │       devm_request_threaded_irq(
      │           irq,
      │           NULL,                     (no hard IRQ handler)
      │           cd_gpio_isr,              (threaded handler)
      │           RISING|FALLING|ONESHOT,
      │           cd_label,
      │           host)
      │       ret < 0 ? → irq = ret (error)
      │
      ├── slot->cd_irq = irq
      │
      └── irq < 0 ?  ──YES──► host->caps |= MMC_CAP_NEEDS_POLL
```

#### تدفق قراءة card-detect `mmc_gpio_get_cd()`

```
mmc_gpio_get_cd(host)
      │
      ├── !ctx || !ctx->cd_gpio ? ──YES──► return -ENOSYS
      │
      ├── gpiod_cansleep(cd_gpio)?
      │     YES → gpiod_get_value_cansleep(cd_gpio)  [may sleep, sleepable context]
      │     NO  → gpiod_get_value(cd_gpio)            [atomic context ok]
      │
      └── return value (0=absent, 1=present)
          [polarity already handled by gpio_desc flags]
```

#### تدفق الـ IRQ عند تحريك الكارت

```
[Hardware: card inserted/removed]
          │
          ▼
   GPIO pin edge detected
   (RISING or FALLING)
          │
          ▼
   Kernel IRQ subsystem
          │
          ▼
   mmc_gpio_cd_irqt(irq, host)       [threaded IRQ handler]
          │
          ├── host->trigger_card_event = true
          │
          ├── mmc_detect_change(host, debounce_jiffies)
          │         │
          │         └── schedules host->detect (delayed_work)
          │                   after cd_debounce_delay_ms
          │
          └── return IRQ_HANDLED
                    │
                    ▼
          [after debounce delay]
          mmc_rescan() ── يفحص وجود الكارت فعلاً
```

---

### 5. استراتيجية الـ Locking

الـ `slot-gpio` layer بالذات **بسيط ومش بيستخدم locks صريحة** في الكود بتاعه. إنما بيعتمد على:

| المستوى | الآلية | التفاصيل |
|---------|--------|----------|
| **GPIO read** | بدون lock | `gpiod_get_value()` / `gpiod_cansleep()` — atomic أو sleepable حسب الـ GPIO controller. الـ GPIO subsystem بيدير sync داخلياً |
| **IRQ registration** | `devm_request_threaded_irq` | الـ IRQ يتسجل مرة واحدة وقت الـ probe، بعدين الـ kernel بيدير الـ IRQ threading تلقائياً |
| **`mmc_detect_change()`** | `delayed_work` | بيستخدم `host->detect` اللي هو `delayed_work`، والـ `workqueue` بيضمن serialization |
| **`host->slot.cd_irq`** | لا lock | بيتكتب مرة وقت الـ initialization قبل `mmc_add_host()`، بعدين read-only |
| **`host->slot.cd_wake_enabled`** | لا lock | بيتعدّل فقط من `mmc_gpio_set_cd_wake()` اللي بتتنادى من PM callbacks — serialized بالطبيعة |
| **`host->caps`** | بيورّث من `mmc_host` locking | أي تعديل على `caps` بيحصل قبل `mmc_add_host()` أو بيتعمل بشكل آمن |
| **`trigger_card_event`** | simple flag | بيتكتب من ISR context وبيتقرأ من `mmc_rescan()` — ممكن يحتاج `READ_ONCE/WRITE_ONCE` لكن الـ kernel بيتعامل معاه بشكل practical |

#### ملاحظة مهمة على الـ threaded IRQ:

```
Hard IRQ context (disabled interrupts)
    ──────────────────────────────────
    [لا يوجد hard handler — NULL]
            │
            ▼
    Kernel threads up IRQ thread
            │
            ▼
    mmc_gpio_cd_irqt() ← IRQF_ONESHOT يضمن إن الـ IRQ line
                          متعملش re-enable لحد ما الـ thread يخلص
                          ده بيحمي من spurious IRQs أثناء الـ debounce
```

---

### ملخص العلاقات الكاملة

```
[Platform/DT] ──────────────────────────────────┐
                                                 │ GPIO lookups
                                                 ▼
[Host Driver]                            [GPIO Subsystem]
     │                                    gpio_desc
     │ mmc_alloc_host()
     ▼
[struct mmc_host]
     │  slot.handler_priv
     └──────────────────► [struct mmc_gpio]
                               │ cd_gpio ──► gpio_desc ──► IRQ number
                               │ ro_gpio ──► gpio_desc
                               │
                          [IRQ Subsystem]
                               │ threaded IRQ
                               ▼
                          mmc_gpio_cd_irqt()
                               │ mmc_detect_change()
                               ▼
                          [MMC Core workqueue]
                               │ mmc_rescan()
                               ▼
                          [Card attach/detach logic]
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

| Function | Category | الغرض |
|---|---|---|
| `mmc_gpio_get_ro()` | Read State | قراءة حالة write-protect GPIO |
| `mmc_gpio_get_cd()` | Read State | قراءة حالة card-detect GPIO |
| `mmc_gpio_set_cd_irq()` | IRQ Config | تحديد رقم الـ IRQ يدوياً للـ CD |
| `mmc_gpiod_request_cd()` | Setup | طلب وتسجيل GPIO لكشف الكارت |
| `mmc_gpiod_request_ro()` | Setup | طلب وتسجيل GPIO لحماية الكتابة |
| `mmc_gpiod_set_cd_config()` | Config | ضبط pinconf على GPIO الـ CD |
| `mmc_gpio_set_cd_wake()` | Power Mgmt | تفعيل/إلغاء wakeup على IRQ الـ CD |
| `mmc_gpiod_request_cd_irq()` | IRQ Setup | تسجيل الـ IRQ handler للـ CD |
| `mmc_host_can_gpio_cd()` | Query | هل الـ host عنده CD GPIO مسجّل؟ |
| `mmc_host_can_gpio_ro()` | Query | هل الـ host عنده RO GPIO مسجّل؟ |

---

### البنية الداخلية المحورية: `struct mmc_gpio`

```c
struct mmc_gpio {
    struct gpio_desc *ro_gpio;          /* descriptor لـ write-protect pin */
    struct gpio_desc *cd_gpio;          /* descriptor لـ card-detect pin */
    irq_handler_t    cd_gpio_isr;       /* custom ISR أو mmc_gpio_cd_irqt */
    char             *ro_label;         /* اسم الـ pin في sysfs/debugfs */
    char             *cd_label;         /* اسم الـ pin في sysfs/debugfs */
    u32              cd_debounce_delay_ms; /* تأخير software debounce */
    int              cd_irq;            /* رقم IRQ المخصص يدوياً، أو -EINVAL */
};
```

هذه الـ struct هي الـ private context المخزّنة في `host->slot.handler_priv`.
كل الـ functions في هذا الـ header تعمل عليها بشكل مباشر.

---

### Group 1: Setup Functions — طلب وتسجيل الـ GPIOs

الغرض من المجموعة: إعداد الـ GPIO descriptors لكل من card-detect وwrite-protect
قبل إضافة الـ host للنظام عبر `mmc_add_host()`.

---

#### `mmc_gpiod_request_cd()`

```c
int mmc_gpiod_request_cd(struct mmc_host *host,
                          const char *con_id,
                          unsigned int idx,
                          bool override_active_level,
                          unsigned int debounce);
```

بتطلب الـ GPIO descriptor المخصص لكشف وجود الكارت وبتسجّله في الـ `mmc_gpio` context.
بتضبط الـ debounce إما على مستوى الـ hardware عبر `gpiod_set_debounce()` أو بتحفظ القيمة
لـ software debounce في الـ IRQ handler. لو الـ `override_active_level` مفعّل
أو `MMC_CAP2_CD_ACTIVE_HIGH` موجود، بتعكس الـ polarity عبر `gpiod_toggle_active_low()`.

**Parameters:**
| Parameter | الوصف |
|---|---|
| `host` | الـ MMC host المستهدف |
| `con_id` | اسم الـ function في الـ device tree (مثلاً `"cd"`)، أو NULL |
| `idx` | index الـ GPIO في الـ consumer list |
| `override_active_level` | لو `true`، يفرض active-low polarity بغض النظر عن الـ DT flags |
| `debounce` | وقت الـ debounce بالـ microseconds، صفر يعني إيقاف |

**Return:** `0` عند النجاح، أو errno سالب (مثلاً `-ENOENT` لو ما فيش GPIO).

**Key details:**
- لازم يتّستدعى **قبل** `mmc_add_host()`، وإلا لازم تستدعي `mmc_gpiod_request_cd_irq()` يدوياً بعدين.
- لو فشل `gpiod_set_debounce()` (الـ hardware ما يدعمش)، القيمة بتتحول لـ ms وتُستخدم كـ software delay.
- ما فيش locking داخلي — المفروض يُستدعى من init context.

**Who calls it:** MMC host drivers أثناء `probe()` مثل `sdhci-of-esdhc`.

---

#### `mmc_gpiod_request_ro()`

```c
int mmc_gpiod_request_ro(struct mmc_host *host,
                          const char *con_id,
                          unsigned int idx,
                          unsigned int debounce);
```

مشابه لـ `mmc_gpiod_request_cd()` لكن للـ write-protect GPIO. بتطلب الـ descriptor
وبتضبط الـ debounce. لو `MMC_CAP2_RO_ACTIVE_HIGH` موجود، بتعكس الـ polarity.
الفرق الجوهري إنه ما فيش `override_active_level` هنا، والـ error من `gpiod_set_debounce`
بيرجعه مباشرة (على خلاف الـ CD اللي بيعمل software fallback).

**Parameters:**
| Parameter | الوصف |
|---|---|
| `host` | الـ MMC host المستهدف |
| `con_id` | اسم الـ function في الـ DT، أو NULL |
| `idx` | index الـ GPIO |
| `debounce` | وقت الـ debounce بالـ microseconds |

**Return:** `0` عند النجاح، أو errno سالب.

**Key details:**
- لو `gpiod_set_debounce()` فشل هنا، الـ function بترجع error مباشرة — مش زي الـ CD اللي عنده fallback.
- ما فيش IRQ مرتبط بالـ RO، القراءة polling فقط.

**Who calls it:** Host drivers أثناء `probe()`.

---

### Group 2: IRQ Configuration — إعداد وتفعيل الـ Interrupts

الغرض من المجموعة: إدارة الـ interrupt المرتبط بـ card-detect GPIO، سواء بالتسجيل
التلقائي أو بتعيين رقم IRQ يدوي.

---

#### `mmc_gpio_set_cd_irq()`

```c
void mmc_gpio_set_cd_irq(struct mmc_host *host, int irq);
```

بتخزّن رقم IRQ مخصص في الـ `mmc_gpio` context. مفيدة لما تكون الـ GPIO line مشتركة
مع وحدة تانية وما تقدرش تستخدم `gpiod_to_irq()` العادي.

**Parameters:**
| Parameter | الوصف |
|---|---|
| `host` | الـ MMC host |
| `irq` | رقم الـ IRQ المخصص، لو أقل من 0 يتجاهله |

**Return:** void.

**Key details:**
- لو `ctx` هو NULL أو `irq < 0`، الـ function بتخرج بدون عمل أي حاجة.
- لازم تُستدعى قبل `mmc_gpiod_request_cd_irq()`.

**Who calls it:** Host drivers اللي عندها IRQ مخصص غير المشتق من الـ GPIO descriptor.

---

#### `mmc_gpiod_request_cd_irq()`

```c
void mmc_gpiod_request_cd_irq(struct mmc_host *host);
```

بتسجّل الـ threaded IRQ handler لـ card-detect. بتختار رقم الـ IRQ بالترتيب التالي:
أولاً IRQ المخزّن يدوياً في `ctx->cd_irq`، ثانياً `gpiod_to_irq()` لو ما فيش
`MMC_CAP_NEEDS_POLL`. لو فشل التسجيل أو ما لقاش IRQ، بتفعّل `MMC_CAP_NEEDS_POLL`.

```
mmc_gpiod_request_cd_irq():
  if (cd_irq already set OR no ctx OR no cd_gpio) → return
  if (ctx->cd_irq >= 0)
      irq = ctx->cd_irq           // manually set IRQ
  else if (!MMC_CAP_NEEDS_POLL)
      irq = gpiod_to_irq(cd_gpio) // derive from GPIO
  if (irq >= 0):
      set ISR = mmc_gpio_cd_irqt (if no custom ISR)
      devm_request_threaded_irq(RISING|FALLING|ONESHOT)
      if failed → irq = error code
  host->slot.cd_irq = irq
  if (irq < 0) → set MMC_CAP_NEEDS_POLL
```

**Parameters:**
| Parameter | الوصف |
|---|---|
| `host` | الـ MMC host |

**Return:** void.

**Key details:**
- بتستخدم `devm_request_threaded_irq()` يعني الـ IRQ بيتحرر تلقائياً عند الـ remove.
- الـ handler `mmc_gpio_cd_irqt` بيفعّل `host->trigger_card_event` وبيستدعي `mmc_detect_change()` مع الـ debounce delay.
- Flags: `IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING | IRQF_ONESHOT` — يعني threaded وعلى الحافتين.
- لو فشل التسجيل، النظام ينتقل تلقائياً لـ polling mode.

**Who calls it:** `mmc_add_host()` داخلياً، أو host driver يدوياً لو استدعى `mmc_gpiod_request_cd()` بعد `mmc_add_host()`.

---

### Group 3: Read State Functions — قراءة حالة الـ GPIOs

الغرض من المجموعة: استيفاء قيمة الـ GPIO الحالية لكل من card-detect وwrite-protect،
مع التعامل الصحيح مع الـ GPIOs اللي تحتاج sleep.

---

#### `mmc_gpio_get_ro()`

```c
int mmc_gpio_get_ro(struct mmc_host *host);
```

بتقرأ حالة الـ write-protect GPIO وبترجع قيمته المنطقية. بتتحقق أولاً هل الـ GPIO
يحتاج sleep عبر `gpiod_cansleep()`، وعلى حسب الإجابة بتستخدم النسخة المناسبة من
`gpiod_get_value()`.

**Parameters:**
| Parameter | الوصف |
|---|---|
| `host` | الـ MMC host |

**Return:**
- `0`: الكارت قابل للكتابة
- `1`: الكارت محمي من الكتابة (read-only)
- `-ENOSYS`: ما فيش RO GPIO مسجّل

**Key details:**
- ممكن تتستدعى من sleepable context بسبب الـ `gpiod_cansleep()` check.
- الـ polarity معالَجة تلقائياً من الـ GPIO descriptor (مش محتاج invert يدوي).

**Who calls it:** الـ MMC core عبر `host->ops->get_ro()` callback.

---

#### `mmc_gpio_get_cd()`

```c
int mmc_gpio_get_cd(struct mmc_host *host);
```

نفس منطق `mmc_gpio_get_ro()` لكن للـ card-detect GPIO. بتفحص وجود `ctx->cd_gpio`
وبتختار بين `gpiod_get_value()` و`gpiod_get_value_cansleep()`.

**Parameters:**
| Parameter | الوصف |
|---|---|
| `host` | الـ MMC host |

**Return:**
- `0`: ما فيش كارت
- `1`: في كارت
- `-ENOSYS`: ما فيش CD GPIO مسجّل

**Key details:**
- نفس خصائص الـ RO من ناحية الـ polarity والـ sleep handling.
- لو الـ host شغّال بـ polling mode، الـ MMC core بتستدعي هذه الـ function دورياً.

**Who calls it:** الـ MMC core عبر `host->ops->get_cd()` callback.

---

### Group 4: Configuration Functions — ضبط الـ GPIO Config

---

#### `mmc_gpiod_set_cd_config()`

```c
int mmc_gpiod_set_cd_config(struct mmc_host *host, unsigned long config);
```

بتطبّق pinconf configuration على الـ GPIO الخاص بـ card-detect بعد تسجيله.
مفيدة لضبط خصائص زي `PIN_CONFIG_BIAS_PULL_UP` اللي ما تتحددش في الـ DT.

**Parameters:**
| Parameter | الوصف |
|---|---|
| `host` | الـ MMC host |
| `config` | الـ packed pinconf config من `pinconf_to_config_packed()` |

**Return:** `0` عند النجاح، أو errno سالب من `gpiod_set_config()`.

**Key details:**
- لازم يُستدعى **بعد** `mmc_gpiod_request_cd()` وإلا `ctx->cd_gpio` هيكون NULL.
- بيمشي مباشرة على الـ `gpio_desc` بدون أي abstraction إضافي.

**Who calls it:** Host drivers اللي محتاجة pin bias أو drive strength خاص.

---

### Group 5: Power Management — إدارة الـ Wake-up

---

#### `mmc_gpio_set_cd_wake()`

```c
int mmc_gpio_set_cd_wake(struct mmc_host *host, bool on);
```

بتفعّل أو بتعطّل قدرة الـ card-detect IRQ على إيقاظ النظام من الـ suspend.
بتتحقق من ثلاث شروط قبل ما تعمل أي حاجة: الـ `MMC_CAP_CD_WAKE` موجود،
الـ IRQ مسجّل (>= 0)، وإن الحالة المطلوبة مختلفة عن الحالية.

```
mmc_gpio_set_cd_wake():
  if (!MMC_CAP_CD_WAKE OR cd_irq < 0 OR on == cd_wake_enabled)
      return 0   // no-op
  if (on):
      enable_irq_wake(cd_irq)
      cd_wake_enabled = (ret == 0)
  else:
      disable_irq_wake(cd_irq)
      cd_wake_enabled = false
```

**Parameters:**
| Parameter | الوصف |
|---|---|
| `host` | الـ MMC host |
| `on` | `true` لتفعيل الـ wakeup، `false` لتعطيله |

**Return:** `0` عند النجاح أو لو الشروط ما اتحققتش، أو errno من `enable_irq_wake()`.

**Key details:**
- بتتبع الحالة في `host->slot.cd_wake_enabled` لتفادي الـ redundant calls.
- لو `enable_irq_wake()` فشل، `cd_wake_enabled` يفضل `false`.
- `disable_irq_wake()` بيتستدعى بدون check على return value — intentional.

**Who calls it:** MMC core أثناء `suspend`/`resume` cycle.

---

### Group 6: Query Functions — استعلام عن الـ GPIO Availability

---

#### `mmc_host_can_gpio_cd()`

```c
bool mmc_host_can_gpio_cd(struct mmc_host *host);
```

بترجع `true` لو الـ host عنده card-detect GPIO مسجّل و descriptor صالح.

**Parameters:**
| Parameter | الوصف |
|---|---|
| `host` | الـ MMC host |

**Return:** `true` لو `ctx->cd_gpio != NULL`، `false` غير كده.

**Key details:** ما فيها locking — بتستدعى من context أمان بعد الـ init.

**Who calls it:** MMC core أو host drivers للتحقق قبل استدعاء `mmc_gpio_get_cd()`.

---

#### `mmc_host_can_gpio_ro()`

```c
bool mmc_host_can_gpio_ro(struct mmc_host *host);
```

نفس `mmc_host_can_gpio_cd()` لكن للـ write-protect GPIO.

**Parameters:**
| Parameter | الوصف |
|---|---|
| `host` | الـ MMC host |

**Return:** `true` لو `ctx->ro_gpio != NULL`، `false` غير كده.

**Who calls it:** MMC core أو host drivers للتحقق قبل استدعاء `mmc_gpio_get_ro()`.

---

### الـ IRQ Handler الداخلي: `mmc_gpio_cd_irqt()`

```c
/* internal — not exported */
static irqreturn_t mmc_gpio_cd_irqt(int irq, void *dev_id)
{
    struct mmc_host *host = dev_id;
    struct mmc_gpio *ctx = host->slot.handler_priv;

    host->trigger_card_event = true;
    /* schedule detection after debounce delay */
    mmc_detect_change(host, msecs_to_jiffies(ctx->cd_debounce_delay_ms));

    return IRQ_HANDLED;
}
```

الـ default threaded ISR للـ card-detect. بتضع `trigger_card_event` flag وبتستدعي
`mmc_detect_change()` مع delay لإتاحة الوقت للـ debounce. الـ default delay هو 200ms.

---

### تدفق الـ Initialization الكامل

```
host driver probe():
    mmc_alloc_host()
        └─ mmc_gpio_alloc()          // allocates mmc_gpio, sets cd_irq = -EINVAL
    mmc_gpiod_request_cd(...)        // gets cd GPIO descriptor, sets debounce
    mmc_gpio_set_cd_irq(...)         // optional: override IRQ number
    mmc_gpiod_request_ro(...)        // gets ro GPIO descriptor
    mmc_gpiod_set_cd_config(...)     // optional: set pin pull-up etc.
    mmc_add_host()
        └─ mmc_gpiod_request_cd_irq()  // registers IRQ handler OR sets NEEDS_POLL

runtime (card insert/remove):
    IRQ fires → mmc_gpio_cd_irqt()
        └─ mmc_detect_change() → MMC core re-scans the slot

suspend/resume:
    mmc_gpio_set_cd_wake(host, true)   // enable wakeup
    mmc_gpio_set_cd_wake(host, false)  // disable wakeup
```
## Phase 5: دليل الـ Debugging الشامل

> **Subsystem**: MMC Slot GPIO — card-detect (CD) + read-only (RO) via GPIO
> **Header**: `include/linux/mmc/slot-gpio.h`

---

## Software Level

### 1. debugfs Entries

| Entry | Path | الوصف |
|-------|------|--------|
| `state` | `/sys/kernel/debug/mmc0/state` | حالة الـ host (idle/transfer/...) |
| `ios` | `/sys/kernel/debug/mmc0/ios` | إعدادات الـ bus الحالية |
| `clock` | `/sys/kernel/debug/mmc0/ios` | تضمن الـ clock المفعّل |
| `cd_gpio` | `/sys/kernel/debug/gpio` | حالة كل GPIO بما فيها CD/RO |

```bash
# قراءة حالة الـ MMC host
cat /sys/kernel/debug/mmc0/ios

# قراءة حالة كل الـ GPIOs (فيها CD و RO)
cat /sys/kernel/debug/gpio | grep -A2 -i "mmc\|cd\|ro"

# مشاهدة state الـ slot
cat /sys/kernel/debug/mmc0/state
```

**مثال output لـ gpio debugfs:**
```
GPIOs 400-415, platform/gpio-controller, gpio-controller:
 gpio-404 (                    |cd              ) in  hi IRQ ACTIVE LOW
 gpio-405 (                    |ro              ) in  lo
```
- `hi` = الـ card مش موجودة (active low)
- `lo` = الـ card موجودة (CD active)
- `ACTIVE LOW` = الـ polarity معكوسة — طبيعي في أغلب slots

---

### 2. sysfs Entries

| Entry | Path | الوصف |
|-------|------|--------|
| `present` | `/sys/bus/mmc/devices/mmc0:*/` | هل كارت موجود |
| `ro` | `/sys/block/mmcblk0/ro` | هل الكارت read-only |
| `gpio direction` | `/sys/class/gpio/gpio<N>/direction` | اتجاه الـ GPIO |
| `gpio value` | `/sys/class/gpio/gpio<N>/value` | قيمة الـ GPIO الحالية |

```bash
# قراءة حالة كل الـ MMC devices
ls /sys/bus/mmc/devices/

# التحقق من وجود كارت
cat /sys/bus/mmc/devices/mmc0\:*/type 2>/dev/null

# قراءة GPIO value يدوياً (لو exported)
# أولاً: اعرف رقم GPIO من debugfs
GPIO_NUM=404
echo $GPIO_NUM > /sys/class/gpio/export 2>/dev/null
cat /sys/class/gpio/gpio${GPIO_NUM}/value
cat /sys/class/gpio/gpio${GPIO_NUM}/direction

# الـ ro لـ block device
cat /sys/block/mmcblk0/ro
```

---

### 3. ftrace — Tracepoints/Events

```bash
# تفعيل tracing للـ MMC subsystem
mount -t tracefs none /sys/kernel/tracing 2>/dev/null

# عرض كل الـ MMC events المتاحة
ls /sys/kernel/tracing/events/mmc/

# تفعيل أهم events للـ GPIO slot
echo 1 > /sys/kernel/tracing/events/mmc/mmc_request_start/enable
echo 1 > /sys/kernel/tracing/events/mmc/mmc_request_done/enable

# تفعيل GPIO events (لمتابعة CD/RO)
echo 1 > /sys/kernel/tracing/events/gpio/enable

# تفعيل IRQ events (لمتابعة CD interrupt)
echo 1 > /sys/kernel/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/tracing/events/irq/irq_handler_exit/enable

# تشغيل الـ trace
echo 1 > /sys/kernel/tracing/tracing_on

# أدخل/أخرج الكارت، ثم اقرأ
cat /sys/kernel/tracing/trace | grep -E "mmc|gpio|cd"

# إيقاف
echo 0 > /sys/kernel/tracing/tracing_on
```

**مثال output:**
```
     kworker/0:2-45    [000] ....  1234.567890: gpio_value: gpio=404 get value=0
     kworker/0:2-45    [000] ....  1234.567920: mmc_request_start: mmc0: cmd=1
```
- `gpio_value: get value=0` بعد إدخال كارت = CD GPIO انخفض (active low → card present)

---

### 4. printk / Dynamic Debug

```bash
# تفعيل dynamic debug لـ MMC slot-gpio subsystem
echo "file slot-gpio.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file mmc.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file host.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لكل ملفات الـ MMC
echo "module mmc_core +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لـ GPIO subsystem
echo "module gpio_sysfs +p" > /sys/kernel/debug/dynamic_debug/control

# مشاهدة الـ output في real-time
dmesg -w | grep -E "mmc|gpio|cd|ro"

# أو عبر kernel log level
dmesg --level=debug | grep mmc
```

**نقاط مهمة في `slot-gpio.c`:**
- `mmc_gpio_get_cd()` — بترجع `0`=card present, `1`=absent, أو `-ENOSYS` لو مفيش GPIO
- `mmc_gpio_get_ro()` — بترجع `0`=writable, `1`=read-only, أو `-ENOSYS`
- `mmc_gpiod_request_cd_irq()` — بتسجل الـ IRQ للـ card-detect

---

### 5. Kernel Config Options للـ Debugging

| Config | الوصف |
|--------|--------|
| `CONFIG_MMC_DEBUG` | تفعيل debug messages في الـ MMC core |
| `CONFIG_DEBUG_GPIO` | تفعيل GPIO debugging و validation |
| `CONFIG_GPIOLIB_IRQCHIP` | دعم IRQ عبر GPIO (لازم للـ CD interrupt) |
| `CONFIG_MMC_SDHCI_OF_ARASAN` | debugging لـ SDHCI controllers شائعة |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug للـ `pr_debug()` |
| `CONFIG_TRACING` | أساس الـ ftrace |
| `CONFIG_MMC_UNSAFE_RESUME` | لـ debugging مشاكل suspend/resume مع CD |
| `CONFIG_PINCTRL_SINGLE` | debugging لـ pin configuration |

```bash
# تحقق من الـ config الحالي
grep -E "CONFIG_MMC_DEBUG|CONFIG_DEBUG_GPIO|CONFIG_GPIOLIB" /boot/config-$(uname -r)
```

---

### 6. جدول Common Error Messages

| رسالة الـ kernel | المعنى | الحل |
|------------------|--------|-------|
| `mmc0: error -110 whilst initialising SD card` | timeout — الكارت مش بيرد | تحقق CD GPIO قيمته صح، تأكد الكارت موجود فعلاً |
| `mmc0: card never left busy state` | الكارت stuck في busy | تحقق الـ power supply للـ slot |
| `mmc_gpio_get_cd: failed, ret=-22` | `-EINVAL` — الـ GPIO descriptor غلط | راجع DTS: `cd-gpios` property صح؟ |
| `Unable to obtain gpiod 'cd'` | الـ GPIO مش متعرّف | تحقق من `con_id` في `mmc_gpiod_request_cd()` |
| `mmc0: got irq -1` | الـ CD IRQ مش شغال | تحقق `mmc_gpio_set_cd_irq()` أو الـ DTS `interrupts` |
| `mmc0: cd gpio irq request failed` | فشل تسجيل الـ IRQ للـ CD | تحقق الـ GPIO يدعم interrupt، راجع `gpio_to_irq()` |
| `mmc0: readonly status changed` | RO GPIO تغيّر | طبيعي لو الكارت اتحطّ/اتشال، أو مشكلة RO GPIO noise |
| `mmc_gpiod_set_cd_config: -524` | `-ENOTSUPP` — الـ GPIO driver مش بيدعم config | جرب pinctrl بدل gpiod config |

---

## Hardware Level

### 1. التحقق إن الـ Hardware State مطابق للـ Kernel State

```bash
# الخطوة 1: اعرف رقم GPIO الخاص بالـ CD
# من DTS أو من debugfs
cat /sys/kernel/debug/gpio | grep -i "cd\|card.detect"

# الخطوة 2: قيس الـ GPIO فيزيائياً بالـ multimeter
# Pin يجب يكون ~0V لما الكارت موجود (active-low CD)
# Pin يجب يكون ~3.3V لما مفيش كارت

# الخطوة 3: قارن مع kernel
GPIO_NUM=404
cat /sys/kernel/debug/gpio | grep "gpio-${GPIO_NUM}"
# لو "hi" و الكارت موجود → polarity problem في DTS
# لو "lo" و الكارت موجود → طبيعي (active-low)

# الخطوة 4: تحقق من الـ IRQ assignment
cat /proc/interrupts | grep -i "mmc\|cd"
```

**ASCII Diagram — CD GPIO Circuit:**
```
Card Slot                GPIO Pin
    |                       |
    |--[CD Switch]--[Pull-up R]--VCC (3.3V)
    |                       |
   GND               GPIO input (kernel reads this)

Card absent  → Switch open  → GPIO = HIGH (1) = no card
Card present → Switch closed → GPIO = LOW  (0) = card present
```

---

### 2. Register Dump Techniques

```bash
# قراءة GPIO controller registers (مثال: Raspberry Pi / BCM)
# عنوان الـ GPIO base من DTS أو /proc/iomem
cat /proc/iomem | grep -i gpio

# devmem2 لقراءة registers مباشرة (لازم تثبّته)
# GPIO_BASE مثلاً 0xFE200000 لـ BCM2711
GPIO_BASE=0xFE200000
devmem2 $GPIO_BASE 32          # GPFSEL0 - Function Select
devmem2 $((GPIO_BASE + 0x34)) 32  # GPLEV0 - Pin Level

# لـ i.MX أو OMAP — استخدم /dev/mem مع dd
# (محتاج CONFIG_DEVMEM=y)

# regmap debugfs (لو الـ GPIO controller بيستخدمه)
ls /sys/kernel/debug/regmap/
```

---

### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس:**
```
Signal      Pin         ما تدور عليه
--------    -------     -----------------------
CD          GPIO_CD     يتغير عند إدخال/إخراج كارت
RO          GPIO_RO     ثابت لما الكارت موجود
VDD_MMC     Power pin   3.3V ثابتة، ripple < 50mV
CLK         CLK pin     بيظهر لما في transfer فقط
```

**إعدادات Logic Analyzer:**
- **Sample rate**: 1 MHz كافي للـ CD/RO (إشارات بطيئة)
- **Trigger**: rising/falling edge على الـ CD pin
- **Debounce time**: راقب الـ bouncing — الـ kernel `debounce` parameter بـ `mmc_gpiod_request_cd()` المفروض يلغيه
- **Capture duration**: 100ms كافي لرصد الـ debounce window

**مثال مشكلة Debouncing:**
```
CD pin (noisy):
    ____    _  __    _________
   |    |__|  |  |__|
   ← bouncing ~5ms →  ← stable

CD pin (after debounce 20ms):
                          ____
_________________________|
                         ↑ kernel sees card insert here
```

---

### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | Pattern في الـ kernel log | التحقق |
|---------|--------------------------|--------|
| CD GPIO لا يتغير عند إدخال كارت | لا يوجد `mmc0: new SD card` | قِس الـ GPIO فيزيائياً بـ multimeter |
| Debounce قصير — false detects | `mmc0: card removed` ثم `mmc0: new SD card` بسرعة | زوّد `debounce` parameter في DTS |
| RO GPIO عكسي | الكارت writable بس kernel يقول read-only | افحص `GPIO_ACTIVE_LOW` في DTS |
| Pull-up مفقود | `mmc0: error -110` متكرر بدون سبب واضح | قِس voltage على CD pin بدون كارت (لازم 3.3V) |
| Ground loop / noise | IRQ كتير بدون كارت | راقب الـ CD pin بـ oscilloscope |
| Power sequencing خاطئ | `mmc0: card never left busy state` | افحص VDD_MMC يطلع قبل CLK |

---

## Practical Commands

### مجموعة كاملة جاهزة للـ copy-paste

```bash
#!/bin/bash
# === MMC Slot GPIO Debug Script ===

echo "=== 1. GPIO States ==="
cat /sys/kernel/debug/gpio | grep -E -A1 "mmc|cd|ro" 2>/dev/null || echo "debugfs not mounted"

echo ""
echo "=== 2. MMC Host IOS ==="
for host in /sys/kernel/debug/mmc*; do
    echo "--- $host ---"
    cat "$host/ios" 2>/dev/null
done

echo ""
echo "=== 3. MMC Devices ==="
ls /sys/bus/mmc/devices/ 2>/dev/null

echo ""
echo "=== 4. Block Device RO Status ==="
for dev in /sys/block/mmcblk*; do
    echo "$dev/ro: $(cat $dev/ro 2>/dev/null)"
done

echo ""
echo "=== 5. IRQ Assignment ==="
cat /proc/interrupts | grep -i mmc

echo ""
echo "=== 6. Recent dmesg (MMC) ==="
dmesg | grep -i mmc | tail -30

echo ""
echo "=== 7. Dynamic Debug — Enable MMC slot-gpio ==="
echo "file slot-gpio.c +p" > /sys/kernel/debug/dynamic_debug/control 2>/dev/null && \
    echo "slot-gpio debug enabled" || echo "dynamic_debug not available"
```

---

### تفعيل ftrace للـ CD events خطوة بخطوة

```bash
# تأكد tracefs مـ mounted
mount | grep tracefs || mount -t tracefs none /sys/kernel/tracing

cd /sys/kernel/tracing

# صفّر الـ trace
echo 0 > tracing_on
echo > trace

# فعّل GPIO + MMC events
echo 1 > events/gpio/enable
echo 1 > events/mmc/enable 2>/dev/null

# ابدأ التسجيل
echo 1 > tracing_on

# === أدخل/أخرج الكارت هنا ===
sleep 5

# أوقف وقرأ
echo 0 > tracing_on
cat trace | grep -E "gpio|mmc" | head -50
```

**مثال output مع تفسير:**
```
# الكارت اتحط
  kworker/u4:2-89   [000] d...  5678.123: gpio_value: gpio=404 get value=0
  kworker/u4:2-89   [000] ....  5678.124: mmc_request_start: mmc0: cmd=0 flags=0x00b5
  kworker/u4:2-89   [000] ....  5678.156: mmc_request_done: mmc0: cmd=0 err=0

# gpio=404 value=0 → CD GPIO انخفض → kernel بدأ initialization
# cmd=0 → GO_IDLE_STATE (RESET)
# err=0 → ناجح
```

---

### سيناريو: كارت بيتعرف غلط (false positive)

```bash
# 1. تحقق من الـ debounce الحالي
grep -r "debounce" /sys/kernel/debug/gpio 2>/dev/null

# 2. راقب الـ CD IRQ count
watch -n 0.5 'cat /proc/interrupts | grep mmc'

# 3. لو الـ IRQ count بيزيد بسرعة بدون كارت = noise
# الحل: زوّد debounce في DTS
# cd-gpios = <&gpio 404 GPIO_ACTIVE_LOW>;
# debounce-delay-ms = <20>;   ← زوّدها

# 4. اختبار بعد تغيير DTS (بدون recompile) عبر configfs لو متاح
# أو عبر:
echo "file slot-gpio.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg -w | grep "debounce\|cd\|gpio"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### سيناريو 1: Industrial Gateway على RK3562 — card-detect مش شغال خالص

**العنوان:** SD card مش بتتعرف على industrial gateway بيشتغل على RK3562

**السياق:**
شركة بتعمل industrial IoT gateway بيشتغل على RK3562، الـ gateway بيحتاج SD card عشان يخزن logs. المنتج وصل لمرحلة الـ mass production وفجأة اكتشفوا إن بعض الوحدات مش بتعرف الـ SD card خالص حتى لو هي موجودة فعلاً.

**المشكلة:**
الـ kernel مش بيطلع interrupt لما بتحط الـ card. `dmesg` بيبين:
```
mmc0: no support for card detection; polling
```
ده معناه الـ host مش عارف يستخدم GPIO interrupt للـ card-detect.

**التحليل:**
الكود في `slot-gpio.h` بيعرّف `mmc_gpiod_request_cd_irq()` و`mmc_host_can_gpio_cd()`. الـ flow بيكون:

```c
/* host driver calls this after requesting the CD GPIO */
mmc_gpiod_request_cd_irq(host);
/*
 * internally checks if host->slot.cd_gpio is valid
 * if not valid → falls back to polling
 */
```

المشكلة إن `mmc_gpiod_request_cd()` اتنادى بـ `con_id = NULL` فبدل ما يدور على `"cd"` في الـ Device Tree، رجع `-ENOENT` بصمت وما حدش لاحظ.

**التحليل التفصيلي:**
```c
int mmc_gpiod_request_cd(struct mmc_host *host,
                         const char *con_id,  /* should be "cd" */
                         unsigned int idx,
                         bool override_active_level,
                         unsigned int debounce);
```

لو `con_id` بـ `NULL`، الـ gpiod subsystem هيدور على default label مش `"cd"` — النتيجة: GPIO مش بيتطلب، الـ IRQ مش بيتسجل، الـ polling بيبدأ.

**الحل:**
```c
/* في الـ driver الخاص بالـ host controller */
ret = mmc_gpiod_request_cd(host,
                           "cd",   /* MUST match DT property: cd-gpios */
                           0,
                           false,
                           200);   /* 200ms debounce */
if (ret && ret != -ENOENT)
    dev_err(host->parent, "failed to request CD GPIO: %d\n", ret);
```

وفي الـ Device Tree:
```dts
&sdmmc0 {
    cd-gpios = <&gpio0 RK_PA4 GPIO_ACTIVE_LOW>;
    /* debounce handled in driver */
};
```

**الدرس المستفاد:**
`con_id` في `mmc_gpiod_request_cd()` لازم يطابق بالظبط اسم الـ property في الـ DT من غير الـ `-gpios` suffix. غلطة صغيرة بتحول الـ interrupt-driven detection لـ polling وده بيأكل CPU بلاش.

---

### سيناريو 2: Android TV Box على Allwinner H616 — write-protect غلط

**العنوان:** SD card بتتعامل كـ read-only على TV box رغم إنها مش محمية

**السياق:**
منتج Android TV box يشتغل على Allwinner H616 SoC. المستخدمين بيشتكوا إن الـ SD card دايماً read-only حتى لو الـ physical write-protect switch مش موجود أو off. المنتج بيستخدم SD card عشان يحدّث الـ firmware.

**المشكلة:**
```
mmc0: card is write-protected
```
الرسالة دي بتظهر حتى مع كارت جديد مفيهوش write-protect switch.

**التحليل:**
الـ header بيعرّف:
```c
int mmc_gpio_get_ro(struct mmc_host *host);
bool mmc_host_can_gpio_ro(struct mmc_host *host);
```

الـ `mmc_gpio_get_ro()` بترجع:
- `0` = كارت مش read-only
- `1` = كارت read-only
- `< 0` = مفيش GPIO للـ RO → الـ host يقرر بنفسه

المشكلة: الـ DT فيه:
```dts
wp-gpios = <&pio 5 6 GPIO_ACTIVE_HIGH>;
```

بس الـ GPIO رقم 6 في الـ H616 مش connected فعلاً لأي حاجة، فقيمته floating وعشان الـ pull-up الداخلي، بيرجع دايماً HIGH = write-protected.

**التحليل التفصيلي:**
```c
int mmc_gpiod_request_ro(struct mmc_host *host,
                         const char *con_id,   /* "wp" */
                         unsigned int idx,
                         unsigned int debounce);
```

لما الـ `mmc_gpio_get_ro()` بتتنادى:
```c
/* simplified internal logic */
if (!mmc_host_can_gpio_ro(host))
    return -ENOSYS; /* no RO GPIO → host decides */

return gpiod_get_value_cansleep(host->slot.ro_gpio);
/* floating pin + internal pull-up → always returns 1 → READ-ONLY */
```

**الحل:**
خيارين:
1. لو مفيش write-protect switch فعلي، شيل الـ `wp-gpios` من الـ DT تماماً
2. لو الـ hardware معاه switch، صلح الـ GPIO رقم والـ active level:

```dts
/* Option 1: no WP switch on this board */
/* simply remove wp-gpios from DT */

/* Option 2: correct GPIO if switch exists */
wp-gpios = <&pio 5 6 GPIO_ACTIVE_LOW>;
/*                    ^^^^^^^^^^^^^^ adjust polarity */
```

للتأكد قبل الحل:
```bash
# check current RO GPIO state
cat /sys/kernel/debug/gpio | grep mmc
# or
gpioget gpiochip0 6
```

**الدرس المستفاد:**
لو الـ hardware مالوش write-protect switch، متحطش `wp-gpios` في الـ DT خالص. الـ `mmc_host_can_gpio_ro()` بيرجع `false` لو مفيش GPIO مسجّل، وده الـ behavior الصح للـ boards اللي مالهاش هذا الـ feature.

---

### سيناريو 3: IoT Sensor على STM32MP1 — wakeup من SD card مش شغال

**العنوان:** الجهاز مش بيصحى من sleep لما بتحط SD card على STM32MP1

**السياق:**
جهاز IoT sensor صناعي بيشتغل على STM32MP157. الجهاز بيدخل في deep sleep عشان يوفر طاقة. المطلوب إن لما تحط SD card، الجهاز يصحى أوتوماتيك ويقرا البيانات. ده كان شغال على نسخة قديمة من الـ kernel بس بعد الـ upgrade بطّل.

**المشكلة:**
الجهاز مش بيصحى من `suspend` لما تحط card. اللوج بيبين إن الـ interrupt بيحصل بس الـ wakeup source مش مسجّل.

**التحليل:**
الـ header بيعرّف:
```c
int mmc_gpio_set_cd_wake(struct mmc_host *host, bool on);
```

الـ function دي مسؤولة إنها تسجّل/تلغي الـ CD GPIO كـ wakeup source للـ system. بعد الـ kernel upgrade، الـ driver الـ STM32 SDMMC بطّل يتنادى `mmc_gpio_set_cd_wake()` في الـ suspend path.

**التحليل التفصيلي:**
```c
/* Expected flow in driver's suspend callback */
static int stm32_sdmmc_suspend(struct device *dev)
{
    struct mmc_host *host = dev_get_drvdata(dev);

    /* This enables the CD GPIO as a wakeup source */
    mmc_gpio_set_cd_wake(host, true);
    /*
     * internally calls:
     * enable_irq_wake(host->slot.cd_irq)
     * which tells the PM subsystem this IRQ can wake the system
     */
    return 0;
}

static int stm32_sdmmc_resume(struct device *dev)
{
    struct mmc_host *host = dev_get_drvdata(dev);

    /* Must disable wakeup after resume */
    mmc_gpio_set_cd_wake(host, false);
    return 0;
}
```

بعد الـ upgrade، الـ `pm_ops` اتغيرت والـ suspend callback مش بيتنادى بالـ order الصح.

**الحل:**
```c
/* Ensure mmc_gpio_set_cd_wake is called properly */
static const struct dev_pm_ops stm32_sdmmc_pm_ops = {
    SET_SYSTEM_SLEEP_PM_OPS(stm32_sdmmc_suspend,
                            stm32_sdmmc_resume)
};

/* In DT, mark CD GPIO as wakeup capable */
```

```dts
/* STM32MP1 DT fix */
&sdmmc1 {
    cd-gpios = <&gpiob 7 GPIO_ACTIVE_LOW>;
    wakeup-source;  /* mark as wakeup source */
};
```

```bash
# Verify wakeup source is registered
cat /sys/bus/platform/devices/58005000.sdmmc/power/wakeup
# should show "enabled"
```

**الدرس المستفاد:**
`mmc_gpio_set_cd_wake()` لازم تتنادى صراحةً في الـ suspend/resume callbacks. مش بتحصل أوتوماتيك. كمان `wakeup-source` في الـ DT ضروري عشان الـ PM framework يعرف إن الـ GPIO ده مسموح يصحى النظام.

---

### سيناريو 4: Automotive ECU على i.MX8 — debounce غلط بيسبب false card detects

**العنوان:** ECU بيشوف SD card بتتوصل وتتفصل بسرعة على i.MX8QM

**السياق:**
automotive ECU بيشتغل على i.MX8QM في سيارة. الـ ECU بيستخدم SD card لتسجيل بيانات الـ CAN bus. في البيئة الـ automotive، في vibration عالي وتقلبات في الـ voltage. المنتج بدأ يسجّل false "card inserted/removed" events في الـ log اللي بتسبب data corruption.

**المشكلة:**
```
mmc0: card removed
mmc0: new SDHC card at address 0001
mmc0: card removed
mmc0: new SDHC card at address 0001
```
ده بيكرر كل بضع ثواني حتى لو الـ card ثابتة جوه الـ slot.

**التحليل:**
الـ header بيعرّف:
```c
int mmc_gpiod_request_cd(struct mmc_host *host,
                         const char *con_id,
                         unsigned int idx,
                         bool override_active_level,
                         unsigned int debounce);  /* in microseconds */
```

الـ `debounce` parameter بيتبعت للـ GPIO subsystem عشان يطفّش الـ noise. في الكود الحالي:

```c
/* current driver code — too short debounce */
ret = mmc_gpiod_request_cd(host, "cd", 0, false,
                           200);  /* 200 microseconds — too short! */
```

الـ vibration في البيئة الـ automotive بيخلي الـ CD pin يـ bounce لمدة أطول من 200µs.

**التحليل التفصيلي:**
```c
/*
 * mmc_gpiod_request_cd internally calls:
 * gpiod_set_debounce(desc, debounce)
 * if hardware debounce not available, kernel does software debounce
 * 200µs is insufficient for automotive vibration environments
 */
```

كمان الـ `override_active_level = false` معناه إن الـ driver بيثق في الـ DT لـ active level. لو الـ DT غلط، كل حاجة بتتقلب.

**الحل:**
```c
/* Fix 1: increase debounce significantly */
ret = mmc_gpiod_request_cd(host, "cd", 0, false,
                           10000); /* 10ms — suitable for automotive */

/* Fix 2: verify active level in DT */
```

```dts
/* i.MX8 automotive ECU DT */
&usdhc2 {
    cd-gpios = <&gpio2 12 GPIO_ACTIVE_LOW>;
    /* card inserted = GPIO low = active */
};
```

```bash
# Monitor card detect events
udevadm monitor --kernel --subsystem-match=block

# Check GPIO debounce (if hardware supports it)
cat /sys/kernel/debug/gpio
```

**الدرس المستفاد:**
الـ `debounce` في `mmc_gpiod_request_cd()` بيتقاس بـ microseconds مش milliseconds. في البيئات الـ automotive والـ industrial اللي فيها vibration، استخدم 10000 (10ms) على الأقل. الـ 200µs default مناسب للـ consumer electronics بس.

---

### سيناريو 5: Custom Board Bring-up على AM62x — CD GPIO بـ active level معكوس

**العنوان:** SD card مش بتتعرف على custom AM62x board أثناء الـ bring-up

**السياق:**
فريق hardware design بيعمل bring-up لـ custom board يشتغل على TI AM625 SoC. الـ schematic فيه CD GPIO متوصل بـ active-high logic (card inserted = GPIO HIGH) بس الـ DT template المنقول من reference design تاني كان فيه active-low.

**المشكلة:**
الـ card موجودة في الـ slot بس:
```
mmc0: no card present
```
وكمان لو شلت الـ card:
```
mmc0: card inserted
```
كل حاجة معكوسة!

**التحليل:**
الـ header بيعرّف:
```c
int mmc_gpiod_request_cd(struct mmc_host *host,
                         const char *con_id,
                         unsigned int idx,
                         bool override_active_level, /* key parameter! */
                         unsigned int debounce);
```

الـ `override_active_level` parameter ده المفتاح. لو `true`، الـ driver بيتجاهل الـ active level في الـ DT ويستخدم الـ hardware detection من الـ host controller نفسه.

**التحليل التفصيلي:**
```c
/*
 * Case 1: DT says GPIO_ACTIVE_LOW but hardware is ACTIVE_HIGH
 * → mmc_gpio_get_cd() returns inverted result
 * → card present = 0, card absent = 1 (WRONG!)
 */

/* mmc_gpio_get_cd() internal simplified logic: */
int mmc_gpio_get_cd(struct mmc_host *host)
{
    if (!mmc_host_can_gpio_cd(host))
        return -ENOSYS;

    /*
     * gpiod_get_value_cansleep respects active low/high
     * from DT — if DT is wrong, result is inverted
     */
    return gpiod_get_value_cansleep(host->slot.cd_gpio);
}
```

الحل الأول غلط كان:
```c
/* WRONG approach during bring-up */
ret = mmc_gpiod_request_cd(host, "cd", 0,
                           true,   /* override active level */
                           200);
/* This ignores DT active level entirely — risky! */
```

**الحل الصح:**
```c
/* CORRECT: fix the DT to match hardware */
ret = mmc_gpiod_request_cd(host, "cd", 0,
                           false,  /* trust DT */
                           200);
```

```dts
/* AM62x custom board — fix active level to match schematic */
&sdhci1 {
    /* Hardware: card inserted = GPIO HIGH */
    cd-gpios = <&main_gpio0 14 GPIO_ACTIVE_HIGH>;
    /*                          ^^^^^^^^^^^^^^^^ changed from ACTIVE_LOW */
};
```

للتحقق السريع أثناء الـ bring-up:
```bash
# Read raw GPIO value while card is inserted
gpioget gpiochip0 14
# Should return 1 if ACTIVE_HIGH and card is present

# Read through MMC subsystem
cat /sys/bus/mmc/devices/mmc0:0001/state
```

لو لازم تـ override مؤقتاً أثناء الـ debugging:
```c
/*
 * Use override_active_level=true ONLY for debugging
 * to bypass DT active level — never in production code
 */
ret = mmc_gpiod_request_cd(host, "cd", 0, true, 200);
```

**الدرس المستفاد:**
`override_active_level` في `mmc_gpiod_request_cd()` أداة debugging مش production feature. الـ fix الصح دايماً في الـ DT. أثناء الـ bring-up، اتأكد إن الـ `GPIO_ACTIVE_HIGH/LOW` في الـ DT بيطابق الـ schematic قبل ما تلوم الـ driver.

---
## Phase 7: مصادر ومراجع

---

### 1. مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [Add dynamic MMC-over-SPI-GPIO driver](https://lwn.net/Articles/290066/) | إضافة driver ديناميكي لـ MMC فوق SPI-GPIO — يشرح ربط MMC بـ GPIO عام |
| [Documentation/gpio.txt](https://lwn.net/Articles/532717/) | الوثيقة الأصلية لـ GPIO legacy API في الـ kernel |
| [Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/) | توثيق الـ gpiod descriptor API الجديد اللي اتبنت عليه `mmc_gpiod_*` |

---

### 2. توثيق الـ Kernel الرسمي

| المسار | المحتوى |
|--------|---------|
| `Documentation/driver-api/mmc/index.rst` | نقطة البداية لكل وثائق MMC subsystem |
| `Documentation/driver-api/mmc/mmc-tools.rst` | أدوات MMC واختبار الـ slots |
| `Documentation/driver-api/gpio/index.rst` | مدخل GPIO driver API |
| `Documentation/driver-api/gpio/driver.rst` | كيفية كتابة GPIO driver — يشمل debounce و IRQ |
| `Documentation/driver-api/gpio/board.rst` | GPIO Mappings — ربط الـ GPIO بـ Device Tree |
| `Documentation/driver-api/gpio/intro.rst` | مقدمة GPIO في الـ kernel |
| `Documentation/driver-api/gpio/legacy.rst` | Legacy GPIO API (قبل gpiod) |
| `Documentation/driver-api/gpio/drivers-on-gpio.rst` | subsystems تستخدم GPIO — يذكر MMC صراحةً |

روابط مباشرة (static.lwn.net mirror):
- [MMC/SD/SDIO card support](https://static.lwn.net/kerneldoc/driver-api/mmc/index.html)
- [Subsystem drivers using GPIO](https://static.lwn.net/kerneldoc/driver-api/gpio/drivers-on-gpio.html)
- [GPIO Driver Interface](https://static.lwn.net/kerneldoc/driver-api/gpio/driver.html)
- [GPIO Mappings (board.rst)](https://static.lwn.net/kerneldoc/driver-api/gpio/board.html)
- [Legacy GPIO Interfaces](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html)
- [GPIO Introduction](https://static.lwn.net/kerneldoc/driver-api/gpio/intro.html)

---

### 3. الملفات المصدرية المباشرة

```
include/linux/mmc/slot-gpio.h   <-- الـ header موضوع الدراسة
drivers/mmc/core/slot-gpio.c    <-- التنفيذ الكامل
drivers/mmc/core/host.c         <-- mmc_host allocation وربط الـ GPIO
include/linux/mmc/host.h        <-- struct mmc_host تعريف
```

- [slot-gpio.c على GitHub (torvalds/linux)](https://github.com/torvalds/linux/blob/master/drivers/mmc/core/slot-gpio.c)

---

### 4. Patches وكوميتات مهمة

| الـ Patch | الأهمية |
|-----------|---------|
| [mmc: rework cd-gpio handling](https://patchwork.kernel.org/patch/11280435/) | توحيد منطق invert لـ CD line في `mmc_gpiod_request_cd()` |
| [mmc: slot-gpio: add gpiod variant to get wp GPIO](https://patchwork.ozlabs.org/project/linux-gpio/patch/1409137253-25189-2-git-send-email-linus.walleij@linaro.org/) | إضافة `mmc_gpiod_request_ro()` — الانتقال من legacy لـ descriptor API |
| [mmc: s3cmci: Use the slot GPIO descriptor](https://patchwork.kernel.org/project/linux-mmc/patch/20181115232145.22130-1-linus.walleij@linaro.org/) | مثال عملي لمنصة تنتقل للـ gpiod API |
| [gpio: OF: Parse MMC-specific CD and WP properties](https://patchwork.kernel.org/project/linux-mmc/patch/20181126141714.18399-1-linus.walleij@linaro.org/) | parsing `cd-gpios` و`wp-gpios` من Device Tree |
| [mmc: sdhci-pci: Allow deferred probe for cd gpio](https://patchwork.kernel.org/project/linux-mmc/patch/1479805418-19603-3-git-send-email-adrian.hunter@intel.com/) | دعم deferred probe لـ card detect GPIO |
| [mmc: mmc_spi: Add Card Detect comments](https://patchwork.kernel.org/project/linux-mmc/patch/20160216040641.31979.6202.sendpatchset@little-apple/) | توثيق حالة CD GPIO في mmc_spi |

---

### 5. صفحات eLinux.org

| الصفحة | المحتوى |
|--------|---------|
| [R-Car/Use-GPIO-for-SD-card-detection](https://elinux.org/R-Car/Use-GPIO-for-SD-card-detection) | استخدام GPIO للـ SD card detection — مثال R-Car embedded |
| [ZipIt MMC](https://elinux.org/ZipIt_MMC) | تفاصيل GPIO pins لـ MMC slot على ZipIt hardware |
| [EVM Second MMC/SD](https://elinux.org/Second_MMC_/_SD) | إضافة MMC slot ثاني على TI EVM |
| [Device Tree Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل لـ DT bindings بما فيها MMC وGPIO |
| [Fix IRQ Domain DT support and gpio IRQ](https://elinux.org/Fix_IRQ_Domain_DT_support_issues_and_gpio_IRQ) | حل مشاكل IRQ لـ GPIO card detect |

---

### 6. كتب مرجعية

| الكتاب | الفصل المهم |
|--------|-------------|
| **Linux Device Drivers (LDD3)** — Rubini, Corbet, Kroah-Hartman | Chapter 14: The Linux Device Model؛ Chapter 6: Advanced Char Driver Operations (IRQ handling) |
| **Linux Kernel Development** — Robert Love (3rd ed.) | Chapter 7: Interrupts and Interrupt Handlers؛ Chapter 17: Devices and Modules |
| **Embedded Linux Primer** — Christopher Hallinan (2nd ed.) | Chapter 10: MTD and Flash Filesystems؛ Chapter 15: Debugging Embedded Linux |
| **Professional Linux Kernel Architecture** — Wolfgang Mauerer | Chapter 6: Device Drivers — يشمل GPIO و interrupt subsystem |

> ملاحظة: LDD3 مجاني على [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

---

### 7. مصادر kernelnewbies.org

- [Linux 3.16 Changelog](https://kernelnewbies.org/Linux_3.16) — أول إصدار يضم تحسينات gpiod API كبيرة
- [Linux 4.5 Changelog](https://kernelnewbies.org/Linux_4.5) — تحسينات MMC و GPIO descriptor
- [KernelNewbies: Driver porting guide](https://kernelnewbies.org/Driver_porting) — دليل نقل الـ drivers للـ gpiod API

---

### 8. أوامر بحث مفيدة

للبحث في source tree:

```bash
# كل الأماكن اللي بتستخدم slot-gpio API
grep -r "mmc_gpiod_request_cd\|mmc_gpio_get_cd" drivers/mmc/

# host drivers اللي بتطلب GPIO للـ card detect
grep -rl "mmc_gpiod_request_cd" drivers/

# Device Tree bindings لـ MMC
grep -r "cd-gpios\|wp-gpios" Documentation/devicetree/bindings/mmc/

# كل الـ IRQ handlers الخاصة بـ card detect
grep -r "mmc_gpio_set_cd_irq\|mmc_gpiod_request_cd_irq" drivers/
```

**Search terms للبحث الخارجي:**
- `mmc_gpiod_request_cd site:lkml.org`
- `MMC slot GPIO card detect debounce Linux kernel`
- `gpiod descriptor API MMC host driver`
- `cd-gpios wp-gpios device tree mmc binding`
- `mmc_host card_detect GPIO interrupt`

---

### 9. Device Tree Bindings الرسمية

```
Documentation/devicetree/bindings/mmc/mmc-controller.yaml
```

الخصائص المرتبطة بـ `slot-gpio.h`:

| الخاصية | الوظيفة |
|---------|---------|
| `cd-gpios` | GPIO لـ card detection — تقرأه `mmc_gpiod_request_cd()` |
| `wp-gpios` | GPIO لـ write protect — تقرأه `mmc_gpiod_request_ro()` |
| `cd-inverted` | عكس منطق الـ CD line |
| `wp-inverted` | عكس منطق الـ WP line |
| `broken-cd` | لما مفيش CD GPIO — polling mode |
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على دالة `mmc_gpio_get_cd()` — دي الدالة اللي بتقرأ حالة GPIO الخاصة بـ card-detect في MMC/SD slots. كل ما حد في الكرنل سأل "في كارت مدخّل؟"، الـ probe بتاعنا بتطلع الـ return value (0 = مفيش كارت، 1 = في كارت، سالب = error).

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * mmc_cd_probe.c
 * kprobe on mmc_gpio_get_cd() — prints card-detect GPIO result
 * every time the kernel polls for SD/MMC card presence.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/mmc/host.h>   /* struct mmc_host */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs");
MODULE_DESCRIPTION("kprobe on mmc_gpio_get_cd to trace card-detect GPIO reads");

/* ----------------------------------------------------------------
 * pre_handler — fires just BEFORE mmc_gpio_get_cd() executes.
 * We read the mmc_host pointer from the first argument register.
 * ---------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64: first arg is in regs->di
     * On arm64:  first arg is in regs->regs[0]
     * We cast it to struct mmc_host* to read the host index.
     */
#ifdef CONFIG_X86_64
    struct mmc_host *host = (struct mmc_host *)regs->di;
#elif defined(CONFIG_ARM64)
    struct mmc_host *host = (struct mmc_host *)regs->regs[0];
#else
    struct mmc_host *host = NULL;
#endif

    if (host)
        /* print the host index so we know which slot is being polled */
        pr_info("mmc_cd_probe: mmc_gpio_get_cd() called — host index=%d\n",
                host->index);
    else
        pr_info("mmc_cd_probe: mmc_gpio_get_cd() called (host ptr unavailable)\n");

    return 0; /* 0 = don't stop execution */
}

/* ----------------------------------------------------------------
 * post_handler — fires just AFTER mmc_gpio_get_cd() returns.
 * regs->ax (x86-64) or regs->regs[0] (arm64) holds the return value.
 * ---------------------------------------------------------------- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    long retval;

#ifdef CONFIG_X86_64
    retval = (long)regs->ax;
#elif defined(CONFIG_ARM64)
    retval = (long)regs->regs[0];
#else
    retval = -ENOSYS;
#endif

    /* decode the return value for human-readable output */
    if (retval == 1)
        pr_info("mmc_cd_probe: result=CARD PRESENT (1)\n");
    else if (retval == 0)
        pr_info("mmc_cd_probe: result=NO CARD (0)\n");
    else
        pr_info("mmc_cd_probe: result=ERROR (%ld)\n", retval);
}

/* ----------------------------------------------------------------
 * kprobe struct — ties symbol name to our handlers
 * ---------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "mmc_gpio_get_cd", /* exported symbol we want to hook */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ----------------------------------------------------------------
 * module_init — register the kprobe
 * ---------------------------------------------------------------- */
static int __init mmc_cd_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("mmc_cd_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("mmc_cd_probe: kprobe planted on %s at %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ----------------------------------------------------------------
 * module_exit — unregister the kprobe cleanly
 * ---------------------------------------------------------------- */
static void __exit mmc_cd_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("mmc_cd_probe: kprobe removed from %s\n", kp.symbol_name);
}

module_init(mmc_cd_probe_init);
module_exit(mmc_cd_probe_exit);
```

---

### Makefile

```makefile
obj-m += mmc_cd_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

| الجزء | الغرض |
|---|---|
| `symbol_name = "mmc_gpio_get_cd"` | بنحدد اسم الدالة اللي عايزين نـhook عليها — الكرنل بيحوّله لعنوان وقت الـ register |
| `pre_handler` | بتتنفذ قبل ما الدالة تشتغل، فبنقدر نعرف مين طلبها (host index) |
| `post_handler` | بتتنفذ بعد رجوع الدالة، فبنقرأ الـ return value من الـ registers مباشرة |
| `regs->di / regs->regs[0]` | الـ ABI بيحط أول argument في register معين حسب الـ architecture — x86-64 بيستخدم `rdi`، arm64 بيستخدم `x0` |
| `register_kprobe` / `unregister_kprobe` | الـ init بيزرع الـ hook، والـ exit بيشيله بشكل نظيف عشان مفيش crash لو الـ module اتـunload |
| `MODULE_LICENSE("GPL")` | لازم تكون GPL عشان الكرنل يسمحلك تشتغل مع الـ exported symbols |

---

### ليه `mmc_gpio_get_cd` بالذات؟

الدالة دي **exported** (`EXPORT_SYMBOL_GPL`) وبتتحط في الـ hot path كل ما الكرنل يحتاج يعرف لو في SD card مدخّلة. مش بتعمل memory allocation أو sleep، فالـ kprobe عليها **آمن** ومش هيأثر على الـ performance بشكل ملحوظ في بيئة تطوير. كمان الـ return value بتاعها (0/1/error) واضح جداً وبيعطي معلومة مفيدة فوراً في الـ dmesg.

---

### تجربة الـ Module

```bash
# بناء وتحميل
make
sudo insmod mmc_cd_probe.ko

# شوف اللوج
dmesg | grep mmc_cd_probe

# إخراج مثال
# mmc_cd_probe: kprobe planted on mmc_gpio_get_cd at ffffffffc0a12340
# mmc_cd_probe: mmc_gpio_get_cd() called — host index=0
# mmc_cd_probe: result=CARD PRESENT (1)

# إزالة نظيفة
sudo rmmod mmc_cd_probe
```
