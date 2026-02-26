## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ SDIO؟

**SDIO** اختصار لـ Secure Digital Input/Output — هو امتداد لمعيار SD الشهير (بطاقات التخزين)، بس بدل ما بيخزّن بيانات بس، بيسمح بتوصيل **أجهزة كاملة** زي WiFi، Bluetooth، كاميرات، GPS، عبر نفس فتحة SD الموجودة في الجهاز.

**الـ MMC/SD/SDIO subsystem** في Linux هو المسؤول عن كل ده.

---

### القصة من الأول

تخيّل معي إنك بتصمم لاب توب صغير أو embedded board. عندك فتحة SD واحدة بس. بدل ما تحطّ فيها كارت ذاكرة، تحطّ **كارت WiFi SDIO** — الكارت ده بيتكلم مع الكيرنل عن طريق نفس بروتوكول SD العادي، لكن بدل بيانات، بيبعت وبيستقبل **أوامر I/O** وـ**interrupts**.

المشكلة: الـ kernel محتاج طريقة عشان:
1. يعرف الكارت ده "مين" — vendor، device ID، class.
2. يربطه بـ driver مناسب.
3. الـ driver يقدر يقرأ ويكتب في registers الكارت.
4. الكارت يقدر يبعت interrupt لما يحتاج attention.

هنا بيجي دور الملف ده.

---

### دور الملف `sdio_func.h`

الملف ده هو **العقد المشترك** بين:
- الـ **SDIO core** (الكيرنل اللي بيدير الكروت).
- الـ **SDIO function driver** (الكود اللي بيشتغل مع WiFi أو Bluetooth أو غيره).

بيعرّف:

| العنصر | الوصف |
|---|---|
| `struct sdio_func` | يمثّل **function واحدة** داخل كارت SDIO (كارت واحد ممكن يكون فيه أكتر من function) |
| `struct sdio_driver` | الـ driver اللي بيتعامل مع function معينة |
| `struct sdio_func_tuple` | بيانات CIS — معلومات إضافية عن الـ function محفوظة في الكارت نفسه |
| `sdio_irq_handler_t` | نوع الـ callback لما الكارت يبعت interrupt |
| I/O functions | `sdio_readb`, `sdio_writeb`, `sdio_memcpy_fromio`... إلخ |
| Power Management | `sdio_get_host_pm_caps`, `sdio_set_host_pm_flags` |

---

### كارت SDIO جوّاه إيه؟

كارت SDIO الواحد بيتقسّم لـ **functions** — زي إنك تفتح USB device وتلاقي فيه أكتر من interface.

```
[ SDIO Card ]
   ├── Function 0  ← دائماً موجودة، بتتحكم في الكارت كله (CIA)
   ├── Function 1  ← مثلاً: WiFi
   └── Function 2  ← مثلاً: Bluetooth
```

كل function بـ `struct sdio_func` منفصل، وممكن يكون لها driver منفصل.

---

### إزاي الـ driver بيشتغل؟ (القصة الكاملة)

```
[ Hardware ]
     │
     ▼
[ MMC Host Controller Driver ]   ← مثلاً: sdhci.c, dw_mmc.c
     │  بيتعامل مع الـ hardware مباشرة
     ▼
[ MMC Core ]  ← core.c, bus.c
     │  بيعرف الكارت ويحدد نوعه (SD / MMC / SDIO)
     ▼
[ SDIO Core ] ← sdio.c, sdio_bus.c, sdio_io.c, sdio_irq.c
     │  بيعمل enumerate للـ functions وبيسجلهم في الـ bus
     ▼
[ SDIO Function Driver ]  ← مثلاً: drivers/net/wireless/brcm80211/
     │  بيستخدم sdio_func.h API عشان يتكلم مع الهاردوير
     ▼
[ Network Stack / Bluetooth Stack / etc. ]
```

---

### الـ `struct sdio_func` عن قرب

```c
struct sdio_func {
    struct mmc_card     *card;         /* الكارت الأب اللي الـ function دي جوّاه */
    struct device        dev;          /* Linux device model — للـ sysfs والـ driver binding */
    sdio_irq_handler_t  *irq_handler;  /* callback لما الكارت يبعت IRQ */
    unsigned int         num;          /* رقم الـ function: 1..7 */

    unsigned char        class;        /* نوع الـ function (WiFi=0x07, BT=0x02, ...) */
    unsigned short       vendor;       /* مثلاً Broadcom = 0x02D0 */
    unsigned short       device;       /* model ID */

    unsigned             max_blksize;  /* أكبر block ممكن في transfer واحد */
    unsigned             cur_blksize;  /* الـ block size الحالي */

    u8                  *tmpbuf;       /* buffer مخصص للـ DMA */
    struct sdio_func_tuple *tuples;    /* linked list من CIS tuples */
};
```

---

### ليه مهم؟

- **WiFi chips** زي Broadcom BCM43xx وRealtek rtw88 SDIO كلها بتعتمد على هذا الـ API.
- **Bluetooth SDIO** modules كمان.
- الـ Raspberry Pi مثلاً بيستخدم BCM43438 WiFi/BT SDIO chip.

---

### الماكروهات المهمة

| الماكرو | الوظيفة |
|---|---|
| `SDIO_DEVICE(vend, dev)` | بيعرّف device ID للـ driver table |
| `SDIO_DEVICE_CLASS(class)` | بيتطابق مع أي device من class معين |
| `sdio_register_driver(drv)` | بيسجّل الـ driver في الـ SDIO bus |
| `module_sdio_driver(drv)` | boilerplate مختصر لـ module_init/exit |
| `sdio_get_drvdata(f)` | بيجيب الـ private data الخاصة بالـ driver |
| `dev_to_sdio_func(d)` | يحوّل `struct device*` لـ `struct sdio_func*` |

---

### الـ Power Management في SDIO

الهيدر بيضم دالتين للـ PM:
- **`sdio_get_host_pm_caps`** — بترجع الـ flags اللي الـ host controller بيدعمها (مثلاً: `MMC_PM_KEEP_POWER`).
- **`sdio_set_host_pm_flags`** — الـ driver بيطلب من الـ core يحافظ على power أو يصحّي من الـ IRQ لما الجهاز بيدخل suspend.

ده مهم جداً للـ WiFi — عشان الـ chip تفضل صاحية وتصحّي الجهاز لما تيجي packet.

---

### الملفات المرتبطة

#### Core (اللب الأساسي)
| الملف | الدور |
|---|---|
| `drivers/mmc/core/sdio.c` | initialization وenumeration للـ SDIO cards |
| `drivers/mmc/core/sdio_bus.c` | الـ SDIO bus: match، probe، remove |
| `drivers/mmc/core/sdio_io.c` | تنفيذ كل دوال I/O: readb، writeb، memcpy... |
| `drivers/mmc/core/sdio_irq.c` | thread بيـpoll أو بيستقبل IRQs من الكروت |
| `drivers/mmc/core/sdio_cis.c` | قراءة CIS tuples من الكارت |
| `drivers/mmc/core/sdio_ops.c` | أوامر MMC منخفضة المستوى خاصة بالـ SDIO |

#### Headers المهمة
| الملف | الدور |
|---|---|
| `include/linux/mmc/sdio_func.h` | **الملف ده** — API للـ function drivers |
| `include/linux/mmc/sdio.h` | تعريفات الـ SDIO standard registers |
| `include/linux/mmc/sdio_ids.h` | vendor/device IDs لكروت SDIO معروفة |
| `include/linux/mmc/card.h` | `struct mmc_card` — الكارت الفيزيائي كله |
| `include/linux/mmc/host.h` | `struct mmc_host` — الـ host controller |
| `include/linux/mmc/pm.h` | `mmc_pm_flag_t` وفلاجات الـ power management |

#### Hardware Drivers (أمثلة)
| الملف | الدور |
|---|---|
| `drivers/mmc/host/sdhci.c` | أشهر SDIO/SD host controller driver |
| `drivers/mmc/host/dw_mmc.c` | DesignWare MMC host (شائع في ARM SoCs) |
| `drivers/mmc/host/bcm2835.c` | Raspberry Pi host controller |

#### Function Drivers (أمثلة)
| الملف | الدور |
|---|---|
| `drivers/net/wireless/brcm80211/brcmfmac/` | Broadcom WiFi SDIO driver |
| `drivers/mmc/core/sdio_uart.c` | SDIO UART function driver |
## Phase 2: شرح الـ SDIO Function Framework

### المشكلة — ليه الـ Subsystem ده موجود أصلاً؟

الـ **SDIO (Secure Digital Input/Output)** بيخلي كارت SD العادي يشتغل كـ peripheral حقيقي — Wi-Fi، Bluetooth، GPS، كاميرا، إلخ. المشكلة إن كارت SDIO الواحد ممكن يحتوي على أكتر من وظيفة (function) جوّاه في نفس الوقت — مثلاً Wi-Fi + Bluetooth في نفس الكارت الجسمي.

من غير framework، كل driver هيضطر يتعامل مع الـ MMC host controller على طول — يعمل arbitration للـ bus، يقرأ CIS (Card Information Structure) بنفسه، يدير الـ IRQ sharing، ويتعامل مع الـ block size negotiation. ده كود متكرر، وخطر، وصعب في الـ multi-function scenario.

**الـ SDIO Function Framework** وُجد عشان يحل بالظبط ده.

---

### الحل — إيه الـ Approach بتاع الـ Kernel؟

الـ kernel بيعامل كل **function** جوّا الـ SDIO card كـ device مستقل على الـ Linux device model. يعني:

- كل function بيبقى ليه `struct sdio_func` — device object كامل.
- في `struct sdio_driver` بيعمل probe/remove تبع الـ function مش الـ card كلها.
- الـ framework هو اللي بيـmanage الـ bus locking (`sdio_claim_host` / `sdio_release_host`)، الـ IRQ routing، والـ CIS parsing.
- الـ driver مش محتاج يعرف حاجة عن الـ MMC layer تحته.

---

### Big Picture Architecture

```
+---------------------------------------------------------------+
|                     SDIO Function Driver                      |
|   (مثلاً: brcmfmac Wi-Fi driver)                              |
|                                                               |
|  sdio_register_driver(&my_driver)                             |
|  probe(sdio_func, sdio_device_id)                             |
|  sdio_readb / sdio_writeb / sdio_memcpy_fromio                |
+------------------------------+--------------------------------+
                               |
                   struct sdio_driver
                   struct sdio_func
                               |
+------------------------------v--------------------------------+
|                  SDIO Function Framework                      |
|              (drivers/mmc/core/sdio*.c)                       |
|                                                               |
|  - Bus type: sdio_bus_type                                    |
|  - CIS parsing → populate sdio_func fields                    |
|  - IRQ thread management                                      |
|  - Host locking (claim/release)                               |
|  - Block size negotiation                                     |
|  - PM flags (MMC_PM_KEEP_POWER, MMC_PM_WAKE_SDIO_IRQ)         |
+------------------------------+--------------------------------+
                               |
                   struct mmc_card
                               |
+------------------------------v--------------------------------+
|                     MMC Core Layer                            |
|              (drivers/mmc/core/core.c)                        |
|                                                               |
|  - Card detection & enumeration                               |
|  - Bus width negotiation                                      |
|  - Clock & power management                                   |
+------------------------------+--------------------------------+
                               |
+------------------------------v--------------------------------+
|                  MMC Host Controller Driver                   |
|         (مثلاً: sdhci.c, dw_mmc.c, sunxi-mmc.c)              |
|                                                               |
|  - Hardware registers                                         |
|  - DMA transfers                                              |
|  - Physical SDIO bus signals                                  |
+---------------------------------------------------------------+
                               |
                      [ SDIO Card Hardware ]
                      Function 0 (CIA) + Function 1..7
```

---

### Analogy حقيقي

فكّر في **USB hub** متصل بيه 3 أجهزة: كيبورد، ماوس، وكاميرا.

| USB World | SDIO World |
|---|---|
| USB hub (جهاز واحد جسمي) | SDIO card (كارت واحد جسمي) |
| USB device descriptor | `struct mmc_card` |
| USB interface (function) | `struct sdio_func` |
| USB interface driver | `struct sdio_driver` |
| USB `usb_register_driver()` | `sdio_register_driver()` |
| USB `probe(usb_interface *)` | `probe(sdio_func *)` |
| Configuration descriptor | CIS tuples → `sdio_func_tuple` |
| Vendor ID / Product ID | `func->vendor` / `func->device` |

زي ما في USB كل interface بيبقى ليه driver مستقل حتى لو في نفس الجهاز، كمان في SDIO كل function بيبقى ليه `sdio_driver` مستقل حتى لو في نفس الكارت.

---

### الـ Core Abstraction

الـ abstraction الأساسية هي `struct sdio_func` — هي الـ unit اللي كل حاجة بتتمحور حواليها:

```c
struct sdio_func {
    struct mmc_card     *card;          /* الـ card الأم */
    struct device        dev;           /* Linux device model integration */
    sdio_irq_handler_t  *irq_handler;   /* callback لما يجي IRQ */
    unsigned int         num;           /* رقم الـ function (1..7) */

    unsigned char        class;         /* standard interface class */
    unsigned short       vendor;        /* vendor ID */
    unsigned short       device;        /* device ID */

    unsigned             max_blksize;   /* أقصى block size مدعوم */
    unsigned             cur_blksize;   /* الـ block size الحالي */

    unsigned             enable_timeout;/* وقت انتظار enable بالـ msec */
    unsigned int         state;         /* SDIO_STATE_PRESENT */

    u8                  *tmpbuf;        /* DMA-able scratch buffer */

    u8                   major_rev;     /* revision info من CIS */
    u8                   minor_rev;
    unsigned             num_info;
    const char         **info;          /* info strings من CIS */

    struct sdio_func_tuple *tuples;     /* raw CIS tuples غير معروفة */
};
```

وبجانبها `struct sdio_driver`:

```c
struct sdio_driver {
    char *name;
    const struct sdio_device_id *id_table; /* matching table */

    int  (*probe) (struct sdio_func *, const struct sdio_device_id *);
    void (*remove)(struct sdio_func *);
    void (*shutdown)(struct sdio_func *);

    struct device_driver drv; /* Linux driver model */
};
```

الـ matching بين driver وfunction بيحصل عن طريق `sdio_device_id`:

```c
struct sdio_device_id {
    __u8            class;       /* SDIO_ANY_ID لو مش مهم */
    __u16           vendor;      /* Vendor ID */
    __u16           device;      /* Device ID */
    kernel_ulong_t  driver_data; /* private data للـ driver */
};
```

الـ macros `SDIO_DEVICE(vend, dev)` و `SDIO_DEVICE_CLASS(dev_class)` بتسهّل بناء الـ id_table.

---

### إيه اللي الـ Framework بيملكه vs. إيه اللي بيفوّضه للـ Driver؟

#### الـ Framework بيملك (owns):

| المسؤولية | الآلية |
|---|---|
| Bus locking | `sdio_claim_host()` / `sdio_release_host()` |
| IRQ thread & dispatch | `sdio_claim_irq()` / `sdio_release_irq()` |
| Function enable/disable | `sdio_enable_func()` / `sdio_disable_func()` |
| Block size negotiation | `sdio_set_block_size()` |
| CIS parsing & tuple storage | `sdio_func_tuple` linked list |
| Device registration في sysfs | `SDIO_STATE_PRESENT` + `struct device` |
| PM capabilities negotiation | `sdio_get_host_pm_caps()` / `sdio_set_host_pm_flags()` |
| Re-tuning control | `sdio_retune_hold_now()` / `sdio_retune_release()` |
| Transfer alignment | `sdio_align_size()` |

#### الـ Driver بيتحمل مسؤولية:

| المسؤولية | الآلية |
|---|---|
| تنفيذ الـ protocol فوق SDIO | `sdio_readb` / `sdio_writeb` / `sdio_memcpy_fromio` / إلخ |
| إدارة الـ private state | `sdio_set_drvdata()` / `sdio_get_drvdata()` |
| تسجيل الـ IRQ handler | `sdio_claim_irq(func, my_handler)` |
| تحديد الـ block size المطلوب | `sdio_set_block_size(func, 512)` |
| إدارة الـ firmware load | مسؤولية الـ driver بالكامل |
| الـ network/char/misc device layer | مسؤولية الـ driver بالكامل |

**ملاحظة مهمة:** الـ `sdio_f0_readb` / `sdio_f0_writeb` بيسمحوا للـ driver بقراءة الـ **Function 0** (الـ CIA — Common I/O Area) اللي هي الـ control function الخاصة بالكارت كله — لكن الصلاحيات محدودة، الـ framework بيحمي العمليات الحساسة.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Macros والـ Constants — Cheatsheet

#### State Flags — `sdio_func::state`

| Flag | القيمة | المعنى |
|------|--------|--------|
| `SDIO_STATE_PRESENT` | `(1 << 0)` | الـ function موجودة في sysfs |

#### PM Flags — `mmc_pm_flag_t`

| Flag | القيمة | المعنى |
|------|--------|--------|
| `MMC_PM_KEEP_POWER` | `(1 << 0)` | احتفظ بالطاقة أثناء suspend |
| `MMC_PM_WAKE_SDIO_IRQ` | `(1 << 1)` | صحّي الـ host لما يجي IRQ من SDIO |

#### Matching Constants — `sdio_device_id`

| Constant | القيمة | المعنى |
|----------|--------|--------|
| `SDIO_ANY_ID` | `(~0)` | wildcard لأي vendor/device/class |

#### Macros للـ sdio_func

| Macro | ما بيعمله |
|-------|-----------|
| `sdio_func_present(f)` | يرجع 1 لو الـ function موجودة في sysfs |
| `sdio_func_set_present(f)` | يحط `SDIO_STATE_PRESENT` في الـ state |
| `sdio_func_id(f)` | يرجع اسم الـ device (string) من `dev_name` |
| `sdio_get_drvdata(f)` | يجيب الـ driver private data |
| `sdio_set_drvdata(f, d)` | يحط الـ driver private data |
| `dev_to_sdio_func(d)` | يحوّل `struct device *` لـ `struct sdio_func *` |
| `SDIO_DEVICE(vend, dev)` | ينشئ `sdio_device_id` بـ vendor+device محدد |
| `SDIO_DEVICE_CLASS(cls)` | ينشئ `sdio_device_id` بـ class محدد فقط |
| `sdio_register_driver(drv)` | يسجل الـ driver (wrapper على `__sdio_register_driver` + `THIS_MODULE`) |
| `module_sdio_driver(drv)` | boilerplate كامل لـ module_init/exit |

---

### 1. أهم الـ Structs

#### `struct sdio_func_tuple`
**الغرض:** تخزين CIS tuple واحدة — المعلومات الخام من الـ Card Information Structure اللي ما تعرفها الـ SDIO core.

| Field | النوع | الوصف |
|-------|-------|--------|
| `next` | `struct sdio_func_tuple *` | pointer للـ tuple التالية (linked list) |
| `code` | `unsigned char` | رقم الـ tuple (CISTPL code) |
| `size` | `unsigned char` | حجم الـ data بالبايت |
| `data[]` | `unsigned char[]` | flexible array — البيانات الفعلية للـ tuple |

**الاتصالات:** `sdio_func` بيحتفظ بـ pointer `tuples` لأول عنصر في الـ linked list.

---

#### `struct sdio_func`
**الغرض:** يمثل SDIO function device واحدة — الوحدة الأساسية في نظام SDIO. كل كارت SDIO ممكن يكون فيه أكتر من function (زي WiFi + Bluetooth في نفس الكارت).

| Field | النوع | الوصف |
|-------|-------|--------|
| `card` | `struct mmc_card *` | الكارت اللي بيتبع ليه |
| `dev` | `struct device` | embedded Linux device (مش pointer!) |
| `irq_handler` | `sdio_irq_handler_t *` | callback لما يجي IRQ |
| `num` | `unsigned int` | رقم الـ function (1-7) |
| `class` | `unsigned char` | الـ standard interface class |
| `vendor` | `unsigned short` | vendor ID |
| `device` | `unsigned short` | device ID |
| `max_blksize` | `unsigned` | أكبر block size مدعوم |
| `cur_blksize` | `unsigned` | الـ block size الحالي المستخدم |
| `enable_timeout` | `unsigned` | timeout بالـ msec لعملية الـ enable |
| `state` | `unsigned int` | state flags (حالياً `SDIO_STATE_PRESENT` بس) |
| `tmpbuf` | `u8 *` | scratch buffer قابل لـ DMA |
| `major_rev` | `u8` | major revision |
| `minor_rev` | `u8` | minor revision |
| `num_info` | `unsigned` | عدد الـ info strings |
| `info` | `const char **` | array من الـ info strings |
| `tuples` | `struct sdio_func_tuple *` | linked list من الـ CIS tuples |

**الاتصالات:** مرتبط بـ `mmc_card` (الأب)، وبيحتوي على `struct device` embedded، وبيشير لـ `sdio_func_tuple` linked list.

---

#### `struct sdio_driver`
**الغرض:** يمثل SDIO function driver — الـ driver اللي بيتعامل مع function معينة.

| Field | النوع | الوصف |
|-------|-------|--------|
| `name` | `char *` | اسم الـ driver |
| `id_table` | `const struct sdio_device_id *` | جدول الـ IDs المدعومة |
| `probe` | `int (*)(sdio_func *, sdio_device_id *)` | يتسمى لما يُكشف device متوافق |
| `remove` | `void (*)(sdio_func *)` | يتسمى لما يتشال الـ device |
| `shutdown` | `void (*)(sdio_func *)` | يتسمى عند shutdown الـ system |
| `drv` | `struct device_driver` | embedded Linux driver struct |

**الاتصالات:** مرتبط بـ `sdio_device_id` (matching table)، وبيحتوي على `struct device_driver` embedded.

---

#### `struct sdio_device_id` (من `mod_devicetable.h`)
**الغرض:** يحدد SDIO device بـ class و vendor و device ID للـ matching.

| Field | النوع | الوصف |
|-------|-------|--------|
| `class` | `__u8` | class أو `SDIO_ANY_ID` |
| `vendor` | `__u16` | vendor ID أو `SDIO_ANY_ID` |
| `device` | `__u16` | device ID أو `SDIO_ANY_ID` |
| `driver_data` | `kernel_ulong_t` | data خاصة بالـ driver |

---

### 2. رسم علاقات الـ Structs (ASCII)

```
                    ┌─────────────────────────────────┐
                    │         struct mmc_card          │
                    │  (الكارت الفيزيكي كامل)          │
                    └──────────────┬──────────────────┘
                                   │  card->
                          func[0]  │  func[1] ... func[7]
                                   │
              ┌────────────────────▼────────────────────┐
              │            struct sdio_func              │
              │  ┌─────────────────────────────────┐    │
              │  │  struct device  dev  (embedded) │    │
              │  │  (يرتبط بـ Linux device model)  │    │
              │  └─────────────────────────────────┘    │
              │                                         │
              │  irq_handler ──► sdio_irq_handler_t()  │
              │                                         │
              │  tuples ──────► sdio_func_tuple         │
              └─────────────────────────────────────────┘
                        ▲
                        │ id_table يحدد
              ┌─────────┴────────────────────────────┐
              │          struct sdio_driver           │
              │  ┌───────────────────────────────┐   │
              │  │  struct device_driver  drv    │   │
              │  │  (embedded — يرتبط بـ bus)   │   │
              │  └───────────────────────────────┘   │
              │  id_table ──► sdio_device_id[]        │
              │  probe()  ──► func_driver_probe()     │
              │  remove() ──► func_driver_remove()    │
              └──────────────────────────────────────┘


  sdio_func_tuple linked list:

  sdio_func.tuples
       │
       ▼
  ┌────────────┐    next    ┌────────────┐    next    ┌────────────┐
  │  tuple[0]  │ ─────────► │  tuple[1]  │ ─────────► │  tuple[N]  │──► NULL
  │  code=0x21 │            │  code=0x22 │            │  code=...  │
  │  size=4    │            │  size=8    │            │  ...       │
  │  data[4]   │            │  data[8]   │            │  data[...]  │
  └────────────┘            └────────────┘            └────────────┘
```

---

### 3. دورة حياة الـ sdio_func (Lifecycle)

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    CREATION & REGISTRATION                       │
  └─────────────────────────────────────────────────────────────────┘

  MMC Core يكتشف الكارت
        │
        ▼
  sdio_init_func()          ← MMC core ينشئ struct sdio_func
  [kmalloc + init fields]
        │
        ▼
  sdio_read_func_cis()      ← يقرأ الـ CIS ويبني sdio_func_tuple list
        │
        ▼
  sdio_func_set_present()   ← يحط SDIO_STATE_PRESENT
        │
        ▼
  device_add(&func->dev)    ← يسجل في Linux device model (sysfs ينشأ)
        │
        ▼
  sdio_bus_match()          ← kernel يطابق مع id_table في sdio_driver
        │
        ▼
  driver->probe(func, id)   ← الـ driver يبدأ شغله


  ┌─────────────────────────────────────────────────────────────────┐
  │                         USAGE                                    │
  └─────────────────────────────────────────────────────────────────┘

  Driver يحجز الـ host:
  sdio_claim_host(func)     ← يأخذ lock على الـ MMC host
        │
        ▼
  sdio_enable_func(func)    ← يفعّل الـ function في الكارت
        │
        ▼
  sdio_set_block_size()     ← يضبط cur_blksize
        │
        ▼
  sdio_claim_irq(func, handler) ← يسجل IRQ handler في func->irq_handler
        │
        ▼
  sdio_read*/sdio_write*()  ← عمليات I/O عادية
        │
        ▼
  sdio_release_host(func)   ← يحرر lock على الـ host


  ┌─────────────────────────────────────────────────────────────────┐
  │                        TEARDOWN                                  │
  └─────────────────────────────────────────────────────────────────┘

  driver->remove(func)      ← الـ driver يوقف شغله
        │
        ▼
  sdio_release_irq(func)    ← يلغي تسجيل IRQ handler
        │
        ▼
  sdio_disable_func(func)   ← يوقف الـ function في الكارت
        │
        ▼
  device_del(&func->dev)    ← يشيل من sysfs
        │
        ▼
  sdio_free_func_cis()      ← يحرر sdio_func_tuple list
        │
        ▼
  sdio_free_func()          ← kfree للـ struct نفسه
```

---

### 4. Call Flow Diagrams

#### تسجيل الـ Driver

```
  Driver Module Load
       │
       ▼
  module_sdio_driver(my_driver)
       │  expands to module_init/module_exit
       ▼
  sdio_register_driver(&my_driver)
       │  macro adds THIS_MODULE
       ▼
  __sdio_register_driver(drv, module)
       │
       ▼
  driver_register(&drv->drv)   ← Linux driver core
       │
       ▼
  sdio_bus_match() يُستدعى لكل device موجود
       │
       └─► إذا match ──► drv->probe(func, id)
```

#### عملية I/O — مثال sdio_readb()

```
  Driver Code
      │
      ▼
  sdio_claim_host(func)
      │  يأخذ mutex على mmc_host
      ▼
  sdio_readb(func, addr, &err)
      │
      ▼
  mmc_io_rw_direct()           ← MMC core
      │
      ▼
  host->ops->request()         ← Host Controller Driver
      │
      ▼
  Hardware SDIO CMD52
      │
      ▼
  Return data to sdio_readb()
      │
      ▼
  sdio_release_host(func)
```

#### معالجة الـ IRQ

```
  Hardware يرفع SDIO IRQ line
       │
       ▼
  MMC Host Controller interrupt handler
       │
       ▼
  sdio_run_irqs() / sdio_irq_thread()
       │
       ▼
  for each func with irq_handler != NULL:
       │
       ▼
  func->irq_handler(func)      ← Driver IRQ callback
       │
       ▼
  Driver reads/clears interrupt source
```

#### دورة الـ PM (Power Management)

```
  System Suspend
       │
       ▼
  sdio_get_host_pm_caps(func)  ← يسأل إيه الـ PM المدعوم
       │
       ▼
  sdio_set_host_pm_flags(func, MMC_PM_KEEP_POWER)
       │  أو MMC_PM_WAKE_SDIO_IRQ
       ▼
  mmc_host suspend path        ← ينفذ الـ PM بناءً على الـ flags

  System Resume
       │
       ▼
  mmc_host resume path
       │
       ▼
  Re-enumerate SDIO functions
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks المستخدمة

| Lock | موقعه | بيحمي إيه |
|------|--------|------------|
| `mmc_host::lock` (mutex) | `struct mmc_host` | كل عمليات الـ bus — CMD/DATA |
| `mmc_host::sdio_irq_thread` | kernel thread | تنفيذ الـ IRQ handlers |
| `device::mutex` (من device model) | `struct device` | الـ probe/remove lifecycle |

#### القواعد

**`sdio_claim_host` / `sdio_release_host`:**
- أي I/O operation (read/write) لازم يكون جوه claim/release.
- بيأخذ mutex على مستوى الـ `mmc_host` كامل — يعني كل الـ functions على نفس الكارت بتتنافس على نفس الـ lock.
- لو بتكتب driver، الترتيب الصح:

```c
sdio_claim_host(func);           /* 1. احجز الـ bus */
    sdio_enable_func(func);      /* 2. فعّل الـ function */
    sdio_set_block_size(func, N);/* 3. اضبط الـ block size */
    sdio_claim_irq(func, handler);/* 4. سجل IRQ */
    /* ... do I/O ... */
    sdio_release_irq(func);      /* 5. الغ IRQ قبل disable */
    sdio_disable_func(func);     /* 6. أوقف الـ function */
sdio_release_host(func);         /* 7. حرر الـ bus */
```

**IRQ Handler:**
- الـ `func->irq_handler` بيتسمى من thread context (مش hard IRQ).
- الـ handler ممنوع يستدعي `sdio_claim_host` — الـ host بيكون محجوز أصلاً من الـ IRQ thread.
- الـ `sdio_claim_irq` / `sdio_release_irq` نفسهم لازم يتسموا جوه `sdio_claim_host`.

**Lock Ordering:**
```
  mmc_host::lock
      └─► (أي عملية SDIO تانية — لا يوجد nested locks)
```
لا في nested locking في مستوى الـ `sdio_func` — كل الـ synchronization بيمشي من خلال الـ host lock الواحد.

**الـ `tmpbuf`:**
- الـ `sdio_func::tmpbuf` هو DMA-safe scratch buffer.
- بيتحمى ضمنياً لأنه بيتستخدم بس جوه I/O operations اللي بتكون جوه `sdio_claim_host`.

**الـ `state` field:**
- الـ bit `SDIO_STATE_PRESENT` بيتحط مرة واحدة عند الإنشاء ومفيش race condition عليه لأنه بيتعمل قبل `device_add`.

---

### ملخص العلاقات

```
  sdio_driver ──(id_table)──► sdio_device_id[]
      │
      └──(drv)──► device_driver ──► sdio_bus_type

  sdio_func ──(card)──► mmc_card ──► mmc_host
      │
      ├──(dev)──► device ──► sdio_bus_type
      │
      └──(tuples)──► sdio_func_tuple ──► next ──► ...

  sdio_driver.probe(sdio_func, sdio_device_id)
      └── driver sets: sdio_set_drvdata(func, priv)
      └── driver calls: sdio_claim_host → sdio_enable_func → I/O ops
```
## Phase 4: شرح الـ Functions

---

## ملخص سريع — Cheatsheet

### Registration & Driver Lifecycle

| Function / Macro | Signature (مختصرة) | الغرض |
|---|---|---|
| `sdio_register_driver` | `macro(drv)` | يسجّل SDIO driver في البص |
| `__sdio_register_driver` | `int (drv, module*)` | الـ actual registration مع module reference |
| `sdio_unregister_driver` | `void (drv)` | يشيل الـ driver من البص |
| `module_sdio_driver` | `macro(__drv)` | boilerplate بديل لـ `module_init`/`module_exit` |

### Function Enable/Disable

| Function | Signature | الغرض |
|---|---|---|
| `sdio_enable_func` | `int (func)` | يفعّل SDIO function |
| `sdio_disable_func` | `int (func)` | يعطّل SDIO function |
| `sdio_set_block_size` | `int (func, blksz)` | يضبط block size |

### Host Locking

| Function | الغرض |
|---|---|
| `sdio_claim_host` | يحجز الـ host controller (mutex-like) |
| `sdio_release_host` | يحرر الـ host controller |

### IRQ Management

| Function | الغرض |
|---|---|
| `sdio_claim_irq` | يسجّل IRQ handler للـ function |
| `sdio_release_irq` | يلغي تسجيل الـ IRQ handler |

### I/O — Single Byte/Word/DWord

| Function | Direction | Width |
|---|---|---|
| `sdio_readb` | Read | 8-bit |
| `sdio_readw` | Read | 16-bit |
| `sdio_readl` | Read | 32-bit |
| `sdio_writeb` | Write | 8-bit |
| `sdio_writew` | Write | 16-bit |
| `sdio_writel` | Write | 32-bit |
| `sdio_writeb_readb` | Write+Read | 8-bit atomic |

### I/O — Block / Stream

| Function | Direction | نوع النقل |
|---|---|---|
| `sdio_memcpy_fromio` | Read | multi-byte, incrementing addr |
| `sdio_readsb` | Read | multi-byte, fixed addr (FIFO) |
| `sdio_memcpy_toio` | Write | multi-byte, incrementing addr |
| `sdio_writesb` | Write | multi-byte, fixed addr (FIFO) |

### Function 0 (CIA) I/O

| Function | الغرض |
|---|---|
| `sdio_f0_readb` | قراءة register من Function 0 |
| `sdio_f0_writeb` | كتابة register في Function 0 (محدود) |

### Power Management

| Function | الغرض |
|---|---|
| `sdio_get_host_pm_caps` | يجيب PM capabilities من الـ host |
| `sdio_set_host_pm_flags` | يضبط PM flags (keep power / wake on IRQ) |

### Retune Control

| Function | الغرض |
|---|---|
| `sdio_retune_crc_disable` | يوقف CRC errors أثناء retune |
| `sdio_retune_crc_enable` | يعيد CRC بعد retune |
| `sdio_retune_hold_now` | يوقف الـ retuning مؤقتاً |
| `sdio_retune_release` | يسمح للـ retuning يرجع |

### Helper Macros

| Macro | الغرض |
|---|---|
| `sdio_func_present(f)` | يتحقق إن الـ function موجودة في sysfs |
| `sdio_func_set_present(f)` | يعلّم الـ function كـ present |
| `sdio_func_id(f)` | يجيب string ID من `dev_name` |
| `sdio_get_drvdata(f)` | يجيب driver private data |
| `sdio_set_drvdata(f,d)` | يحط driver private data |
| `dev_to_sdio_func(d)` | يحوّل `struct device*` لـ `struct sdio_func*` |
| `SDIO_DEVICE(v,d)` | يبني `sdio_device_id` entry لـ vendor+device |
| `SDIO_DEVICE_CLASS(c)` | يبني `sdio_device_id` entry لـ class فقط |

---

## Category 1: Driver Registration

**الغرض:** ربط الـ SDIO driver بالـ Linux bus model. الكيرنل يعمل match تلقائي بين الـ driver وأي function تتوافق مع الـ `id_table`.

---

### `sdio_register_driver` (macro)

```c
#define sdio_register_driver(drv) \
    __sdio_register_driver(drv, THIS_MODULE)
```

Macro wrapper يمرر `THIS_MODULE` تلقائياً لـ `__sdio_register_driver`. لازم تستخدم الـ macro دي مش الـ `__` version مباشرة عشان الـ module reference صح.

- **`drv`**: مؤشر لـ `struct sdio_driver` الخاص بالـ driver.
- **Return**: `int` — صفر لو نجح، سالب لو فشل.
- **من يناديها**: `module_init()` أو `module_sdio_driver` macro.

---

### `__sdio_register_driver`

```c
extern int __sdio_register_driver(struct sdio_driver *drv,
                                   struct module *owner);
```

الـ actual implementation للتسجيل. بتسجّل الـ `struct device_driver` الداخلية في SDIO bus، وبتحدد الـ `probe`/`remove`/`shutdown` callbacks. الكيرنل يعمل bus scan وينادي `probe` لكل function موجودة وبتتطابق مع الـ `id_table`.

- **`drv`**: الـ driver المراد تسجيله.
- **`owner`**: الـ module المالك — لازم يكون `THIS_MODULE`.
- **Return**: `0` success، `-EINVAL` لو الـ `id_table` null، أو error من `driver_register`.
- **Side effects**: يضيف الـ driver لـ sysfs، ويعمل probe لكل function متطابقة.

---

### `sdio_unregister_driver`

```c
extern void sdio_unregister_driver(struct sdio_driver *drv);
```

يشيل الـ driver من SDIO bus. بيتسبب في نداء `remove` callback على كل function كانت مرتبطة بالـ driver. لازم تتنادى في `module_exit`.

- **`drv`**: الـ driver المراد إزالته.
- **Return**: `void`.
- **Key detail**: بيضمن إن كل الـ resources اتحررت قبل ما الـ module يتسلخ من الكيرنل.

---

### `module_sdio_driver` (macro)

```c
#define module_sdio_driver(__sdio_driver) \
    module_driver(__sdio_driver, sdio_register_driver, \
                  sdio_unregister_driver)
```

Boilerplate macro يولّد `module_init` و `module_exit` تلقائياً. الـ drivers اللي ما عندهاش initialization خاصة بتستخدمه.

```c
/* Example usage */
static struct sdio_driver my_sdio_drv = {
    .name     = "my_wifi",
    .id_table = my_sdio_ids,
    .probe    = my_probe,
    .remove   = my_remove,
};
module_sdio_driver(my_sdio_drv);
```

---

## Category 2: Host Locking

**الغرض:** الـ MMC/SDIO host controller مورد shared. أي عملية I/O أو configuration لازم تتم وانت ماسك الـ host lock عشان تضمن exclusive access.

---

### `sdio_claim_host`

```c
extern void sdio_claim_host(struct sdio_func *func);
```

يحجز الـ host controller المرتبط بالـ `func`. بينفذ mutex/semaphore acquire داخلياً. **لازم** تنادي دي قبل أي I/O operation أو `sdio_enable_func` / `sdio_set_block_size`.

- **`func`**: الـ SDIO function — بيجيب منها `func->card->host`.
- **Return**: `void` — blocking call، بيستنى لو حد تاني ماسك الـ host.
- **Locking**: بتعمل nest — نفس الـ thread مش لازم يناديها مرتين قبل `release`.
- **من يناديها**: Driver في `probe`, `remove`, أو أي runtime I/O path.

---

### `sdio_release_host`

```c
extern void sdio_release_host(struct sdio_func *func);
```

يحرر الـ host lock اللي أخده `sdio_claim_host`. **لازم** تتنادى بعد كل `claim` في نفس الـ context.

- **`func`**: نفس الـ function اللي استخدمتها في `claim_host`.
- **Return**: `void`.
- **Key detail**: لو نسيت `release` — deadlock مضمون.

---

## Category 3: Function Enable / Disable

**الغرض:** كل SDIO function لازم تتفعّل صراحةً قبل ما الـ driver يقدر يعمل I/O عليها. الـ enable بيحط الـ IOEX bit في CIA register.

---

### `sdio_enable_func`

```c
extern int sdio_enable_func(struct sdio_func *func);
```

يفعّل الـ SDIO function عن طريق set الـ I/O Enable bit في Function 0's CCCR. بعدين بيستنى لحد `func->enable_timeout` ms لحد ما الـ I/O Ready bit يتست. **Requires host claimed.**

- **`func`**: الـ function المراد تفعيلها.
- **Return**: `0` نجاح، `-ETIMEDOUT` لو الـ function ما ردتش، سالب تاني لو error في I/O.
- **Side effects**: الـ function تبقى ready لاستقبال I/O commands.
- **من يناديها**: Driver's `probe` callback.

```
sdio_enable_func flow:
  1. sdio_f0_readb(CCCR_IOEx)     --> read current enable bits
  2. Set bit for func->num
  3. sdio_f0_writeb(CCCR_IOEx)    --> write back
  4. Poll CCCR_IORx until func bit set OR timeout
```

---

### `sdio_disable_func`

```c
extern int sdio_disable_func(struct sdio_func *func);
```

يعطّل الـ function عن طريق clear الـ I/O Enable bit. **Requires host claimed.**

- **`func`**: الـ function المراد تعطيلها.
- **Return**: `0` نجاح، سالب لو فيه error.
- **من يناديها**: Driver's `remove` callback أو error paths في `probe`.

---

### `sdio_set_block_size`

```c
extern int sdio_set_block_size(struct sdio_func *func, unsigned blksz);
```

يضبط الـ block size للـ block mode transfers على الـ function دي. لو `blksz == 0`، بيستخدم `func->card->host->max_blk_size`. بيكتب الـ value في FBR register للـ function. **Requires host claimed.**

- **`func`**: الـ target function.
- **`blksz`**: الـ block size المطلوبة بالـ bytes. لازم تكون أصغر من `func->max_blksize`.
- **Return**: `0` نجاح، `-EINVAL` لو الـ size أكبر من المسموح، سالب تاني لو I/O error.
- **Side effects**: يحدّث `func->cur_blksize`.

---

## Category 4: IRQ Management

**الغرض:** الـ SDIO IRQ mechanism بيشتغل عن طريق shared interrupt line. الـ host driver بيكتشف أي function عملت IRQ، وبينادي الـ registered handler.

---

### `sdio_claim_irq`

```c
extern int sdio_claim_irq(struct sdio_func *func,
                           sdio_irq_handler_t *handler);
```

يسجّل IRQ handler لـ function معينة. بيحط المؤشر في `func->irq_handler` وبيفعّل الـ interrupt في الـ CCCR. **Requires host claimed.**

- **`func`**: الـ function اللي عايزها تستقبل IRQs.
- **`handler`**: Callback من النوع `void (*)(struct sdio_func *)` — بيتنادى في interrupt context.
- **Return**: `0` نجاح، `-EBUSY` لو handler مسجّل بالفعل، سالب لو error.
- **Key detail**: الـ handler بيتشغّل من dedicated `sdio_irq` kernel thread — مش hard IRQ context — فممكن تعمل فيه sleeping ops.

---

### `sdio_release_irq`

```c
extern int sdio_release_irq(struct sdio_func *func);
```

يلغي تسجيل الـ IRQ handler ويعطّل الـ interrupt للـ function دي. **Requires host claimed.**

- **`func`**: الـ function المراد إلغاء IRQ تاعها.
- **Return**: `0` نجاح، سالب لو error.
- **من يناديها**: `remove` callback أو cleanup paths.

---

## Category 5: I/O Helper

### `sdio_align_size`

```c
extern unsigned int sdio_align_size(struct sdio_func *func,
                                     unsigned int sz);
```

يحسب الـ transfer size المحاذية (aligned) الأقرب لـ `sz` بناءً على قدرات الـ host والـ function (block size). لازم تستخدمها قبل ما تعمل DMA buffer allocation.

- **`func`**: الـ function المعنية.
- **`sz`**: الـ size الأصلية بالـ bytes.
- **Return**: الـ aligned size — دايماً >= `sz`.
- **Key detail**: بتأخذ بالاعتبار الـ `cur_blksize` والـ host DMA alignment constraints.

---

## Category 6: Single-Register I/O

**الغرض:** قراءة/كتابة registers بشكل مباشر. كلها تشترط host claimed. الـ err_ret parameter بيبلّغ عن أي error بدلاً من الـ return value في حالة الـ read functions.

---

### `sdio_readb`

```c
extern u8 sdio_readb(struct sdio_func *func,
                     unsigned int addr, int *err_ret);
```

يقرأ byte واحد من address `addr` في space الـ function.

- **`func`**: الـ SDIO function.
- **`addr`**: الـ 17-bit function address.
- **`err_ret`**: مؤشر لـ int بيتحط فيه الـ error code. ممكن يكون NULL لو مش محتاج.
- **Return**: الـ byte المقروء. لو error، القيمة undefined والـ `*err_ret` سالب.

---

### `sdio_readw`

```c
extern u16 sdio_readw(struct sdio_func *func,
                      unsigned int addr, int *err_ret);
```

يقرأ 16-bit word — بيعمل عملياً `readb` مرتين (little-endian). نفس parameters الـ `sdio_readb`.

- **Return**: الـ 16-bit value، little-endian.

---

### `sdio_readl`

```c
extern u32 sdio_readl(struct sdio_func *func,
                      unsigned int addr, int *err_ret);
```

يقرأ 32-bit dword — `readb` × 4، little-endian. نفس parameters السابقة.

- **Return**: الـ 32-bit value، little-endian.

---

### `sdio_writeb`

```c
extern void sdio_writeb(struct sdio_func *func, u8 b,
                        unsigned int addr, int *err_ret);
```

يكتب byte واحد على address `addr`.

- **`b`**: الـ value المراد كتابتها.
- **`err_ret`**: بيتحط فيه الـ error لو حصل. ممكن NULL.
- **Return**: `void`.

---

### `sdio_writew`

```c
extern void sdio_writew(struct sdio_func *func, u16 b,
                        unsigned int addr, int *err_ret);
```

يكتب 16-bit word — `writeb` × 2، little-endian.

---

### `sdio_writel`

```c
extern void sdio_writel(struct sdio_func *func, u32 b,
                        unsigned int addr, int *err_ret);
```

يكتب 32-bit dword — `writeb` × 4، little-endian.

---

### `sdio_writeb_readb`

```c
extern u8 sdio_writeb_readb(struct sdio_func *func,
                             u8 write_byte,
                             unsigned int addr,
                             int *err_ret);
```

عملية **atomic** write-then-read على نفس الـ address. بتستخدم SDIO CMD52 مع الـ RAW (Read After Write) flag set. مفيدة جداً لـ read-modify-write على registers زي interrupt mask.

- **`write_byte`**: القيمة المكتوبة.
- **`addr`**: الـ register address.
- **Return**: القيمة الـ register بعد الكتابة (من الـ hardware).
- **Key detail**: Atomic من ناحية الـ card — مفيش race condition بين الـ write والـ read.

---

## Category 7: Block / Stream I/O

**الغرض:** نقل كميات كبيرة من البيانات بكفاءة. الـ `memcpy` variants بتزوّد الـ address بعد كل byte (suitable for RAM/registers). الـ `readsb`/`writesb` variants بتفضل على نفس الـ address (suitable for FIFOs).

---

### `sdio_memcpy_fromio`

```c
extern int sdio_memcpy_fromio(struct sdio_func *func,
                               void *dst,
                               unsigned int addr,
                               int count);
```

ينقل `count` bytes من card address `addr` لـ buffer `dst`. بيزوّد الـ address بعد كل byte (CMD53 incremental). بيستخدم DMA لو الـ host يدعمه.

- **`dst`**: الـ destination buffer — لازم يكون DMA-capable (or `func->tmpbuf` للـ small sizes).
- **`addr`**: Starting address في الـ function space.
- **`count`**: عدد الـ bytes.
- **Return**: `0` نجاح، سالب لو error.
- **Locking**: Requires host claimed.

---

### `sdio_readsb`

```c
extern int sdio_readsb(struct sdio_func *func,
                       void *dst,
                       unsigned int addr,
                       int count);
```

نفس `sdio_memcpy_fromio` لكن بـ **fixed address** — CMD53 مع `FIXED` flag. مناسب لقراءة FIFO register. الـ `addr` مش بيتزود.

- **Parameters**: نفس `sdio_memcpy_fromio`.
- **Return**: `0` نجاح، سالب error.

---

### `sdio_memcpy_toio`

```c
extern int sdio_memcpy_toio(struct sdio_func *func,
                             unsigned int addr,
                             void *src,
                             int count);
```

ينقل `count` bytes من buffer `src` لـ card address `addr`. Incrementing address. لاحظ إن ترتيب الـ parameters مختلف عن `_fromio` — الـ `addr` جه قبل `src`.

- **`addr`**: Destination address في الـ function space.
- **`src`**: الـ source buffer.
- **`count`**: عدد الـ bytes.
- **Return**: `0` نجاح، سالب error.

---

### `sdio_writesb`

```c
extern int sdio_writesb(struct sdio_func *func,
                        unsigned int addr,
                        void *src,
                        int count);
```

نفس `sdio_memcpy_toio` لكن بـ **fixed address** — مناسب للـ TX FIFO.

---

## Category 8: Function 0 (CIA) I/O

**الغرض:** Function 0 هي الـ Common I/O Area (CIA) الخاصة بالـ card نفسها. فيها الـ CCCR وغيره. الـ write عليها محدود — SDIO spec بيسمح بـ write على registers معينة بس.

---

### `sdio_f0_readb`

```c
extern unsigned char sdio_f0_readb(struct sdio_func *func,
                                    unsigned int addr,
                                    int *err_ret);
```

يقرأ byte من Function 0 address space. بيستخدم CMD52 مع function number = 0. **Requires host claimed.**

- **`func`**: أي function على الـ card (بيجيب منها الـ card pointer فقط).
- **`addr`**: الـ CIA address (مثل CCCR registers).
- **`err_ret`**: Error output.
- **Return**: الـ byte المقروء.

---

### `sdio_f0_writeb`

```c
extern void sdio_f0_writeb(struct sdio_func *func,
                            unsigned char b,
                            unsigned int addr,
                            int *err_ret);
```

يكتب byte في Function 0 address. **محدود جداً** — الـ SDIO spec بيسمح بالكتابة على CCCR addresses معينة فقط (مثل IO_ABORT). لو حاولت تكتب على address مش مسموح — بيرجع `-EINVAL`.

- **Key detail**: الـ implementation بيتحقق إن الـ address في range المسموح قبل الكتابة.

---

## Category 9: Power Management

**الغرض:** إدارة سلوك الكارت أثناء system suspend. الـ SDIO card ممكن تفضل شغّالة أثناء suspend (لـ wake-on-WLAN مثلاً).

---

### `sdio_get_host_pm_caps`

```c
extern mmc_pm_flag_t sdio_get_host_pm_caps(struct sdio_func *func);
```

بيرجع الـ PM capabilities اللي الـ host controller يدعمها.

- **`func`**: الـ function المعنية.
- **Return**: `mmc_pm_flag_t` bitmask:
  - `MMC_PM_KEEP_POWER` — الـ host يقدر يفضّل power أثناء suspend
  - `MMC_PM_WAKE_SDIO_IRQ` — الـ host يقدر يصحى لو الـ card عملت IRQ
- **من يناديها**: Driver في `suspend` callback عشان يعرف إيه المتاح.

---

### `sdio_set_host_pm_flags`

```c
extern int sdio_set_host_pm_flags(struct sdio_func *func,
                                   mmc_pm_flag_t flags);
```

يطلب من الـ host يفعّل PM features معينة أثناء الـ suspend القادم.

- **`flags`**: Combination من `MMC_PM_KEEP_POWER` و/أو `MMC_PM_WAKE_SDIO_IRQ`.
- **Return**: `0` نجاح، `-EINVAL` لو طلبت flag مش مدعومة من الـ host.
- **Key detail**: لازم تتحقق من `sdio_get_host_pm_caps` أولاً قبل ما تطلب flags.
- **من يناديها**: Driver في `suspend` callback.

```c
/* Typical PM usage in driver suspend */
mmc_pm_flag_t caps = sdio_get_host_pm_caps(func);
if (caps & MMC_PM_KEEP_POWER)
    sdio_set_host_pm_flags(func, MMC_PM_KEEP_POWER |
                                  MMC_PM_WAKE_SDIO_IRQ);
```

---

## Category 10: Retune Control

**الغرض:** الـ retuning هي عملية الكيرنل بتعيد calibrate فيها الـ timing بين الـ host والـ card (ضروري لـ HS200/SDR104). أحياناً الـ driver محتاج يوقف الـ retuning مؤقتاً أثناء critical I/O sequence.

---

### `sdio_retune_crc_disable`

```c
extern void sdio_retune_crc_disable(struct sdio_func *func);
```

يوقف الـ CRC error checking المؤدي لـ trigger retuning. بيستخدم أثناء تسلسل commands معين محتاج consistency. **Requires host claimed.**

- **`func`**: الـ function المعنية.
- **Key detail**: بيزوّد counter داخلي — لكل `crc_disable` لازم `crc_enable` مقابلها.

---

### `sdio_retune_crc_enable`

```c
extern void sdio_retune_crc_enable(struct sdio_func *func);
```

يعيد تفعيل CRC error detection للـ retuning triggering. **Requires host claimed.**

---

### `sdio_retune_hold_now`

```c
extern void sdio_retune_hold_now(struct sdio_func *func);
```

يوقف الـ retuning من إنه يحصل فوراً. بيضمن إن أي pending retune يتأخر لحد ما `sdio_retune_release` يتنادى. مفيد لـ time-critical sequences.

- **Key detail**: Increments hold counter — مش nested safe بشكل تلقائي، الـ driver مسؤول عن الـ balance.

---

### `sdio_retune_release`

```c
extern void sdio_retune_release(struct sdio_func *func);
```

يسمح للـ retuning يحصل تاني بعد `sdio_retune_hold_now`. لو كان فيه pending retune — ممكن يحصل فوراً.

---

## Category 11: Helper Macros

**الغرض:** تبسيط الوصول لـ state وبيانات الـ function.

---

### `sdio_func_present` / `sdio_func_set_present`

```c
#define sdio_func_present(f)     ((f)->state & SDIO_STATE_PRESENT)
#define sdio_func_set_present(f) ((f)->state |= SDIO_STATE_PRESENT)
```

الـ `SDIO_STATE_PRESENT` flag بيشير إن الـ function اتضافت لـ sysfs. الـ core بيستخدمهم — الـ drivers نادراً بيحتاجوهم.

---

### `sdio_func_id`

```c
#define sdio_func_id(f) (dev_name(&(f)->dev))
```

بيرجع الـ string name من الـ device (مثل `"mmc1:0001:1"`). مفيد للـ logging.

---

### `sdio_get_drvdata` / `sdio_set_drvdata`

```c
#define sdio_get_drvdata(f)    dev_get_drvdata(&(f)->dev)
#define sdio_set_drvdata(f, d) dev_set_drvdata(&(f)->dev, d)
```

Standard driver private data get/set. الـ driver بيحفظ `struct my_priv*` في `probe` وبيجيبها في كل callback.

```c
/* probe */
struct my_priv *priv = kzalloc(sizeof(*priv), GFP_KERNEL);
sdio_set_drvdata(func, priv);

/* any callback */
struct my_priv *priv = sdio_get_drvdata(func);
```

---

### `dev_to_sdio_func`

```c
#define dev_to_sdio_func(d) container_of(d, struct sdio_func, dev)
```

يحوّل `struct device*` generic لـ `struct sdio_func*`. بيستخدمه الـ bus code والـ drivers اللي بتشتغل مع `struct device` مباشرة (مثل sysfs attribute callbacks).

---

### `SDIO_DEVICE` / `SDIO_DEVICE_CLASS`

```c
#define SDIO_DEVICE(vend, dev) \
    .class = SDIO_ANY_ID, \
    .vendor = (vend), .device = (dev)

#define SDIO_DEVICE_CLASS(dev_class) \
    .class = (dev_class), \
    .vendor = SDIO_ANY_ID, .device = SDIO_ANY_ID
```

بيتستخدموا في `id_table` initialization:

```c
static const struct sdio_device_id my_sdio_ids[] = {
    { SDIO_DEVICE(0x0271, 0x0301) },  /* match by vendor+device */
    { SDIO_DEVICE_CLASS(SDIO_CLASS_BT_A) }, /* match any Bluetooth type A */
    { /* end sentinel */ },
};
MODULE_DEVICE_TABLE(sdio, my_sdio_ids);
```

---

## ملاحظات عامة مهمة

| قاعدة | التفاصيل |
|---|---|
| **Host lock إجباري** | كل I/O operations (قراءة/كتابة/enable/IRQ) تحتاج `sdio_claim_host` أولاً |
| **IRQ handler context** | بيشتغل في kernel thread (`ksdioirqd`) — مش hard IRQ — ممكن تعمل sleeping I/O |
| **DMA buffers** | الـ buffers اللي بتتمرر لـ `memcpy_fromio`/`toio` لازم DMA-safe — استخدم `sdio_align_size` |
| **Function 0 write** | محدود بـ addresses معينة في CCCR — محاولة write على address تاني = `-EINVAL` |
| **PM sequence** | `get_host_pm_caps` أولاً، فـ `set_host_pm_flags` — دايماً في `suspend` callback |
| **Retune balance** | كل `hold_now` لازم `release` مقابلها — وكل `crc_disable` لازم `crc_enable` |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs Entries

الـ MMC/SDIO subsystem بيعرض entries مهمة في debugfs:

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# شوف كل الـ MMC hosts
ls /sys/kernel/debug/mmc*/

# أهم الـ entries:
# /sys/kernel/debug/mmc0/ios         — حالة الـ I/O settings (clock, bus width, timing)
# /sys/kernel/debug/mmc0/clock_delay — تأخير الـ clock (بعض الـ controllers)
# /sys/kernel/debug/mmc0/caps        — الـ capabilities المدعومة
# /sys/kernel/debug/mmc0/caps2       — capabilities إضافية
```

```bash
# اقرأ الـ ios (مهم لـ SDIO debugging)
cat /sys/kernel/debug/mmc0/ios
```

مثال للـ output:
```
clock:          25000000 Hz
actual clock:   25000000 Hz
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       2 (push-pull)
chip select:    0 (don't care)
power mode:     2 (on)
bus width:      2 (4 bits)
timing spec:    2 (sd high-speed)
signal voltage: 0 (3.30 V)
driver type:    0 (driver type B)
```

**التفسير**: لو `clock` مختلف عن `actual clock` → ممكن يكون في مشكلة hardware في الـ clock source.

---

#### 2. sysfs Entries

الـ SDIO function device بيظهر في sysfs تحت bus الـ `sdio`:

```bash
# اعرض كل الـ SDIO devices
ls /sys/bus/sdio/devices/

# كل device اسمه على صيغة: mmc<host>:<card>:<func>
# مثال: mmc0:0001:1

# الـ attributes الأساسية لكل function:
ls /sys/bus/sdio/devices/mmc0:0001:1/
```

| Sysfs Attribute | المعنى |
|---|---|
| `vendor` | الـ vendor ID (hex) |
| `device` | الـ device ID (hex) |
| `class` | الـ standard interface class |
| `modalias` | الـ modalias لتحميل الـ driver التلقائي |
| `enable` | هل الـ function enabled |
| `driver/` | رابط للـ driver المرتبط |

```bash
# اقرأ vendor و device للـ function
cat /sys/bus/sdio/devices/mmc0:0001:1/vendor
cat /sys/bus/sdio/devices/mmc0:0001:1/device
cat /sys/bus/sdio/devices/mmc0:0001:1/class

# تحقق من الـ modalias (مهم لمشاكل الـ driver binding)
cat /sys/bus/sdio/devices/mmc0:0001:1/modalias
# output: sdio:c07v0271d0301

# شوف الـ block size الحالي
cat /sys/bus/sdio/devices/mmc0:0001:1/block_size  # لو موجود
```

---

#### 3. ftrace — Tracepoints و Events

```bash
# شوف الـ MMC events المتاحة
ls /sys/kernel/debug/tracing/events/mmc/

# أهم الـ events للـ SDIO:
# mmc_request_start     — بداية كل request
# mmc_request_done      — نهاية كل request
# mmc_cmd_*             — تفاصيل الـ commands
```

```bash
# فعّل الـ MMC tracing كامل
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable

# أو بشكل انتقائي للـ requests بس
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_request_start/enable
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_request_done/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل العملية اللي بتعمل المشكلة هنا...

# وقف واقرأ الـ trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

مثال للـ output:
```
kworker/0:2-45    [000] ....  123.456789: mmc_request_start: mmc0: start struct mmc_request[...]
kworker/0:2-45    [000] ....  123.456800: mmc_request_done:  mmc0: end struct mmc_request[...] err=0
```

**التفسير**: لو `err` مش صفر → في مشكلة في الـ I/O. لو الـ request مبتخلصش → deadlock محتمل في `sdio_claim_host`.

```bash
# trace function-level للـ SDIO core
echo sdio_* > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. printk / Dynamic Debug

```bash
# فعّل dynamic debug لكل الـ MMC/SDIO subsystem
echo 'module mmc_core +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mmc/core/sdio*.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mmc/core/sdio_io.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mmc/core/sdio_irq.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mmc/core/sdio_bus.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug entries الفعّالة
cat /sys/kernel/debug/dynamic_debug/control | grep sdio

# فعّل لـ driver بعينه (مثلاً driver wifi على SDIO)
echo 'module brcmfmac +p' > /sys/kernel/debug/dynamic_debug/control
```

```bash
# رفع مستوى الـ printk للـ MMC في kernel cmdline:
# mmc_core.debug=1  (لو الـ parameter موجود)

# أو عبر sysfs (لو الـ driver بيدعمها)
echo 1 > /sys/module/mmc_core/parameters/use_spi_crc  # مثال parameter
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_MMC_DEBUG` | يفعّل debug messages في الـ MMC core |
| `CONFIG_DYNAMIC_DEBUG` | يتيح dynamic debug control |
| `CONFIG_FTRACE` | يتيح function tracing |
| `CONFIG_MMC_UNSAFE_RESUME` | لتشخيص مشاكل الـ suspend/resume |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في `sdio_claim_host` |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف spinlock misuse |
| `CONFIG_KASAN` | يكتشف memory corruption في الـ tmpbuf وغيره |
| `CONFIG_UBSAN` | يكتشف undefined behavior |
| `CONFIG_MMC_SDHCI_OF_ARASAN` | (مثال) debug للـ controller specific |

```bash
# تحقق إن CONFIG_MMC_DEBUG مفعّل
zcat /proc/config.gz | grep MMC_DEBUG
# أو
cat /boot/config-$(uname -r) | grep MMC_DEBUG
```

---

#### 6. Common Error Messages → المعنى → الحل

| Error Message / Log | المعنى | الحل |
|---|---|---|
| `mmc0: error -110 whilst initialising SDIO card` | timeout أثناء الـ initialization | افحص الـ power supply والـ clock. جرب تخفيض الـ clock speed |
| `sdio_enable_func: error -110` | الـ function مش بترد على الـ enable command | تحقق من الـ enable_timeout في `sdio_func`. ممكن hardware مش شغال |
| `mmc0: Got data interrupt 0x00000002 even though no data operation was in progress` | spurious interrupt | مشكلة في الـ interrupt routing أو hardware glitch |
| `mmc0: Timeout waiting for hardware interrupt` | الـ hardware controller مش بيعمل interrupt | افحص الـ IRQ connections وتأكد الـ controller clock شغال |
| `mmc0: card does not support CMD52` | الـ SDIO card مش standard أو في مشكلة تهيئة | تأكد الـ card فعلاً SDIO وليس SD memory |
| `mmc0: card is not responding to CMD5` | الـ card مش SDIO أو في مشكلة power | افحص الـ VDD وتسلسل الـ power-up |
| `sdio_claim_irq: Function 1 already has an IRQ handler` | double registration للـ IRQ | تأكد الـ driver مش بيعمل `sdio_claim_irq` مرتين |
| `mmc0: CRC error` | مشكلة في سلامة البيانات | مشكلة layout أو interference. قلل الـ clock |
| `request timed out` | الـ MMC request استغرق وقت أطول من المتوقع | deadlock محتمل في `sdio_claim_host` أو hardware freezes |

---

#### 7. Strategic Points لـ dump_stack() / WARN_ON()

```c
/* في sdio_enable_func() — بعد فشل enable */
if (ret) {
    WARN_ON(ret == -ETIMEDOUT);  /* detect timeout patterns */
    dev_err(&func->dev, "enable failed: %d\n", ret);
}

/* تحقق من state قبل I/O operations */
WARN_ON(!sdio_func_present(func));

/* في sdio_claim_irq() — تحقق من عدم وجود handler مسبق */
WARN_ON(func->irq_handler != NULL);

/* في sdio_set_block_size() — تحقق من الحدود */
WARN_ON(blksz > func->max_blksize);

/* في cleanup paths — تحقق إن الـ IRQ released */
WARN_ON_ONCE(func->irq_handler);

/* في probe/remove لضمان التماثل */
static int mydrv_probe(struct sdio_func *func, ...) {
    sdio_claim_host(func);
    ret = sdio_enable_func(func);
    WARN_ON(ret);  /* يطبع stack trace لو فشل */
    sdio_release_host(func);
}
```

**نقاط استراتيجية للـ dump_stack():**
- بعد `sdio_claim_host()` مباشرة لو الـ timeout ظهر (يساعد تحديد مين ماسك الـ host)
- في بداية الـ IRQ handler لو في spurious interrupts
- في أي code path بيوصل لـ `sdio_disable_func()` بدون enable مسبق

---

### Hardware Level

#### 1. التحقق من توافق الـ Hardware State مع الـ Kernel State

```bash
# قارن الـ clock الـ software بالـ actual hardware clock
cat /sys/kernel/debug/mmc0/ios | grep -E "clock|actual"

# تحقق من الـ bus width المتفق عليه
cat /sys/kernel/debug/mmc0/ios | grep "bus width"
# يجب يتطابق مع إعداد الـ hardware controller

# تحقق من حالة الـ function (enabled أو لا)
# عن طريق قراءة CCCR register Function Enable (FEN) — Function 0, address 0x02
# يمكن عمله عبر أدوات مثل mmc-utils
mmc sdio read-byte /dev/mmcblk0 0 0x02
# Bit 1 = Function 1 enabled
```

الـ CCCR (Card Common Control Register) layout الأساسي:

```
Addr 0x00 — CCCR/SDIO Revision
Addr 0x02 — I/O Enable (FEN bits)
Addr 0x03 — I/O Ready (RDY bits)
Addr 0x04 — INT Enable
Addr 0x05 — INT Pending
Addr 0x06 — I/O Abort
Addr 0x07 — Bus Interface Control (bus width)
Addr 0x08 — Card Capability
```

---

#### 2. Register Dump Techniques

```bash
# استخدم mmc-utils لقراءة الـ SDIO registers مباشرة
# تثبيت: apt install mmc-utils

# اقرأ CCCR register (Function 0)
mmc sdio read-byte /dev/mmcblk0 0 0x00   # CCCR revision
mmc sdio read-byte /dev/mmcblk0 0 0x02   # I/O Enable
mmc sdio read-byte /dev/mmcblk0 0 0x03   # I/O Ready
mmc sdio read-byte /dev/mmcblk0 0 0x04   # INT Enable

# اقرأ FBR (Function Basic Register) للـ Function 1
# FBR للـ function N يبدأ من 0x100 * N
mmc sdio read-byte /dev/mmcblk0 0 0x100  # Function 1 Class
mmc sdio read-byte /dev/mmcblk0 0 0x110  # Function 1 Block Size Low
mmc sdio read-byte /dev/mmcblk0 0 0x111  # Function 1 Block Size High
```

```c
/* في الـ kernel driver — dump registers في probe() */
static void mydrv_dump_regs(struct sdio_func *func)
{
    int err;
    u8 val;

    sdio_claim_host(func);

    /* read CCCR I/O Enable */
    val = sdio_f0_readb(func, SDIO_CCCR_IOEx, &err);
    pr_info("CCCR IOEx=0x%02x err=%d\n", val, err);

    /* read device-specific registers */
    val = sdio_readb(func, MY_DEV_STATUS_REG, &err);
    pr_info("Device status=0x%02x err=%d\n", val, err);

    sdio_release_host(func);
}
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**الـ signals المهمة للـ monitoring:**

```
SDIO Interface Pins:
┌─────────────┬───────────────────────────────────┐
│ Signal      │ ما تراقبه                         │
├─────────────┼───────────────────────────────────┤
│ CLK         │ Frequency, Jitter, Duty cycle      │
│ CMD         │ Command/Response timing             │
│ DAT0-DAT3   │ Data integrity, Bus width usage    │
│ VDD         │ Voltage levels & noise             │
│ IRQ (DAT1)  │ Interrupt assertion patterns        │
└─────────────┴───────────────────────────────────┘
```

**Logic Analyzer Setup:**
- **Sample rate**: 10x الـ clock frequency على الأقل (مثلاً 250 MSa/s لـ 25MHz SDIO)
- **Trigger**: على الـ CMD line لالتقاط بداية الـ transactions
- **Protocol decoder**: فعّل الـ SDIO/MMC decoder لو متاح (Sigrok/PulseView بيدعمه)

```bash
# استخدم sigrok مع logic analyzer
sigrok-cli -d fx2lafw --config samplerate=24000000 \
  --channels CLK,CMD,DAT0,DAT1,DAT2,DAT3 \
  --protocol-decoders sdcard_sd \
  --output-file sdio_capture.sr
```

**Oscilloscope Tips:**
- راقب الـ VDD للـ SDIO card — لازم يكون stable (3.3V أو 1.8V حسب الـ signaling voltage)
- افحص الـ CLK edges — rise/fall time لازم يكون أقل من 20% من الـ period
- راقب الـ CMD line أثناء الـ initialization — شوف CMD5 response

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة Hardware | Pattern في الـ Kernel Log |
|---|---|
| **ضعف الـ Power Supply** | `mmc0: error -110 whilst initialising` متكرر بعد الـ boot |
| **مشكلة الـ Clock** | `mmc0: Timeout waiting for hardware interrupt` + الـ actual clock = 0 |
| **خط CMD مقطوع/ضعيف** | `mmc0: card is not responding to CMD52` |
| **CRC errors في الـ data** | `mmc0: CRC error` متكرر خصوصاً بسرعات عالية |
| **مشكلة Pull-up Resistors** | الـ card يُكشف ثم يختفي `mmc0: card removed` |
| **Interrupt line مشكلة** | الـ driver يعمل لكن بدون استجابة لـ events (polling فقط يشتغل) |
| **Voltage switching فاشل** | `mmc0: Signal voltage switch failed` عند محاولة 1.8V |
| **الـ card مش محمولة صح** | متقطع `mmc0: card is present, but controller reports it is not` |

---

### Practical Commands

#### الأوامر الجاهزة للنسخ

```bash
#!/bin/bash
# === SDIO Full Debug Script ===

echo "=== 1. SDIO Devices ==="
ls -la /sys/bus/sdio/devices/

echo "=== 2. MMC Host IOS ==="
for host in /sys/kernel/debug/mmc*/ios; do
    echo "--- $host ---"
    cat "$host"
done

echo "=== 3. SDIO Device Details ==="
for dev in /sys/bus/sdio/devices/*/; do
    echo "--- $dev ---"
    for attr in vendor device class modalias; do
        printf "%-12s: %s\n" "$attr" "$(cat ${dev}${attr} 2>/dev/null)"
    done
done

echo "=== 4. Kernel Messages ==="
dmesg | grep -iE 'mmc|sdio' | tail -50

echo "=== 5. Loaded SDIO Drivers ==="
ls /sys/bus/sdio/drivers/
```

```bash
# === Enable Full SDIO Tracing ===
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace                          # clear buffer
echo 1 > events/mmc/enable            # enable all MMC events
echo 8192 > buffer_size_kb            # increase buffer
echo 1 > tracing_on

echo "Reproduce the issue now..."
sleep 10  # أو انتظر حدوث المشكلة

echo 0 > tracing_on
cat trace | grep -E 'mmc|sdio' | head -100
```

```bash
# === Dynamic Debug Activation ===
# فعّل logging لكل SDIO core files
for f in sdio_bus sdio_cis sdio_io sdio_irq sdio_ops sdio; do
    echo "file drivers/mmc/core/${f}.c +pflmt" \
        > /sys/kernel/debug/dynamic_debug/control 2>/dev/null && \
        echo "Enabled: ${f}.c" || echo "Not found: ${f}.c"
done

# شاهد الـ kernel log بشكل مباشر
dmesg -w | grep -i sdio
```

```bash
# === SDIO IRQ Debug ===
# تحقق من الـ interrupt statistics
cat /proc/interrupts | grep -i mmc
cat /proc/interrupts | grep -i sdio

# شوف الـ interrupt thread لو موجود
ps aux | grep sdio_irq
```

```bash
# === Block Size Debug ===
# تحقق من الـ block size المتفق عليه
for dev in /sys/bus/sdio/devices/*/; do
    echo -n "$dev -> max_blksize: "
    # من الـ sysfs لو متاح
    cat "${dev}block_size" 2>/dev/null || echo "N/A"
done

# عبر dmesg أثناء الـ probe
dmesg | grep -i "block size"
```

```bash
# === Check SDIO PM Capabilities ===
# تحقق من دعم الـ power management للـ SDIO
cat /sys/bus/sdio/devices/mmc0:0001:1/power/wakeup
cat /sys/bus/sdio/devices/mmc0:0001:1/power/runtime_status

# فعّل runtime PM debug
echo 1 > /sys/bus/sdio/devices/mmc0:0001:1/power/pm_qos_resume_latency
```

```bash
# === Quick Sanity Check Script ===
echo "=== SDIO Sanity Check ==="

# 1. هل في SDIO devices؟
count=$(ls /sys/bus/sdio/devices/ 2>/dev/null | wc -l)
echo "SDIO functions found: $count"

# 2. هل الـ drivers مرتبطة؟
echo "Bound drivers:"
for dev in /sys/bus/sdio/devices/*/; do
    driver=$(readlink ${dev}driver 2>/dev/null | xargs basename 2>/dev/null)
    echo "  $(basename $dev) -> ${driver:-UNBOUND}"
done

# 3. آخر errors في الـ log
echo "Recent MMC errors:"
dmesg | grep -iE 'mmc.*error|sdio.*fail|mmc.*timeout' | tail -10

# 4. الـ clock الحالي
echo "Current MMC clocks:"
grep "actual clock" /sys/kernel/debug/mmc*/ios 2>/dev/null
```

#### مثال Output وتفسيره

```
# output من script الـ sanity check:
SDIO functions found: 2
Bound drivers:
  mmc0:0001:1 -> brcmfmac          ← Function 1 مرتبطة بـ WiFi driver ✓
  mmc0:0001:2 -> UNBOUND            ← Function 2 مش مرتبطة! ← مشكلة
Recent MMC errors:
  mmc0: error -84 whilst initialising SDIO card   ← -EILSEQ = CRC error
Current MMC clocks:
  actual clock: 0 Hz                ← الـ clock واقف! مشكلة hardware
```

**التفسير والخطوات**:
1. Function 2 `UNBOUND` → ممكن ما فيش driver مثبّت لها أو الـ modalias مش مطابق — تحقق بـ `modprobe` أو `lsmod`
2. `-EILSEQ` CRC errors → قلل الـ clock speed أو افحص الـ PCB routing
3. `actual clock: 0` → الـ controller clock source مش شغال — افحص الـ device tree / clock gating

## Phase 6: سيناريوهات من الحياة العملية

---

### سيناريو 1: Wi-Fi SDIO مش بيشتغل على Industrial Gateway بـ RK3562

**العنوان:** `probe` بيفشل صامت على RTL8189 SDIO Wi-Fi

**السياق:**
بورد industrial gateway بيشغّل Buildroot على RK3562. الـ Wi-Fi chip هو RTL8189FS متوصّل على SDIO. المنتج وصل للـ factory و الـ Wi-Fi مش بيظهر خالص.

**المشكلة:**
الـ driver بـ `dmesg` مش بيظهر أي error، لكن `ip link` مش بيرجّع أي wireless interface. الـ `lsmod` بيأكّد إن الـ module اتحمل.

**التحليل:**
بنراجع الـ `sdio_driver` struct في الـ header:

```c
struct sdio_driver {
    char *name;
    const struct sdio_device_id *id_table;  /* matching table */

    int (*probe)(struct sdio_func *, const struct sdio_device_id *);
    void (*remove)(struct sdio_func *);
    ...
};
```

الـ kernel بيطابق الـ vendor/device IDs من `id_table` مع اللي بيجي من الـ card. لو الـ `id_table` فيها vendor ID غلط، `probe` مش هيتسمّي أصلاً — من غير error.

بنتحقق من الـ DT:
```bash
cat /sys/bus/sdio/devices/*/uevent
# SDIO_ID=024C:8179   ← RTL8179, مش 8189!
```

الـ driver `id_table`:
```c
static const struct sdio_device_id rtl8189_ids[] = {
    { SDIO_DEVICE(0x024C, 0x8189) },  /* RTL8189FS */
    { }
};
```

الـ chip على البورد فعلاً RTL8179، مش 8189. الـ vendor غلط في الـ BOM.

**الحل:**
```bash
# تأكيد الـ ID الفعلي
cat /sys/bus/sdio/devices/mmc1\:0001\:1/vendor
# 0x024c
cat /sys/bus/sdio/devices/mmc1\:0001\:1/device
# 0x8179
```

إضافة الـ ID الصح في الـ `id_table`:
```c
static const struct sdio_device_id rtl8189_ids[] = {
    { SDIO_DEVICE(0x024C, 0x8189) },
    { SDIO_DEVICE(0x024C, 0x8179) },  /* add RTL8179 variant */
    { }
};
MODULE_DEVICE_TABLE(sdio, rtl8189_ids);
```

**الدرس المستفاد:**
لازم دايماً تتحقق من `vendor`/`device` في `sdio_func` قبل ما توصل للـ factory. `SDIO_DEVICE()` macro بيحدد الـ matching بدقة — أي اختلاف في الـ ID بيخلّي الـ `probe` متستدعيش خالص.

---

### سيناريو 2: Kernel Panic عشان DMA على STM32MP1 IoT Sensor

**العنوان:** `tmpbuf` مش DMA-safe بيودّي Kernel Panic

**السياق:**
IoT sensor node بيشغّل sensor hub Wi-Fi chip (CYW43012) على SDIO متوصّل بـ STM32MP1. الجهاز بيـcrash بعد تشغيل Wi-Fi بدقائق في بيئة stress test.

**المشكلة:**
```
kernel BUG at mm/slub.c:3954!
Unable to handle kernel NULL pointer dereference
PC is at sdio_io_rw_ext_helper+0x...
```

**التحليل:**
بنبص على `sdio_func`:
```c
struct sdio_func {
    ...
    u8  *tmpbuf;  /* DMA:able scratch buffer */
    ...
};
```

الـ comment صريح: `tmpbuf` لازم يكون DMA-able. الـ driver كان بيعمل:
```c
/* WRONG: stack buffer, not DMA-safe */
u8 local_buf[4];
func->tmpbuf = local_buf;  /* BUG! stack address */
```

على STM32MP1 مع IOMMU، الـ DMA بيحاول يوصل لـ stack address — مش valid في الـ IOMMU mapping — crash.

الـ sdio core بيستخدم `tmpbuf` في `sdio_readb`/`sdio_writeb` كـ bounce buffer للـ DMA transfers:
```c
extern u8 sdio_readb(struct sdio_func *func, unsigned int addr, int *err_ret);
/* internally uses func->tmpbuf for single-byte DMA */
```

**الحل:**
```c
/* In probe: allocate proper DMA-coherent buffer */
func->tmpbuf = kmalloc(4, GFP_KERNEL | GFP_DMA);
if (!func->tmpbuf)
    return -ENOMEM;

/* In remove: free it */
kfree(func->tmpbuf);
func->tmpbuf = NULL;
```

ملحوظة: الـ MMC core عادةً بيعمل ده أوتوماتيك، لكن لو الـ driver اتلاعب في `tmpbuf` يدوياً — لازم DMA-safe memory.

**الدرس المستفاد:**
الـ comment في `sdio_func.h` ("DMA:able scratch buffer") مش تزويقي — ده contract. أي حاجة بتتكتب على `tmpbuf` لازم تكون من `kmalloc(GFP_DMA)` أو `dma_alloc_coherent()`.

---

### سيناريو 3: Android TV Box بـ Allwinner H616 — SDIO Wi-Fi بيصحى من Sleep غلط

**العنوان:** `sdio_set_host_pm_flags` مش بيتسمّي صح — Wi-Fi بيموت بعد suspend

**السياق:**
Android TV box بـ Allwinner H616 و AP6256 Wi-Fi (BCM43456 on SDIO). المستخدمين بيشتكوا إن Wi-Fi بيقطع بعد ما الـ TV يدخل standby وبعدين يصحى.

**المشكلة:**
الـ Wi-Fi بيشتغل تمام قبل الـ suspend. بعد الـ resume، الـ interface موجود بس مش بيقدر يـconnect. Reboot بيحل المشكلة.

**التحليل:**
بنفحص الـ PM flags API:
```c
extern mmc_pm_flag_t sdio_get_host_pm_caps(struct sdio_func *func);
extern int sdio_set_host_pm_flags(struct sdio_func *func, mmc_pm_flag_t flags);
```

الـ driver بيحتاج يطلب `MMC_PM_KEEP_POWER` عشان الـ Wi-Fi chip يفضل powered during suspend ويقدر يعمل wake-on-WLAN.

بنتحقق من الـ driver:
```c
static int bcm_wifi_suspend(struct device *dev)
{
    struct sdio_func *func = dev_to_sdio_func(dev);  /* macro from header */
    /* BUG: forgot to set PM flags */
    return 0;
}
```

الـ `dev_to_sdio_func()` macro موجود في الـ header:
```c
#define dev_to_sdio_func(d) container_of(d, struct sdio_func, dev)
```

لكن الـ flags مش اتبعتش. الـ host controller بيقطع الـ power عن SDIO أثناء suspend.

**الحل:**
```c
static int bcm_wifi_suspend(struct device *dev)
{
    struct sdio_func *func = dev_to_sdio_func(dev);
    mmc_pm_flag_t caps;

    /* check what host supports */
    caps = sdio_get_host_pm_caps(func);
    if (!(caps & MMC_PM_KEEP_POWER)) {
        dev_err(&func->dev, "host can't keep power during suspend\n");
        return -EINVAL;
    }

    /* request power kept during suspend */
    return sdio_set_host_pm_flags(func, MMC_PM_KEEP_POWER);
}
```

كمان في الـ DT لازم:
```dts
&sdio {
    keep-power-in-suspend;
    wakeup-source;
};
```

**الدرس المستفاد:**
`sdio_get_host_pm_caps()` و `sdio_set_host_pm_flags()` اتصمموا عشان يتفاوض الـ driver مع الـ host. لازم تتحقق من الـ caps الأول قبل ما تطلب أي flag — مش كل host controllers بتدعم `KEEP_POWER`.

---

### سيناريو 4: i.MX8 Automotive ECU — IRQ Storm بيعطّل الـ CAN Bus

**العنوان:** `sdio_claim_irq` مع handler غلط بيودّي IRQ storm

**السياق:**
Automotive ECU بـ i.MX8QM بيشغّل SDIO Wi-Fi (NXP IW612) للـ V2X communication. في production testing، الـ CAN bus بيتوقف عن الشغل بعد 30 ثانية من تشغيل Wi-Fi.

**المشكلة:**
```bash
dmesg | grep -i irq
# irq 123: nobody cared
# handlers: [<sdio_card_irq>]
# Disabling IRQ #123
```

الـ CPU utilization بترتفع لـ 100% لحظة قبل ما الـ IRQ يتـdisable.

**التحليل:**
الـ header بيعرّف:
```c
typedef void (sdio_irq_handler_t)(struct sdio_func *);

extern int sdio_claim_irq(struct sdio_func *func, sdio_irq_handler_t *handler);
extern int sdio_release_irq(struct sdio_func *func);
```

الـ `irq_handler` بيتخزن في `sdio_func`:
```c
struct sdio_func {
    ...
    sdio_irq_handler_t *irq_handler;  /* IRQ callback */
    ...
};
```

الـ driver الـ handler بتاعه:
```c
static void iw612_irq_handler(struct sdio_func *func)
{
    /* BUG: reads interrupt status but never clears it! */
    u8 status = sdio_readb(func, REG_INT_STATUS, NULL);
    if (status & INT_DATA_READY)
        queue_work(priv->wq, &priv->rx_work);
    /* missing: sdio_writeb to clear the interrupt */
}
```

الـ SDIO interrupt line بيفضل asserted لأن السجل ما اتمسحش. الـ kernel بيدعي الـ handler تاني وتاني في loop — IRQ storm.

**الحل:**
```c
static void iw612_irq_handler(struct sdio_func *func)
{
    int err;
    u8 status;

    sdio_claim_host(func);  /* required before any I/O */

    /* read then CLEAR the interrupt register */
    status = sdio_readb(func, REG_INT_STATUS, &err);
    if (!err && (status & INT_DATA_READY)) {
        /* clear by writing 1 to the bit */
        sdio_writeb(func, INT_DATA_READY, REG_INT_CLEAR, &err);
        queue_work(priv->wq, &priv->rx_work);
    }

    sdio_release_host(func);
}
```

**الدرس المستفاد:**
الـ `sdio_irq_handler_t` callback لازم دايماً تـclear الـ interrupt source في الـ chip قبل ما ترجع. فشل في ده بيودّي IRQ storm بيأكل الـ CPU ويأثر على subsystems تانية زي CAN.

---

### سيناريو 5: AM62x Custom Board Bring-Up — SDIO Function مش بتتعرّف

**العنوان:** `enable_timeout` و Block Size غلط بيمنعوا الـ function initialization

**السياق:**
Custom board جديد بـ TI AM62x بيشغّل WL1837MOD (Texas Instruments Wi-Fi/BT SDIO combo). أول bring-up، الـ Wi-Fi مش بيظهر، بس الـ SDIO card نفسها بتتعرّف.

**المشكلة:**
```bash
dmesg
# mmc1: new ultra high speed SDR104 SDIO card at address 0001
# mmc1: card 0001: sdio_enable_func failed: -110  ← ETIMEDOUT
```

**التحليل:**
بنبص على الـ `sdio_func` struct:
```c
struct sdio_func {
    ...
    unsigned max_blksize;    /* maximum block size */
    unsigned cur_blksize;    /* current block size */
    unsigned enable_timeout; /* max enable timeout in msec */
    ...
};
```

الـ `sdio_enable_func()` بتستخدم `enable_timeout` عشان تستنّى الـ chip يجهّز نفسه. الـ WL1837 بيحتاج حتى 1000ms للـ enable على بعض الـ configurations.

```c
extern int sdio_enable_func(struct sdio_func *func);
```

بنتحقق من default timeout في الـ core:
```bash
# default enable_timeout is 100ms — too short for WL1837
```

كمان `cur_blksize` اتضبطت على 512 بس الـ chip firmware بيتوقع 256:

```c
extern int sdio_set_block_size(struct sdio_func *func, unsigned blksz);
```

**الحل:**
في الـ `probe` function بتاع الـ driver:
```c
static int wl18xx_probe(struct sdio_func *func,
                        const struct sdio_device_id *id)
{
    int ret;

    /* Increase enable timeout for WL1837 */
    func->enable_timeout = 1000;  /* 1 second */

    sdio_claim_host(func);

    ret = sdio_enable_func(func);
    if (ret) {
        dev_err(&func->dev, "enable failed: %d\n", ret);
        goto out;
    }

    /* Set correct block size */
    ret = sdio_set_block_size(func, 256);
    if (ret) {
        dev_err(&func->dev, "set block size failed: %d\n", ret);
        goto disable;
    }

    sdio_release_host(func);

    /* store driver data */
    sdio_set_drvdata(func, priv);  /* macro from header */
    return 0;

disable:
    sdio_disable_func(func);
out:
    sdio_release_host(func);
    return ret;
}
```

كمان نتحقق من الـ DT:
```dts
&sdhci1 {
    status = "okay";
    bus-width = <4>;
    cap-sd-highspeed;
    ti,otap-del-sel-sdr104 = <0x4>;
    /* make sure clk is stable before card init */
    cd-debounce-delay-ms = <200>;
};
```

**الدرس المستفاد:**
`enable_timeout` في `sdio_func` مش read-only — الـ driver ممكن يضبطه قبل ما يستدعي `sdio_enable_func()`. البورد bring-up لازم يراجع دايماً الـ chip datasheet للـ power-on timing requirements، وكمان `max_blksize`/`cur_blksize` لازم يتوافقوا مع الـ firmware expectations.
## Phase 7: مصادر ومراجع

---

### 1. مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [SDIO support coming](https://lwn.net/Articles/242744/) | إعلان Pierre Ossman عن دعم SDIO قادم لـ kernel 2.6.24 |
| [MMC updates](https://lwn.net/Articles/253888/) | تحديثات الـ MMC layer شاملة SDIO و SPI support |
| [StarFive's SDIO/eMMC driver support](https://lwn.net/Articles/923377/) | إضافة designware mobile storage host controller driver |
| [rtw88: Add SDIO support](https://lwn.net/Articles/926679/) | إضافة SDIO support لـ Realtek rtw88 wireless driver |
| [Add support for Meson MX "SDIO" MMC controller](https://lwn.net/Articles/734836/) | دعم Meson MX controller |
| [mmc: core: add SD4.0 support](https://lwn.net/Articles/642336/) | إضافة SD 4.0 support للـ MMC core |
| [1/4 MMC layer](https://lwn.net/Articles/82765/) | تاريخي: بنية الـ MMC layer الأولى |
| [Linux Device Drivers, Third Edition (LDD3)](https://lwn.net/Kernel/LDD3/) | النسخة الكاملة من LDD3 على LWN |

---

### 2. التوثيق الرسمي للـ kernel

كل المسارات دي relative لـ `Documentation/` في kernel source tree:

```
Documentation/driver-api/mmc/index.rst       # نقطة البداية للـ MMC/SD/SDIO subsystem
Documentation/driver-api/mmc/mmc-dev-attrs.rst  # block device attributes
Documentation/driver-api/mmc/mmc-dev-parts.rst  # device partitions
Documentation/driver-api/mmc/mmc-async-req.rst  # async request handling
Documentation/driver-api/mmc/mmc-tools.rst      # user-space tools
```

روابط online:

| المرجع | الرابط |
|--------|--------|
| MMC/SD/SDIO card support | [docs.kernel.org/driver-api/mmc/index.html](https://docs.kernel.org/driver-api/mmc/index.html) |
| Driver implementer's API guide | [static.lwn.net/kerneldoc/driver-api/index.html](https://static.lwn.net/kerneldoc/driver-api/index.html) |
| Devres - Managed Device Resource | [static.lwn.net/kerneldoc/driver-api/driver-model/devres.html](https://static.lwn.net/kerneldoc/driver-api/driver-model/devres.html) |
| SD and MMC Device Partitions | [kernel.org/doc/html/latest/driver-api/mmc/mmc-dev-parts.html](https://www.kernel.org/doc/html/latest/driver-api/mmc/mmc-dev-parts.html) |

---

### 3. ملفات المصدر الأساسية في kernel

```
include/linux/mmc/sdio_func.h     # struct sdio_func, struct sdio_driver — ملف الـ phase ده
include/linux/mmc/sdio.h          # SDIO register definitions
include/linux/mmc/sdio_ids.h      # vendor/device IDs
include/linux/mmc/card.h          # struct mmc_card
include/linux/mmc/host.h          # struct mmc_host

drivers/mmc/core/sdio.c           # core SDIO implementation
drivers/mmc/core/sdio_func.c      # sdio_func operations
drivers/mmc/core/sdio_irq.c       # IRQ handling
drivers/mmc/core/sdio_bus.c       # SDIO bus driver
drivers/mmc/core/sdio_io.c        # I/O operations (readb/writeb/etc.)
drivers/mmc/core/sdio_cis.c       # CIS tuple parsing
```

---

### 4. Kernel Commits المهمة

ابحث في `git log --oneline drivers/mmc/` أو على [git.kernel.org](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/drivers/mmc) عن:

| الموضوع | Search term |
|---------|-------------|
| Pierre Ossman's original SDIO implementation | `git log --author="Pierre Ossman" -- drivers/mmc/` |
| SDIO IRQ threading | `sdio_irq: implement threaded IRQ` |
| Power management hooks | `sdio: add PM suspend/resume` |
| rtw88 SDIO | `rtw88: Add SDIO support` |

---

### 5. كتب موصى بيها

| الكتاب | الفصل المهم | الملاحظة |
|--------|------------|---------|
| **Linux Device Drivers, 3rd Ed.** (Corbet, Rubini, Kroah-Hartman) | Ch. 14: The Linux Device Model | أساسي لفهم `struct device` و `struct bus_type` اللي بيبني عليهم SDIO |
| **Linux Kernel Development, 3rd Ed.** (Robert Love) | Ch. 17: Devices and Modules | يشرح device driver model بشكل مبسط |
| **Embedded Linux Primer, 2nd Ed.** (Christopher Hallinan) | Ch. 15: Embedded Drivers | أمثلة عملية على SDIO في embedded systems |
| **Understanding the Linux Kernel** (Bovet & Cesati) | Ch. 13: I/O Architecture | خلفية hardware I/O |

---

### 6. eLinux.org

| الصفحة | الرابط |
|--------|--------|
| Libertas SDIO driver | [elinux.org/Libertas_SDIO](https://elinux.org/Libertas_SDIO) |
| SDIO KS7010 testing | [elinux.org/Tests:SDIO-KS7010](https://elinux.org/Tests:SDIO-KS7010) |
| SDIO with UHS testing | [elinux.org/Tests:SDIO-with-UHS](https://elinux.org/Tests:SDIO-with-UHS) |

---

### 7. kernelnewbies.org

| الصفحة | الملاحظة |
|--------|---------|
| [Linux 2.6.24 changes](https://kernelnewbies.org/Linux_2_6_24) | الـ release اللي دخل فيه SDIO support بشكل رسمي، بيشرح الـ MMC layer refactoring |

---

### 8. Search Terms للبحث عن مزيد

```
# في kernel mailing list (lore.kernel.org):
site:lore.kernel.org "sdio_func" driver
site:lore.kernel.org "sdio_register_driver"

# في git.kernel.org:
sdio_claim_host sdio_release_host
sdio_memcpy_fromio implementation
sdio_irq_handler_t

# مصطلحات تقنية مفيدة:
"SDIO CIS tuple parsing"
"SDIO function driver probe"
"mmc_card SDIO"
"sdio_f0 function 0"
"SDIO block mode vs byte mode"
"SDIO 1-bit 4-bit bus width"
"SDIO power management suspend resume"
"SDIO DMA buffer alignment"
```

---

### 9. مراجع المواصفات

| المواصفة | المصدر |
|----------|--------|
| SDIO Specification (Part E1) | [sdcard.org](https://www.sdcard.org/developers/sd-standard-overview/) — مدفوع، بس الـ summary مجاني |
| SD Physical Layer Simplified Specification | متاح مجانًا على sdcard.org |
| JEDEC eMMC Standard (JESD84) | [jedec.org](https://www.jedec.org/standards-documents/docs/jesd84-b51) |
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على `sdio_enable_func` — دي function مصدّرة بتتسمى كل ما driver SDIO بيحاول يفعّل function على كارت SDIO. ده بيخلينا نشوف أي device بيتفعّل ومين بيطلبه.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * sdio_enable_probe.c
 * Kprobe on sdio_enable_func() — logs vendor/device/func-num on every call.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/mmc/sdio_func.h>   /* struct sdio_func */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("SDIO Watcher");
MODULE_DESCRIPTION("Kprobe on sdio_enable_func — log SDIO function activations");

/* ---------------------------------------------------------------
 * pre-handler: runs just before sdio_enable_func() executes.
 * First argument (struct sdio_func *func) is in RDI on x86-64.
 * --------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Grab the first argument — pointer to sdio_func */
    struct sdio_func *func = (struct sdio_func *)regs->di;

    if (!func)
        return 0;

    /*
     * طباعة vendor ID و device ID ورقم الـ function.
     * sdio_func_id() بيرجع اسم الـ device زي "mmc1:0001:1".
     */
    pr_info("sdio_probe: sdio_enable_func called — id=%s vendor=0x%04x device=0x%04x func_num=%u blksz=%u\n",
            sdio_func_id(func),
            func->vendor,
            func->device,
            func->num,
            func->cur_blksize);

    return 0;  /* 0 = let the real function run normally */
}

/* ---------------------------------------------------------------
 * post-handler: runs after sdio_enable_func() returns.
 * بنستخدمه عشان نطبع إن الـ function اتفعّلت فعلاً.
 * --------------------------------------------------------------- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * Return value في RAX على x86-64.
     * 0 = success, negative = error.
     */
    long ret = (long)regs->ax;

    pr_info("sdio_probe: sdio_enable_func returned %ld (%s)\n",
            ret, ret == 0 ? "OK" : "FAIL");
}

/* ---------------------------------------------------------------
 * تعريف الـ kprobe struct — بنحدد اسم الـ symbol اللي هنعمل probe عليه
 * --------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name    = "sdio_enable_func",
    .pre_handler    = handler_pre,
    .post_handler   = handler_post,
};

/* ---------------------------------------------------------------
 * module_init — بيسجل الـ kprobe مع الـ kernel
 * --------------------------------------------------------------- */
static int __init sdio_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("sdio_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    /*
     * kp.addr بيكون populated بعد register_kprobe بعنوان الـ symbol الفعلي.
     */
    pr_info("sdio_probe: kprobe planted at %pS (%px)\n",
            kp.addr, kp.addr);
    return 0;
}

/* ---------------------------------------------------------------
 * module_exit — بيشيل الـ kprobe عشان منيش crash لو الـ module اتشال
 * --------------------------------------------------------------- */
static void __exit sdio_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("sdio_probe: kprobe removed from sdio_enable_func\n");
}

module_init(sdio_probe_init);
module_exit(sdio_probe_exit);
```

---

### Makefile

```makefile
obj-m := sdio_enable_probe.o

KDIR  ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل قسم

| القسم | الغرض |
|---|---|
| `#include <linux/mmc/sdio_func.h>` | عشان نوصل لـ `struct sdio_func` وكل حقولها زي `vendor`, `device`, `num` |
| `handler_pre` | بيشتغل قبل `sdio_enable_func` ويطبع بيانات الـ SDIO function — ده المكان المفيد لأن الـ argument لسه ماتغيرش |
| `regs->di` | على x86-64 أول argument بيتبعت في register `RDI`، فبنقرأ الـ pointer من هناك مباشرة |
| `handler_post` | بيطبع return value من `RAX` عشان نعرف إذا كان الـ enable نجح ولا لأ |
| `kp.symbol_name` | الـ kernel بيحوّل الاسم لعنوان تلقائياً عند `register_kprobe` |
| `module_init / module_exit` | تسجيل وإلغاء تسجيل الـ probe — لازم يتعملوا صح عشان الـ kernel ميcrashش عند rmmod |

---

### تشغيل وملاحظة

```bash
# بناء
make

# تحميل
sudo insmod sdio_enable_probe.ko

# مشاهدة اللوج (وصّل أي SDIO device أو حمّل driver WiFi)
sudo dmesg -w | grep sdio_probe

# إزالة
sudo rmmod sdio_enable_probe
```

مثال على output متوقع لما WiFi chip SDIO بيتفعّل:

```
sdio_probe: kprobe planted at sdio_enable_func+0x0/0x80 (ffffffffc0123456)
sdio_probe: sdio_enable_func called — id=mmc1:0001:1 vendor=0x02d0 device=0x4329 func_num=1 blksz=512
sdio_probe: sdio_enable_func returned 0 (OK)
```

**vendor=0x02d0** هو Broadcom — وده شائع في chips زي BCM4329 اللي بتكون SDIO WiFi.
