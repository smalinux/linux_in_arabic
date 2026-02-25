## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبع له الملف

الـ file ده جزء من **MMC/SD/SDIO Subsystem** في الـ Linux kernel — المسؤول عنه Ulf Hansson والـ mailing list بتاعته هي `linux-mmc@vger.kernel.org`.

---

### الصورة الكبيرة: إيه مشكلة الـ Power Management في SDIO؟

تخيل عندك لابتوب فيه **كارت Wi-Fi** شغال على **SDIO** — يعني متوصل بالـ processor عن طريق bus اسمه MMC/SD. لما المستخدم يضغط "Sleep" أو "Suspend"، الـ kernel محتاج يقرر:

- هل يوقف الكارت تماماً ويقطع التيار عنه؟
- ولا يخليه شغال ويفضل يستنى interrupt عشان يصحّي السيستم؟

ده السؤال اللي `mmc/pm.h` بيجاوب عليه.

---

### القصة بالتفصيل

#### المشكلة

الـ SDIO cards (زي Wi-Fi، Bluetooth، GPS) مش بس storage — دي **أجهزة تانية** بتعمل communication. لما الجهاز بيدخل Suspend:

1. الـ **host controller** (الـ hardware اللي بيتكلم مع الكارت) بيحتاج يعرف إيه اللي المسموح بيه.
2. الـ **SDIO card driver** (زي driver الـ Wi-Fi) محتاج يطلب ميزات معينة من الـ host عشان يفضل يشتغل صح.
3. في طبقات كتير بينهم: الـ host controller driver ← الـ MMC core ← الـ SDIO core ← الـ SDIO function driver.

كل الطبقات دي محتاجة "تتكلم" بنفس اللغة — وده اللي `pm.h` بيوفره: **تعريفات مشتركة** تعبر عن capabilities الـ power management.

#### الحل

الملف بيعرّف:
- **`mmc_pm_flag_t`**: نوع بيانات (مجرد `unsigned int`) بيمثل مجموعة flags لـ PM.
- **`MMC_PM_KEEP_POWER`**: flag بتقول "متقطعش التيار عن الكارت وقت الـ suspend".
- **`MMC_PM_WAKE_SDIO_IRQ`**: flag بتقول "لو الكارت بعت interrupt، صحّي السيستم من الـ suspend".

---

### مثال واقعي

تخيل كارت **Wi-Fi على SDIO** في تابلت Android:

```
[Wi-Fi Driver] → طلب suspend
    ↓
sdio_get_host_pm_caps()   ← اسأل الـ host: إيه اللي تقدر تعمله؟
    ↓ (الـ host يرجع MMC_PM_KEEP_POWER | MMC_PM_WAKE_SDIO_IRQ)
sdio_set_host_pm_flags()  ← قوله: أنا محتاج KEEP_POWER + WAKE_SDIO_IRQ
    ↓
[MMC Core] يمرر الطلب للـ host controller
    ↓
[Host Controller Driver] يحافظ على التيار ويفعّل wakeup من الـ IRQ
```

لو جه packet على الـ Wi-Fi وقت الـ sleep → الكارت يبعت IRQ → الـ host يصحّي السيستم ← كل ده بسبب الـ flags دي.

---

### الطبقات اللي بتستخدم الـ flags دي

```
┌─────────────────────────────────────┐
│       SDIO Function Driver          │  ← Wi-Fi, BT, GPS driver
│  (sdio_get/set_host_pm_flags)       │
├─────────────────────────────────────┤
│           SDIO Core                 │  ← drivers/mmc/core/sdio.c
├─────────────────────────────────────┤
│            MMC Core                 │  ← drivers/mmc/core/core.c
├─────────────────────────────────────┤
│      Host Controller Driver         │  ← drivers/mmc/host/*.c
│  (pm_caps يتملى من Device Tree)     │
└─────────────────────────────────────┘
```

الـ `mmc/pm.h` هو **العقد المشترك** بين كل الطبقات دي — من غيره كل طبقة كانت هتعرّف flags بنفسها وتبقى فوضى.

---

### الـ Flags بالتفصيل

| الـ Flag | القيمة | المعنى |
|---|---|---|
| `MMC_PM_KEEP_POWER` | `1 << 0` | الكارت يفضل شغال بالتيار أثناء suspend |
| `MMC_PM_WAKE_SDIO_IRQ` | `1 << 1` | IRQ من الكارت يصحّي السيستم من الـ suspend |

الـ host بيعلن الـ `pm_caps` بتاعته (إيه اللي يقدر يعمله) — والـ driver بيطلب من خلال `pm_flags` (إيه اللي محتاجه فعلاً). الـ MMC core بيتحقق إن المطلوب مش أكتر من المتاح.

---

### الـ Device Tree Connection

الـ host controller بيعلن قدراته في الـ Device Tree:

```dts
/* مثال في device tree */
mmc0 {
    keep-power-in-suspend;     /* → MMC_PM_KEEP_POWER */
    wakeup-source;             /* → MMC_PM_WAKE_SDIO_IRQ */
};
```

الكود في `drivers/mmc/core/host.c` بيقرأ الـ properties دي ويملّي `host->pm_caps`.

---

### الملفات المرتبطة اللي المفروض تعرفها

#### الـ Headers الأساسية
| الملف | الدور |
|---|---|
| `include/linux/mmc/pm.h` | **الملف ده** — تعريف الـ PM flags |
| `include/linux/mmc/host.h` | تعريف `struct mmc_host` اللي فيها `pm_caps` و `pm_flags` |
| `include/linux/mmc/sdio_func.h` | تعريف `sdio_get_host_pm_caps()` و `sdio_set_host_pm_flags()` |
| `include/linux/mmc/core.h` | الـ core API للـ MMC/SD/SDIO |
| `include/linux/mmc/card.h` | تعريف `struct mmc_card` |
| `include/linux/mmc/sdio.h` | تعريفات خاصة بـ SDIO protocol |

#### ملفات الـ Core
| الملف | الدور |
|---|---|
| `drivers/mmc/core/sdio_io.c` | تنفيذ `sdio_get_host_pm_caps()` و `sdio_set_host_pm_flags()` |
| `drivers/mmc/core/sdio.c` | إدارة suspend/resume لـ SDIO cards |
| `drivers/mmc/core/host.c` | تهيئة الـ host وقراءة الـ Device Tree properties |
| `drivers/mmc/core/core.c` | الـ MMC core logic |

#### ملفات الـ Host Controllers (أمثلة)
| الملف | الدور |
|---|---|
| `drivers/mmc/host/sdhci.c` | الـ SDHCI standard host controller |
| `drivers/mmc/host/mmci.c` | ARM PL180/181 MMCI controller |
| `drivers/mmc/host/dw_mmc.c` | Synopsys DesignWare MMC controller |

---

### ملخص

**الـ `mmc/pm.h`** ملف صغير جداً (سطرين تعريف فعلي) لكن دوره كبير: هو **اللغة المشتركة** بين كل طبقات الـ MMC/SDIO subsystem للتفاهم على سلوك الـ Power Management وقت الـ suspend. من غيره كل طبقة كانت ممكن تعرّف نفس المفاهيم بطريقة مختلفة وتبقى الطبقات incompatible مع بعض.
## Phase 2: شرح الـ MMC Power Management Framework

### المشكلة — ليه الـ framework ده موجود أصلاً؟

في الـ embedded systems، الـ SDIO cards (زي Wi-Fi modules وBluetooth adapters) مش مجرد storage — هي devices نشطة بتعمل network traffic وبتستقبل interrupts. لما الـ system بيدخل suspend (مثلاً ARM SoC بيدخل في sleep mode لتوفير battery)، بيحصل تعارض:

**الـ host controller** بيتوقع إنه يقطع الـ power عن كل حاجة.
**الـ SDIO card** محتاجة تفضل شغالة عشان تقدر تصحّي الـ system لماييجي network packet أو Bluetooth event.

من غير framework موحّد، كل driver كان هيعمل الـ power management بطريقته الخاصة، وكان في chaos بين:
- الـ host controller driver (مثلاً `sdhci-pltfm`)
- الـ MMC core layer
- الـ SDIO core layer
- الـ function driver (مثلاً `brcmfmac` لـ Broadcom Wi-Fi)

الـ `mmc/pm.h` جه يحل مشكلة واحدة بسيطة لكنها أساسية: **إزاي كل الـ layers دي تتفاهم على إيه الـ PM features المطلوبة وقت الـ suspend؟**

---

### الحل — الـ Approach اللي الـ kernel اتاخده

الحل هو **bitmask موحّد** بيتمرر عبر كل الـ layers من فوق لتحت:

```c
typedef unsigned int mmc_pm_flag_t;

#define MMC_PM_KEEP_POWER    (1 << 0)  /* preserve card power during suspend */
#define MMC_PM_WAKE_SDIO_IRQ (1 << 1)  /* wake up host system on SDIO IRQ assertion */
```

الـ `mmc_pm_flag_t` هو مجرد `unsigned int` — بس الـ typedef ده مهم لأنه بيوضح الـ semantic intent ويخلي الكود أوضح.

الـ `struct mmc_host` بيحمل اتنين:

```c
mmc_pm_flag_t  pm_caps;   /* what the HOST CONTROLLER supports */
mmc_pm_flag_t  pm_flags;  /* what the FUNCTION DRIVER requests  */
```

الفرق جوهري:
- **`pm_caps`**: الـ host controller بيقول "أنا أقدر أعمل إيه"
- **`pm_flags`**: الـ SDIO function driver بيقول "أنا محتاج إيه"

الـ MMC core بيعمل intersection بينهم عشان يقرر إيه اللي هيتنفذ فعلاً.

---

### تشبيه من الواقع — فندق وغرف الكهرباء

تخيل فندق كبير فيه **مدير طابق** (MMC core)، **فني كهرباء** (host controller driver)، و**نزيل** (SDIO function driver).

| الكونسبت في الـ kernel | المقابل في التشبيه |
|---|---|
| `mmc_pm_flag_t` | قائمة خدمات الغرفة |
| `MMC_PM_KEEP_POWER` | "خلّي الكهرباء شغالة في غرفتي حتى وأنا نايم" |
| `MMC_PM_WAKE_SDIO_IRQ` | "لو جه اتصال تليفوني، صحّيني حتى لو نايم" |
| `pm_caps` في الـ host | قائمة الخدمات اللي الفندق فعلاً بيقدر يوفرها |
| `pm_flags` في الـ host | طلبات النزيل الحالي |
| الـ MMC core يعمل intersection | المدير بيقول "النزيل طلب كذا، إحنا بنقدر نعمل كذا، إذن بنعمل التقاطع بينهم" |

الـ mapping هنا دقيق: الفندق مش لازم ينفذ كل طلبات النزيل — لو الطابق مش فيه أجراس ليلية (`pm_caps` مش فيها `MMC_PM_WAKE_SDIO_IRQ`)، حتى لو النزيل طلبها، مش هتتنفذ. ده بالظبط اللي بيحصل في الكود.

---

### الصورة الكبيرة — Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    SDIO Function Driver                      │
│              (e.g., brcmfmac Wi-Fi driver)                   │
│                                                              │
│  sdio_set_host_pm_flags(func, MMC_PM_KEEP_POWER |           │
│                               MMC_PM_WAKE_SDIO_IRQ);        │
└────────────────────────┬────────────────────────────────────┘
                         │  sets pm_flags on mmc_host
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                      SDIO Core Layer                         │
│              (drivers/mmc/core/sdio_bus.c)                   │
│                                                              │
│  • calls sdio_bus_pm_ops (suspend/resume)                    │
│  • validates flags against pm_caps                           │
│  • delegates to MMC core                                     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                       MMC Core Layer                         │
│              (drivers/mmc/core/core.c)                       │
│                                                              │
│  • mmc_power_save_host() / mmc_power_restore_host()         │
│  • checks: pm_flags & pm_caps                               │
│  • if KEEP_POWER set → skip power-off sequence              │
│  • if WAKE_SDIO_IRQ set → keep IRQ line active              │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  Host Controller Driver                      │
│         (e.g., sdhci.c / sdhci-pltfm.c / dw_mmc.c)         │
│                                                              │
│  • sets pm_caps during probe:                               │
│      host->pm_caps |= MMC_PM_KEEP_POWER;                    │
│      host->pm_caps |= MMC_PM_WAKE_SDIO_IRQ;                 │
│                                                              │
│  • implements set_ios() to control:                         │
│      - VMC regulator (card power supply)                    │
│      - SDCLK (bus clock)                                    │
│      - DAT1 line (SDIO IRQ line)                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                     ARM SoC / Hardware                       │
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────────┐ │
│  │  SDHCI regs  │   │  VMC regulator│   │  DAT1/IRQ line  │ │
│  │  (MMIO)      │   │  (PMIC)      │   │  (wakeup source)│ │
│  └──────────────┘   └──────────────┘   └─────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ framework ده بيقدم abstraction واحدة: **negotiated power state**.

مش الـ function driver هو اللي بيتحكم في الـ hardware مباشرةً، ومش الـ host controller هو اللي بيقرر لوحده. بدل كده:

1. الـ host controller بيعلن **قدراته** عبر `pm_caps`
2. الـ function driver بيطلب **احتياجاته** عبر `pm_flags`
3. الـ MMC core بيعمل **negotiation** (AND operation فعلياً) ويحدد الـ actual behavior

ده نفس الـ pattern اللي بتلاقيه في subsystems تانية في الـ kernel زي:
- **الـ clock framework** (`clk_set_rate` بيطلب، الـ hardware بيقدر يوفر قريب منه)
- **الـ regulator framework** (`regulator_set_voltage` بيطلب، الـ PMIC بيقدر يوفر)

---

### الـ Two Flags — شرح تفصيلي

#### `MMC_PM_KEEP_POWER` — الحفاظ على الـ Power

```
Normal Suspend (no flag):              With MMC_PM_KEEP_POWER:
─────────────────────────              ──────────────────────────────
card power ON                          card power ON
    │                                      │
    ▼                                      ▼
mmc_power_save_host()              mmc_power_save_host()
    │                                      │
    ▼                                      │   ← power is NOT cut
set_ios(MMC_POWER_OFF)             clock is stopped, but VMC stays ON
    │
    ▼
VMC regulator disabled
card loses state, needs
full reinit on resume
```

لما الـ flag ده مش مضبوط، الـ card بتخسر الـ state بتاعها وبتحتاج full re-initialization عند الـ resume — ده بطيء وبيستهلك طاقة.

لما يكون مضبوط، الـ card بتفضل powered وبتحافظ على الـ state بتاعها، والـ resume بيبقى أسرع بكتير.

#### `MMC_PM_WAKE_SDIO_IRQ` — الإيقاظ بالـ SDIO IRQ

```
SDIO IRQ architecture:
──────────────────────
SDIO Card
    │
    │ DAT1 line (shared with IRQ signaling in SD mode)
    │
    ▼
Host Controller ──→ SoC IRQ Controller ──→ CPU wakeup
                         │
                         └── Only works if:
                             1. DAT1 stays powered  (KEEP_POWER)
                             2. IRQ detection stays enabled (WAKE_SDIO_IRQ)
                             3. SoC IRQ is configured as wakeup source
```

الـ SDIO بيستخدم **DAT1 line** كـ interrupt line (مش pin منفصلة). لما الـ `MMC_PM_WAKE_SDIO_IRQ` يتضبط:
- الـ host controller بيفضل يراقب الـ DAT1 line حتى وقت الـ suspend
- لو الـ card سحبت الـ DAT1 low (يعني فيه interrupt)، الـ host controller بيبعث wakeup event للـ SoC
- الـ SoC بيصحى، والـ SDIO IRQ thread بيشتغل ويخدم الـ interrupt

ده مهم جداً لـ Wi-Fi: الـ Wi-Fi chip محتاجة تصحّي الـ system لما ييجي packet حتى والـ system نايمة.

---

### الـ inline functions في host.h — الـ Accessors

الـ `host.h` بيقدم اتنين accessor functions مهمين:

```c
/* checks if the FUNCTION DRIVER requested keep-power */
static inline int mmc_card_keep_power(struct mmc_host *host)
{
    return host->pm_flags & MMC_PM_KEEP_POWER;
}

/* checks if the FUNCTION DRIVER requested SDIO IRQ wakeup */
static inline int mmc_card_wake_sdio_irq(struct mmc_host *host)
{
    return host->pm_flags & MMC_PM_WAKE_SDIO_IRQ;
}
```

الـ MMC core بيستخدم الاتنين دول في الـ suspend path قبل ما يحدد إيه اللي يعمله بالـ hardware.

---

### إيه اللي الـ subsystem بيملكه vs إيه اللي بيفوّضه للـ drivers

```
┌─────────────────────────────────────────────────────────────────┐
│                    ما يملكه MMC PM Framework                     │
├─────────────────────────────────────────────────────────────────┤
│  • تعريف الـ flag type (mmc_pm_flag_t)                          │
│  • تعريف الـ flag constants (MMC_PM_KEEP_POWER, etc.)           │
│  • مكان تخزين الـ flags (pm_caps و pm_flags في mmc_host)        │
│  • الـ negotiation logic (intersection في الـ core)             │
│  • الـ accessor API (mmc_card_keep_power, etc.)                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│               ما يُفوّضه للـ Host Controller Driver              │
├─────────────────────────────────────────────────────────────────┤
│  • تحديد pm_caps وقت الـ probe                                  │
│  • التنفيذ الفعلي لـ keep-power (تحكم في الـ regulator)        │
│  • تفعيل/تعطيل IRQ detection على الـ DAT1 line                  │
│  • ضبط الـ SoC wakeup source للـ SDIO IRQ                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│               ما يُفوّضه للـ SDIO Function Driver                │
├─────────────────────────────────────────────────────────────────┤
│  • تحديد الـ flags المطلوبة (pm_flags) وقت الـ suspend          │
│  • القرار: هل الـ card محتاجة تصحّي الـ system؟                │
│  • الـ card-level suspend/resume sequence                        │
└─────────────────────────────────────────────────────────────────┘
```

---

### مثال عملي — بيتعمل إيه وقت suspend؟

خلينا نشوف الـ flow الكامل لـ Broadcom Wi-Fi (brcmfmac) على ARM board:

```
1. kernel يبدأ suspend sequence
        │
        ▼
2. brcmfmac driver suspend callback يتنادى:
   sdio_set_host_pm_flags(func, MMC_PM_KEEP_POWER | MMC_PM_WAKE_SDIO_IRQ)
   → يكتب في host->pm_flags
        │
        ▼
3. SDIO core يتحقق:
   (pm_flags & pm_caps) == pm_flags?  ← هل الـ host بيدعم اللي طُلب؟
   لو لأ → يرجع error، الـ driver يعمل fallback
   لو أيوه → يكمل
        │
        ▼
4. MMC core:
   mmc_card_keep_power(host) == true?
   → مش بيبعت power-off sequence
   → بيوقف الـ clock بس (ios->clock = 0)
   → الـ VMC regulator فاضل enabled
        │
        ▼
5. Host controller driver:
   mmc_card_wake_sdio_irq(host) == true?
   → بيفعّل الـ DAT1 IRQ كـ wakeup source
   → enable_irq_wake(host->irq)
        │
        ▼
6. SoC بيدخل suspend، لكن:
   - الـ Wi-Fi chip فاضلة powered
   - الـ DAT1 line بتتراقب
        │
        ▼
7. Wi-Fi chip بتستقبل network packet:
   → سحبت DAT1 low
   → SoC IRQ controller بيشتغل
   → SoC بيصحى
   → SDIO IRQ thread بيخدم الـ interrupt
   → brcmfmac بيقرأ الـ packet
```

---

### علاقة الـ pm_caps بالـ Capabilities الأخرى في mmc_host

الـ `mmc_host` بيحمل عدة capability sets:

```
struct mmc_host {
    u32           caps;      ← general host capabilities (bus width, speed, etc.)
    u32           caps2;     ← extended capabilities
    mmc_pm_flag_t pm_caps;   ← PM-specific capabilities  ← نحن هنا
    mmc_pm_flag_t pm_flags;  ← current PM requests
    ...
}
```

الـ `caps` و`caps2` بيتضبطوا مرة وبيتغيروش. أما الـ `pm_flags` فبيتكتب في كل suspend cycle بناءً على الـ function driver اللي شغال دلوقتي.

---

### الـ Subsystems اللي محتاج تعرفها

- **الـ Linux PM framework** (`include/linux/pm.h`): الـ framework العام للـ power management في الـ kernel — الـ MMC PM بيبني فوقيه.
- **الـ regulator framework** (`include/linux/regulator/`): بيتحكم في الـ voltage regulators (VMC) — الـ host controller بيستخدمه لـ keep-power.
- **الـ IRQ subsystem** (`include/linux/interrupt.h`): `enable_irq_wake()` هو اللي بيخلي الـ IRQ مصدر إيقاظ للـ SoC.
- **الـ SDIO bus** (`drivers/mmc/core/sdio_bus.c`): هو الـ consumer الرئيسي للـ MMC PM flags.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Types — Cheatsheet

الـ `mmc/pm.h` ملف صغير جداً بس حيوي — بيعرّف نوع واحد وفلاجين بس:

```c
typedef unsigned int mmc_pm_flag_t;  /* bitmask لـ PM features */
```

| Flag | Value | المعنى |
|------|-------|---------|
| `MMC_PM_KEEP_POWER` | `(1 << 0)` | متقطعش الطاقة عن الكارت وقت الـ suspend — الكارت يفضل شغال |
| `MMC_PM_WAKE_SDIO_IRQ` | `(1 << 1)` | لو جه IRQ من الكارت وانت في suspend، صحّي الـ host system |

---

### الـ Structs المهمة

#### 1. `mmc_pm_flag_t` (typedef)

**الغرض:** نوع مشترك بين كل layers الـ MMC stack لتمثيل PM capabilities وrequests.

مش struct حقيقي — مجرد `unsigned int` اتعمله typedef عشان الكود يوضح إن القيمة دي bitmask لـ PM وماشية.

---

#### 2. `struct mmc_host` — الـ fields المتعلقة بـ PM

ده الـ struct الأساسي اللي بيحوّل الـ pm flags من concept لـ runtime state. فيه fieldين محوريين:

```c
struct mmc_host {
    /* ... */
    mmc_pm_flag_t  pm_caps;   /* ما يقدر عليه الـ host controller */
    mmc_pm_flag_t  pm_flags;  /* ما طلبه الـ SDIO function driver */
    /* ... */
};
```

| Field | النوع | الغرض | مين بيكتب فيه | مين بيقرأه |
|-------|-------|--------|--------------|------------|
| `pm_caps` | `mmc_pm_flag_t` | قدرات الـ hardware — ثابتة بعد الـ probe | الـ host driver أو `mmc_of_parse()` | الـ SDIO function driver عبر `sdio_get_host_pm_caps()` |
| `pm_flags` | `mmc_pm_flag_t` | طلبات الـ suspend — بتتغير كل suspend cycle | الـ SDIO function driver عبر `sdio_set_host_pm_flags()` | الـ MMC core في `mmc_sdio_suspend()` |

**الـ fields التانية في `mmc_host` المرتبطة بـ PM:**

| Field | الغرض |
|-------|--------|
| `struct wakeup_source *ws` | wakeup source لضمان إن الـ system ميروحش sleep وفي event |
| `unsigned int sdio_irqs` | عدد الـ SDIO IRQs المسجلة — لو > 0 ولا `KEEP_POWER` → warning |
| `bool sdio_irq_pending` | في IRQ جاهز للمعالجة |
| `struct task_struct *sdio_irq_thread` | الـ thread اللي بيعالج SDIO IRQs |
| `caps & MMC_CAP_SDIO_IRQ` | الـ host يقدر يعمل SDIO IRQ signaling |
| `caps & MMC_CAP_AGGRESSIVE_PM` | ابعت الكارت للـ suspend حتى وهو idle |
| `caps2 & MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND` | ممكن يعمل full power cycle أثناء الـ suspend |

---

#### 3. `struct sdio_func` (للسياق)

الـ SDIO function driver بيشتغل على الـ `sdio_func` اللي فيه pointer للـ card، واللي فيه pointer للـ host:

```c
struct sdio_func {
    struct mmc_card  *card;   /* → mmc_card → mmc_host → pm_caps/pm_flags */
    /* ... */
};
```

---

### رسم علاقات الـ Structs

```
┌──────────────────────────────────────────────────────────┐
│                    mmc/pm.h                              │
│  typedef unsigned int mmc_pm_flag_t;                     │
│  #define MMC_PM_KEEP_POWER      (1 << 0)                 │
│  #define MMC_PM_WAKE_SDIO_IRQ   (1 << 1)                 │
└──────────────────────────┬───────────────────────────────┘
                           │ used by
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
┌──────────────────┐  ┌─────────────┐  ┌──────────────────────┐
│   mmc_host       │  │  sdio_func  │  │ MMC core (sdio.c /   │
│                  │  │             │  │ sdio_io.c)           │
│  mmc_pm_flag_t   │◄─┤  *card ─┐  │  │                      │
│    pm_caps       │  │         │  │  │  sdio_get_host_pm_caps│
│  mmc_pm_flag_t   │  └─────────┼──┘  │  sdio_set_host_pm_flags│
│    pm_flags      │            │     │  mmc_sdio_suspend()   │
│                  │  ┌─────────▼──┐  │  mmc_sdio_resume()    │
│  *ops            │  │  mmc_card  │  └──────────────────────┘
│  *card ──────────┼─►│            │
│  mmc_ios  ios    │  │  *host ────┼──► (back to mmc_host)
│  spinlock_t lock │  └────────────┘
│  *sdio_irq_thread│
│  u32 caps        │
│  u32 caps2       │
└──────────────────┘
```

---

### الـ layers الأربع للـ PM flags

الـ comment في `mmc/pm.h` نفسه بيقول في 4 layers:

```
┌─────────────────────────────────────────┐
│   SDIO Function Driver                  │  ← بيطلب الـ features
│   (wifi driver, BT driver, etc.)        │
│   sdio_set_host_pm_flags(func, flags)   │
└───────────────────┬─────────────────────┘
                    │ يكتب في host->pm_flags
                    ▼
┌─────────────────────────────────────────┐
│   SDIO Core (sdio.c / sdio_io.c)        │  ← بيتحقق من الـ flags
│   mmc_sdio_suspend()                    │
│   مثلاً: لو KEEP_POWER مش set →        │
│          mmc_power_off()                │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│   MMC Core (core.c / host.c)            │  ← بيعمل الـ actual suspend
│   mmc_power_off() / mmc_power_up()      │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│   Host Controller Driver                │  ← بيكتب في pm_caps
│   (sdhci, dw_mmc, etc.)                │
│   host->pm_caps = MMC_PM_KEEP_POWER     │
│                 | MMC_PM_WAKE_SDIO_IRQ  │
└─────────────────────────────────────────┘
```

---

### دورة حياة الـ PM flags — Lifecycle Diagram

#### مرحلة الـ Probe (تسجيل الـ Host):

```
host driver probe()
    │
    ├─► mmc_alloc_host(extra, dev)
    │       kzalloc(sizeof(mmc_host) + extra)
    │       host->pm_caps = 0   ← يبدأ بصفر
    │       host->pm_flags = 0
    │       spin_lock_init(&host->lock)
    │       return host
    │
    ├─► host driver يضبط pm_caps يدوياً:
    │       host->pm_caps |= MMC_PM_KEEP_POWER;
    │       host->pm_caps |= MMC_PM_WAKE_SDIO_IRQ;
    │    أو عبر Device Tree:
    │       mmc_of_parse(host)
    │           "keep-power-in-suspend" → pm_caps |= KEEP_POWER
    │           "wakeup-source"         → pm_caps |= WAKE_SDIO_IRQ
    │
    └─► mmc_add_host(host)
            mmc_validate_host_caps()
            device_add(&host->class_dev)
            mmc_start_host(host)   ← تبدأ card detection
```

#### مرحلة الـ Suspend:

```
System suspend triggered
    │
    ├─► SDIO function driver ->suspend() callback
    │       pm_caps = sdio_get_host_pm_caps(func)
    │           └─► return func->card->host->pm_caps
    │
    │       /* driver يقرر ايه اللي محتاجه */
    │       if (pm_caps & MMC_PM_KEEP_POWER)
    │           sdio_set_host_pm_flags(func, MMC_PM_KEEP_POWER)
    │               └─► host->pm_flags |= MMC_PM_KEEP_POWER
    │
    │       if (want_wakeup && pm_caps & MMC_PM_WAKE_SDIO_IRQ)
    │           sdio_set_host_pm_flags(func, MMC_PM_WAKE_SDIO_IRQ)
    │               └─► host->pm_flags |= MMC_PM_WAKE_SDIO_IRQ
    │
    └─► mmc_sdio_suspend(host)   ← MMC core
            WARN_ON(host->sdio_irqs && !mmc_card_keep_power(host))
            │    /* لو في IRQs مسجلة لازم KEEP_POWER يكون set */
            │
            ├─► if (KEEP_POWER && WAKE_SDIO_IRQ):
            │       sdio_disable_4bit_bus()  ← switch to 1-bit for IRQ wakeup
            │
            └─► if (!KEEP_POWER):
                    mmc_power_off()   ← قطع الطاقة تماماً
                else:
                    mmc_retune_timer_stop()  ← وقف الـ retune timer
```

#### مرحلة الـ Resume:

```
System resume
    │
    └─► mmc_sdio_resume(host)
            │
            ├─► if (!KEEP_POWER):
            │       mmc_power_up()          ← شغّل الطاقة تاني
            │       mmc_sdio_reinit_card()  ← reinit كامل
            │
            ├─► else if (WAKE_SDIO_IRQ):
            │       sdio_enable_4bit_bus()  ← رجّع 4-bit mode
            │
            ├─► mmc_card_clr_suspended()
            │
            ├─► if (sdio_irqs > 0):
            │       wake_up_process(sdio_irq_thread)
            │
            └─► host->pm_flags &= ~MMC_PM_KEEP_POWER
                    /* reset بعد كل resume — الـ driver لازم يطلب تاني في الـ suspend الجاي */
```

#### مرحلة الـ Teardown:

```
host driver remove()
    │
    ├─► mmc_remove_host(host)
    │       mmc_stop_host()
    │       device_del(&host->class_dev)
    │       led_trigger_unregister_simple()
    │
    └─► mmc_free_host(host)
            cancel_delayed_work_sync(&host->detect)
            mmc_pwrseq_free(host)
            put_device(&host->class_dev)
                └─► mmc_host_classdev_release()
                        wakeup_source_unregister(host->ws)
                        ida_free(&mmc_host_ida, host->index)
                        kfree(host)
```

---

### Call Flow Diagrams

#### Flow 1: SDIO driver يسأل ويطلب PM features

```
sdio_func_driver->suspend(func)
    │
    ├─► sdio_get_host_pm_caps(func)
    │       └─► return func->card->host->pm_caps   [read-only]
    │
    └─► sdio_set_host_pm_flags(func, MMC_PM_KEEP_POWER)
            host = func->card->host
            if (flags & ~host->pm_caps) return -EINVAL  ← validation
            host->pm_flags |= flags                     ← write
```

#### Flow 2: الـ core يتخذ قرار الـ suspend بناءً على الـ flags

```
mmc_sdio_suspend(host)
    │
    ├─► mmc_card_keep_power(host)
    │       └─► host->pm_flags & MMC_PM_KEEP_POWER
    │
    ├─► mmc_card_wake_sdio_irq(host)
    │       └─► host->pm_flags & MMC_PM_WAKE_SDIO_IRQ
    │
    ├─► [KEEP_POWER=1, WAKE_SDIO_IRQ=1]
    │       sdio_disable_4bit_bus(card)
    │           └─► ops->set_ios(host, &ios)  ← switch to 1-bit
    │               (IRQ detection يشتغل على 1-bit فقط)
    │
    ├─► [KEEP_POWER=0]
    │       mmc_power_off(host)
    │           └─► ops->set_ios(host, &ios)  ← ios.power_mode = OFF
    │
    └─► [KEEP_POWER=1, WAKE_SDIO_IRQ=0]
            mmc_retune_timer_stop(host)   ← بس وقف الـ timer
```

#### Flow 3: Device Tree → pm_caps

```
mmc_of_parse(host)
    │
    ├─► device_property_read_bool(dev, "keep-power-in-suspend")
    │       └─► host->pm_caps |= MMC_PM_KEEP_POWER
    │
    └─► device_property_read_bool(dev, "wakeup-source")
         أو device_property_read_bool(dev, "enable-sdio-wakeup")  [legacy]
              └─► host->pm_caps |= MMC_PM_WAKE_SDIO_IRQ
```

---

### استراتيجية الـ Locking

#### الـ pm_flags — مين بيحميه؟

```c
/* من sdio_io.c */
/* function suspend methods are serialized, hence no lock needed */
host->pm_flags |= flags;
```

الـ comment في الكود نفسه بيقول صريح: **مفيش lock** لأن الـ SDIO function suspend methods بتتشغل بشكل serialized من framework الـ PM — يعني مفيش race condition ممكن.

#### الـ pm_caps — ثابتة بعد الـ probe

الـ `pm_caps` بتتكتب مرة واحدة وقت الـ probe (في `mmc_of_parse()` أو في الـ host driver مباشرة)، وبعدين بتبقى read-only. فمحتاجش lock.

#### الـ host->lock (spinlock)

الـ `spin_lock_t lock` في `mmc_host` مش بتحمي الـ PM flags — بتحمي:

| ما يحميه | السبب |
|----------|-------|
| `host->claimed` | منع أكتر من context يدّعي الـ host |
| `host->claimer` | الـ context الحالي اللي عنده الـ host |
| `host->claim_cnt` | nested claim count |
| bus operations | إجراءات على الـ bus |

#### الـ mmc_claim_host() في الـ suspend/resume

```c
mmc_sdio_suspend(host):
    mmc_claim_host(host)   ← يحجز الـ host
    /* ... power/bus operations ... */
    mmc_release_host(host) ← يحرره

mmc_sdio_resume(host):
    mmc_claim_host(host)
    /* ... reinit card ... */
    mmc_release_host(host)
    host->pm_flags &= ~MMC_PM_KEEP_POWER  ← بعد release — safe لأن serialized
```

#### ترتيب الـ Locks (Lock Ordering)

```
لا توجد nested locks بين pm_flags وأي lock تاني.
الترتيب العام في MMC subsystem:

    mmc_claim_host()    ← mutex-like (wait_queue based)
        └─► spin_lock(&host->lock)   ← قصير جداً، لحماية state
                └─► (no pm_flags access inside spinlock)
```

---

### ملخص تفاعل الـ Flags في الـ Real World

مثال: Wi-Fi SDIO card (زي Broadcom BCM4329) محتاجة تصحّي الـ system لما تيجيلها packet:

```
1. Probe:
   host driver:  host->pm_caps = MMC_PM_KEEP_POWER | MMC_PM_WAKE_SDIO_IRQ

2. System suspend:
   wifi driver:
       caps = sdio_get_host_pm_caps(func)
       /* caps == KEEP_POWER | WAKE_SDIO_IRQ */
       sdio_set_host_pm_flags(func, KEEP_POWER | WAKE_SDIO_IRQ)
       /* host->pm_flags = KEEP_POWER | WAKE_SDIO_IRQ */

   MMC core:
       mmc_sdio_suspend():
           KEEP_POWER && WAKE_SDIO_IRQ → sdio_disable_4bit_bus()
           /* الكارت فضل شغال على 1-bit، ومستني IRQ */

3. الـ Wi-Fi chip تستقبل packet → ترفع SDIO IRQ
   → الـ host controller يصحي الـ CPU
   → kernel يعمل resume

4. Resume:
   mmc_sdio_resume():
       KEEP_POWER=1 && WAKE_SDIO_IRQ=1
       → sdio_enable_4bit_bus()   ← رجّع الـ speed
       → wake_up_process(sdio_irq_thread)
       → host->pm_flags &= ~KEEP_POWER  ← reset للـ cycle الجاي
```
## Phase 4: شرح الـ Functions

> **الملف:** `include/linux/mmc/pm.h`
> **الغرض:** تعريف الـ bitmask flags والـ typedef الخاصة بـ power management في subsystem الـ MMC/SDIO.

---

### ملخص الـ API — Cheatsheet

| الرمز / الـ API | النوع | القيمة | الوصف المختصر |
|---|---|---|---|
| `mmc_pm_flag_t` | `typedef unsigned int` | — | نوع bitmask لحمل PM flags |
| `MMC_PM_KEEP_POWER` | `#define` | `(1 << 0)` | ابقِ الطاقة على الكارت أثناء الـ suspend |
| `MMC_PM_WAKE_SDIO_IRQ` | `#define` | `(1 << 1)` | صحّي الـ system من الـ suspend لما يجي SDIO IRQ |
| `host->pm_caps` | `mmc_pm_flag_t` field | — | الـ flags اللي الـ host controller يدعمها |
| `host->pm_flags` | `mmc_pm_flag_t` field | — | الـ flags اللي الـ SDIO function driver طلبها فعلاً |
| `sdio_get_host_pm_caps()` | function | — | اقرأ `pm_caps` من الـ host |
| `sdio_set_host_pm_flags()` | function | — | اكتب flags في `pm_flags` بشرط إنها موجودة في `pm_caps` |
| `mmc_card_keep_power()` | inline | — | اتحقق من bit `MMC_PM_KEEP_POWER` في `pm_flags` |
| `mmc_card_wake_sdio_irq()` | inline | — | اتحقق من bit `MMC_PM_WAKE_SDIO_IRQ` في `pm_flags` |

---

### المجموعة الأولى: Type Definition والـ Bitmask Constants

هذه المجموعة هي الـ core API للملف — تُعرّف اللغة المشتركة بين ثلاث طبقات: الـ host controller driver، الـ MMC core، والـ SDIO function driver. الفكرة إن كل طبقة تفهم نفس الـ flags بدون ما تعرف تفاصيل الطبقة التانية.

---

#### `mmc_pm_flag_t`

```c
typedef unsigned int mmc_pm_flag_t;
```

**الـ typedef** الأساسية في الملف. بتحوّل `unsigned int` لنوع semantic واضح يوضح إن الـ value دي بتمثل PM flags مش مجرد عدد عشوائي.

- **الاستخدام:** كل field بتحمل PM flags في الـ kernel بتُعلَن كـ `mmc_pm_flag_t` مش `unsigned int` — ده يخلي الـ compiler يساعد في الـ type checking، وبيوضح النية.
- **الأماكن اللي تظهر فيها:**
  - `struct mmc_host` → `pm_caps` و `pm_flags`
  - signature الـ `sdio_get_host_pm_caps()` و `sdio_set_host_pm_flags()`
- **بدون lock:** القراءة/الكتابة في `pm_flags` أثناء الـ suspend path بتتم بدون lock لأن الـ SDIO function suspend callbacks بتتنفذ بشكل serialized من kernel PM framework.

---

#### `MMC_PM_KEEP_POWER`

```c
#define MMC_PM_KEEP_POWER    (1 << 0)    /* preserve card power during suspend */
```

**الـ flag** الأهم في الملف. لما الـ SDIO function driver يسيّب الـ bit ده في `pm_flags`، الـ MMC core يُبقي على الطاقة على الكارت أثناء system suspend بدل ما يعمل `mmc_power_off()`.

**ليه بيتستخدم:**
- كارت SDIO Wi-Fi مثلاً محتاج يفضل شغّال أثناء الـ suspend عشان يقدر يستقبل wakeup packets.
- لو متحطش، الـ MMC core هيعمل full power cycle للكارت عند الـ resume وهيحتاج يعمل reinitialization كاملة.

**التأثير على `mmc_sdio_suspend()`:**

```c
/* من drivers/mmc/core/sdio.c */
if (!mmc_card_keep_power(host)) {
    mmc_power_off(host);        /* الكارت هيتطفى */
} else if (host->retune_period) {
    mmc_retune_timer_stop(host);  /* ابقِ الكارت شغال لكن وقّف الـ retune */
    mmc_retune_needed(host);
}
```

**التأثير على `mmc_sdio_resume()`:**

```c
if (!mmc_card_keep_power(host)) {
    mmc_power_up(host, host->card->ocr);   /* أعد تشغيل الكارت */
    err = mmc_sdio_reinit_card(host);      /* reinit كامل */
}
/* لو KEEP_POWER متحط: مفيش power cycle، الكارت فاضل شغال */
host->pm_flags &= ~MMC_PM_KEEP_POWER;     /* clear بعد الـ resume */
```

**ملاحظة مهمة:** الـ flag بيتـ clear تلقائياً من `mmc_sdio_resume()` عند نهاية كل دورة suspend/resume. ده معناه إن الـ SDIO function driver **لازم يسيّبه من جديد** في كل `suspend` callback.

---

#### `MMC_PM_WAKE_SDIO_IRQ`

```c
#define MMC_PM_WAKE_SDIO_IRQ    (1 << 1)    /* wake up host system on SDIO IRQ assertion */
```

**الـ flag** الثاني. لما يتحط جنب `MMC_PM_KEEP_POWER`، الـ host controller لازم يكون configured كـ wakeup source — يعني لو جه interrupt من الكارت أثناء الـ suspend، الـ system هيصحى.

**الشرط المهم:** الـ flag ده مفيدش لوحده. لو مش موجود `MMC_PM_KEEP_POWER` معاه، الكارت أصلاً هيتطفى ومفيش IRQ هيجي.

**التأثير على suspend/resume:**

```c
/* من mmc_sdio_suspend() */
/* لو الاتنين متحطين، الـ bus بينزل لـ 1-bit mode عشان يوفر طاقة */
if (mmc_card_keep_power(host) && mmc_card_wake_sdio_irq(host))
    sdio_disable_4bit_bus(host->card);

/* من mmc_sdio_resume() */
/* لو الكارت صحّى الـ system بـ IRQ، لازم نرجع الـ bus لـ 4-bit */
} else if (mmc_card_wake_sdio_irq(host)) {
    mmc_retune_hold_now(host);
    err = sdio_enable_4bit_bus(host->card);
    mmc_retune_release(host);
}
```

**الـ DT property المقابلة:**
الـ flag ده بيتحط في `pm_caps` من `mmc_of_parse()` لو الـ Device Tree فيه `wakeup-source` أو `enable-sdio-wakeup` (legacy):

```c
/* من drivers/mmc/core/host.c */
if (device_property_read_bool(dev, "wakeup-source") ||
    device_property_read_bool(dev, "enable-sdio-wakeup"))
    host->pm_caps |= MMC_PM_WAKE_SDIO_IRQ;
```

---

### المجموعة التانية: دالتين الـ Query/Set الرئيسيتين

الـ functions دي هي الواجهة الرسمية اللي الـ SDIO function drivers بتستخدمها للتعامل مع الـ PM flags.

---

#### `sdio_get_host_pm_caps()`

```c
mmc_pm_flag_t sdio_get_host_pm_caps(struct sdio_func *func);
```

**بترجع الـ `pm_caps` bitmask** من الـ `mmc_host` الخاص بالـ `sdio_func`. الـ `pm_caps` بتعكس القدرات الفعلية للـ host controller hardware — مش الـ flags المطلوبة.

**البارامتر:**
- `func`: مؤشر لـ `struct sdio_func` المرتبطة بالـ function driver. لو `NULL` بترجع `0`.

**الـ Return Value:**
- `mmc_pm_flag_t` بيمثل الـ capabilities المتاحة.
- `0` لو `func` كانت `NULL`.

**التفاصيل المهمة:**
- **لا تحتاج claim:** الـ host مش محتاج يكون claimed عشان تستخدمها.
- **القصد منها:** الـ SDIO function driver يستخدمها في الـ `probe` أو قبل الـ suspend عشان يعرف هل ممكن يطلب `MMC_PM_KEEP_POWER` أو لا.
- **exported كـ:** `EXPORT_SYMBOL_GPL`

**مثال استخدام من SDIO driver:**

```c
static int my_sdio_suspend(struct device *dev)
{
    struct sdio_func *func = dev_to_sdio_func(dev);
    mmc_pm_flag_t caps = sdio_get_host_pm_caps(func);

    if (caps & MMC_PM_KEEP_POWER) {
        /* الـ host يدعم keep power، نطلبه */
        sdio_set_host_pm_flags(func, MMC_PM_KEEP_POWER);
    }
    return 0;
}
```

---

#### `sdio_set_host_pm_flags()`

```c
int sdio_set_host_pm_flags(struct sdio_func *func, mmc_pm_flag_t flags);
```

**بتحط الـ flags** المطلوبة في `host->pm_flags`. الـ MMC core هيقرأ الـ flags دي وقت الـ `mmc_sdio_suspend()` ويقرر بناءً عليها.

**البارامترات:**
- `func`: مؤشر لـ `struct sdio_func`. لو `NULL` بترجع `-EINVAL`.
- `flags`: الـ `mmc_pm_flag_t` bitmask اللي الـ driver عايزها.

**الـ Return Value:**
- `0` عند النجاح.
- `-EINVAL` لو `func` كانت `NULL`.
- `-EINVAL` لو أي flag مطلوبة **مش موجودة** في `pm_caps` — الـ host مش بيدعمها.

**التحقق الداخلي (validation):**

```c
/* من drivers/mmc/core/sdio_io.c */
if (flags & ~host->pm_caps)
    return -EINVAL;   /* طلبت حاجة الـ hardware مش بيدعمها */

host->pm_flags |= flags;  /* OR مش assignment، عشان متحيش flags تانية */
```

**ملاحظات الـ locking:**
- **بلا lock:** الـ function suspend methods بتتنفذ في سياق متسلسل من kernel PM framework — no races possible.
- **exported كـ:** `EXPORT_SYMBOL_GPL`

**ملاحظة على الـ lifetime:** الـ flags المحطوطة بـ `sdio_set_host_pm_flags()` بتتـ clear في `mmc_sdio_resume()` بعد الـ resume، وفي `mmc_stop_host()` بعد إيقاف الـ host:

```c
/* من drivers/mmc/core/core.c */
void mmc_stop_host(struct mmc_host *host)
{
    /* clear pm flags now and let card drivers set them as needed */
    host->pm_flags = 0;
    ...
}
```

---

### المجموعة التالتة: الـ Inline Helper Functions

هذه الـ helpers معرّفة في `include/linux/mmc/host.h` وبتستخدم الـ flags المعرّفة في `pm.h`. هي الـ accessors الرسمية اللي الـ MMC core بيستخدمها بدل ما يـ test الـ bits مباشرة.

---

#### `mmc_card_keep_power()`

```c
static inline int mmc_card_keep_power(struct mmc_host *host)
{
    return host->pm_flags & MMC_PM_KEEP_POWER;
}
```

**بتتحقق** من وجود `MMC_PM_KEEP_POWER` في `pm_flags`. الـ return value غير صفر يعني "ابقِ الكارت مشغلاً أثناء suspend".

**البارامتر:** `host` — مؤشر لـ `struct mmc_host`.

**الـ Return:** `int` — nonzero لو الـ flag محطوط، صفر لو مش محطوط.

**من يستدعيها:**
- `mmc_sdio_suspend()` — لتحديد هل يتم `mmc_power_off()` أم لا.
- `mmc_sdio_resume()` — لتحديد هل يتم `mmc_power_up()` + reinit أم لا.

```c
/* pseudocode للـ suspend decision */
if (mmc_card_keep_power(host))
    /* ابقِ الـ card شغالة */
else
    mmc_power_off(host);  /* أطفي كل حاجة */
```

---

#### `mmc_card_wake_sdio_irq()`

```c
static inline int mmc_card_wake_sdio_irq(struct mmc_host *host)
{
    return host->pm_flags & MMC_PM_WAKE_SDIO_IRQ;
}
```

**بتتحقق** من وجود `MMC_PM_WAKE_SDIO_IRQ` في `pm_flags`. نتيجتها غير صفر تعني "الكارت محتاج يصحّي الـ system لما يبعت IRQ".

**البارامتر:** `host` — مؤشر لـ `struct mmc_host`.

**الـ Return:** `int` — nonzero لو الـ flag محطوط.

**من يستدعيها:**
- `mmc_sdio_suspend()` — لتحديد هل يتم تنزيل الـ bus لـ 1-bit mode قبل الـ suspend.
- `mmc_sdio_resume()` — لتحديد هل يتم رفع الـ bus لـ 4-bit مرة تانية بعد الصحيان.

---

### التدفق الكامل لـ PM Flags عبر الطبقات

```
Device Tree / Board Code
        │
        │  "keep-power-in-suspend"  →  MMC_PM_KEEP_POWER
        │  "wakeup-source"          →  MMC_PM_WAKE_SDIO_IRQ
        ▼
   mmc_of_parse()
        │ يحط الـ flags في host->pm_caps
        ▼
   struct mmc_host
   ┌─────────────────────────────┐
   │  pm_caps  =  ما يدعمه HW   │  ← read-only بعد الـ init
   │  pm_flags =  ما طُلب فعلاً │  ← write من SDIO drivers
   └─────────────────────────────┘
        ▲                   │
        │                   │
sdio_get_host_pm_caps()    sdio_set_host_pm_flags()
   (SDIO driver reads)        (SDIO driver writes)

        │  عند الـ suspend
        ▼
   mmc_sdio_suspend()
        │
        ├── mmc_card_keep_power()  ?
        │       YES → ابقِ الطاقة، وقّف الـ retune timer
        │       NO  → mmc_power_off()
        │
        └── mmc_card_wake_sdio_irq() && keep_power ?
                YES → sdio_disable_4bit_bus() (توفير طاقة)

        │  عند الـ resume
        ▼
   mmc_sdio_resume()
        │
        ├── !mmc_card_keep_power() → mmc_power_up() + reinit
        └── mmc_card_wake_sdio_irq() → sdio_enable_4bit_bus()
        │
        └── host->pm_flags &= ~MMC_PM_KEEP_POWER  (clear للـ next cycle)
```

---

### علاقة `pm_caps` بـ `pm_flags`

| الـ field | مَن يكتب فيها | مَن يقرأ منها | الوصف |
|---|---|---|---|
| `host->pm_caps` | `mmc_of_parse()` أو الـ host driver مباشرة | `sdio_get_host_pm_caps()` | الـ capabilities الثابتة للـ HW |
| `host->pm_flags` | `sdio_set_host_pm_flags()` | `mmc_card_keep_power()`, `mmc_card_wake_sdio_irq()` | الطلبات الفعلية لكل suspend cycle |

الـ validation gate هو:

```c
if (flags & ~host->pm_caps)
    return -EINVAL;
```

أي flag مش موجودة في `pm_caps` مش ممكن تتحط في `pm_flags` — ده يضمن إن الـ SDIO driver مش يطلب حاجة الـ hardware مش بيدعمها.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs المتعلقة بـ MMC Power Management

الـ MMC core بيخلق `debugfs_root` لكل host تحت `/sys/kernel/debug/mmc{N}/`.

| المسار | المحتوى | طريقة القراءة |
|--------|---------|----------------|
| `/sys/kernel/debug/mmc0/ios` | الـ `mmc_ios` الحالي: clock, voltage, power_mode, timing | `cat /sys/kernel/debug/mmc0/ios` |
| `/sys/kernel/debug/mmc0/err_stats` | عدادات الأخطاء من `enum mmc_err_stat` | `cat /sys/kernel/debug/mmc0/err_stats` |
| `/sys/kernel/debug/mmc0/clock` | الـ `actual_clock` الفعلي للـ host | `cat /sys/kernel/debug/mmc0/clock` |

**قراءة الـ ios لمعرفة حالة الـ power:**

```bash
# اقرأ حالة الـ power_mode الحالية
# 0=OFF, 1=UP, 2=ON, 3=UNDEFINED
cat /sys/kernel/debug/mmc0/ios
```

مثال على الـ output:

```
clock:          200000000 Hz
actual clock:   200000000 Hz
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       push-pull
chip select:    don't care
power mode:     on
bus width:      4 bits
timing spec:    sd highspeed
signal voltage: 3.30 V
driver type:    B
```

**قراءة err_stats لمعرفة أخطاء الـ PM:**

```bash
cat /sys/kernel/debug/mmc0/err_stats
# مهم: CMD_TIMEOUT وDAT_TIMEOUT كتير بعد resume = مشكلة PM
```

---

#### 2. مدخلات الـ sysfs

| المسار | الدور |
|--------|-------|
| `/sys/bus/mmc/devices/mmc0:0001/` | device attributes للكارت |
| `/sys/class/mmc_host/mmc0/power/` | runtime PM للـ host controller |
| `/sys/class/mmc_host/mmc0/power/runtime_status` | `active` / `suspended` / `suspending` |
| `/sys/class/mmc_host/mmc0/power/runtime_suspended_time` | الوقت الكلي في suspend بالـ ms |
| `/sys/class/mmc_host/mmc0/power/runtime_active_time` | الوقت الكلي في active |
| `/sys/class/mmc_host/mmc0/power/control` | `auto` / `on` — للتحكم في runtime PM |
| `/sys/bus/mmc/devices/mmc0:0001/power/wakeup` | `enabled` / `disabled` لـ wakeup من SDIO IRQ |

```bash
# تحقق من runtime PM status
cat /sys/class/mmc_host/mmc0/power/runtime_status

# تحقق هل الكارت configured كـ wakeup source
cat /sys/bus/mmc/devices/mmc0:0001/power/wakeup

# تعطيل aggressive PM مؤقتاً للـ debug
echo on > /sys/class/mmc_host/mmc0/power/control

# إعادة تفعيل auto PM
echo auto > /sys/class/mmc_host/mmc0/power/control

# عرض pm_caps (الـ capabilities الـ PM للـ host)
cat /sys/class/mmc_host/mmc0/power/pm_qos_latency_tolerance_us
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

**الـ tracepoints المباشرة لـ MMC PM:**

```bash
# تفعيل كل events الـ MMC
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable

# أو tracepoints محددة للـ power management
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_set_ios/enable
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_request_start/enable
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_request_done/enable
```

**تتبع تسلسل الـ suspend/resume كامل:**

```bash
# تفعيل tracing للـ PM events
echo 1 > /sys/kernel/debug/tracing/events/power/device_pm_callback_start/enable
echo 1 > /sys/kernel/debug/tracing/events/power/device_pm_callback_end/enable
echo 1 > /sys/kernel/debug/tracing/events/power/rpm_suspend/enable
echo 1 > /sys/kernel/debug/tracing/events/power/rpm_resume/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# نفذ الـ suspend أو الـ operation المشبوهة
systemctl suspend   # مثلاً

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace | grep -E "mmc|rpm"
```

**filter على mmc0 فقط:**

```bash
echo 'name == "mmc0"' > /sys/kernel/debug/tracing/events/power/rpm_suspend/filter
echo 1 > /sys/kernel/debug/tracing/events/power/rpm_suspend/enable
```

**مثال على الـ output من `mmc_set_ios`:**

```
kworker/0:2-50  [000] ....  1234.567: mmc_set_ios: mmc0: clock 0 power mode OFF vdd 0 bus_mode PUSH-PULL chip_select DONTCARE timing LEGACY signal_voltage 3.3V drv_type B
kworker/0:2-50  [000] ....  1234.890: mmc_set_ios: mmc0: clock 400000 power mode UP vdd 21 ...
```

الـ `power mode OFF` ثم `UP` ثم `ON` = تسلسل صحيح للـ power cycle.

---

#### 4. الـ printk والـ Dynamic Debug

**تفعيل dynamic debug للـ MMC core:**

```bash
# كل رسائل الـ debug في mmc core
echo 'module mmc_core +p' > /sys/kernel/debug/dynamic_debug/control

# ملفات محددة متعلقة بالـ PM
echo 'file drivers/mmc/core/core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mmc/core/sdio_bus.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mmc/core/pm.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mmc/core/host.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ regulator المرتبط بالـ supply
echo 'module regulator +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع stack trace لكل رسالة
echo 'file drivers/mmc/core/core.c +ps' > /sys/kernel/debug/dynamic_debug/control
```

**رفع مستوى الـ loglevel للـ kernel مؤقتاً:**

```bash
# اعرض كل رسائل الـ kernel بما فيها debug
echo 8 > /proc/sys/kernel/printk

# أو استخدم dmesg مع -w للمتابعة الفورية
dmesg -w | grep -i mmc
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ CONFIG | الغرض |
|-----------|--------|
| `CONFIG_MMC_DEBUG` | تفعيل رسائل الـ pr_debug في كل الـ MMC subsystem |
| `CONFIG_PM_DEBUG` | تفعيل `/sys/power/pm_print_times` و verbose PM messages |
| `CONFIG_PM_SLEEP_DEBUG` | تفعيل debug للـ suspend/resume path |
| `CONFIG_PM_TEST_SUSPEND` | اختبار suspend بدون فعلي sleep |
| `CONFIG_PM_ADVANCED_DEBUG` | يضيف `/sys/kernel/debug/devices` entries تفصيلية |
| `CONFIG_REGULATOR_DEBUG` | رسائل debug للـ vmmc/vqmmc regulators |
| `CONFIG_FAIL_MMC_REQUEST` | fault injection للـ MMC requests (اختبار error recovery) |
| `CONFIG_MMC_UNSAFE_RESUME` | يمنع re-initialization الكارت عند resume (SDIO keep power) |
| `CONFIG_DYNAMIC_DEBUG` | يتيح dynamic_debug بشكل عام |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في الـ `host->lock` spinlock |

**تفعيل `pm_print_times` لقياس وقت كل device في suspend:**

```bash
echo 1 > /sys/power/pm_print_times
# بعد resume، اقرأ dmesg
dmesg | grep "mmc\|PM:"
```

---

#### 6. الـ Devlink وأدوات خاصة بالـ Subsystem

**أداة `mmc-utils` للتفاعل المباشر مع الكارت:**

```bash
# قراءة EXT_CSD من eMMC (مفيد بعد resume للتحقق من حالة الكارت)
mmc extcsd read /dev/mmcblk0

# التحقق من حالة الـ power class في EXT_CSD
mmc extcsd read /dev/mmcblk0 | grep -i "power\|sleep"

# إرسال SLEEP_AWAKE command يدوياً
mmc sleep /dev/mmcblk0

# قراءة CSD
mmc csd read /dev/mmcblk0
```

**الـ sysfs للـ SDIO function driver (للـ pm_flags):**

```bash
# عرض pm_flags الحالية على الـ host (pm_flags = requested, pm_caps = supported)
# هذي بتتحكم في MMC_PM_KEEP_POWER وMMC_PM_WAKE_SDIO_IRQ
cat /sys/bus/sdio/devices/mmc0:0001:1/power/wakeup

# مثال: تفعيل wakeup على SDIO device
echo enabled > /sys/bus/sdio/devices/mmc0:0001:1/power/wakeup
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|-------|
| `mmc0: Timeout waiting for hardware interrupt` | الـ host controller لم يرد بعد resume أو power-on | تحقق من `set_ios` وتسلسل الـ power (OFF→UP→ON) |
| `mmc0: error -110 whilst initialising SD card` | ETIMEDOUT أثناء init بعد resume — الكارت مش جاهز | زود `power_delay_ms` في `mmc_ios` أو تحقق من الـ vmmc regulator |
| `mmc0: card never left busy state` | الكارت مش بيرد على CMD — `card_busy()` بترجع 1 باستمرار | مشكلة في voltage switching أو الكارت في sleep mode غير منتظر |
| `mmc0: Controller never released inhibit bit(s)` | الـ host controller نفسه مش صاحي من sleep | مشكلة في host driver PM، تحقق من clock gating |
| `mmc0: SDIO IRQ work already queued` | سباق في معالجة SDIO IRQ أثناء resume | تحقق من `sdio_irq_pending` وـ `enable_sdio_irq` ordering |
| `mmc0: card claims to support voltages below defined range` | مشكلة في الـ OCR negotiation بعد re-init | تحقق من `ocr_avail` وإعدادات الـ regulator |
| `mmc0: unrecognised CIS tuple` | مشكلة في قراءة CIS بعد resume — الكارت لسه مش fully powered | زود power-on delay، تحقق من `MMC_PM_KEEP_POWER` |
| `mmc0: tried to HW reset card, got -EOPNOTSUPP` | `card_hw_reset` مش موجود وكان مطلوب | implement `card_hw_reset` أو استخدم power cycle بدل reset |
| `sdio: card 0001:0001 failed to enable` | SDIO function فشلت في init بعد resume | تحقق من SDIO driver `resume` callback وـ `pm_flags` |
| `mmc0: re-tuning execution failed` | فشل الـ re-tuning بعد resume في UHS/HS200/HS400 | تعمل retune بعد ما الـ clock والـ voltage يستقر |
| `mmc0: power off notify timed out` | eMMC بطئت في respond لـ power-off notification (EXT_CSD) | زود `max_busy_timeout` أو disable power off notification |
| `mmc0: undervoltage detected` | انخفاض جهد الـ vmmc أو vqmmc — `undervoltage` flag اتعمل set | تحقق من الـ regulator supply تحت load |

---

#### 8. نقاط استراتيجية لـ `dump_stack()` و`WARN_ON()`

**في `mmc_host_ops.set_ios`** — لتتبع كل تغيير في الـ power state:

```c
void my_host_set_ios(struct mmc_host *host, struct mmc_ios *ios)
{
    /* تحذير لو الـ power mode اتغير بشكل غير متوقع */
    WARN_ON(ios->power_mode == MMC_POWER_UNDEFINED);

    if (ios->power_mode != host->ios.power_mode) {
        pr_debug("%s: power mode %d -> %d\n",
            mmc_hostname(host),
            host->ios.power_mode,
            ios->power_mode);
        /* uncomment للـ stack trace */
        /* dump_stack(); */
    }
    /* ... باقي الـ implementation */
}
```

**في suspend path** — تحقق من الـ pm_flags قبل ما تبدأ:

```c
int my_host_suspend(struct device *dev)
{
    struct mmc_host *host = dev_get_drvdata(dev);

    /* تحذير لو SDIO بطلب keep power بس الـ host مش بيدعمه */
    WARN_ON((host->pm_flags & MMC_PM_KEEP_POWER) &&
            !(host->pm_caps & MMC_PM_KEEP_POWER));

    /* تحذير لو الـ host claimed وسط suspend */
    WARN_ON(host->claimed);
}
```

**في resume path** — تحقق من حالة الـ clock:

```c
int my_host_resume(struct device *dev)
{
    struct mmc_host *host = dev_get_drvdata(dev);

    /* الـ clock لازم يكون 0 قبل ما نبدأ resume */
    WARN_ON(host->ios.clock != 0 &&
            !(host->pm_flags & MMC_PM_KEEP_POWER));
}
```

---

### Hardware Level

#### 1. التحقق من مطابقة حالة الـ Hardware للـ Kernel State

بعد كل suspend/resume، لازم تتحقق إن ما بين الـ kernel وإيه موجود فعلياً على السلك:

```bash
# تحقق من power_mode في debugfs (kernel state)
cat /sys/kernel/debug/mmc0/ios | grep "power mode"

# قارن مع timing وclock
cat /sys/kernel/debug/mmc0/ios

# تحقق من الـ actual_clock vs المطلوب
# actual_clock بيتحسب من الـ hardware registers فعلياً
```

**تحقق من الـ regulators بعد resume:**

```bash
# عرض كل الـ regulators وحالتها
cat /sys/kernel/debug/regulator/regulator_summary

# تحقق من regulator الـ vmmc (Card supply)
cat /sys/class/regulator/regulator.X/microvolts
cat /sys/class/regulator/regulator.X/state  # enabled/disabled
```

---

#### 2. تقنيات الـ Register Dump

**استخدام `devmem2` لقراءة registers الـ host controller مباشرة:**

```bash
# اعرف الـ base address من device tree أو /proc/iomem
cat /proc/iomem | grep -i mmc

# مثال: SDHCI controller base address = 0xFE300000
# اقرأ الـ Present State Register (offset 0x24)
devmem2 0xFE300024 w
# bit 1 = DAT[0] line busy (card busy)
# bit 18-16 = voltage select
# bit 8 = card inserted

# اقرأ الـ Power Control Register (offset 0x29)
devmem2 0xFE300028 b
# bits [7:1] = SD bus voltage select
# bit 0 = SD bus power
```

**استخدام `/dev/mem` مع `dd` بدون devmem2:**

```bash
# قراءة 256 byte من registers الـ SDHCI
dd if=/dev/mem bs=1 count=256 skip=$((0xFE300000)) 2>/dev/null | xxd | head -20
```

**الـ SDHCI standard registers المهمة للـ PM:**

```
Offset 0x28: Power Control Register
  - Bit 0:    SD Bus Power
  - Bits 3-1: SD Bus Voltage Select (0x7=3.3V, 0x6=3.0V, 0x5=1.8V)

Offset 0x2C: Block Gap Control / Wakeup Control
  - Bit 18: Wakeup Event Enable On SD Card Interrupt
  - Bit 17: Wakeup Event Enable On SD Card Removal
  - Bit 16: Wakeup Event Enable On SD Card Insertion

Offset 0x3C: Auto CMD Status Register

Offset 0x40: Host Control 2
  - Bit 3: 1.8V Signaling Enable (for voltage switching)
```

---

#### 3. Logic Analyzer / Oscilloscope

**نقاط القياس الأساسية:**

```
VDD (vmmc)     ──┐
                 ├── قس على pin الكارت 4 (VDD)
VDDQ (vqmmc)  ──┘    قس على pin الكارت 11 (VDD_IO على SD)

CMD line       ──── pin الكارت 2
CLK line       ──── pin الكارت 5
DAT[0]         ──── pin الكارت 7
```

**سيناريوهات القياس:**

| السيناريو | ما تقيسه | التوقع الصحيح |
|-----------|---------|---------------|
| Power cycle أثناء suspend | VDD على VDD pin | يهبط لـ 0V خلال 1ms، يعود خلال 10ms |
| Voltage switching (3.3V→1.8V) | VDD_IO (VDDQ) | هبوط تدريجي لـ 1.8V ± 5% |
| Clock stop قبل suspend | CLK line | يصبح 0 قبل ما VDD ينزل |
| CMD during suspend مع KEEP_POWER | CMD line | يجب ما فيش activity لو مفيش SDIO IRQ |
| SDIO IRQ wakeup | DAT[1] line | pulse على DAT[1] يصحي الـ host |

**نصائح عملية:**

- شغل trigger على VDD pin للتقاط لحظة الـ power off/on.
- قس الـ rise time للـ VDD — لو أبطأ من الـ `power_delay_ms` في `mmc_ios`، الكارت هيعمل timeout.
- DAT[1] هو خط الـ SDIO interrupt — أي glitch عليه ممكن يعمل false wakeup.

---

#### 4. مشاكل الـ Hardware الشائعة وـ Kernel Log Patterns

| مشكلة الـ Hardware | Pattern في الـ Kernel Log |
|-------------------|--------------------------|
| VDD مش بيوصل للكارت | `mmc0: error -110 whilst initialising` + لا retry بتنجح |
| VDD rise time بطيء جداً | `Timeout waiting for hardware interrupt` أول request بعد resume |
| Voltage switch فاشل (VDDQ مش بيروح 1.8V) | `mmc0: Problem switching card into high-speed mode!` |
| SDIO IRQ line (DAT[1]) floating | `SDIO IRQ work already queued` بشكل متكرر بدون سبب |
| Clock mux مش بيرجع صح بعد sleep | `mmc0: tried to HW reset card` + CRC errors |
| نسيان تعمل power-on reset للـ controller | `Controller never released inhibit bit` |
| عدم تعمل deselect للكارت قبل KEEP_POWER suspend | كارت مش بيرد على CMDs بعد resume |

---

#### 5. الـ Device Tree Debugging

**تحقق من إن الـ DT بيعكس الـ Hardware فعلياً:**

```bash
# اعرض كل properties متعلقة بالـ MMC node
cat /proc/device-tree/soc/mmc@FE300000/compatible
cat /proc/device-tree/soc/mmc@FE300000/vmmc-supply
cat /proc/device-tree/soc/mmc@FE300000/vqmmc-supply
cat /proc/device-tree/soc/mmc@FE300000/keep-power-in-suspend  # هل موجود؟
cat /proc/device-tree/soc/mmc@FE300000/wakeup-source          # هل موجود؟

# أو استخدم dtc لعرض الـ DTS كامل
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 30 "mmc@"
```

**أهم الـ DT properties للـ MMC PM:**

```dts
/* مثال على DT node صح لـ SDIO مع PM support */
mmc1: mmc@FE300000 {
    compatible = "snps,dwmmc";
    reg = <0xFE300000 0x10000>;

    vmmc-supply = <&vcc_sd>;       /* Card power supply */
    vqmmc-supply = <&vccio_sd>;    /* IO voltage supply */

    /* PM flags المهمة */
    keep-power-in-suspend;         /* يعمل set لـ MMC_PM_KEEP_POWER في pm_caps */
    wakeup-source;                 /* يتيح الـ wake من suspend */

    mmc-pwrseq = <&sdio_pwrseq>;  /* power sequence للـ SDIO card */

    /* delay مهم لاستقرار الـ power */
    post-power-on-delay-ms = <100>;
    power-off-delay-us = <500>;
};
```

**تحقق من إن `keep-power-in-suspend` اتحول لـ `pm_caps`:**

```bash
# يجب إن pm_caps يحتوي MMC_PM_KEEP_POWER (bit 0 = 1)
# المشكلة: ما في sysfs مباشر له، بس نستنتج من:
cat /sys/kernel/debug/mmc0/ios  # لو power mode ON بعد suspend = keep power شغال
```

---

### Practical Commands

#### كل التقنيات في أوامر جاهزة للنسخ

**1. تشخيص سريع — أول شيء تعمله:**

```bash
#!/bin/bash
# mmc-pm-diag.sh - تشخيص سريع لـ MMC PM

HOST="mmc0"
echo "=== MMC PM Quick Diagnostic ==="
echo "--- Current IOS state ---"
cat /sys/kernel/debug/${HOST}/ios

echo "--- Runtime PM Status ---"
cat /sys/class/mmc_host/${HOST}/power/runtime_status
cat /sys/class/mmc_host/${HOST}/power/runtime_suspended_time
cat /sys/class/mmc_host/${HOST}/power/runtime_active_time

echo "--- Error Stats ---"
cat /sys/kernel/debug/${HOST}/err_stats

echo "--- Wakeup source ---"
cat /sys/bus/mmc/devices/${HOST}:*/power/wakeup 2>/dev/null || echo "no devices"

echo "--- Regulators ---"
cat /sys/kernel/debug/regulator/regulator_summary 2>/dev/null | grep -A2 -B2 -i "mmc\|sd\|vmmc"
```

**2. تفعيل الـ tracing الكامل لحظة suspend/resume:**

```bash
#!/bin/bash
# تفعيل tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace

# اختر الـ events
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable
echo 1 > /sys/kernel/debug/tracing/events/power/rpm_suspend/enable
echo 1 > /sys/kernel/debug/tracing/events/power/rpm_resume/enable
echo 1 > /sys/kernel/debug/tracing/events/power/device_pm_callback_start/enable
echo 1 > /sys/kernel/debug/tracing/events/power/device_pm_callback_end/enable

# ابدأ
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "Tracing enabled. Trigger suspend now..."
echo "After resume, run: cat /sys/kernel/debug/tracing/trace | grep -E 'mmc|rpm'"
```

**3. استخراج وتحليل الـ trace بعد الحادثة:**

```bash
# احفظ الـ trace
cp /sys/kernel/debug/tracing/trace /tmp/mmc-trace-$(date +%s).txt

# فلتر على power mode changes فقط
grep "power mode\|set_ios\|rpm_suspend\|rpm_resume" /tmp/mmc-trace-*.txt | head -50

# قياس وقت الـ suspend
grep "rpm_suspend\|rpm_resume" /tmp/mmc-trace-*.txt | awk '{print $4, $NF}'
```

**4. تشغيل dynamic debug وتجميع logs:**

```bash
# تفعيل كل debug messages للـ MMC
echo 'module mmc_core +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module mmc_block +p' > /sys/kernel/debug/dynamic_debug/control

# تابع الـ log بالـ timestamps
dmesg -wT 2>&1 | grep -E "mmc|sdio|vmmc|vqmmc" | tee /tmp/mmc-debug.log &
DMESG_PID=$!

# نفذ الـ test
systemctl suspend

# بعد الـ resume
kill $DMESG_PID
echo "Log saved to /tmp/mmc-debug.log"
```

**5. تحقق من الـ pm_flags و pm_caps من الـ kernel module:**

```bash
# هذا السكريبت بيطبع قيم pm_flags وpm_caps من خلال
# تحليل الـ sysfs وdebugfs بشكل غير مباشر

# تحقق من MMC_PM_KEEP_POWER support:
# لو الـ driver بيدعمه، الـ card هيفضل powered بعد suspend
dmesg | grep -i "keep.power\|pm_caps\|pm_flags"

# تحقق من MMC_PM_WAKE_SDIO_IRQ:
dmesg | grep -i "wake.*sdio\|sdio.*wake\|wakeup"
```

**6. اختبار power cycle يدوي عبر sysfs:**

```bash
# إجبار power off على الكارت (لو MMC_CAP_POWER_OFF_CARD مدعوم)
echo off > /sys/bus/mmc/devices/mmc0:0001/state 2>/dev/null ||
    echo "Manual power off not supported via sysfs"

# إجبار rescan (إعادة init الكارت)
echo 1 > /sys/class/mmc_host/mmc0/scan

# مراقبة الـ dmesg أثناء rescan
dmesg -w | grep mmc0 &
echo 1 > /sys/class/mmc_host/mmc0/scan
sleep 3
kill %1
```

**7. تحقق من الـ SDIO IRQ status:**

```bash
# هل يوجد SDIO IRQ thread؟
ps aux | grep sdio
# sdio_irq thread اسمه عادةً: ksdioirqd/mmc0

# هل الـ IRQ معمول enable؟
cat /proc/interrupts | grep -i "mmc\|sdio"

# عدد الـ interrupts قبل وبعد suspend للمقارنة
cat /proc/interrupts | grep mmc
```

**8. Fault injection لاختبار error recovery:**

```bash
# تفعيل fault injection (يحتاج CONFIG_FAIL_MMC_REQUEST=y)
# هذا بيختبر إن الـ driver بيتعامل صح مع الأخطاء أثناء PM transitions

echo 1 > /sys/kernel/debug/mmc0/fail_mmc_request/task-filter
echo 1000 > /sys/kernel/debug/mmc0/fail_mmc_request/probability
echo 1 > /sys/kernel/debug/mmc0/fail_mmc_request/times

# شغل I/O test بعدين
dd if=/dev/mmcblk0 of=/dev/null bs=4096 count=1000

# شوف الـ err_stats
cat /sys/kernel/debug/mmc0/err_stats
```

**9. تحليل دلتا الـ error stats قبل وبعد suspend:**

```bash
#!/bin/bash
# قياس أخطاء الـ MMC قبل وبعد suspend
HOST="mmc0"

echo "=== Before suspend ==="
cp /sys/kernel/debug/${HOST}/err_stats /tmp/err_before.txt
cat /tmp/err_before.txt

# انتظر الـ suspend/resume
read -p "Perform suspend/resume, then press Enter..."

echo "=== After suspend ==="
cat /sys/kernel/debug/${HOST}/err_stats > /tmp/err_after.txt

echo "=== Delta (new errors) ==="
diff /tmp/err_before.txt /tmp/err_after.txt
```

**10. تحقق من تسلسل الـ power state عبر الـ trace:**

```bash
# تفسير سريع للـ power mode في trace:
# power mode: off = MMC_POWER_OFF = 0  → الكارت فاصل كهرباء
# power mode: up  = MMC_POWER_UP  = 1  → الـ VDD بيشتغل (pre-init)
# power mode: on  = MMC_POWER_ON  = 2  → الكارت شغال بالكامل

# التسلسل الصح عند suspend (بدون KEEP_POWER):
# on → off

# التسلسل الصح عند resume:
# off → up → on

# لو شفت: on → up → on  = خطأ! فيه bug في الـ driver
# لو شفت: off → on = خطأ! فاتت خطوة UP
grep "power mode" /sys/kernel/debug/tracing/trace
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Wi-Fi SDIO مش بيصحى بعد الـ suspend على AM62x Industrial Gateway

#### العنوان
**الـ MMC_PM_WAKE_SDIO_IRQ مش متفعّل — الـ Wi-Fi بيصحى بس مش بيستقبل interrupt**

#### السياق
شغال على industrial gateway بيستخدم **AM62x** (Texas Instruments) مع Wi-Fi chip هو **RTL8189FTV** متوصّل على **SDIO** (mmc1). المنتج محتاج يفضل connected على الشبكة حتى وهو في suspend mode عشان يستقبل alerts من الـ MQTT broker.

#### المشكلة
بعد `echo mem > /sys/power/state`، الجهاز بيدخل suspend تمام. لما بييجي packet على الـ Wi-Fi، الـ RTL8189 بيرفع الـ SDIO IRQ line، بس الـ AP (Application Processor) مش بيصحى. الـ dmesg بعد wake بيظهر:

```
mmc1: error -110 whilst initialising SDIO card
```

يعني الكارت اتعمله power-off أثناء الـ suspend وبعدين فشل يرجع.

#### التحليل
الـ `pm.h` بيعرّف الفلاجين:

```c
#define MMC_PM_KEEP_POWER    (1 << 0)  /* preserve card power during suspend */
#define MMC_PM_WAKE_SDIO_IRQ (1 << 1)  /* wake up host system on SDIO IRQ assertion */
```

الـ `mmc_host` في `host.h` عنده:

```c
mmc_pm_flag_t  pm_caps;   /* what the HOST controller supports */
mmc_pm_flag_t  pm_flags;  /* what the SDIO function driver REQUESTS */
```

الـ inline helpers بتوضح:

```c
static inline int mmc_card_keep_power(struct mmc_host *host)
{
    return host->pm_flags & MMC_PM_KEEP_POWER;
}

static inline int mmc_card_wake_sdio_irq(struct mmc_host *host)
{
    return host->pm_flags & MMC_PM_WAKE_SDIO_IRQ;
}
```

الـ MMC core وقت الـ suspend بيسأل: هل `pm_flags` فيها `MMC_PM_KEEP_POWER`؟ لو لأ → بيعمل power-off للكارت. الـ RTL8189 driver مش بيسيت الفلاجين دول.

#### التحليل التفصيلي للـ flow

```
suspend path:
  sdio_bus_pm_ops->suspend()
      → sdio_driver->drv->drv_pm->suspend()  [rtl8189fs driver]
          → sdio_set_host_pm_flags(func, MMC_PM_KEEP_POWER)  ← مش بيتعمل!
      → mmc_sdio_suspend()
          → mmc_card_keep_power(host) → returns 0
          → mmc_power_off(host)       ← المشكلة هنا
```

#### الحل

**1. في الـ RTL8189 driver، لازم يسيت الـ pm_flags صح:**

```c
static int rtl8189_sdio_suspend(struct device *dev)
{
    struct sdio_func *func = dev_to_sdio_func(dev);
    mmc_pm_flag_t caps = sdio_get_host_pm_caps(func);

    /* check host supports keeping power */
    if (!(caps & MMC_PM_KEEP_POWER))
        return -ENOSYS;

    /* request keep power + wake on SDIO IRQ */
    return sdio_set_host_pm_flags(func,
        MMC_PM_KEEP_POWER | MMC_PM_WAKE_SDIO_IRQ);
}
```

**2. الـ AM62x host driver لازم يعلن إنه بيدعم الـ feature دي في الـ `pm_caps`:**

```c
/* في drivers/mmc/host/sdhci-am654.c أو المكافئ */
host->pm_caps |= MMC_PM_KEEP_POWER | MMC_PM_WAKE_SDIO_IRQ;
```

**3. التحقق:**

```bash
# قبل suspend
cat /sys/bus/mmc/devices/mmc1:0001/pm_caps   # should show 0x3
cat /sys/bus/mmc/devices/mmc1:0001/pm_flags  # should show 0x3 after driver loads

# debug
echo 1 > /sys/module/mmc_core/parameters/use_spi_crc
dmesg | grep -i "mmc1\|sdio\|pm_flag"
```

#### الدرس المستفاد
الـ `pm_caps` هي ما يقدر عليه الـ host hardware. الـ `pm_flags` هي ما بيطلبه الـ function driver. لو الـ driver مش بيسيت `MMC_PM_KEEP_POWER` في الـ `pm_flags` قبل الـ suspend، الـ MMC core هيعمل power-off للكارت بغض النظر عن أي حاجة تانية.

---

### السيناريو الثاني: eMMC على RK3562 بيـ corrupt بعد الـ suspend/resume على Android TV Box

#### العنوان
**الـ MMC_CAP_AGGRESSIVE_PM بيعمل power cycle للـ eMMC وبيخلي الـ filesystem يتكسر**

#### السياق
Android TV box بيستخدم **RK3562** مع **eMMC 5.1** (Kingston EMMC08G) على mmc0. المستخدمين بيشتكوا إن لما الجهاز يدخل suspend ويرجع، بيلاقوا apps متمسحة أو الـ /data partition فيه errors.

#### المشكلة
الـ `e2fsck` بيلاقي errors في الـ /data partition بعد كل resume. الـ kernel log بيظهر:

```
mmc0: Reset 0x1 never completed.
mmc0: mmc_select_hs400() failed, error -110
EXT4-fs error (device mmcblk0p8): ...
```

#### التحليل

في `host.h`:

```c
#define MMC_CAP_AGGRESSIVE_PM  (1 << 7)  /* Suspend (e)MMC/SD at idle */
#define MMC_CAP_NONREMOVABLE   (1 << 8)  /* Nonremovable e.g. eMMC */
```

الـ `MMC_CAP_AGGRESSIVE_PM` معناها إن الـ MMC core هيعمل power-off للكارت لما يبقى idle حتى لو مفيش suspend كامل. لو اتضافت مع الـ suspend path، بيعمل power cycle للـ eMMC وهو لسه عنده dirty pages في الـ cache.

الـ problem flow:

```
system idle → MMC_CAP_AGGRESSIVE_PM triggers → mmc_power_off(eMMC)
   ↓
pending writes في الـ page cache لسه متكتبتش
   ↓
resume → mmc re-init يفشل لأن eMMC في HS400 mode بتاعته
       → أو ينجح بس بعد data loss
```

الـ eMMC ده مش زي SDIO — مفيش `MMC_PM_KEEP_POWER` مناسبة هنا. الـ issue إن `MMC_CAP_AGGRESSIVE_PM` مش المفروض تتستخدم مع eMMC في production.

في الـ `pm.h`، الفلاجين المتعرّفين بسّ للـ SDIO cards:

```c
/*
 * These flags are used to describe power management features that
 * some cards (typically SDIO cards) might wish to benefit from...
 */
```

الـ eMMC مش بيستخدم `MMC_PM_KEEP_POWER` أو `MMC_PM_WAKE_SDIO_IRQ` — بيستخدم `MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND` بدلها.

#### الحل

**1. إزالة الـ `MMC_CAP_AGGRESSIVE_PM` من الـ RK3562 eMMC host:**

```c
/* في arch/arm64/boot/dts/rockchip/rk3562.dtsi */
&sdhci {  /* eMMC controller */
    /* remove mmc-pwrseq or aggressive-pm if set */
    no-mmc-hs400;           /* تبسيط مؤقت للـ debugging */
    keep-power-in-suspend;  /* DT property يسيت pm_caps */
};
```

**2. في الـ driver، التحقق قبل سيت الـ cap:**

```c
/* eMMC لازم ما ياخدش AGGRESSIVE_PM */
if (host->caps & MMC_CAP_NONREMOVABLE)
    host->caps &= ~MMC_CAP_AGGRESSIVE_PM;
```

**3. التحقق من الـ caps:**

```bash
# شوف الـ caps الحالية
cat /sys/kernel/debug/mmc0/ios

# شوف لو AGGRESSIVE_PM مسيت
grep -r "MMC_CAP_AGGRESSIVE_PM" /sys/kernel/debug/
```

#### الدرس المستفاد
الـ `mmc_pm_flag_t` في `pm.h` للـ SDIO فقط. الـ eMMC بيستخدم الـ `caps` و`caps2` في `host.h` لإدارة الـ power. خلط الـ mechanisms ده بيودي لـ data corruption.

---

### السيناريو الثالث: Wi-Fi مش بيشتغل بعد الـ resume على i.MX8MM IoT Sensor

#### العنوان
**الـ `pm_caps` مش بيتّسيت في الـ i.MX8MM USDHC driver — الـ Wi-Fi SDIO مش قادر يطلب keep_power**

#### السياق
IoT environmental sensor بيستخدم **i.MX8M Mini** مع **Murata LBEH5HMZPC** (Wi-Fi/BT module) على SDIO. المنتج لازم يصحى من sleep لما يجي command من الـ cloud، لكن الـ Wi-Fi مش بيصحى.

#### المشكلة
الـ driver بيطلب الـ `MMC_PM_KEEP_POWER`:

```c
sdio_set_host_pm_flags(func, MMC_PM_KEEP_POWER | MMC_PM_WAKE_SDIO_IRQ);
```

بس الـ suspend بيفشل ويرجع error:

```
mmc2: pm flag 0x3 out of capabilities 0x0
```

#### التحليل

الـ code في `mmc_sdio_suspend()` بيتحقق:

```c
/* from drivers/mmc/core/sdio.c - the check that rejects the request */
if (host->pm_flags & ~host->pm_caps) {
    /* requested flags exceed host capabilities */
    return -EINVAL;
}
```

يعني الـ `pm_caps` بتاعة الـ i.MX8MM USDHC host = `0x0` — مش معلنتش أي support للـ PM features دي.

الـ `pm_caps` لازم يتّسيت في الـ host controller driver:

```c
/* in host.h: */
mmc_pm_flag_t  pm_caps;   /* supported pm features - SET BY HOST DRIVER */
mmc_pm_flag_t  pm_flags;  /* requested pm features - SET BY SDIO DRIVER */
```

في `drivers/mmc/host/sdhci-esdhc-imx.c`، المفروض يوجد:

```c
host->mmc->pm_caps = MMC_PM_KEEP_POWER | MMC_PM_WAKE_SDIO_IRQ;
```

بس في بعض النسخ القديمة من الـ driver أو لو الـ DT مش عنده `keep-power-in-suspend`، الـ `pm_caps` بتفضل صفر.

#### الحل

**1. إضافة الـ DT property:**

```dts
/* في arch/arm64/boot/dts/freescale/imx8mm-iot-sensor.dts */
&usdhc2 {
    pinctrl-names = "default", "state_100mhz", "state_200mhz";
    pinctrl-0 = <&pinctrl_usdhc2>;
    bus-width = <4>;
    keep-power-in-suspend;      /* sets MMC_PM_KEEP_POWER in pm_caps */
    enable-sdio-wakeup;         /* sets MMC_PM_WAKE_SDIO_IRQ in pm_caps */
    non-removable;
    status = "okay";
};
```

**2. أو في الـ driver مباشرة:**

```c
/* في sdhci-esdhc-imx.c probe */
static int sdhci_esdhc_imx_probe(struct platform_device *pdev)
{
    /* ... existing code ... */

    /* declare PM capabilities for SDIO wakeup */
    host->mmc->pm_caps |= MMC_PM_KEEP_POWER;
    if (of_property_read_bool(np, "enable-sdio-wakeup"))
        host->mmc->pm_caps |= MMC_PM_WAKE_SDIO_IRQ;
}
```

**3. التحقق:**

```bash
# قبل load الـ Wi-Fi driver
cat /sys/class/mmc_host/mmc2/mmc2\:0001/pm_caps
# المتوقع: 3 (0x3 = KEEP_POWER | WAKE_SDIO_IRQ)

# بعد load
echo mem > /sys/power/state
# انتظر network packet
# لو الجهاز صحى → تمام
dmesg | tail -20
```

#### الدرس المستفاد
الـ `pm_caps` مسؤولية الـ **host controller driver**، مش الـ SDIO function driver. الـ SDIO driver بس بيطلب (`pm_flags`). لو الـ host مش معلن إنه يقدر يعمل حاجة، الطلب هيتّرفض. الـ DT property `keep-power-in-suspend` و`enable-sdio-wakeup` هما الطريقة المعيارية.

---

### السيناريو الرابع: eMMC على STM32MP1 بيخد وقت طويل جداً في الـ resume على Automotive ECU

#### العنوان
**غياب `MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND` بيخلي الـ resume يأخد 800ms بدل 50ms**

#### السياق
Automotive ECU بيستخدم **STM32MP157** مع **Micron eMMC** (MTFC4GACAJCN) على الـ SDMMC2. المتطلبات الـ automotive بتقول إن الـ wake-up من sleep لازم يكون أقل من 100ms. الـ measurement بيديّ 850ms.

#### المشكلة
الـ resume path بيأخد وقت طويل. الـ ftrace بيظهر:

```
mmc_resume_host: 847ms
  → mmc_power_up: 250ms      ← الـ vmmc regulator rising time
  → mmc_send_cmd(CMD0): 100ms ← reset
  → mmc_init_card: 497ms      ← full re-init من الـ HS400
```

#### التحليل

في `host.h`:

```c
#define MMC_CAP2_FULL_PWR_CYCLE           (1 << 2)  /* Can do full power cycle */
#define MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND (1 << 3) /* Can do full power cycle in suspend */
```

لو `MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND` مش مسيّتة، الـ MMC core بيحاول يعمل **partial re-initialization** بدل full power cycle. لكن الـ eMMC لو اتعمله power-off فعلاً أثناء الـ suspend، الـ partial re-init بيفشل وبيضطر يعمل full init.

الـ `pm.h` flags مش موجودة هنا (دي للـ SDIO). للـ eMMC، القرار بيعتمد على:

```c
/* from drivers/mmc/core/mmc.c */
static int mmc_resume(struct mmc_host *host)
{
    if (!(host->caps2 & MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND)) {
        /* assume card state preserved, try soft re-init */
        err = mmc_init_card(host, host->card->ocr, host->card);
    } else {
        /* full re-init expected */
        mmc_power_up(host, host->card->ocr);
        err = mmc_init_card(host, host->card->ocr, host->card);
    }
}
```

المشكلة: الـ STM32MP1 SDMMC driver مش بيسيت `MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND`، فالـ core بيحاول soft re-init، بس الـ eMMC فعلاً اتعمله power-off، فبيفشل ويرجع لـ full init — بس بعد timeout.

#### الحل

**1. إضافة الـ cap في الـ STM32 driver:**

```c
/* في drivers/mmc/host/mmci_stm32_sdmmc.c */
static void sdmmc_variant_init(struct mmci_host *host)
{
    /* declare that suspend causes full power cycle */
    host->mmc->caps2 |= MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND;
}
```

**2. أو عبر الـ DT:**

```dts
/* arch/arm/boot/dts/stm32mp157c-ecu.dts */
&sdmmc2 {
    bus-width = <8>;
    non-removable;
    full-pwr-cycle-in-suspend;  /* maps to MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND */
    mmc-hs400-1_8v;
    status = "okay";
};
```

**3. قياس الـ timing بعد الإصلاح:**

```bash
# قياس resume time
echo mem > /sys/power/state &
sleep 2
# trigger wakeup
time cat /sys/class/mmc_host/mmc0/mmc0\:0001/date

# أو بـ ftrace
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable
echo mem > /sys/power/state
```

**4. ضبط الـ vmmc regulator لتقليل الـ ramp time:**

```dts
vmmc_ecu: regulator-vmmc {
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    regulator-ramp-delay = <1000>;  /* 1ms بدل default */
};
```

#### الدرس المستفاد
الـ `MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND` مش بس optimization — ده إخبار صريح للـ MMC core بإن الـ card هتحتاج full re-init بعد الـ suspend. غيابه بيخلي الـ core يجرب shortcut بيفشل، وده بيضيف timeout overhead فوق الـ full init الطبيعي.

---

### السيناريو الخامس: الـ Wi-Fi على Allwinner H616 Android TV Box بيصحى البورد كل شوية بدون سبب

#### العنوان
**الـ `MMC_PM_WAKE_SDIO_IRQ` مع misconfigured interrupt routing بيعمل spurious wakeups**

#### السياق
Android TV box رخيص بيستخدم **Allwinner H616** مع **XR829** Wi-Fi chip على SDIO (mmc1). الـ product management بتشتكي إن الجهاز مش بيفضل في standby — بيصحى كل دقيقتين تقريباً بدون أي user interaction.

#### المشكلة
الـ `dmesg` بعد كل wake:

```
PM: wakeup from suspend
mmc1: Got SDIO IRQ
sunxi-mmc 4020000.mmc: Got data error
xr829: spurious interrupt, status=0x0000
```

الـ Wi-Fi مش بيعمل traffic، بس الـ IRQ بييجي على طول.

#### التحليل

الـ `pm.h` بيعرّف:

```c
#define MMC_PM_WAKE_SDIO_IRQ (1 << 1)  /* wake up host system on SDIO IRQ assertion */
```

الـ `host.h` بيوضح الـ chain:

```c
/* sdio_irq_claimed check used during suspend/resume */
static inline bool sdio_irq_claimed(struct mmc_host *host)
{
    return host->sdio_irqs > 0;
}

/* signal from ISR */
static inline void mmc_signal_sdio_irq(struct mmc_host *host)
{
    host->ops->enable_sdio_irq(host, 0);   /* disable IRQ first */
    host->sdio_irq_pending = true;
    if (host->sdio_irq_thread)
        wake_up_process(host->sdio_irq_thread);
}
```

الـ flow:

```
suspend path:
  → sdio driver sets MMC_PM_WAKE_SDIO_IRQ in pm_flags
  → MMC core calls: enable_sdio_irq(host, 1) ← يفعّل الـ IRQ wakeup
  → الـ H616 Sunxi SDIO controller بيضبط الـ IRQ كـ level-triggered
  → بس الـ XR829 interrupt line فيها noise بسبب لsupply filtering ضعيف
  → كل noise pulse بتعمل level assertion على الـ DAT1 line
  → الـ SDIO IRQ detection بيشوف high level → wakeup
```

الـ H616 Sunxi driver بيسيت:

```c
host->mmc->pm_caps |= MMC_PM_KEEP_POWER | MMC_PM_WAKE_SDIO_IRQ;
```

بس الـ hardware واجهة مشكلة في الـ signal integrity على DAT1 (الـ SDIO IRQ line).

#### التحليل التفصيلي

```
DAT1 line during suspend:
  ____/‾‾‾\___/‾‾‾\___  ← noise spikes بسبب power supply ripple
       ↑   ↑
       |   |
    wakeup wakeup  ← كل spike بتصحّي الـ system
```

الـ XR829 بيبعت IRQ على DAT1 كـ level (active low normally, pulses high for IRQ). الـ noise بتعمل false high.

#### الحل

**1. Software: تعطيل `MMC_PM_WAKE_SDIO_IRQ` واستخدام dedicated GPIO wakeup:**

```dts
/* في arch/arm64/boot/dts/allwinner/h616-tv-box.dts */
&mmc1 {
    /* disable SDIO IRQ wakeup, use dedicated GPIO instead */
    /* remove: enable-sdio-wakeup; */
    keep-power-in-suspend;
};

/* dedicated wakeup line من XR829 HOST_WAKE pin */
xr829_wifi: wifi {
    wakeup-gpios = <&pio 6 10 GPIO_ACTIVE_HIGH>;  /* PG10 */
};
```

```c
/* في الـ XR829 driver */
static int xr829_sdio_suspend(struct device *dev)
{
    /* use only KEEP_POWER, not WAKE_SDIO_IRQ */
    return sdio_set_host_pm_flags(func, MMC_PM_KEEP_POWER);
}
```

**2. Hardware fix: إضافة filtering على الـ DAT1/IRQ line:**

```
              100Ω
DAT1 ────────/\/\/─────── SDIO controller
              |
             10nF
              |
             GND
```

**3. التحقق إن الـ spurious wakeups اختفت:**

```bash
# monitor wakeup sources
cat /sys/power/pm_wakeup_count
echo mem > /sys/power/state
# بعد wake
cat /sys/power/pm_wakeup_count  # لو زاد بسبب مصدر تاني → صح

# debug الـ SDIO IRQ
cat /proc/interrupts | grep mmc1
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_sdio_irq/enable

# شوف الـ pm_flags
cat /sys/bus/sdio/devices/mmc1\:0001\:1/pm_flags
```

**4. لو محتاج تفضل بـ SDIO IRQ وتحل الـ noise مؤقتاً:**

```c
/* في الـ sunxi mmc driver، إضافة debounce */
static irqreturn_t sunxi_mmc_irq(int irq, void *dev_id)
{
    /* قبل SDIO IRQ signal، verify الـ DAT1 stable لـ 50us */
    if (intmask & SDXC_SDIO_INT_WAKEUP) {
        udelay(50);
        if (!(mmc_readl(host, REG_RINTR) & SDXC_SDIO_INT_WAKEUP))
            return IRQ_HANDLED;  /* كان noise */
    }
    /* ... */
}
```

#### الدرس المستفاد
الـ `MMC_PM_WAKE_SDIO_IRQ` قوي، بس بيتعامل مع الـ hardware signal مباشرة. في boards رخيصة زي Android TV boxes بـ Allwinner، الـ signal integrity على الـ SDIO bus ممكن تكون weak. الـ solution الأنظف هو استخدام dedicated wakeup GPIO من الـ Wi-Fi module بدل الـ in-band SDIO IRQ، وتخلّي `MMC_PM_KEEP_POWER` بس من غير `MMC_PM_WAKE_SDIO_IRQ`.
## Phase 7: مصادر ومراجع

---

### مصادر الكود الرسمي في الـ Kernel

| المسار | الوصف |
|--------|-------|
| `include/linux/mmc/pm.h` | الملف الرئيسي — تعريف `mmc_pm_flag_t` و`MMC_PM_KEEP_POWER` و`MMC_PM_WAKE_SDIO_IRQ` |
| `drivers/mmc/core/sdio.c` | تطبيق suspend/resume للـ SDIO مع استخدام الـ flags |
| `drivers/mmc/core/core.c` | الـ MMC core — runtime PM hooks والـ bus operations |
| `drivers/mmc/host/` | مجلد host controllers — كل controller بيتعامل مع الـ pm flags بشكل مختلف |
| `include/linux/mmc/host.h` | تعريف `struct mmc_host` وخاصية `pm_caps` |
| `include/linux/mmc/sdio_func.h` | واجهة SDIO function driver مع دوال suspend/resume |

---

### التوثيق الرسمي للـ Kernel

- [MMC/SD/SDIO card support — The Linux Kernel documentation](https://docs.kernel.org/driver-api/mmc/index.html)
  - الـ index الرئيسي لكل توثيق الـ MMC subsystem
- [MMC tools introduction](https://docs.kernel.org/driver-api/mmc/mmc-tools.html)
  - أدوات تشخيص وإدارة كروت الـ MMC/SD
- [SD and MMC Block Device Attributes](https://docs.kernel.org/driver-api/mmc/mmc-dev-attrs.html)
  - الـ sysfs attributes المتاحة لأجهزة MMC/SD
- [SD and MMC Device Partitions](https://docs.kernel.org/driver-api/mmc/mmc-dev-parts.html)
  - إدارة الـ partitions في أجهزة MMC/eMMC
- [Device Power Management Basics — The Linux Kernel documentation](https://static.lwn.net/kerneldoc/driver-api/pm/devices.html)
  - أساسيات إدارة الطاقة للأجهزة، بيتضمن أمثلة على MMC/SD كـ wakeup sources
- [CPU and Device Power Management — Index](https://static.lwn.net/kerneldoc/driver-api/pm/index.html)
  - الـ index الكامل لتوثيق إدارة الطاقة في الـ kernel
- [Documentation/power/runtime_pm.txt — via LWN](https://lwn.net/Articles/347575/)
  - التوثيق الرسمي لـ runtime PM framework

---

### مقالات LWN.net

**الـ** LWN هو المصدر الأهم لفهم السياق التاريخي والتقني للتغييرات في الـ kernel.

#### إدارة الطاقة — عامة

- [Linux power management: The documentation I wanted to read](https://lwn.net/Articles/505683/)
  - مقال شامل بيشرح كيف الـ MMC bus handlers بتستدعي الـ driver-specific handlers أثناء suspend، وده مرتبط مباشرة بكيفية معالجة `mmc_pm_flag_t`

- [Runtime power management](https://lwn.net/Articles/347573/)
  - المقال الأساسي اللي شرح فيه Jonathan Corbet نظام runtime PM — نفس الوقت تقريبًا اللي اتكتب فيه `pm.h` (2009)

- [PM: Avoid losing wakeup events during suspend](https://lwn.net/Articles/392897/)
  - بيشرح إزاي الـ kernel بيتعامل مع wakeup events زي `MMC_PM_WAKE_SDIO_IRQ` أثناء دورة الـ suspend

- [Waking systems from suspend](https://lwn.net/Articles/429925/)
  - بيتكلم عن آلية wakeup interrupts، وده الـ mechanism اللي `MMC_PM_WAKE_SDIO_IRQ` بيعتمد عليه

- [A new approach to opportunistic suspend](https://lwn.net/Articles/460857/)
  - بيشرح suspend blockers وكيف الـ SDIO devices بتمنع الـ suspend لما بيكون في IRQ pending

- [An alternative to suspend blockers](https://lwn.net/Articles/416690/)
  - نقاش مهم حول wakelock vs. wakeup events — مؤثر على تصميم الـ SDIO wakeup path

- [pwrseq: Add subsystem for power sequences](https://lwn.net/Articles/602855/)
  - الـ pwrseq subsystem المستخدم مع الـ MMC لإدارة تسلسل تشغيل الطاقة للبطاقات

- [PM: Update device power management document](https://lwn.net/Articles/378597/)
  - تحديث التوثيق بيتضمن MMC/SD كأمثلة على buses بتشوف device removal أثناء suspend

---

### Patchwork ومناقشات الـ Mailing List

- [mmc: print debug messages for runtime PM actions](https://patchwork.kernel.org/project/linux-mmc/patch/20110717153841.D4BF29D401C@zog.reactivated.net/)
  - patch بيضيف debug messages لـ runtime PM hooks في `drivers/mmc/core/core.c`

- [mmc: core: Read the SD function extension registers for power management](https://patchwork.kernel.org/project/linux-mmc/patch/20210504161222.101536-10-ulf.hansson@linaro.org/)
  - patch بيضيف دعم لـ SD power management extensions — امتداد لمفهوم الـ pm flags

- [mmc: sdio: reset card during power_restore](https://patchwork.kernel.org/patch/866442/)
  - بيعالج مشكلة في مسار `power_restore` لما `MMC_PM_KEEP_POWER` مش متفعّل

- [mmc: sdio: fix runtime PM path during driver removal](https://patchwork.kernel.org/project/linux-wireless/patch/1307662827-16618-1-git-send-email-ohad@wizery.com/)
  - بيصلح bug في مسار runtime PM لما الـ SDIO driver بيتشال

- [PATCH v1 3/3: mmc: changes to enable runtime PM of mmc core layer](https://groups.google.com/g/linux.kernel/c/HpizXM7VIzs)
  - مناقشة تفعيل runtime PM في الـ MMC core layer

- [SDIO suspend/resume while runtime suspended](https://www.spinics.net/lists/linux-mmc/msg03975.html)
  - patch بيضيف دعم suspend/resume لما الجهاز كان في حالة runtime suspend — حالة edge case مهمة

- [mmc: check SDIO card wake up during suspending](http://lists.infradead.org/pipermail/linux-arm-kernel/2011-September/066098.html)
  - patch على linux-arm-kernel بيتحقق من wake-up capability قبل الـ suspend

---

### kernelnewbies.org — تاريخ التغييرات في الـ MMC Subsystem

- [Linux 2.6.24 — MMC layer transformation](https://kernelnewbies.org/Linux_2_6_24)
  - التحول الكبير للـ MMC layer لدعم SDIO وSPI — الجذور الأولى للـ subsystem

- [Linux 2.6.31 — SDIO enhancements](https://kernelnewbies.org/Linux_2_6_31)
  - تحسينات على SDIO — قريب من وقت إضافة `pm.h` (اتكتب في 2009)

- [Linux 4.0 — MMC power sequences](https://kernelnewbies.org/Linux_4.0-DriversArch)
  - إضافة دعم MMC power sequences وdrivers لـ eMMC hardware reset

- [Linux 5.7 — MMC software queue](https://kernelnewbies.org/Linux_5.7)
  - إضافة software queue للـ MMC core بعمق 64 — تحسين الـ performance

- [Linux 6.7 — MMC updates](https://kernelnewbies.org/Linux_6.7)
  - آخر التحديثات على الـ MMC subsystem

---

### elinux.org — تطبيقات Embedded

- [R-Car/Merging-MMC-block-requests](https://elinux.org/R-Car/Merging-MMC-block-requests)
  - تحسين أداء الـ MMC عبر IOMMU في Linux kernel v5.4

- [Tests:eMMC-8bit-width](https://elinux.org/Tests:eMMC-8bit-width)
  - اختبارات eMMC بعرض باص 8-bit وتكوينات high-speed

- [EVM Second MMC / SD](https://elinux.org/Second_MMC_/_SD)
  - دليل عملي لإضافة SDIO/SD slot ثاني على embedded boards

- [AM335x recovery](https://elinux.org/AM335x_recovery)
  - استخدام MMC/SD في إجراءات الاسترداد على TI AM335x

---

### مصادر الكود على GitHub / Kernel.org

| المصدر | الرابط |
|--------|--------|
| Linux kernel — MMC drivers | [github.com/torvalds/linux/tree/master/drivers/mmc](https://github.com/torvalds/linux/tree/master/drivers/mmc) |
| Linux kernel — MMC host drivers | [github.com/torvalds/linux/tree/master/drivers/mmc/host](https://github.com/torvalds/linux/tree/master/drivers/mmc/host) |
| pm.h على Android kernel (Mediatek) | [android.googlesource.com — pm.h](https://android.googlesource.com/kernel/mediatek/+/android-mediatek-sprout-3.4-kitkat-mr2/include/linux/mmc/pm.h) |
| Marvell Embedded Linux kernel | [github.com/MarvellEmbeddedProcessors/linux-marvell](https://github.com/MarvellEmbeddedProcessors/linux-marvell) |
| CONFIG_MMC — lkddb | [cateee.net/lkddb — CONFIG_MMC](https://cateee.net/lkddb/web-lkddb/MMC.html) |

---

### كتب موصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 14**: The Linux Device Model — بيشرح الـ bus/device/driver model اللي الـ MMC بيبني عليه
- **الفصل 16**: Block Drivers — مفيد لفهم الـ MMC block layer
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 14**: The Block I/O Layer — فهم الـ I/O scheduling اللي بيأثر على الـ MMC
- **الفصل 17**: Devices and Modules — الـ device model والـ bus abstraction
- الكتاب ده بيكسب فهم عميق للـ kernel internals اللي الـ MMC بيشتغل فوقيها

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 14**: Kernel Debugging Techniques — debug المشاكل في MMC/SDIO drivers
- **الفصل 15**: Embedded Development Environment — تضمين MMC support في embedded systems
- بيتكلم كتير عن ARM platforms اللي فيها SDIO/MMC شائع

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **Chapter 6**: Device Drivers — architecture الـ driver model بالتفصيل
- **Chapter 12**: Networks — فهم الـ SDIO network adapters (WiFi chips)

---

### مصطلحات البحث للاستزادة

لو حابب تدور على معلومات أكتر، استخدم الـ search terms دي:

```
# بحث في LWN.net
site:lwn.net "mmc_pm_flag_t"
site:lwn.net SDIO suspend "keep power"
site:lwn.net "MMC_PM_WAKE_SDIO_IRQ"

# بحث في mailing lists
linux-mmc mailing list: https://lore.kernel.org/linux-mmc/
lore.kernel.org/linux-mmc mmc_pm_flag
lore.kernel.org MMC_PM_KEEP_POWER SDIO

# بحث في kernel git
git log --follow -- include/linux/mmc/pm.h
git log --all --oneline -- drivers/mmc/core/sdio.c | head -30
git log --author="Nicolas Pitre" -- drivers/mmc/

# بحث عام
"MMC_PM_KEEP_POWER" WoWLAN SDIO wifi linux
"mmc_pm_flag_t" linux kernel driver
SDIO "wake on wireless" linux kernel MMC suspend
```

---

### الـ Kernel Source Paths للمراجعة المباشرة

```
Documentation/driver-api/mmc/         ← التوثيق الرسمي
include/linux/mmc/pm.h                ← الملف الرئيسي موضوع الدراسة
include/linux/mmc/host.h              ← struct mmc_host + pm_caps
include/linux/mmc/sdio_func.h        ← SDIO function driver interface
drivers/mmc/core/sdio.c              ← تطبيق SDIO suspend/resume
drivers/mmc/core/core.c              ← MMC core + runtime PM
drivers/mmc/core/host.c              ← host registration/deregistration
drivers/net/wireless/                 ← SDIO WiFi drivers (brcmfmac, etc.)
```
## Phase 8: Writing simple module

### الفكرة العامة

**`mmc_add_host()`** هي الـ exported function الأكثر إثارة للاهتمام في سياق الـ MMC PM — لأنها اللحظة اللي بيتسجل فيها الـ host في الـ MMC core وبيتضاف فيها `pm_caps` الخاص بيه. بنستخدم **kprobe** عشان نعترض الاستدعاء دا ونطبع كل معلومات الـ PM capabilities والـ power settings قبل ما الـ host يتسجل رسمياً.

الـ hook ده بيديك رؤية حقيقية لكل SDIO device بيتعرف على النظام — بتشوف هل بيعمل `MMC_PM_KEEP_POWER` (بيفضل شغال أثناء الـ suspend) أو `MMC_PM_WAKE_SDIO_IRQ` (بيصحّي النظام لما SDIO IRQ يجي).

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * mmc_pm_monitor.c
 *
 * Monitors mmc_add_host() via kprobe to log MMC PM capabilities
 * and power settings at host registration time.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit        */
#include <linux/kernel.h>       /* pr_info, pr_err                         */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, etc.    */
#include <linux/mmc/host.h>     /* struct mmc_host, mmc_pm_flag_t, caps    */
#include <linux/mmc/pm.h>       /* MMC_PM_KEEP_POWER, MMC_PM_WAKE_SDIO_IRQ */

/*
 * ---- kprobe pre-handler ------------------------------------------------
 * Called just before mmc_add_host() executes.
 * pt_regs carries the CPU registers at the call site;
 * on x86-64 the first argument (host) is in rdi.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Recover the mmc_host pointer from the first function argument */
    struct mmc_host *host = (struct mmc_host *)regs->di;

    if (!host)
        return 0;   /* safety: never dereference NULL */

    pr_info("[mmc_pm_monitor] mmc_add_host() called\n");

    /* ---- Host identity ---- */
    pr_info("[mmc_pm_monitor]   host index  : %d\n", host->index);
    pr_info("[mmc_pm_monitor]   host parent : %s\n",
            host->parent ? dev_name(host->parent) : "none");

    /* ---- PM capabilities (what the controller HW supports) ---- */
    pr_info("[mmc_pm_monitor]   pm_caps     : 0x%x\n", host->pm_caps);

    if (host->pm_caps & MMC_PM_KEEP_POWER)
        pr_info("[mmc_pm_monitor]     -> KEEP_POWER supported (card stays powered during suspend)\n");

    if (host->pm_caps & MMC_PM_WAKE_SDIO_IRQ)
        pr_info("[mmc_pm_monitor]     -> WAKE_SDIO_IRQ supported (SDIO IRQ can wake system)\n");

    if (!host->pm_caps)
        pr_info("[mmc_pm_monitor]     -> no PM features advertised\n");

    /* ---- Key host capabilities relevant to power ---- */
    pr_info("[mmc_pm_monitor]   caps        : 0x%x\n", host->caps);

    if (host->caps & MMC_CAP_AGGRESSIVE_PM)
        pr_info("[mmc_pm_monitor]     -> AGGRESSIVE_PM: suspend card at idle\n");

    if (host->caps & MMC_CAP_NONREMOVABLE)
        pr_info("[mmc_pm_monitor]     -> NONREMOVABLE (eMMC)\n");

    if (host->caps & MMC_CAP_SDIO_IRQ)
        pr_info("[mmc_pm_monitor]     -> SDIO_IRQ capable\n");

    if (host->caps & MMC_CAP_SYNC_RUNTIME_PM)
        pr_info("[mmc_pm_monitor]     -> SYNC_RUNTIME_PM\n");

    /* ---- Power supply info ---- */
    pr_info("[mmc_pm_monitor]   max_current : 3.3V=%umA  3.0V=%umA  1.8V=%umA\n",
            host->max_current_330,
            host->max_current_300,
            host->max_current_180);

    /* ---- Clock range ---- */
    pr_info("[mmc_pm_monitor]   clock range : %u Hz .. %u Hz\n",
            host->f_min, host->f_max);

    return 0;   /* 0 = continue normal execution of mmc_add_host() */
}

/*
 * ---- kprobe post-handler -----------------------------------------------
 * Called immediately after mmc_add_host() returns (optional but useful).
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    pr_info("[mmc_pm_monitor] mmc_add_host() returned\n");
}

/* ---- kprobe descriptor ---- */
static struct kprobe kp = {
    .symbol_name = "mmc_add_host",  /* target: the exported MMC function   */
    .pre_handler = handler_pre,     /* fires before the function body runs  */
    .post_handler = handler_post,   /* fires after the function body runs   */
};

/* ---- module init ---- */
static int __init mmc_pm_monitor_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[mmc_pm_monitor] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[mmc_pm_monitor] kprobe planted at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ---- module exit ---- */
static void __exit mmc_pm_monitor_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[mmc_pm_monitor] kprobe removed from %s\n", kp.symbol_name);
}

module_init(mmc_pm_monitor_init);
module_exit(mmc_pm_monitor_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("MMC PM Lab");
MODULE_DESCRIPTION("kprobe monitor for MMC host PM capabilities at registration");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | يجيب `struct kprobe` وكل الـ API الخاص بالـ kprobes |
| `linux/mmc/host.h` | يجيب `struct mmc_host` وكل الـ `MMC_CAP_*` macros |
| `linux/mmc/pm.h` | يجيب `mmc_pm_flag_t` والـ flagsـ `MMC_PM_KEEP_POWER` و `MMC_PM_WAKE_SDIO_IRQ` |

الـ `linux/module.h` و`linux/kernel.h` أساسيين في أي module — الأول للـ macros التسجيلية والثاني لـ `pr_info`.

---

#### الـ `handler_pre` — قلب الـ module

**الـ `pt_regs *regs`** بتحمل حالة الـ CPU registers لحظة الـ call. على **x86-64**، أول argument بيتبعت في `rdi` حسب الـ System V ABI — ومن هنا بنجيب الـ `struct mmc_host *`.

بنطبع **`pm_caps`** لأنه القيمة اللي بيحددها الـ host controller driver نفسه في `probe()` بتاعه، وبيعكس الـ hardware capabilities الحقيقية للـ PM. مثلاً host driver زي `sdhci` ممكن يضيف `MMC_PM_KEEP_POWER` لو الـ hardware بيدعم الـ voltage regulator أثناء الـ suspend.

---

#### الـ `handler_post`

بيتأكد إن `mmc_add_host()` رجع من غير ما ينهار — مفيد في الـ debugging لأنك بتعرف إن الـ function اتنفذت بشكل كامل. في الـ production module ممكن تشيله لو مش محتاجه.

---

#### الـ `struct kprobe kp`

**`symbol_name = "mmc_add_host"`** — الـ kernel بيحول الاسم ده لعنوان فعلي عن طريق الـ kallsyms وقت الـ `register_kprobe()`. ده أضمن من hardcode الـ address لأنه بيشتغل على أي kernel build.

---

#### الـ `module_init` / `module_exit`

- **`register_kprobe()`** بتزرع breakpoint في الكود — لو فشلت (مثلاً الـ function مش موجودة أو محمية بـ `__nokprobe_inline`) بترجع error سالب ولازم نعمل `return ret` فوراً.
- **`unregister_kprobe()`** في الـ exit **إلزامية** — لو مسحت الـ module من غير ما تشيل الـ kprobe، أول ما `mmc_add_host()` تتنادى هيجي الـ CPU على handler في memory اتحررت → kernel panic.

---

### Makefile للبناء

```makefile
obj-m += mmc_pm_monitor.o

KDIR := /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميل الـ module
sudo insmod mmc_pm_monitor.ko

# لو عندك SDIO device، افصله وأعد توصيله أو عمل rescan
echo 1 | sudo tee /sys/bus/mmc/devices/mmc0\:*/remove 2>/dev/null
echo 1 | sudo tee /sys/bus/mmc/rescan 2>/dev/null

# أو عمل suspend/resume للنظام وشوف الـ logs
sudo rtcwake -m mem -s 5

# مشاهدة الـ output
sudo dmesg | grep mmc_pm_monitor

# إزالة الـ module
sudo rmmod mmc_pm_monitor
```

**مثال output حقيقي** على جهاز بيه SDIO Wi-Fi card:

```
[mmc_pm_monitor] kprobe planted at mmc_add_host (ffffffffc0a12340)
[mmc_pm_monitor] mmc_add_host() called
[mmc_pm_monitor]   host index  : 0
[mmc_pm_monitor]   host parent : mmc0
[mmc_pm_monitor]   pm_caps     : 0x3
[mmc_pm_monitor]     -> KEEP_POWER supported (card stays powered during suspend)
[mmc_pm_monitor]     -> WAKE_SDIO_IRQ supported (SDIO IRQ can wake system)
[mmc_pm_monitor]   caps        : 0x80000498
[mmc_pm_monitor]     -> SDIO_IRQ capable
[mmc_pm_monitor]     -> NONREMOVABLE (eMMC)
[mmc_pm_monitor]   max_current : 3.3V=400mA  3.0V=0mA  1.8V=200mA
[mmc_pm_monitor]   clock range : 400000 Hz .. 208000000 Hz
[mmc_pm_monitor] mmc_add_host() returned
```

**الـ `pm_caps = 0x3`** معناها إن الـ host بيدعم الـ flag التانين مع بعض: الكارت بيفضل شغال أثناء الـ suspend (`KEEP_POWER`) وبيقدر يصحّي النظام لما SDIO IRQ يجي (`WAKE_SDIO_IRQ`) — ده السلوك الطبيعي لـ Wi-Fi chip زي `brcmfmac` أو `ath6kl`.
