## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

**الـ** `include/linux/of_dma.h` هو header يربط بين نظامين أساسيين في Linux kernel:
- **Device Tree (OF = Open Firmware)**: وصف الـ hardware في ملف نصي (شجرة الأجهزة).
- **DMA Engine**: إطار العمل الذي يُدير نقل البيانات بين الأجهزة والذاكرة بدون تدخل الـ CPU.

### القسم الذي ينتمي إليه

**الـ** `DMA GENERIC OFFLOAD ENGINE SUBSYSTEM` — المُشرف عليه: `Vinod Koul <vkoul@kernel.org>`.

---

### القصة: لماذا يوجد هذا الملف؟

تخيّل أنك تبني هاتفاً مُدمجاً (SoC). داخل هذا الـ SoC يوجد:
- **UART** لاستقبال البيانات التسلسلية.
- **SPI** لقراءة بيانات من sensor.
- **متحكم DMA** قادر على نقل البيانات بين هذه الأجهزة والذاكرة تلقائياً دون إشغال الـ CPU.

المشكلة: كيف يعرف الـ UART driver أيّ **DMA channel** يستخدم من بين عشرات القنوات المتاحة؟

الحل التقليدي كان: **تشفير الأرقام مباشرة في الكود** (hardcoding) — وهذا كارثة لأن كل SoC مختلف.

الحل الحديث: **Device Tree** — ملف نصي يصف الـ hardware، ويقول فيه:

```
uart0: uart@12340000 {
    dmas = <&dma_controller 5>, <&dma_controller 6>;
    dma-names = "rx", "tx";
};
```

هذا يعني: الـ UART يستخدم القناة 5 للاستقبال و6 للإرسال من المتحكم `dma_controller`.

**الـ** `of_dma.h` يُعرّف الـ API الذي يجعل هذا "الترجمة من Device Tree إلى DMA channel حقيقي" ممكناً.

---

### الصورة الكبيرة

```
Device Tree (dts file)
        |
        | "uart0 needs DMA channel 5"
        v
  of_dma_request_slave_channel()      <-- الـ driver يطلب channel باسمه "rx"
        |
        v
  of_dma_find_controller()            <-- يبحث عن المتحكم المسجّل
        |
        v
  of_dma_xlate() / of_dma_simple_xlate()  <-- تُترجم رقم القناة إلى struct dma_chan
        |
        v
  DMA Engine (dmaengine subsystem)    <-- يُنفّذ النقل الفعلي
```

---

### Structs الأساسية

#### `struct of_dma`
يُمثّل **متحكم DMA واحد** كما وصفه الـ Device Tree:

```c
struct of_dma {
    struct list_head    of_dma_controllers; /* ربطه بقائمة المتحكمات المسجّلة */
    struct device_node  *of_node;           /* عقدة Device Tree لهذا المتحكم */
    struct dma_chan *(*of_dma_xlate)        /* دالة الترجمة: رقم -> channel */
                    (struct of_phandle_args *, struct of_dma *);
    void *(*of_dma_route_allocate)          /* للـ DMA routers فقط */
                    (struct of_phandle_args *, struct of_dma *);
    struct dma_router   *dma_router;        /* مؤشر للـ router إن وُجد */
    void                *of_dma_data;       /* بيانات خاصة بالمتحكم */
};
```

#### `struct of_dma_filter_info`
يُستخدم مع `of_dma_simple_xlate` — يحمل قدرات الـ DMA ودالة الفلترة للعثور على قناة مناسبة:

```c
struct of_dma_filter_info {
    dma_cap_mask_t  dma_cap;   /* ما الذي يستطيع هذا الـ DMA فعله؟ */
    dma_filter_fn   filter_fn; /* دالة لتحديد القناة المناسبة */
};
```

---

### الـ API الرئيسي

| الدالة | الغرض |
|---|---|
| `of_dma_controller_register()` | يُسجّل متحكم DMA في النظام |
| `of_dma_controller_free()` | يُلغي تسجيل متحكم DMA |
| `of_dma_router_register()` | يُسجّل DMA router (جهاز وسيط يُوزّع القنوات) |
| `of_dma_request_slave_channel()` | الـ driver يطلب قناة DMA بالاسم (مثل "rx") |
| `of_dma_simple_xlate()` | دالة ترجمة جاهزة للمتحكمات البسيطة |
| `of_dma_xlate_by_chan_id()` | ترجمة عبر رقم القناة مباشرة |

---

### الفرق بين Controller وRouter

- **DMA Controller**: يمتلك قنوات DMA مباشرة ويُنفّذ النقل.
- **DMA Router**: جهاز وسيط (مثل multiplexer) يُوزّع القنوات بين أجهزة متعددة قبل وصولها للمتحكم الحقيقي — شائع في SoCs من TI وFreeScale.

---

### ملفات يجب معرفتها

| الملف | الدور |
|---|---|
| `include/linux/of_dma.h` | **هذا الملف** — الـ API والـ structs |
| `drivers/dma/of-dma.c` | التنفيذ الفعلي لجميع الدوال |
| `include/linux/dmaengine.h` | تعريفات `dma_chan`، `dma_cap_mask_t`، `dma_filter_fn` |
| `include/linux/of.h` | تعريفات `device_node`، `of_phandle_args` |
| `drivers/dma/dmaengine.c` | إطار عمل الـ DMA Engine |
| `drivers/dma/*.c` | drivers متحكمات DMA الفعلية (pl08x, bcm2835, imx-sdma, ...) |
| `Documentation/devicetree/bindings/dma/` | كيفية وصف DMA في الـ Device Tree |

---

### ملاحظة تصميمية

الملف يستخدم نمط `#ifdef CONFIG_DMA_OF` لتوفير **stub functions** تُعيد `-ENODEV` عندما لا يكون الـ DMA OF مُفعّلاً في kernel config — هذا يمنع الـ drivers من الانهيار على أنظمة بدون Device Tree.
## Phase 2: شرح الـ OF-DMA Framework

### المشكلة التي يحلّها هذا الـ Subsystem

في أنظمة ARM المدمجة، كل SoC يحتوي على DMA controller واحد أو أكثر. كل controller يملك عدة channels. الـ driver لأي peripheral (UART، SPI، I2S...) يحتاج إلى DMA channel بمواصفات محددة.

**المشكلة المزدوجة:**

1. **مشكلة الـ Hardcoding:** قديماً كان الـ driver يحدد الـ channel برقمه الصريح في الكود — يعني كود UART يقول "خذ channel رقم 5 من DMA controller رقم 0". هذا يربط الـ driver بـ SoC بعينه ويكسر كل إمكانية لإعادة الاستخدام.

2. **مشكلة الـ Discovery:** لا يوجد آلية معيارية تتيح لأي driver أن يسأل: "أعطني DMA channel يدعم slave transfers ومتصل بهذا الـ peripheral" — دون معرفة مسبقة بتفاصيل الـ hardware.

الحل التقليدي كان `dma_request_channel()` مع `dma_filter_fn` — لكنه لا يزال يتطلب من الـ driver معرفة خصائص الـ platform، وهو أمر لا يناسب عالم Device Tree حيث الـ hardware description يجب أن تكون خارج الكود.

---

### الحل: OF-DMA كـ Translation Layer

**الـ OF-DMA framework** هو الطبقة التي تربط بين وصف الـ hardware في الـ Device Tree (`dmas` property) وبين الـ DMA engine API الفعلي (`struct dma_chan`).

الفكرة الجوهرية: بدل أن يعرف الـ driver كيف يطلب الـ channel، يقرأ الـ kernel الـ Device Tree ويستدعي دالة **xlate** (translation) مسجّلة من قِبَل DMA controller driver لتحويل الـ DT args إلى `dma_chan` فعلي.

---

### المثيل الواقعي: مكتب توزيع الموظفين

تخيّل شركة كبيرة فيها:
- **أقسام** (UART driver, SPI driver, I2S driver) — كل قسم يحتاج موظفاً بمواصفات.
- **وكالات توظيف** (DMA controllers) — كل وكالة تملك موظفين (DMA channels).
- **مدير الـ HR** (OF-DMA framework) — يستقبل طلبات التوظيف ويوجّهها للوكالة الصحيحة.
- **ملف الموظف في الـ DT** (Device Tree `dmas` property) — مكتوب فيه: "هذا القسم يحتاج موظفاً من وكالة رقم 2، دوره channel رقم 4".

| الـ analogy | الـ kernel concept |
|---|---|
| القسم الذي يطلب موظفاً | الـ slave driver (UART, SPI, ...) |
| ملف الطلب المكتوب في HR | `dmas = <&dma0 4>` في الـ DT |
| مدير الـ HR | `of_dma_request_slave_channel()` |
| سجل الوكالات المعتمدة | القائمة `of_dma_list` (linked list من `of_dma`) |
| عقد كل وكالة مع HR | `of_dma_controller_register()` |
| طريقة كل وكالة لتفسير الطلب | `of_dma_xlate` function pointer |
| الموظف الذي يُسلَّم للقسم | `struct dma_chan *` |

الـ mapping كامل ودقيق — مدير الـ HR لا يعرف تفاصيل كل وكالة، هو فقط يمرر الطلب ويأخذ الموظف. هذا بالضبط ما يفعله OF-DMA.

---

### الصورة الكبيرة: أين يقع OF-DMA في الـ Kernel؟

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    Slave Driver (e.g. UART)                  │
  │    of_dma_request_slave_channel(np, "tx")                   │
  └────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                     OF-DMA Framework                         │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │  of_dma_list  (global linked list of of_dma structs) │   │
  │  │                                                      │   │
  │  │  ┌─────────────┐   ┌─────────────┐   ┌───────────┐  │   │
  │  │  │  of_dma[0]  │──▶│  of_dma[1]  │──▶│ of_dma[2] │  │   │
  │  │  │  of_node=A  │   │  of_node=B  │   │ of_node=C │  │   │
  │  │  │  xlate=fn1  │   │  xlate=fn2  │   │ xlate=fn3 │  │   │
  │  │  └─────────────┘   └─────────────┘   └───────────┘  │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  1. ابحث في of_dma_list عن of_dma الذي of_node يطابق      │
  │     الـ phandle في الـ DT                                  │
  │  2. استدعِ of_dma->of_dma_xlate(dma_spec, ofdma)           │
  │  3. أعِد الـ dma_chan الناتج                               │
  └────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                DMA Engine API (dmaengine.h)                  │
  │                                                             │
  │   dma_request_channel() / dma_get_slave_channel()           │
  └────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
  ┌─────────────────────────────────────────────────────────────┐
  │              DMA Controller Driver (e.g. pl330, edma)       │
  │   يملك: struct dma_device + قائمة من struct dma_chan        │
  │   يُسجّل نفسه عبر: of_dma_controller_register()            │
  └─────────────────────────────────────────────────────────────┘
                           │
                           ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                       Hardware                              │
  │         DMA Controller (PL330, eDMA, XDMAC, ...)           │
  │         Channels 0..N connected to peripherals              │
  └─────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: struct of_dma

هذه هي الـ struct المحورية في الـ framework:

```c
struct of_dma {
    struct list_head    of_dma_controllers;  /* ربط في القائمة العالمية */
    struct device_node *of_node;             /* من أيّ DT node سُجِّل */

    /* دالة التحويل: DT args → dma_chan */
    struct dma_chan *(*of_dma_xlate)(struct of_phandle_args *, struct of_dma *);

    /* للـ DMA Router فقط */
    void *(*of_dma_route_allocate)(struct of_phandle_args *, struct of_dma *);
    struct dma_router *dma_router;

    void *of_dma_data; /* بيانات خاصة بالـ controller */
};
```

**كيف تترابط الـ structs:**

```
  struct of_dma
  ┌─────────────────────────────────┐
  │ of_dma_controllers ─────────────┼──▶ (linked list entry)
  │                                 │
  │ of_node ────────────────────────┼──▶ struct device_node
  │                                 │    ┌──────────────────┐
  │                                 │    │ name: "dma"      │
  │                                 │    │ phandle: 0x1a    │
  │                                 │    │ full_name: ...   │
  │                                 │    └──────────────────┘
  │                                 │
  │ of_dma_xlate() ─────────────────┼──▶ function pointer
  │    receives:                    │    (e.g. of_dma_simple_xlate)
  │      struct of_phandle_args     │
  │      ┌──────────────────────┐   │
  │      │ np: device_node *    │   │
  │      │ args_count: 1        │   │
  │      │ args[0]: 4  (ch id)  │   │
  │      └──────────────────────┘   │
  │    returns: struct dma_chan *    │
  │                                 │
  │ of_dma_data ────────────────────┼──▶ void * (platform-specific)
  └─────────────────────────────────┘
```

---

### struct of_dma_filter_info: جسر مع الـ Legacy API

```c
struct of_dma_filter_info {
    dma_cap_mask_t  dma_cap;   /* bitmap: ما الـ capabilities المطلوبة */
    dma_filter_fn   filter_fn; /* callback: هل هذا الـ channel مناسب؟ */
};
```

هذه الـ struct تُستخدم في `of_dma_simple_xlate()` لتحويل الـ DT args إلى طلب DMA عبر الـ legacy `dma_request_channel()` API. بمعنى آخر: جسر بين عالم الـ Device Tree وعالم الـ DMA engine القديم.

**الـ dma_cap_mask_t** (من `dmaengine.h`):
```c
typedef struct { DECLARE_BITMAP(bits, DMA_TX_TYPE_END); } dma_cap_mask_t;
```
هو bitmap يحدد نوع الـ DMA transaction المطلوب (مثلاً `DMA_SLAVE` لـ peripheral transfers).

**الـ dma_filter_fn**:
```c
typedef bool (*dma_filter_fn)(struct dma_chan *chan, void *filter_param);
```
دالة يستدعيها الـ DMA engine على كل channel متاح، ترجع `true` إذا كان الـ channel مناسباً للـ requester.

---

### الـ DMA Router: حالة خاصة

بعض الـ SoCs (مثل OMAP، Keystone) تملك **DMA router** — وهو multiplexer hardware يقف بين الـ peripherals والـ DMA controller الفعلي. يتيح توجيه أي peripheral إلى أي channel ديناميكياً.

```
  Peripheral A ──┐
  Peripheral B ──┼──▶ [ DMA Router ] ──▶ DMA Controller
  Peripheral C ──┘         │
                       (struct dma_router)
                       route_free()
```

```c
struct dma_router {
    struct device *dev;
    void (*route_free)(struct device *dev, void *route_data);
};
```

الـ `of_dma_route_allocate` في `struct of_dma` هو الدالة التي تحجز مسار في الـ router (تضبط الـ multiplexer)، بينما `dma_router->route_free` تحرّره عند الانتهاء.

---

### الـ API: ماذا يملك الـ Framework وماذا يفوّض؟

#### ما يملكه OF-DMA:

| الوظيفة | الدالة |
|---|---|
| تسجيل DMA controller في القائمة | `of_dma_controller_register()` |
| إزالة DMA controller من القائمة | `of_dma_controller_free()` |
| تسجيل DMA router | `of_dma_router_register()` |
| طلب channel بالاسم من الـ DT | `of_dma_request_slave_channel()` |

#### ما يفوّضه للـ Driver:

| الوظيفة | المسؤول |
|---|---|
| تفسير الـ DT args (`args[0]`, `args[1]`...) | DMA controller driver عبر `of_dma_xlate` |
| إدارة الـ channels فعلياً (alloc/free) | DMA engine driver |
| ضبط الـ router hardware | DMA router driver عبر `of_dma_route_allocate` |
| تنفيذ الـ DMA transfer | DMA engine (خارج نطاق OF-DMA كلياً) |

#### دالتان جاهزتان (Generic xlate functions):

**`of_dma_simple_xlate()`** — للـ controllers التي تستخدم legacy filter:
```c
/* تستخدم of_dma_filter_info لاستدعاء dma_request_channel() */
struct dma_chan *of_dma_simple_xlate(struct of_phandle_args *dma_spec,
                                     struct of_dma *ofdma);
```

**`of_dma_xlate_by_chan_id()`** — للـ controllers التي تعرّف channels بـ ID مباشر:
```c
/* تأخذ args[0] كـ channel ID وتعيد الـ channel المقابل مباشرة */
struct dma_chan *of_dma_xlate_by_chan_id(struct of_phandle_args *dma_spec,
                                         struct of_dma *ofdma);
```

---

### دورة الحياة الكاملة: من الـ DT إلى الـ dma_chan

```
  Device Tree:
  ┌──────────────────────────────────────┐
  │  uart0: uart@12340000 {              │
  │      dmas = <&pdma 12>, <&pdma 13>; │
  │      dma-names = "rx", "tx";         │
  │  };                                  │
  │                                      │
  │  pdma: dma@11c00000 {                │
  │      compatible = "ti,edma3";        │
  │  };                                  │
  └──────────────────────────────────────┘
          │
          │  Boot time: DMA controller driver probe
          ▼
  of_dma_controller_register(pdma_node, edma_of_xlate, edma_data)
          │
          │  Creates struct of_dma, adds to of_dma_list
          ▼
  ┌──────────────────────────────┐
  │  of_dma_list                 │
  │  └── of_dma {                │
  │        of_node  = pdma_node  │
  │        xlate    = edma_xlate │
  │        of_data  = edma_data  │
  │      }                       │
  └──────────────────────────────┘
          │
          │  UART driver probe
          ▼
  of_dma_request_slave_channel(uart0_node, "tx")
          │
          │  1. of_parse_phandle_with_args() → of_phandle_args{np=pdma, args[0]=13}
          │  2. ابحث في of_dma_list عن of_dma حيث of_node == pdma_node
          │  3. استدعِ edma_xlate(&args, ofdma)
          │  4. edma_xlate تنظر في args[0]=13 وترجع dma_chan للـ channel 13
          ▼
  struct dma_chan * ← تُسلَّم لـ UART driver لاستخدامها
```

---

### الـ CONFIG_DMA_OF Guard

كل الـ API محاط بـ `#ifdef CONFIG_DMA_OF`. عند عدم تفعيله، كل الدوال تصبح `static inline` ترجع `-ENODEV` أو `ERR_PTR(-ENODEV)` — يعني الـ drivers تعمل بلا OF-DMA بشكل نظيف دون تغيير في كودهم. هذا النمط شائع في الـ kernel لدعم architectures لا تستخدم Device Tree (x86 مثلاً).
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Config Options والـ Flags

| الـ Option / الـ Macro | القيمة / الأثر |
|---|---|
| `CONFIG_DMA_OF` | يُفعّل كامل subsystem الـ OF-DMA؛ بدونه كل الدوال تُصبح stubs ترجع `-ENODEV` |
| `MAX_PHANDLE_ARGS` | يساوي `NR_FWNODE_REFERENCE_ARGS` — الحد الأقصى لعدد الـ args في `of_phandle_args` |
| `of_dma_router_free` | macro يُعيد استخدام `of_dma_controller_free` لتحرير الـ router |
| `of_dma_xlate_by_chan_id` | يُعرَّف بـ `NULL` عند غياب `CONFIG_DMA_OF` |

---

### الـ Enums والـ Typedefs المستخدمة

| النوع | تعريفه | الغرض |
|---|---|---|
| `dma_cap_mask_t` | `struct { DECLARE_BITMAP(bits, DMA_TX_TYPE_END); }` | bitmap لقدرات الـ DMA channel |
| `dma_filter_fn` | `bool (*)(struct dma_chan *chan, void *filter_param)` | callback لتصفية الـ channels عند الطلب |
| `phandle` | `u32` | مُعرّف عقدة في Device Tree |
| `dma_cookie_t` | `s32` | معرّف transaction داخل الـ DMA engine |

---

### الـ Structs الأساسية

#### 1. `struct of_dma`

**الغرض**: يُمثّل **DMA controller** مُسجَّلاً عبر Device Tree. كل controller يُنشئ instance منه ويُضيفه إلى قائمة عالمية.

| الحقل | النوع | الشرح |
|---|---|---|
| `of_dma_controllers` | `struct list_head` | ربط هذا الـ controller بالقائمة العالمية لكل الـ controllers |
| `of_node` | `struct device_node *` | عقدة الـ Device Tree المرتبطة بهذا الـ controller |
| `of_dma_xlate` | function pointer | يُحوّل `of_phandle_args` إلى `dma_chan` فعلي — قلب عملية الـ translation |
| `of_dma_route_allocate` | function pointer | يُخصّص مسار (route) داخل الـ DMA router بدلاً من إرجاع channel مباشر |
| `dma_router` | `struct dma_router *` | مؤشر للـ router إن كان هذا الـ controller من نوع router |
| `of_dma_data` | `void *` | بيانات خاصة بالـ driver تُمرَّر لدالة الـ xlate |

**الارتباط**: يُشير إلى `device_node` (هوية في Device Tree) وإلى `dma_router` (منطق التوجيه).

---

#### 2. `struct of_dma_filter_info`

**الغرض**: حُزمة بيانات تُمرَّر كـ `filter_param` عند استخدام `of_dma_simple_xlate` — تجمع بين قدرات الـ channel المطلوبة ودالة التصفية.

| الحقل | النوع | الشرح |
|---|---|---|
| `dma_cap` | `dma_cap_mask_t` | bitmap للقدرات المطلوبة (مثلاً: `DMA_SLAVE`) |
| `filter_fn` | `dma_filter_fn` | الدالة التي تُقرر قبول/رفض channel معين |

**الارتباط**: تُستخدم كـ `of_dma_data` داخل `of_dma`، وتُمرَّر إلى `dma_request_channel()` في الـ DMA engine.

---

#### 3. `struct of_phandle_args` (من `linux/of.h`)

**الغرض**: يحمل نتيجة تحليل **phandle مع arguments** من الـ Device Tree — أي `<&dma_controller channel_id flags>`.

| الحقل | النوع | الشرح |
|---|---|---|
| `np` | `struct device_node *` | العقدة المُشار إليها بالـ phandle (أي الـ DMA controller) |
| `args_count` | `int` | عدد الـ arguments المرفقة بالـ phandle |
| `args[]` | `uint32_t[MAX_PHANDLE_ARGS]` | قيم الـ arguments (مثل: channel ID، flags) |

---

#### 4. `struct dma_chan` (من `linux/dmaengine.h`)

**الغرض**: يُمثّل **DMA channel** واحداً — الوحدة الأساسية لنقل البيانات.

| الحقل | النوع | الشرح |
|---|---|---|
| `device` | `struct dma_device *` | الـ controller المالك لهذا الـ channel |
| `slave` | `struct device *` | الجهاز الطالب (مثل UART، SPI) |
| `cookie` | `dma_cookie_t` | آخر transaction ID أُعطي للعميل |
| `completed_cookie` | `dma_cookie_t` | آخر transaction مكتمل |
| `chan_id` | `int` | معرّف الـ channel في sysfs |
| `device_node` | `struct list_head` | ربطه بقائمة channels في الـ dma_device |
| `router` | `struct dma_router *` | إن كان مُوجَّهاً عبر router |
| `route_data` | `void *` | بيانات المسار الخاص بالـ router |
| `private` | `void *` | بيانات خاصة باتحاد client-channel |

---

#### 5. `struct dma_router` (من `linux/dmaengine.h`)

**الغرض**: يُمثّل **DMA router** — جهاز وسيط يُوجّه طلبات DMA بين أجهزة متعددة وـ controller واحد.

| الحقل | النوع | الشرح |
|---|---|---|
| `dev` | `struct device *` | الجهاز الفيزيائي للـ router |
| `route_free` | `void (*)(struct device *, void *)` | callback لتحرير مسار عند انتهاء الاستخدام |

---

#### 6. `struct device_node` (من `linux/of.h`)

**الغرض**: يُمثّل **عقدة في Device Tree** — هوية كل جهاز في الشجرة.

| الحقل | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | اسم العقدة |
| `phandle` | `phandle` | معرّف فريد للإشارة إلى هذه العقدة من عقد أخرى |
| `full_name` | `const char *` | المسار الكامل في الشجرة |
| `properties` | `struct property *` | قائمة خصائص العقدة |
| `parent` | `struct device_node *` | العقدة الأب |
| `child` | `struct device_node *` | أول عقدة ابن |
| `sibling` | `struct device_node *` | العقدة الأخت التالية |
| `_flags` | `unsigned long` | flags داخلية (initialized, populated...) |

---

### مخطط العلاقات بين الـ Structs

```
Device Tree (DTS)
   │
   │  dmas = <&edma 12 0>;
   │
   ▼
struct of_phandle_args
   ├── np ──────────────────────────► struct device_node   (الـ DMA controller node)
   ├── args_count = 2
   └── args[] = {12, 0}              (channel ID, flags)
          │
          │  passed to xlate function
          ▼
struct of_dma                         (registered controller entry)
   ├── of_dma_controllers ◄──────────── global list (of_dma_list)
   ├── of_node ──────────────────────► struct device_node
   ├── of_dma_xlate() ───────────────► returns ──► struct dma_chan
   ├── of_dma_route_allocate() ──────► returns ──► route_data (void*)
   ├── dma_router ───────────────────► struct dma_router
   │                                      ├── dev
   │                                      └── route_free()
   └── of_dma_data ──────────────────► struct of_dma_filter_info
                                           ├── dma_cap (bitmap)
                                           └── filter_fn()

struct dma_chan                        (النتيجة النهائية للطالب)
   ├── device ────────────────────────► struct dma_device
   ├── slave ─────────────────────────► struct device (طالب DMA)
   ├── router ────────────────────────► struct dma_router
   └── device_node ───────────────────► list inside dma_device->channels
```

---

### دورة حياة الـ DMA Controller عبر OF

```
## Lifecycle: Controller Registration → Channel Request → Teardown

[Driver Probe]
      │
      ▼
of_dma_controller_register(np, xlate_fn, driver_data)
      │
      ├── kmalloc(struct of_dma)
      ├── ofdma->of_node      = np
      ├── ofdma->of_dma_xlate = xlate_fn
      ├── ofdma->of_dma_data  = driver_data
      └── list_add_tail(&ofdma->of_dma_controllers, &of_dma_list)
                                        │
                                        ▼
                               [Controller Ready in Global List]

[Client Driver / Device]
      │
      ▼
of_dma_request_slave_channel(client_np, "tx")
      │
      ├── of_parse_phandle_with_args(np, "dmas", "#dma-cells", ...)
      │         → fills struct of_phandle_args { np=ctrl_node, args={12,0} }
      │
      ├── search of_dma_list for matching of_node == ctrl_node
      │
      └── ofdma->of_dma_xlate(&dma_spec, ofdma)
                │
                └── [e.g. of_dma_simple_xlate]
                        │
                        ├── dma_request_channel(cap, filter_fn, &spec)
                        └── returns struct dma_chan*
                                        │
                                        ▼
                               [Channel in use by client]

[Driver Remove / Channel Release]
      │
      ├── dma_release_channel(chan)          ← client releases channel
      │
      └── of_dma_controller_free(np)
              │
              ├── find ofdma by np in of_dma_list
              ├── list_del(&ofdma->of_dma_controllers)
              └── kfree(ofdma)
```

---

### مخطط تدفق الاستدعاءات

#### تدفق تسجيل الـ Controller

```
driver->probe()
  └── of_dma_controller_register(np, of_dma_simple_xlate, &filter_info)
        └── [of_dma.c]
              ├── allocate struct of_dma
              ├── set fields (of_node, of_dma_xlate, of_dma_data)
              └── list_add_tail → of_dma_list   [protected by of_dma_lock mutex]
```

#### تدفق طلب الـ Channel

```
slave_driver->probe()
  └── of_dma_request_slave_channel(np, "rx")
        └── [of_dma.c]
              ├── of_parse_phandle_with_args()
              │     └── parses DT: dmas = <&edma 12 0>
              │           → of_phandle_args { np=edma_node, args={12,0} }
              │
              ├── mutex_lock(&of_dma_lock)
              ├── list_for_each_entry(ofdma, &of_dma_list, of_dma_controllers)
              │     └── if ofdma->of_node == dma_spec.np → found
              ├── mutex_unlock(&of_dma_lock)
              │
              └── ofdma->of_dma_xlate(&dma_spec, ofdma)
                    │
                    ├── [of_dma_simple_xlate]
                    │     ├── cast of_dma_data → of_dma_filter_info
                    │     ├── dma_cap_copy(cap, info->dma_cap)
                    │     └── dma_request_channel(cap, info->filter_fn, &dma_spec)
                    │           └── [dmaengine core]
                    │                 └── returns struct dma_chan*
                    │
                    └── [of_dma_xlate_by_chan_id]
                          ├── read args[0] as chan_id
                          └── iterate dma_device->channels
                                └── match chan_id → return dma_chan*
```

#### تدفق الـ Router

```
of_dma_router_register(np, route_allocate_fn, dma_router)
  └── allocate struct of_dma
        ├── of_dma_route_allocate = route_allocate_fn
        ├── dma_router = dma_router
        └── list_add_tail → of_dma_list

[channel request hits router]
  └── ofdma->of_dma_route_allocate(&dma_spec, ofdma)
        ├── router allocates physical route
        ├── returns route_data (void*)
        └── dmaengine sets chan->router     = dma_router
                          chan->route_data = route_data

[channel release]
  └── dma_router->route_free(dev, chan->route_data)
```

---

### استراتيجية الـ Locking

| الـ Lock | النوع | ما يحميه |
|---|---|---|
| `of_dma_lock` | `mutex` | القائمة العالمية `of_dma_list` — يمنع race بين register/free/request |
| `dma_list_mutex` (في dmaengine) | `mutex` | قائمة الـ `dma_device_list` العالمية |
| `chan->lock` (per-channel) | `spinlock` أو `mutex` | حالة الـ channel الفردي (client_count, cookie) |

**ترتيب الـ Locks** (lock ordering لتجنب deadlock):

```
of_dma_lock
    └── (then, if needed) dma_list_mutex
            └── (then, if needed) chan->lock
```

- **`of_dma_lock`** يُحتسب دائماً أولاً لأن `of_dma_request_slave_channel` يمسكه ثم يستدعي الـ DMA engine الذي يمسك `dma_list_mutex`.
- لا يجوز عكس الترتيب — الـ DMA engine لا يستدعي OF functions أثناء مسك lock.
- دوال الـ `xlate` تُستدعى بعد الإفراج عن `of_dma_lock` لتجنب احتجاز الـ mutex خلال عمليات قد تتأخر.
## Phase 4: شرح الـ Functions

---

### ملخص — Cheatsheet

#### الـ Structs الأساسية

| Struct | الغرض |
|--------|--------|
| `struct of_dma` | تمثّل **DMA controller** أو **router** مسجَّل عبر Device Tree |
| `struct of_dma_filter_info` | تحمل `dma_cap_mask_t` و `dma_filter_fn` لاستخدامها كـ `of_dma_data` في `of_dma_simple_xlate` |

#### جدول الـ Functions

| Function | الفئة | الغرض المختصر |
|----------|-------|---------------|
| `of_dma_controller_register` | Registration | تسجيل DMA controller مع OF subsystem |
| `of_dma_controller_free` | Cleanup | إلغاء تسجيل DMA controller |
| `of_dma_router_register` | Registration | تسجيل DMA router مع OF subsystem |
| `of_dma_router_free` | Cleanup | macro يُعيد استخدام `of_dma_controller_free` |
| `of_dma_request_slave_channel` | Runtime | طلب DMA channel باسمه من Device Tree |
| `of_dma_simple_xlate` | Helper/Xlate | ترجمة phandle args إلى `dma_chan` بأسلوب بسيط |
| `of_dma_xlate_by_chan_id` | Helper/Xlate | ترجمة phandle args باستخدام `chan_id` مباشرةً |

---

### فئة 1: Registration — تسجيل الـ Controllers والـ Routers

هذه المجموعة مسؤولة عن إضافة DMA controller أو DMA router إلى القائمة العامة `of_dma_list` التي يديرها kernel. عند استدعاء أي منها، يُخصَّص `struct of_dma` ويُملأ بالبيانات ثم يُضاف إلى القائمة المحمية بـ `mutex`. أي طلب لاحق لـ DMA channel عبر `of_dma_request_slave_channel` سيبحث في هذه القائمة.

---

#### `of_dma_controller_register`

```c
int of_dma_controller_register(struct device_node *np,
        struct dma_chan *(*of_dma_xlate)(struct of_phandle_args *,
                                        struct of_dma *),
        void *data);
```

**ما تفعله:** تُسجّل DMA controller مع الـ OF DMA subsystem عن طريق تخصيص `struct of_dma` وتعبئته بـ `np` وـ `of_dma_xlate` وـ `data`، ثم إضافته إلى `of_dma_list` المحمية بـ `of_dma_lock` (mutex). يُستدعى عادةً من `probe()` الخاصة بـ DMA controller driver.

**البارامترات:**

| Parameter | النوع | الشرح |
|-----------|-------|-------|
| `np` | `struct device_node *` | الـ device node الخاص بالـ DMA controller في Device Tree |
| `of_dma_xlate` | function pointer | دالة الترجمة التي تحوّل `of_phandle_args` إلى `dma_chan`؛ المعيار هو `of_dma_simple_xlate` أو `of_dma_xlate_by_chan_id` |
| `data` | `void *` | بيانات خاصة بالـ driver تُمرَّر لـ `xlate` عند استدعائها؛ تُخزَّن في `of_dma->of_dma_data` |

**القيمة المُعادة:** `0` عند النجاح، أو `-ENOMEM` إذا فشل تخصيص `struct of_dma`.

**تفاصيل مهمة:**
- يحتفظ الـ subsystem بـ reference على `np` عبر `of_node_get()` داخلياً.
- الـ `of_dma_lock` (mutex) يحمي `of_dma_list` — لا يجوز الاستدعاء من context يحمل lock آخر يُسبّب deadlock.
- لا يُستدعى من interrupt context.

**Pseudocode Flow:**
```
of_dma_controller_register(np, xlate, data):
    ofdma = kzalloc(sizeof(*ofdma))
    if !ofdma → return -ENOMEM
    ofdma->of_node       = np
    ofdma->of_dma_xlate  = xlate
    ofdma->of_dma_data   = data
    mutex_lock(&of_dma_lock)
    list_add_tail(&ofdma->of_dma_controllers, &of_dma_list)
    mutex_unlock(&of_dma_lock)
    return 0
```

---

#### `of_dma_router_register`

```c
int of_dma_router_register(struct device_node *np,
        void *(*of_dma_route_allocate)(struct of_phandle_args *,
                                       struct of_dma *),
        struct dma_router *dma_router);
```

**ما تفعله:** مشابهة تماماً لـ `of_dma_controller_register` لكنها مخصصة لـ **DMA routers** — أجهزة وسيطة (مثل crossbar switches) تُوجِّه DMA requests بين الأجهزة الطرفية وقنوات DMA الفعلية. تُخصَّص `struct of_dma` وتُملأ بـ `of_dma_route_allocate` وـ `dma_router` بدلاً من `of_dma_xlate`.

**البارامترات:**

| Parameter | النوع | الشرح |
|-----------|-------|-------|
| `np` | `struct device_node *` | الـ device node للـ DMA router في Device Tree |
| `of_dma_route_allocate` | function pointer | تُخصِّص مسار التوجيه وتُعيد `void *route_data` أو `ERR_PTR` عند الفشل |
| `dma_router` | `struct dma_router *` | بنية الـ router التي تحمل `route_free` callback لتحرير المسار لاحقاً |

**القيمة المُعادة:** `0` عند النجاح، أو `-ENOMEM` عند الفشل.

**تفاصيل مهمة:**
- الـ `of_dma_route_allocate` تُستدعى لاحقاً من `of_dma_request_slave_channel` لإعداد مسار التوجيه قبل إعادة الـ `dma_chan`.
- الـ `dma_router->route_free` تُستدعى عند تحرير الـ channel لإلغاء التوجيه.

---

### فئة 2: Cleanup — إلغاء التسجيل

#### `of_dma_controller_free`

```c
void of_dma_controller_free(struct device_node *np);
```

**ما تفعله:** تبحث في `of_dma_list` عن الـ entry المرتبطة بـ `np`، تُزيلها من القائمة، ثم تُحرّر الذاكرة. تُستدعى من `remove()` أو `devm` cleanup عند إزالة DMA controller driver.

**البارامترات:**

| Parameter | النوع | الشرح |
|-----------|-------|-------|
| `np` | `struct device_node *` | نفس الـ device node المُستخدم في `of_dma_controller_register` |

**القيمة المُعادة:** لا شيء (`void`).

**تفاصيل مهمة:**
- تحتاج إلى `of_dma_lock` mutex — لا تستدعِ من atomic context.
- إذا لم يُعثر على `np` في القائمة، الدالة تعود بصمت دون crash.
- **الـ `of_dma_router_free`** هو مجرد `#define of_dma_router_free of_dma_controller_free` — نفس الدالة تماماً.

---

### فئة 3: Runtime — طلب الـ Channels

#### `of_dma_request_slave_channel`

```c
struct dma_chan *of_dma_request_slave_channel(struct device_node *np,
                                              const char *name);
```

**ما تفعله:** هي نقطة الدخول الرئيسية لطلب DMA channel من Device Tree. تبحث عن property اسمها `dma-names` في node الجهاز للعثور على index الاسم المطلوب، ثم تقرأ الـ phandle المقابل من property `dmas`، ثم تبحث في `of_dma_list` عن الـ controller المسجَّل المقابل وتستدعي `of_dma_xlate` (أو تُهيئ مسار التوجيه أولاً إن كان router) للحصول على الـ `dma_chan`.

**البارامترات:**

| Parameter | النوع | الشرح |
|-----------|-------|-------|
| `np` | `struct device_node *` | الـ device node للجهاز الطالب (consumer) — مثلاً node خاص بـ UART أو SPI |
| `name` | `const char *` | اسم الـ channel كما يظهر في `dma-names` بالـ DTS؛ مثلاً `"tx"` أو `"rx"` |

**القيمة المُعادة:** `struct dma_chan *` عند النجاح، أو `ERR_PTR(-errno)` عند الفشل (`-ENODEV` إن لم يُعثر على controller، `-EPROBE_DEFER` إن لم يُسجَّل بعد).

**تفاصيل مهمة:**
- هذه الدالة هي wrapper فوق `of_dma_request_slave_channel` الداخلية التي تستدعي `of_parse_phandle_with_args`.
- دعم الـ `-EPROBE_DEFER` أساسي لضمان الترتيب الصحيح لـ probe في حال تحميل الـ DMA controller driver بعد الـ consumer.
- إذا كان الـ entry المُعثور عليه في `of_dma_list` عبارة عن **router**، يُستدعى `of_dma_route_allocate` أولاً لتأسيس مسار التوجيه، ثم `of_dma_xlate` على الـ controller الفعلي.
- لا تستدعَ من interrupt context.

**مثال DTS:**
```dts
/* في DTS الخاص بـ UART */
uart0: uart@01c28000 {
    dmas = <&dma 6>, <&dma 7>;   /* phandle + channel args */
    dma-names = "rx", "tx";
};
```

```c
/* في driver الـ UART */
chan = of_dma_request_slave_channel(dev->of_node, "tx");
```

**Pseudocode Flow:**
```
of_dma_request_slave_channel(np, name):
    index = of_property_match_string(np, "dma-names", name)
    if index < 0 → return ERR_PTR(index)

    of_parse_phandle_with_args(np, "dmas", "#dma-cells",
                               index, &dma_spec)

    mutex_lock(&of_dma_lock)
    list_for_each(of_dma_list):
        if entry->of_node == dma_spec.np:
            if entry->of_dma_route_allocate:
                route_data = entry->of_dma_route_allocate(&dma_spec, entry)
                /* setup router, get actual controller channel */
            chan = entry->of_dma_xlate(&dma_spec, entry)
            break
    mutex_unlock(&of_dma_lock)

    return chan  /* or ERR_PTR */
```

---

### فئة 4: Helpers / Xlate — دوال الترجمة

هذه الدوال تُنفَّذ كـ callbacks تُسجَّلها الـ DMA controller drivers مع `of_dma_controller_register`. تتلقى `of_phandle_args` (التي تحمل args من property `dmas` في DTS) وتُعيد الـ `dma_chan` المناسب.

---

#### `of_dma_simple_xlate`

```c
struct dma_chan *of_dma_simple_xlate(struct of_phandle_args *dma_spec,
                                     struct of_dma *ofdma);
```

**ما تفعله:** أبسط تنفيذ لـ xlate callback. تستخدم `of_dma_data` (التي يجب أن تكون من نوع `struct of_dma_filter_info *`) للحصول على `dma_cap_mask` وـ `filter_fn`، ثم تستدعي `dma_request_channel()` مع قيمة `args[0]` من الـ phandle كـ filter parameter. مناسبة للـ controllers التي يُحدَّد فيها الـ channel برقم واحد فقط في DTS.

**البارامترات:**

| Parameter | النوع | الشرح |
|-----------|-------|-------|
| `dma_spec` | `struct of_phandle_args *` | يحمل `args[0]` = رقم الـ DMA request line كما في DTS |
| `ofdma` | `struct of_dma *` | الـ entry المسجَّل؛ `ofdma->of_dma_data` يجب أن يشير إلى `struct of_dma_filter_info` |

**القيمة المُعادة:** `struct dma_chan *` عند النجاح، أو `NULL` إن لم يُعثر على channel مناسب.

**تفاصيل مهمة:**
- تتطلب أن `of_dma_data` يشير إلى `struct of_dma_filter_info` تحمل `dma_cap_mask_t` وـ `dma_filter_fn`.
- تستدعي `dma_request_channel(info->dma_cap, info->filter_fn, &dma_spec->args[0])` داخلياً.
- مناسبة للـ platforms التي تستخدم `#dma-cells = <1>` في DTS (رقم request line واحد فقط).

**مثال استخدام في DMA controller driver:**
```c
static struct of_dma_filter_info my_dma_info = {
    .filter_fn = my_dma_filter,
};
dma_cap_set(DMA_SLAVE, my_dma_info.dma_cap);

of_dma_controller_register(pdev->dev.of_node,
                            of_dma_simple_xlate,
                            &my_dma_info);
```

---

#### `of_dma_xlate_by_chan_id`

```c
struct dma_chan *of_dma_xlate_by_chan_id(struct of_phandle_args *dma_spec,
                                         struct of_dma *ofdma);
```

**ما تفعله:** xlate callback أكثر مباشرةً — تبحث عن الـ `dma_chan` الذي `chan_id` الخاص به يساوي `dma_spec->args[0]`، مباشرةً من قائمة channels الخاصة بالـ `dma_device` المُخزَّن في `of_dma_data`. لا تحتاج إلى `filter_fn`.

**البارامترات:**

| Parameter | النوع | الشرح |
|-----------|-------|-------|
| `dma_spec` | `struct of_phandle_args *` | `args[0]` = الـ `chan_id` المطلوب |
| `ofdma` | `struct of_dma *` | `ofdma->of_dma_data` يجب أن يشير إلى `struct dma_device *` |

**القيمة المُعادة:** `struct dma_chan *` عند النجاح بعد استدعاء `dma_get_slave_channel()`، أو `NULL` إن لم يُعثر على channel بذلك الـ `chan_id`.

**تفاصيل مهمة:**
- تُكرِّر على `dma_device->channels` وتقارن `chan->chan_id` بـ `dma_spec->args[0]`.
- عند العثور، تستدعي `dma_get_slave_channel()` لتحجز الـ channel وترفع reference count.
- مناسبة للـ controllers التي تعرِّف channels بـ IDs ثابتة (مثل `#dma-cells = <1>` بمعنى channel ID مباشر).
- في حالة عدم توفر `CONFIG_DMA_OF`، الـ macro يُعرَّف كـ `NULL` — الـ callers يجب أن يتحققوا من ذلك.

**مقارنة بين xlate functions:**

| | `of_dma_simple_xlate` | `of_dma_xlate_by_chan_id` |
|---|---|---|
| **الـ `of_dma_data` المطلوب** | `struct of_dma_filter_info *` | `struct dma_device *` |
| **آلية البحث** | عبر `dma_request_channel` + `filter_fn` | مباشرة بـ `chan_id` |
| **الـ `#dma-cells`** | `<1>` (request line) | `<1>` (channel ID) |
| **المرونة** | أعلى — filter يمكنه أي منطق | أبسط — مقارنة مباشرة |

---

### الـ Stub Implementations (عند `!CONFIG_DMA_OF`)

عندما لا يُفعَّل `CONFIG_DMA_OF`، تُستبدل جميع الدوال بـ static inline stubs:

| Function | القيمة المُعادة في الـ Stub |
|----------|---------------------------|
| `of_dma_controller_register` | `-ENODEV` |
| `of_dma_controller_free` | لا شيء (empty body) |
| `of_dma_router_register` | `-ENODEV` |
| `of_dma_request_slave_channel` | `ERR_PTR(-ENODEV)` |
| `of_dma_simple_xlate` | `NULL` |
| `of_dma_xlate_by_chan_id` | `NULL` (عبر `#define`) |

هذا النمط يضمن أن الـ consumer drivers تُكمَّل compilation بدون `#ifdef` في كل مكان — تتعامل مع `ERR_PTR(-ENODEV)` أو `NULL` بشكل موحَّد.

---

### الـ Data Flow الكامل — ASCII Diagram

```
DMA Controller Driver probe():
  of_dma_controller_register(np, xlate_fn, data)
         │
         ▼
  ┌─────────────────────────────────────────────────┐
  │              of_dma_list (global)               │
  │  ┌──────────────────────────────────────────┐   │
  │  │  struct of_dma                           │   │
  │  │  ├── of_node      → controller's np      │   │
  │  │  ├── of_dma_xlate → xlate_fn             │   │
  │  │  └── of_dma_data  → driver private data  │   │
  │  └──────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────┘
         │
         │  (later, consumer driver probe)
         ▼
  of_dma_request_slave_channel(consumer_np, "tx")
         │
         ├─ parse "dma-names" → find index of "tx"
         ├─ parse "dmas"      → get of_phandle_args
         │                       (controller np + args[])
         ├─ lookup of_dma_list by controller np
         └─ call of_dma->of_dma_xlate(&args, of_dma)
                    │
                    ├─ [of_dma_simple_xlate]
                    │   dma_request_channel(cap, filter, &args[0])
                    │
                    └─ [of_dma_xlate_by_chan_id]
                        find chan where chan->chan_id == args[0]
                        dma_get_slave_channel(chan)
                    │
                    ▼
              struct dma_chan *   ← يُعاد إلى الـ consumer
```
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem المعني هو **OF-DMA** (Open Firmware DMA helpers)، المسؤول عن ربط الـ Device Tree بنظام الـ DMA engine في Linux. يتمحور حول `of_dma_controller_register`، `of_dma_request_slave_channel`، والـ `of_dma_xlate` callbacks. الأعطال الشائعة تقع عند تسجيل الـ controller، أو عند تحليل الـ `dmas` property في الـ DT، أو عند تخصيص الـ channel.

---

### Software Level

#### 1. مدخلات الـ debugfs ذات الصلة

الـ DMA engine يعرض معلوماته تحت `/sys/kernel/debug/dmaengine/`:

```
/sys/kernel/debug/dmaengine/
├── summary          ← ملخص كل DMA controllers و channels المسجّلة
```

```bash
# اقرأ ملخص كل controllers و channels
cat /sys/kernel/debug/dmaengine/summary
```

مثال على الخرج وتفسيره:

```
dma0 (pl330) [ slave caps: 0x6 ]
   dma0chan0  [  in-use ]  CYCLIC
   dma0chan1  [  available ]
dma1 (edma) [ slave caps: 0x2 ]
   dma1chan0  [  in-use ]  SLAVE
```

- `in-use` → channel محجوز بواسطة driver
- `available` → channel حر لم يُطلب بعد
- اسم الـ controller بعد `(...)` يطابق `struct dma_device.dev.driver->name`

إذا لم يظهر الـ controller هنا رغم تهيئته، فالمشكلة في `of_dma_controller_register` أو في عدم استدعاء `dma_async_device_register`.

---

#### 2. مدخلات الـ sysfs ذات الصلة

```
/sys/bus/platform/devices/<dev>/
    dma_mask          ← الـ DMA address mask للجهاز
/sys/class/dma/
    dma0chan0/
        ├── bytes_transferred
        ├── in_use
        └── memcpy_count
```

```bash
# تحقق من كل channels المرئية للنظام
ls /sys/class/dma/

# اقرأ حالة channel معينة
cat /sys/class/dma/dma0chan0/in_use

# تحقق من الـ DMA mask للجهاز (مثلاً UART يستخدم OF-DMA)
cat /sys/bus/platform/devices/ff140000.serial/dma_mask 2>/dev/null || \
    grep -r "dma" /sys/bus/platform/devices/ff140000.serial/
```

---

#### 3. الـ ftrace: tracepoints و events

الـ subsystem يستخدم events من `dmaengine` و `of`:

```bash
# اعرض كل events المتاحة للـ DMA
grep -r "dma" /sys/kernel/tracing/available_events | grep -v "^#"

# فعّل تتبع طلبات الـ DMA channel
echo 1 > /sys/kernel/tracing/events/dma_fence/enable

# تتبع أحداث الـ OF parsing (device tree)
echo 1 > /sys/kernel/tracing/events/enable
# أو بشكل أدق:
echo 'dmaengine:*' > /sys/kernel/tracing/set_event

# شغّل الـ tracer
echo 1 > /sys/kernel/tracing/tracing_on
# ... اختبر الكود ...
cat /sys/kernel/tracing/trace
echo 0 > /sys/kernel/tracing/tracing_on
```

لتتبع استدعاء `of_dma_request_slave_channel` بالذات، استخدم **function tracer**:

```bash
echo function > /sys/kernel/tracing/current_tracer
echo of_dma_request_slave_channel > /sys/kernel/tracing/set_ftrace_filter
echo of_dma_controller_register >> /sys/kernel/tracing/set_ftrace_filter
echo of_dma_find_controller >> /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on
# ... trigger the driver probe ...
cat /sys/kernel/tracing/trace
```

---

#### 4. الـ printk و dynamic debug

الـ OF-DMA يستخدم `pr_debug` و `dev_dbg`. لتفعيل الرسائل:

```bash
# تفعيل dynamic debug لملف of_dma.c بالكامل
echo 'file drivers/dma/of-dma.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو لكل الـ DMA drivers
echo 'file drivers/dma/* +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل رسائل OF (device tree parsing)
echo 'file drivers/of/base.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/of/property.c +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق مما تم تفعيله
grep "of.dma\|of_dma" /sys/kernel/debug/dynamic_debug/control
```

لإضافة `printk` مؤقت في كود خاص بك أثناء التطوير:

```c
/* في of_dma_request_slave_channel أو driver probe */
pr_info("OF-DMA: requesting channel '%s' from node '%s'\n",
        name, np->full_name);

/* تحقق من نتيجة of_parse_phandle_with_args */
pr_debug("OF-DMA: dma-spec args_count=%d, args[0]=%u\n",
         dma_spec.args_count, dma_spec.args[0]);
```

---

#### 5. خيارات الـ kernel config للـ debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_DMA_OF` | تفعيل OF-DMA helpers (شرط أساسي) |
| `CONFIG_DMA_ENGINE` | تفعيل DMA engine core |
| `CONFIG_DMATEST` | module اختبار DMA channels |
| `CONFIG_DEBUG_DMA_API` | تحقق من صحة استخدام DMA API |
| `CONFIG_DMA_API_DEBUG` | تتبع mappings وكشف leaks |
| `CONFIG_DMA_API_DEBUG_SG` | تحقق إضافي على scatter-gather |
| `CONFIG_OF_DEBUG` | رسائل debug لـ Device Tree parsing |
| `CONFIG_OF_UNITTEST` | unit tests للـ OF core |
| `CONFIG_GENERIC_IRQ_DEBUGFS` | debugfs للـ IRQ (مفيد مع DMA interrupts) |
| `CONFIG_DEBUG_SHIRQ` | كشف مشاكل shared IRQs |
| `CONFIG_KASAN` | كشف memory errors في الـ DMA buffers |

```bash
# تحقق من الـ config الحالي للـ kernel
zcat /proc/config.gz | grep -E "DMA_OF|DMA_ENGINE|DMA_API|OF_DEBUG"
# أو
grep -E "DMA_OF|DMA_ENGINE|DMA_API|OF_DEBUG" /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ subsystem

**أداة DMATEST**: لاختبار channels مباشرةً

```bash
# حمّل module الاختبار
modprobe dmatest

# اختبر channel محدد (مثلاً dma0chan0)
echo dma0chan0 > /sys/module/dmatest/parameters/channel
echo 2000      > /sys/module/dmatest/parameters/timeout
echo 1         > /sys/module/dmatest/parameters/iterations
echo 1         > /sys/module/dmatest/parameters/run

# اقرأ النتائج
dmesg | grep -i dmatest
```

**فحص الـ OF-DMA registration عبر /proc**:

```bash
# تحقق من الـ controllers المسجلة في قائمة of_dma_list
# (لا توجد واجهة مباشرة، لكن يمكن استخدام /proc/devices)
cat /proc/devices | grep -i dma

# فحص الـ device tree nodes التي تحتوي على dmas property
find /proc/device-tree -name "dmas" 2>/dev/null
# أو
find /sys/firmware/devicetree/base -name "dmas" | while read f; do
    echo "=== $(dirname $f) ==="; hexdump -C "$f"; done
```

**أداة dtc لقراءة الـ DT**:

```bash
# حوّل الـ DTB المحمّل إلى DTS قابل للقراءة
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A5 "dmas"
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---|---|---|
| `of_dma_request_slave_channel: dma-names property of node '...' missing or empty` | الـ `dmas-names` property غائبة في الـ DT | أضف `dma-names = "tx", "rx";` في الـ DTS |
| `of_dma_request_slave_channel: can't find DMA controller '...'` | الـ DMA controller لم يُسجّل بعد (race condition) أو phandle خاطئ | تحقق من `&dma0` في الـ DTS وتأكد من probe order |
| `of_dma_controller_register: DMA controller already registered` | تسجيل مزدوج لنفس الـ `device_node` | تحقق من عدم استدعاء `of_dma_controller_register` مرتين |
| `dma_request_chan: dma channel ... not found` | `dmaengine` لم يجد channel بالاسم المطلوب | تحقق من تطابق اسم الـ channel في `dma-names` مع ما يطلبه الـ driver |
| `-EPROBE_DEFER` عند `of_dma_request_slave_channel` | الـ DMA controller لم يُحمَّل بعد | طبيعي — الـ driver سيُعاد probe لاحقاً؛ تحقق إذا استمر |
| `Got channel NULL from of_dma_find_controller` | الـ `of_dma_xlate` callback أعادت NULL | خطأ في تفسير `of_phandle_args`؛ راجع args[0] مقابل عدد الـ channels |
| `DMA-API: device driver has pending DMA allocations while released from device` | memory leak في الـ DMA buffers | فعّل `CONFIG_DMA_API_DEBUG` وتتبع الـ map/unmap calls |
| `dmaengine: failed to get slave channel` | فشل عام في طلب channel | راجع سجل الأخطاء السابق لهذه الرسالة |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في of_dma_controller_register — تحقق من صحة المدخلات */
int of_dma_controller_register(struct device_node *np,
    struct dma_chan *(*of_dma_xlate)(struct of_phandle_args *, struct of_dma *),
    void *data)
{
    WARN_ON(!np);           /* node فارغ = خطأ برمجي */
    WARN_ON(!of_dma_xlate); /* callback فارغ = لن يعمل أي طلب */
    ...
}

/* في of_dma_request_slave_channel — تتبع فشل التحليل */
static struct dma_chan *of_dma_request_slave_channel(...)
{
    ...
    ret = of_parse_phandle_with_args(np, "dmas", "#dma-cells",
                                     index, &dma_spec);
    if (ret) {
        /* هنا dump_stack مفيد لمعرفة من طلب الـ channel */
        dump_stack();
        return ERR_PTR(ret);
    }
    ...
    /* تحقق من صحة الـ xlate result */
    WARN_ON_ONCE(IS_ERR(chan) && PTR_ERR(chan) != -EPROBE_DEFER);
}

/* في of_dma_xlate callback (في driver الـ controller) */
struct dma_chan *my_dma_xlate(struct of_phandle_args *dma_spec,
                               struct of_dma *ofdma)
{
    /* تحقق من عدد الـ args المتوقع */
    WARN_ON(dma_spec->args_count != 1); /* مثلاً إذا #dma-cells = <1> */

    u32 chan_id = dma_spec->args[0];
    WARN_ON(chan_id >= MAX_CHANNELS);
    ...
}
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ hardware مع حالة الـ kernel

الخطوات الأساسية للتحقق:

```bash
# 1. تحقق من أن الـ DMA controller ظهر كـ platform device
ls /sys/bus/platform/devices/ | grep -i dma

# 2. تحقق من أن الـ driver تم bind بنجاح
cat /sys/bus/platform/devices/<dma-controller>/driver_override 2>/dev/null
ls -la /sys/bus/platform/devices/<dma-controller>/driver

# 3. تحقق من الـ clock المغذي للـ DMA controller
cat /sys/kernel/debug/clk/<dma-clk>/clk_enable_count
cat /sys/kernel/debug/clk/<dma-clk>/clk_rate

# 4. تحقق من الـ power domain
cat /sys/bus/platform/devices/<dma-controller>/power/runtime_status
```

مطابقة الـ hardware مع الـ DT:

```bash
# اقرأ عنوان الـ DMA controller من الـ DT
cat /sys/firmware/devicetree/base/<dma-node>/reg | hexdump -C

# قارن مع الـ resource المحجوزة في الـ kernel
cat /proc/iomem | grep -i dma
```

---

#### 2. تقنيات الـ Register Dump

```bash
# باستخدام devmem2 (يجب تثبيته)
# مثال: DMA controller على عنوان 0xFF6D0000 (RK3399 DMAC)
devmem2 0xFF6D0000 w   # اقرأ الـ status register
devmem2 0xFF6D0004 w   # اقرأ الـ configuration register

# باستخدام /dev/mem مباشرة (يحتاج CONFIG_DEVMEM=y)
dd if=/dev/mem bs=4 count=64 skip=$((0xFF6D0000/4)) 2>/dev/null | hexdump -C

# باستخدام io utility (من package io-utils أو busybox)
io -4 -r 0xFF6D0000    # قراءة 32-bit register

# طريقة أكثر أماناً: debugfs register dump (إذا دعمه الـ driver)
cat /sys/kernel/debug/<dma-driver>/registers 2>/dev/null
```

مثال على تفسير registers لـ PL330 DMA controller:

```
Offset 0x000: DS   — DMA Status Register
              0x0 = Stopped, 0x1 = Executing, 0x2 = Cache miss
              0x3 = Updating PC, 0x4 = Waiting for event
Offset 0x004: DPC  — DMA Program Counter
Offset 0x020: INTEN — Interrupt Enable Register
Offset 0x02C: INTMIS — Interrupt Status (masked)
```

---

#### 3. نصائح Logic Analyzer و Oscilloscope

عند تتبع مشاكل الـ OF-DMA على مستوى الـ hardware:

**بروتوكولات يجب مراقبتها:**
- **AXI/AHB bus**: نشاط الـ DMA transfers (عنوان، بيانات، burst size)
- **DMAREQ/DMAACK signals**: طلب الـ DMA من الجهاز الطرفي (مثل UART، SPI)
- **IRQ line**: تحقق من رفع الـ interrupt عند اكتمال الـ transfer

```
مثال على إعداد Logic Analyzer لـ DMA request/acknowledge:

Channel 0: UART_DMAREQ  (طلب الـ DMA من UART)
Channel 1: DMA_DMAACK   (إقرار الـ DMA controller)
Channel 2: DMA_IRQ      (interrupt عند الانتهاء)
Trigger: rising edge على Channel 0

التوقع الصحيح:
DMAREQ  ‾‾‾|___|‾‾‾
DMAACK  ____|‾‾‾|___   (يتبع بتأخير بسيط < 2 clock cycles)
IRQ     ________|‾|_   (عند اكتمال الـ transfer)

مشكلة: إذا ظهر DMAREQ لكن لا DMAACK → الـ controller لم يُفعَّل
مشكلة: إذا ظهر DMAACK لكن لا IRQ  → مشكلة في الـ interrupt routing
```

**Oscilloscope لقياس timing:**
- قس وقت الـ transfer الفعلي مقابل المتوقع حسب الـ burst size والـ clock
- تحقق من voltage levels للـ DMAREQ signal (يجب أن يكون CMOS level صحيح)

---

#### 4. مشاكل الـ hardware الشائعة وأنماط الـ kernel log

| مشكلة الـ hardware | نمط الـ kernel log | التشخيص |
|---|---|---|
| الـ DMA clock معطل | `of_dma_request_slave_channel: can't find DMA controller` أو probe يفشل صامتاً | `cat /sys/kernel/debug/clk/*/clk_enable_count` |
| الـ power domain مقفل | `dma_request_chan: dma channel not found` بعد suspend/resume | `cat /sys/bus/platform/devices/<dev>/power/runtime_status` |
| DMAREQ line غير موصولة | لا transfer يحدث رغم تهيئة ناجحة للـ channel | Logic analyzer على الـ DMAREQ pin |
| عنوان خاطئ في الـ DT | `devm_ioremap_resource: cannot reserve memory region` | قارن `reg` في DTS مع datasheet |
| الـ burst size يتجاوز الـ FIFO | data corruption أو اهتزاز | قلل `dma-burst-size` في DTS |
| خلل في الـ AXI interconnect | `DMA-API: mapping error` أو `iommu fault` | راجع IOMMU logs، افحص الـ bus topology |
| interrupt لا يصل للـ CPU | الـ transfer يكتمل لكن callback لا يُستدعى أبداً | `cat /proc/interrupts` وتحقق من تزايد العداد |

---

#### 5. تصحيح الـ Device Tree للـ OF-DMA

**التحقق من صحة الـ DT:**

```bash
# اعرض الـ DT الحالي المحمّل في شكل مقروء
dtc -I dtb -O dts -o /tmp/current.dts /sys/firmware/fdt 2>/dev/null

# ابحث عن كل nodes التي تحتوي على dmas property
grep -A3 "dmas" /tmp/current.dts

# تحقق من وجود #dma-cells في controller node
grep -B5 -A10 "dma-controller" /tmp/current.dts
```

**مثال على DTS صحيح:**

```dts
/* DMA controller node */
dma0: dma-controller@ff6d0000 {
    compatible = "arm,pl330", "arm,primecell";
    reg = <0x0 0xff6d0000 0x0 0x4000>;
    interrupts = <GIC_SPI 0 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&cru ACLK_DMAC0>;
    clock-names = "apb_pclk";
    #dma-cells = <1>;           /* ← مطلوب لـ of_dma_xlate */
    arm,pl330-periph-burst;
};

/* Device يستخدم DMA */
uart0: serial@ff180000 {
    compatible = "rockchip,rk3399-uart", "snps,dw-apb-uart";
    reg = <0x0 0xff180000 0x0 0x100>;
    dmas = <&dma0 0>, <&dma0 1>;    /* ← phandle + channel ID */
    dma-names = "tx", "rx";          /* ← يطابق طلب driver */
};
```

**أخطاء DT شائعة:**

```bash
# خطأ 1: #dma-cells غائب
# kernel log: "missing #dma-cells"
grep "#dma-cells" /sys/firmware/devicetree/base/dma-controller*/\#dma-cells \
    2>/dev/null | hexdump -C
# القيمة يجب أن تكون 0x00000001 (big-endian) إذا #dma-cells = <1>

# خطأ 2: phandle خاطئ
# تحقق من الـ phandle الفعلي للـ controller
hexdump -C /sys/firmware/devicetree/base/dma-controller@ff6d0000/phandle
# قارنه مع الـ phandle المستخدم في dmas property للجهاز
hexdump -C /sys/firmware/devicetree/base/serial@ff180000/dmas

# خطأ 3: dma-names لا يطابق ما يطلبه الـ driver
# الـ driver يطلب: dma_request_chan(dev, "tx")
# لكن DT يحتوي: dma-names = "dma-tx"  ← خطأ
xxd /sys/firmware/devicetree/base/serial@ff180000/dma-names
```

---

### Practical Commands

#### مجموعة الأوامر الجاهزة للنسخ

**سيناريو 1: التحقق من تسجيل الـ DMA controller**

```bash
#!/bin/bash
# تحقق سريع من حالة OF-DMA

echo "=== DMA Controllers في kernel ==="
cat /sys/kernel/debug/dmaengine/summary 2>/dev/null || \
    echo "debugfs غير متاح، تحقق من: mount -t debugfs none /sys/kernel/debug"

echo ""
echo "=== DMA Channels الحالية ==="
ls /sys/class/dma/ 2>/dev/null

echo ""
echo "=== DMA في Device Tree ==="
find /sys/firmware/devicetree/base -name "dma-names" 2>/dev/null | while read f; do
    dir=$(dirname "$f")
    node=$(basename "$dir")
    names=$(strings "$f" | tr '\n' ',')
    echo "  $node: $names"
done

echo ""
echo "=== رسائل الـ kernel المتعلقة بـ DMA ==="
dmesg | grep -iE "dma|of_dma" | tail -30
```

**سيناريو 2: تتبع طلب channel بـ ftrace**

```bash
#!/bin/bash
# تتبع of_dma_request_slave_channel

TRACE=/sys/kernel/tracing

# تنظيف
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace

# إعداد
echo function > $TRACE/current_tracer
cat > $TRACE/set_ftrace_filter << 'EOF'
of_dma_request_slave_channel
of_dma_find_controller
of_dma_controller_register
of_dma_controller_free
of_parse_phandle_with_args
EOF

echo 1 > $TRACE/tracing_on
echo "الآن حمّل الـ driver أو أعد probe الجهاز..."
echo "مثال: echo <dev> > /sys/bus/platform/drivers/<drv>/bind"
echo "اضغط ENTER عند الانتهاء"
read

echo 0 > $TRACE/tracing_on
echo "=== نتائج الـ trace ==="
cat $TRACE/trace | grep -v "^#" | head -50
```

**سيناريو 3: تفعيل dynamic debug ومراقبة النتائج**

```bash
#!/bin/bash
# تفعيل كل رسائل debug لـ OF-DMA

# تأكد من تركيب debugfs
mountpoint -q /sys/kernel/debug || mount -t debugfs none /sys/kernel/debug

# فعّل رسائل of-dma
echo 'file drivers/dma/of-dma.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل رسائل OF property parsing
echo 'file drivers/of/property.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/of/base.c +p' > /sys/kernel/debug/dynamic_debug/control

# راقب الـ kernel log في real-time
echo "مراقبة kernel log... (Ctrl+C للإيقاف)"
dmesg -w | grep -iE "of.dma|of_dma|dmaengine|dma.*chan"
```

**سيناريو 4: DMATEST للتحقق من صحة الـ channel**

```bash
#!/bin/bash
# اختبار DMA channel مباشرة

CHANNEL=${1:-"dma0chan0"}
echo "اختبار channel: $CHANNEL"

modprobe dmatest 2>/dev/null || true

# إعداد معاملات الاختبار
echo "$CHANNEL"  > /sys/module/dmatest/parameters/channel
echo "9000"      > /sys/module/dmatest/parameters/timeout
echo "10"        > /sys/module/dmatest/parameters/iterations
echo "4096"      > /sys/module/dmatest/parameters/test_buf_size

# تشغيل الاختبار
echo "1" > /sys/module/dmatest/parameters/run

# انتظر وأظهر النتائج
sleep 5
dmesg | grep -i dmatest | tail -20
```

**سيناريو 5: تصحيح الـ DT بالكامل للـ OF-DMA**

```bash
#!/bin/bash
# تفريغ وتحليل الـ DT للـ DMA

DTB_OUT=/tmp/current-dma.dts

# تحويل الـ DTB الحالي
dtc -I dtb -O dts -o "$DTB_OUT" /sys/firmware/fdt 2>/dev/null
echo "تم حفظ الـ DTS في: $DTB_OUT"

echo ""
echo "=== كل الـ DMA Controllers ==="
grep -n "dma-controller" "$DTB_OUT" | head -20

echo ""
echo "=== كل الأجهزة التي تستخدم DMA ==="
grep -n "dma-names" "$DTB_OUT"

echo ""
echo "=== #dma-cells للـ controllers ==="
grep -n "#dma-cells" "$DTB_OUT"

echo ""
echo "=== فحص phandle consistency ==="
# استخرج الـ phandles للـ DMA controllers
grep -B5 "dma-controller" "$DTB_OUT" | grep "phandle\|label\|dma-controller"
```

**سيناريو 6: مراقبة الـ interrupts للـ DMA**

```bash
#!/bin/bash
# تحقق من تلقي الـ DMA interrupts

echo "=== قبل الـ transfer ==="
grep -i dma /proc/interrupts

echo ""
echo "ابدأ الـ DMA transfer الآن، ثم اضغط ENTER..."
read

echo "=== بعد الـ transfer ==="
grep -i dma /proc/interrupts

echo ""
echo "إذا لم تتغير الأرقام، الـ interrupt لا يصل للـ CPU"
echo "تحقق من: 1) GIC configuration  2) DT interrupts property  3) IRQ routing"
```

**مثال على خرج سيناريو 4 وتفسيره:**

```
# خرج ناجح:
[  42.123456] dmatest: Added 1 threads using dma0chan0
[  42.234567] dmatest: dma0chan0-copy0: summary 10 tests, 0 failures, 1234 iops, 5048 KB/s (0)

# خرج فاشل:
[  42.123456] dmatest: Added 1 threads using dma0chan0
[  42.234567] dmatest: dma0chan0-copy0: result #1: 'data mismatch' with src_off=0x7 dst_off=0x0 len=0x1000

# تفسير الفشل:
# - data mismatch → الـ transfer يعمل لكن البيانات تالفة
# - src_off/dst_off → يشير لمشكلة alignment
# - الحل: تحقق من الـ burst size وإعدادات الـ coherency في الـ DT
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: بوابة صناعية على RK3562 — UART بالـ DMA لا يعمل عند الإقلاع

#### العنوان
**الـ `of_dma_request_slave_channel` يرجع `-EPROBE_DEFER` بشكل لانهائي على UART3**

#### السياق
منتج: بوابة صناعية Modbus RTU تعمل على Rockchip RK3562.
الـ UART3 مربوط بـ RS-485 transceiver، والـ driver يستخدم DMA لتقليل تأخر الـ interrupt.
الـ kernel هو 6.6-LTS، والـ DMA controller هو `rockchip-dma`.

#### المشكلة
الـ driver يُسجَّل بنجاح، لكن:
```
serial 100a0000.serial: dma channel rx not available, running without rx DMA
```
يظهر هذا بشكل دائم، وأداء الاستقبال يسوء عند تحميل عالٍ.

#### التحليل
**الـ `of_dma_request_slave_channel`** في `of_dma.h` يُعرَّف هكذا:
```c
extern struct dma_chan *of_dma_request_slave_channel(struct device_node *np,
                                                     const char *name);
```
هذه الدالة تمشي على قائمة `of_dma_controllers` وتبحث عن controller مطابق عبر `of_dma_xlate`.
عندما يُستدعى قبل أن يُسجّل الـ DMA controller نفسه عبر `of_dma_controller_register`، ترجع `ERR_PTR(-EPROBE_DEFER)`.

لكن المشكلة هنا أعمق: الـ DT node للـ UART يحتوي على خطأ في اسم الـ channel:

```dts
/* خطأ: الاسم لا يطابق ما يتوقعه الـ driver */
dmas = <&dmac 20>, <&dmac 21>;
dma-names = "tx", "rx_err";   /* يجب أن يكون "rx" وليس "rx_err" */
```

**الـ `of_dma_request_slave_channel`** تبحث عن الاسم بالضبط:
```c
/* داخل of_dma.c — تستخدم of_property_match_string ثم of_parse_phandle_with_args */
```
عندما لا يُوجد `"rx"` في `dma-names`، ترجع `ERR_PTR(-ENODEV)`، لكن الـ UART driver يُفسّرها كـ defer وينتهي بـ fallback إلى PIO mode.

#### الحل

**1. تصحيح الـ DT:**
```dts
/* arch/arm64/boot/dts/rockchip/rk3562-gateway.dts */
&uart3 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&uart3_xfer>;
    dmas = <&dmac 20>, <&dmac 21>;
    dma-names = "tx", "rx";   /* تصحيح الاسم */
};
```

**2. التحقق من الـ DT المحلل في runtime:**
```bash
# التحقق من الـ node المحلل
cat /proc/device-tree/serial@100a0000/dma-names | tr '\0' '\n'

# التحقق من تسجيل الـ DMA controller
ls /sys/bus/platform/drivers/rockchip-dma/
```

**3. تتبع الـ xlate:**
```bash
# تفعيل dynamic debug
echo "file of_dma.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg | grep of_dma
```

#### الدرس المستفاد
**الـ `of_dma_request_slave_channel`** تعتمد على مطابقة نصية دقيقة للاسم في `dma-names`. أي خطأ إملائي في الـ DT يُنتج `ENODEV` صامتاً. دائماً تحقق من الاسم الذي يتوقعه الـ driver في كوده (عادةً `"tx"` و`"rx"`) مقابل ما هو موجود في الـ DTS.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI audio عبر DMA يتوقف

#### العنوان
**الـ `of_dma_xlate_by_chan_id` يُعيد channel خاطئ لـ I2S/HDMI audio**

#### السياق
منتج: Android TV Box يستخدم Allwinner H616.
الـ audio path: I2S controller → HDMI TX → speaker.
الـ DMA controller: `sun6i-dma` مع 16 channel.
المشكلة ظهرت بعد تحديث الـ kernel من 5.15 إلى 6.1.

#### المشكلة
الصوت يصدر بشكل متقطع أو مشوه تماماً عند تشغيل محتوى 4K. الـ log يُظهر:
```
sun4i-i2s 5096000.i2s: DMA prepare playback failed: -22
```

#### التحليل
الـ `of_dma_xlate_by_chan_id` تُعرَّف في `of_dma.h`:
```c
extern struct dma_chan *of_dma_xlate_by_chan_id(struct of_phandle_args *dma_spec,
        struct of_dma *ofdma);
```

هذه الدالة تأخذ `dma_spec->args[0]` كـ channel ID وتبحث عنه في الـ controller.
الـ `of_phandle_args` مُعرَّف في `linux/of.h`:
```c
struct of_phandle_args {
    struct device_node *np;   /* pointer للـ DMA controller node */
    int args_count;           /* عدد الـ arguments */
    uint32_t args[MAX_PHANDLE_ARGS];  /* args[0] = channel ID */
};
```

في الـ DT القديم (kernel 5.15):
```dts
dmas = <&dma 6>;    /* channel 6 = I2S0 TX */
```

في kernel 6.1 تغيّر ترقيم الـ channels في `sun6i-dma`:
```dts
/* الصواب للـ kernel 6.1 */
dmas = <&dma 14>;   /* channel ID تغيّر بسبب إعادة تنظيم الـ driver */
```

`of_dma_xlate_by_chan_id` تُمرّر الـ ID مباشرة إلى `dma_request_channel`، وإذا كان خاطئاً تأخذ channel مختلف أو ترجع `NULL`.

#### التحليل العميق
```bash
# فحص الـ DMA channels المتاحة
cat /sys/kernel/debug/dmaengine/summary

# فحص الـ channel المطلوب فعلياً
grep -r "I2S\|i2s" /sys/kernel/debug/dmaengine/
```

الـ `of_dma_data` في `struct of_dma` يحمل pointer للـ controller-private data، وعند استخدام `of_dma_xlate_by_chan_id`، يُفترض أن `args[0]` يُعطي index مباشراً في جدول الـ channels — وهذا يختلف بين إصدارات الـ driver.

#### الحل

```dts
/* arch/arm64/boot/dts/allwinner/sun50i-h616-tvbox.dts */
&i2s0 {
    status = "okay";
    /* تحديث channel IDs لتتوافق مع sun6i-dma في kernel 6.1 */
    dmas = <&dma 14>, <&dma 15>;
    dma-names = "tx", "rx";
};
```

**التحقق:**
```bash
# بعد التحديث — التأكد من allocation صحيح
echo "file sun6i-dma.c +p" > /sys/kernel/debug/dynamic_debug/control
aplay -D hw:0,0 test.wav 2>&1 | head -20
```

#### الدرس المستفاد
**الـ `of_dma_xlate_by_chan_id`** لا تعمل abstraction — تمرر الـ ID حرفياً للـ controller. عند upgrade الـ kernel، تحقق دائماً من changelog الـ DMA driver للسوك المستخدم. الـ `of_dma_simple_xlate` أكثر أماناً لأنها تستخدم الـ `dma_cap_mask_t` من `of_dma_filter_info`.

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — SPI بالـ DMA يعلق kernel

#### العنوان
**الـ `of_dma_controller_register` يُستدعى مرتين فيسبب panic**

#### السياق
منتج: IoT node لقراءة حساسات ADC عبر SPI على STM32MP157C-DK2.
الـ DMA controller: STM32 MDMA + DMA1/DMA2.
المنتج يستخدم custom kernel module لـ SPI accelerator.

#### المشكلة
```
kernel BUG at kernel/irq/manage.c:XXX
Call trace:
  of_dma_controller_register+0x...
  stm32_mdma_probe+0x...
```

الـ kernel يـcrash عند تحميل الـ module للمرة الثانية بعد `rmmod`/`insmod`.

#### التحليل
الـ `of_dma_controller_register` في `of_dma.h`:
```c
extern int of_dma_controller_register(struct device_node *np,
        struct dma_chan *(*of_dma_xlate)(struct of_phandle_args *, struct of_dma *),
        void *data);
```

هذه الدالة تُنشئ `struct of_dma` جديد وتُضيفه إلى **قائمة عالمية** `of_dma_list`:

```c
/* من of_dma.c — المنطق الداخلي */
struct of_dma *ofdma = kmalloc(sizeof(*ofdma), GFP_KERNEL);
ofdma->of_node = of_node_get(np);        /* زيادة reference count */
ofdma->of_dma_xlate = of_dma_xlate;
ofdma->of_dma_data = data;
list_add_tail(&ofdma->of_dma_controllers, &of_dma_list);
```

المشكلة: الـ module الـ custom نسي استدعاء `of_dma_controller_free` في `remove()`:

```c
/* كود خاطئ في الـ module */
static int my_dma_probe(struct platform_device *pdev)
{
    of_dma_controller_register(pdev->dev.of_node,
                                my_xlate_fn, priv);
    return 0;
}

static int my_dma_remove(struct platform_device *pdev)
{
    /* BUG: لم يستدعِ of_dma_controller_free */
    return 0;
}
```

عند `insmod` مرة ثانية، يُضاف entry جديد للـ node نفسه في القائمة، مع أن `of_node` لا يزال مُستخدماً من الـ entry القديم، مما يُفسد الـ reference counting ويُسبب use-after-free.

#### الحل

```c
/* تصحيح الـ remove function */
static int my_dma_remove(struct platform_device *pdev)
{
    /* يجب دائماً تحرير الـ registration عند الـ remove */
    of_dma_controller_free(pdev->dev.of_node);
    return 0;
}
```

**التحقق قبل الـ insmod:**
```bash
# التأكد من عدم وجود تسجيل مسبق
grep -r "my_dma" /sys/kernel/debug/dmaengine/
ls /proc/device-tree/ | grep dma
```

**فحص تسريب الـ of_node:**
```bash
# kmemleak للكشف عن memory leak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak | grep of_dma
```

#### الدرس المستفاد
**الـ `of_dma_controller_register`** يزيد الـ reference count على `device_node` عبر `of_node_get`. المقابل **الـ `of_dma_controller_free`** يُنقصه. في كل `probe()` يجب أن يقابله `of_dma_controller_free()` في `remove()`. هذا النمط symmetrical إلزامي في كل kernel driver يستخدم OF DMA registration.

---

### السيناريو 4: Automotive ECU على i.MX8QM — DMA Router لـ ASRC audio mixing

#### العنوان
**الـ `of_dma_router_register` والـ `of_dma_route_allocate` لا تُنتج channel صحيح لـ ASRC**

#### السياق
منتج: ECU ترفيهي في سيارة يستخدم NXP i.MX8QM.
الـ audio subsystem: ASRC (Asynchronous Sample Rate Converter) مربوط بـ SDMA3.
الـ DMA router يُستخدم لتوجيه audio streams من مصادر متعددة إلى ASRC.

#### المشكلة
عند تشغيل مسارات audio متعددة في نفس الوقت (navigation + media + phone):
```
imx-audmix sound: Failed to route DMA channel for ASRC pair 2
asoc-audio-graph-card sound: ASoC: no backend DAIs enabled for fe-dai-2
```

#### التحليل
**الـ `of_dma_router_register`** في `of_dma.h`:
```c
extern int of_dma_router_register(struct device_node *np,
        void *(*of_dma_route_allocate)(struct of_phandle_args *, struct of_dma *),
        struct dma_router *dma_router);
```

وعكسه:
```c
#define of_dma_router_free of_dma_controller_free
```

الـ `of_dma_route_allocate` callback مختلف عن `of_dma_xlate`:
- `of_dma_xlate`: يرجع `struct dma_chan *` مباشرة
- `of_dma_route_allocate`: يرجع `void *` (private router data) ثم الـ framework يبني channel فوقه

الـ DT للـ ASRC:
```dts
/* i.MX8QM ASRC DT — الخطأ في عدد الـ args */
asrc0: asrc@59000000 {
    dmas = <&sdma3 2 23 1 0>,   /* pair 0 — 4 args */
           <&sdma3 3 23 1 0>,   /* pair 1 */
           <&sdma3 4 23 1 0>;   /* pair 2 — فقط 3 channels لـ 3 pairs */
    dma-names = "pair0", "pair1", "pair2";
    fsl,asrc-rate = <8000>;
};
```

`of_dma_route_allocate` تُحلّل `dma_spec->args` وتتوقع 4 arguments لكل channel في SDMA3:
```c
/* من fsl_sdma router — مثال توضيحي */
static void *sdma_route_allocate(struct of_phandle_args *dma_spec,
                                  struct of_dma *ofdma)
{
    if (dma_spec->args_count != 4)   /* يتوقع: channel, script, priority, flags */
        return ERR_PTR(-EINVAL);
    /* ... */
}
```

عند نفاد الـ SDMA channels المتاحة (كل pair تحتاج Tx + Rx = 6 channels لـ 3 pairs)، يفشل الـ allocation بصمت.

#### الحل

**1. التحقق من عدد الـ SDMA channels:**
```bash
# فحص الـ channels المستخدمة
cat /sys/kernel/debug/dmaengine/summary | grep sdma3

# فحص ما إذا كانت الـ channels تُحرَّر بعد الاستخدام
watch -n1 'cat /sys/kernel/debug/dmaengine/summary | grep -c "in-use"'
```

**2. تصحيح الـ DT لضمان كفاية الـ channels:**
```dts
/* زيادة الـ SDMA channels المتاحة أو تقليل الـ ASRC pairs */
&sdma3 {
    fsl,sdma-ram-script-name = "imx/sdma/sdma-imx7d.bin";
};

&asrc0 {
    /* استخدام channel ranges منفصلة لتجنب التعارض */
    dmas = <&sdma3 2 23 1 0>,
           <&sdma3 4 23 1 0>,
           <&sdma3 6 23 1 0>;
};
```

**3. إضافة error handling في الـ driver:**
```c
/* فحص الـ router allocation error */
struct dma_chan *chan = of_dma_request_slave_channel(dev->of_node, "pair2");
if (IS_ERR(chan)) {
    dev_err(dev, "pair2 DMA failed: %ld\n", PTR_ERR(chan));
    /* fallback إلى software mixing */
}
```

#### الدرس المستفاد
**الـ `of_dma_router_register`** مناسب للـ DMA routers مثل ASRC في i.MX التي تحتاج intermediate allocation قبل channel assignment. الـ `of_dma_route_allocate` يجب أن يتحقق من `args_count` بصرامة. في بيئات automotive حيث الـ streams متعددة، يجب حساب الـ DMA channel budget مسبقاً.

---

### السيناريو 5: Custom Board Bring-up على AM62x — I2C بالـ DMA لا يظهر في sysfs

#### العنوان
**`CONFIG_DMA_OF` مُعطّل فيسبب fallback لكل الـ `of_dma_*` functions**

#### السياق
منتج: custom industrial board على Texas Instruments AM62x (Sitara).
الـ I2C3 مربوط بـ EEPROM ومستشعرات متعددة.
فريق الـ BSP يبني kernel مخصص لأول مرة من defconfig.

#### المشكلة
الـ I2C يعمل لكن بأداء ضعيف جداً، والـ DMA لا يُستخدم:
```bash
$ cat /sys/kernel/debug/dmaengine/summary
# لا يظهر أي channel مرتبط بـ i2c3
```

لا توجد رسائل خطأ — الـ system يعمل بـ PIO mode بصمت تام.

#### التحليل
الـ `of_dma.h` يحتوي على guard مهم جداً:

```c
#ifdef CONFIG_DMA_OF
extern int of_dma_controller_register(...);
extern void of_dma_controller_free(...);
/* ... باقي الدوال الحقيقية */
#else
/* stub implementations */
static inline int of_dma_controller_register(...) { return -ENODEV; }
static inline void of_dma_controller_free(...) {}
static inline struct dma_chan *of_dma_request_slave_channel(...) {
    return ERR_PTR(-ENODEV);
}
static inline struct dma_chan *of_dma_simple_xlate(...) { return NULL; }
#define of_dma_xlate_by_chan_id NULL
#endif
```

عندما يكون `CONFIG_DMA_OF=n`، **كل** هذه الدوال تصبح stubs صامتة. الـ `of_dma_request_slave_channel` ترجع `ERR_PTR(-ENODEV)` وأغلب drivers تقرأ هذا وتنتقل لـ PIO mode دون تحذير.

**فحص الـ config:**
```bash
zcat /proc/config.gz | grep DMA_OF
# CONFIG_DMA_OF is not set  ← هذا هو السبب
```

الـ `.config` الناتج من `am62x_defconfig` يتطلب تفعيل `DMA_OF` يدوياً لأنه لا يُفعَّل تلقائياً في بعض الـ defconfigs.

#### الحل

**1. تفعيل `CONFIG_DMA_OF`:**
```bash
# في menuconfig
make ARCH=arm64 menuconfig
# Device Drivers → DMA Engine support → DMA OF helpers [*]

# أو مباشرة في .config
echo "CONFIG_DMA_OF=y" >> arch/arm64/configs/am62x_custom_defconfig
```

**2. التأكد من dependencies:**
```bash
# CONFIG_DMA_OF يعتمد على CONFIG_OF و CONFIG_DMADEVICES
zcat /proc/config.gz | grep -E "^CONFIG_(OF|DMADEVICES|DMA_OF)="
# يجب أن تكون الثلاثة =y
```

**3. التحقق بعد إعادة البناء:**
```bash
# بعد build جديد مع CONFIG_DMA_OF=y
dmesg | grep -i "dma.*i2c\|i2c.*dma"
cat /sys/kernel/debug/dmaengine/summary | grep i2c
```

**4. التحقق في الـ DT أيضاً:**
```dts
/* k3-am625-custom.dts */
&i2c3 {
    status = "okay";
    clock-frequency = <400000>;
    /* هذه الـ properties تعمل فقط إذا كان CONFIG_DMA_OF=y */
    dmas = <&main_udmap 0xc200>, <&main_udmap 0x4200>;
    dma-names = "tx", "rx";
};
```

#### الدرس المستفاد
**الـ `#ifdef CONFIG_DMA_OF`** في `of_dma.h` هو أول مكان تنظر فيه عند bring-up جديد. إذا كان `CONFIG_DMA_OF=n`، كل الـ of_dma infrastructure تصبح no-ops صامتة — لا panic، لا error واضح، فقط أداء منخفض. في أي custom BSP، أضف `CONFIG_DMA_OF=y` إلى الـ defconfig مع `CONFIG_DMADEVICES=y` كـ baseline لأي SoC حديث.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقالة | الوصف |
|---------|-------|
| [of: Add generic device tree DMA helpers](https://lwn.net/Articles/487197/) | المقالة الأساسية التي تُغطي تقديم `of_dma` helpers للـ Device Tree، وتشرح تسجيل الـ DMA controllers وآلية الـ `of_dma_xlate` |
| [dmaengine redux](https://lwn.net/Articles/307264/) | نقاش مهم حول إعادة هيكلة الـ dmaengine subsystem وأسباب التصميم |
| [Dancing the DMA two-step](https://lwn.net/Articles/997563/) | تغطية حديثة لتطور الـ DMA mapping API في النواة |
| [A new DMA-mapping API](https://lwn.net/Articles/1020437/) | مقالة تُغطي الـ API الجديدة لإدارة IOVA space بشكل مباشر |
| [Preview of DMA mask changes](https://lwn.net/Articles/561854/) | تغييرات الـ DMA mask وتأثيرها على الـ Device Tree bindings |
| [dmaengine: enhance dmaengine to support DMA device hotplug](https://lwn.net/Articles/497107/) | دعم الـ hotplug للـ DMA controllers بعد تقديم `of_dma` |
| [CMA & device tree, once again](https://lwn.net/Articles/605348/) | تكامل الـ CMA مع الـ Device Tree في سياق الـ DMA |

---

### التوثيق الرسمي للنواة (`Documentation/`)

الملفات التالية موجودة في شجرة مصدر النواة:

```
Documentation/driver-api/dmaengine/
├── client.rst          # DMA Engine API Guide — كيفية استخدام OF helpers من جانب العميل
├── provider.rst        # DMAengine controller documentation — تسجيل الـ controller
├── index.rst           # فهرس الـ DMAEngine documentation
└── dmatest.rst         # اختبار الـ DMA channels

Documentation/devicetree/bindings/dma/
└── *.yaml              # Device Tree bindings لمختلف الـ DMA controllers

Documentation/core-api/
└── dma-api-howto.rst   # Dynamic DMA mapping Guide
└── dma-api.rst         # مرجع شامل لـ DMA API
```

الروابط المباشرة للتوثيق المنشور:

- [DMAengine controller documentation (provider.rst)](https://static.lwn.net/kerneldoc/driver-api/dmaengine/provider.html)
- [DMA Engine API Guide (client.rst)](https://static.lwn.net/kerneldoc/driver-api/dmaengine/client.html)
- [DMAEngine documentation index](https://static.lwn.net/kerneldoc/driver-api/dmaengine/index.html)
- [Dynamic DMA mapping Guide](https://docs.kernel.org/core-api/dma-api-howto.html)

---

### الملفات المصدرية ذات الصلة في النواة

الـ header الذي تمت دراسته:

```
include/linux/of_dma.h          # التعريفات والـ structs الرئيسية
```

التنفيذ والملفات المرتبطة:

```
drivers/dma/of-dma.c            # تنفيذ of_dma_controller_register و of_dma_request_slave_channel
include/linux/dmaengine.h       # تعريفات dma_chan و dma_cap_mask_t و dma_filter_fn
drivers/dma/dmaengine.c         # النواة الأساسية لـ dmaengine framework
Documentation/devicetree/bindings/dma/dma.txt   # الـ binding spec للـ dmas property
```

الرابط المباشر للتنفيذ على GitHub:
- [drivers/dma/of-dma.c على torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/dma/of-dma.c)
- [Documentation/devicetree/bindings/dma](https://github.com/torvalds/linux/tree/master/Documentation/devicetree/bindings/dma)

---

### Commits مهمة في تاريخ `of_dma`

الـ commit الأصلي الذي أدخل الـ `of_dma` helpers كان بواسطة **Benoit Cousson** و **Matt Porter** لدعم الـ OMAP DMA controller تحت Device Tree. البحث في git log:

```bash
# للبحث عن تاريخ الملف
git log --follow --oneline include/linux/of_dma.h

# للبحث عن الـ commit الأصلي
git log --all --grep="of_dma" --oneline drivers/dma/of-dma.c | tail -5
```

نقطة بحث مفيدة في patchwork:
- [RFC,v12,7/7: dma: mpc512x: register for device tree channel lookup](https://patchwork.kernel.org/project/linux-dmaengine/patch/1398261209-5578-8-git-send-email-a13xp0p0v88@gmail.com/) — مثال على تسجيل driver حقيقي مع `of_dma_controller_register`

---

### نقاشات Mailing List

للاطلاع على نقاشات الـ mailing list الخاصة بـ `of_dma`:

- **القائمة الرئيسية**: `linux-dmaengine@vger.kernel.org`
- **الأرشيف**: [https://lore.kernel.org/linux-dmaengine/](https://lore.kernel.org/linux-dmaengine/)
- **بحث مباشر**: [https://lore.kernel.org/linux-dmaengine/?q=of_dma](https://lore.kernel.org/linux-dmaengine/?q=of_dma)

---

### Kernelnewbies.org

صفحات تُوثق إضافات الـ DMA engine عبر إصدارات النواة:

| الصفحة | ما تحتويه |
|--------|-----------|
| [Linux_4.3](https://kernelnewbies.org/Linux_4.3) | إضافات DMA engine في kernel 4.3 (sun4i، ti-dma-crossbar) |
| [Linux_5.10](https://kernelnewbies.org/Linux_5.10) | تحديثات الـ DMA subsystem في kernel 5.10 |
| [Linux_5.2](https://kernelnewbies.org/Linux_5.2) | ميزات DMA الجديدة في kernel 5.2 |
| [Linux_5.8](https://kernelnewbies.org/Linux_5.8) | تحسينات DMA في kernel 5.8 |

---

### Elinux.org

| الصفحة | الموضوع |
|--------|---------|
| [BeagleBone PRU DMA](https://elinux.org/BeagleBoard/GSoC/BeagleBone_PRU_DMA) | استخدام `of_dma` مع EDMA على AM335x SoC — مثال عملي حقيقي |
| [Tests:I2C-core-DMA](https://elinux.org/Tests:I2C-core-DMA) | اختبار الـ DMA buffers مع I2C على Renesas R-Car |
| [DMA Optimization in Encryption](https://elinux.org/images/d/d0/Ribiere-Dma_Optimization_BayLibre_Guillene_v4.pdf) | تحسين الـ DMA configuration في حالات الاستخدام المضمّنة |
| [Freescale MPC5200](https://elinux.org/Freescale_MPC5200) | مثال على DMA controller قديم مع Device Tree integration |

---

### توثيق الـ STM32 كمثال عملي

الـ [DMA device tree configuration على STM32MPU wiki](https://wiki.st.com/stm32mpu/wiki/DMA_device_tree_configuration) يُقدم مثالاً حقيقياً ومفصلاً لكيفية استخدام `dmas` و `dma-names` properties في الـ Device Tree مع `of_dma_request_slave_channel`.

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)

| الفصل | الموضوع |
|-------|---------|
| Chapter 15: Memory Mapping and DMA | الأساس النظري للـ DMA في Linux، الـ DMA mapping API، coherent vs streaming |

الرابط المجاني: [Chapter 15 على O'Reilly](https://www.oreilly.com/library/view/linux-device-drivers/0596005903/ch15.html)

> **ملاحظة**: LDD3 لا يتناول `of_dma` مباشرةً لأنه سابق لـ Device Tree، لكن فهم الـ DMA API الأساسية ضروري قبل دراسة `of_dma`.

#### Linux Kernel Development — Robert Love (3rd Edition)

- الفصل الخاص بالـ Memory Management يُغطي الـ DMA zones وإدارة الذاكرة المرتبطة
- لا يتناول `of_dma` مباشرةً لكنه يُرسّخ فهم الـ memory model

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- الفصل المخصص للـ Bootloader وتمرير الـ Device Tree
- الفصل الخاص بالـ Board Support Package يُغطي تكامل الـ DMA controllers مع الـ platform drivers

#### Professional Linux Kernel Architecture — Wolfgang Mauerer

- Part IV: Memory Management — الفصل الخاص بـ DMA memory والـ coherency

---

### مصطلحات البحث

للعثور على معلومات إضافية استخدم الكلمات التالية:

```
of_dma_controller_register
of_dma_request_slave_channel
of_dma_xlate
dma-names device tree binding
DMA phandle specifier Linux
dmaengine slave channel request
CONFIG_DMA_OF
of_phandle_args DMA
dma_router Linux kernel
Device Tree DMA binding specification
```

---

### ملخص المصادر الأساسية

```
الأولوية  المصدر
─────────────────────────────────────────────────────────
★★★★★    lwn.net/Articles/487197  — المقالة الأصلية لـ of_dma helpers
★★★★★    Documentation/driver-api/dmaengine/provider.rst
★★★★★    drivers/dma/of-dma.c    — التنفيذ الفعلي
★★★★     Documentation/driver-api/dmaengine/client.rst
★★★★     Documentation/core-api/dma-api-howto.rst
★★★      LDD3 Chapter 15
★★★      elinux.org BeagleBone PRU DMA
★★       kernelnewbies.org Linux_4.3
★★       lore.kernel.org linux-dmaengine archives
```
## Phase 8: Writing simple module

### الهدف

سنكتب وحدة kernel تراقب استدعاءات `of_dma_request_slave_channel` عبر **kprobe**، وهي الدالة المُصدَّرة المسؤولة عن طلب قناة DMA من Device Tree. عند كل استدعاء سنطبع اسم الـ `device_node` والـ `name` المطلوب، مما يكشف أي driver يطلب أي قناة DMA في وقت التشغيل.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_of_dma.c
 * Trace calls to of_dma_request_slave_channel via kprobe.
 * Prints the device_node full_name and channel name on every call.
 */

#include <linux/module.h>      /* module_init / module_exit / MODULE_* macros */
#include <linux/kernel.h>      /* pr_info / KERN_INFO */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/of.h>          /* struct device_node — has full_name field */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("Trace of_dma_request_slave_channel calls with kprobe");

/* ---------------------------------------------------------------
 * pre-handler: called just before of_dma_request_slave_channel
 *
 * Prototype of target:
 *   struct dma_chan *of_dma_request_slave_channel(
 *                       struct device_node *np,
 *                       const char *name);
 *
 * On x86-64:  np   -> regs->di  (first  arg)
 *             name -> regs->si  (second arg)
 * On ARM64:   np   -> regs->regs[0]
 *             name -> regs->regs[1]
 * We use regs_get_kernel_argument() which is arch-agnostic.
 * --------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Extract the two pointer arguments in an arch-agnostic way */
    struct device_node *np   =
        (struct device_node *)regs_get_kernel_argument(regs, 0);
    const char         *name =
        (const char *)regs_get_kernel_argument(regs, 1);

    /* Guard against NULL — early boot or stub callers may pass NULL */
    if (!np || !name)
        return 0;

    pr_info("of_dma: request — node=\"%s\"  channel=\"%s\"\n",
            np->full_name ? np->full_name : "(null)",
            name);

    return 0; /* 0 = continue normal execution */
}

/* ---------------------------------------------------------------
 * The kprobe descriptor — we hook by symbol name so no address
 * arithmetic is needed; the kernel resolves it at register time.
 * --------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "of_dma_request_slave_channel",
    .pre_handler = handler_pre,
};

/* ---------------------------------------------------------------
 * module_init: register the kprobe so the hook becomes active
 * --------------------------------------------------------------- */
static int __init kprobe_of_dma_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        /* Common failure: CONFIG_DMA_OF not set, or symbol not exported */
        pr_err("of_dma kprobe: register failed, err=%d\n", ret);
        return ret;
    }

    pr_info("of_dma kprobe: planted at %pS\n", kp.addr);
    return 0;
}

/* ---------------------------------------------------------------
 * module_exit: MUST unregister before the module's .text is freed,
 * otherwise a call to of_dma_request_slave_channel after rmmod
 * would jump into unmapped memory and crash the kernel.
 * --------------------------------------------------------------- */
static void __exit kprobe_of_dma_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("of_dma kprobe: removed from %pS\n", kp.addr);
}

module_init(kprobe_of_dma_init);
module_exit(kprobe_of_dma_exit);
```

---

### شرح كل قسم

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | توفير `module_init` / `module_exit` والـ macros الخاصة بالوحدات |
| `linux/kprobes.h` | تعريف `struct kprobe` ودوال `register_kprobe` / `unregister_kprobe` |
| `linux/of.h` | تعريف `struct device_node` للوصول إلى `full_name` داخل الـ callback |

**الـ** `linux/kernel.h` يُضمَّن ضمنيًا عبر `linux/module.h` في معظم الـ configs، لكن إدراجه صريحًا يضمن توفر `pr_info` بشكل مستقل عن ترتيب التضمين.

---

#### الـ `handler_pre` — قلب الوحدة

`regs_get_kernel_argument(regs, N)` هي الطريقة المحمولة لقراءة الـ argument رقم N من سجلات الـ CPU، وتعمل على x86-64 وARM64 وRISC-V دون تعديل. نتحقق من `NULL` قبل الوصول إلى الـ pointers لأن بعض الـ drivers في مرحلة التهيئة المبكرة قد تمرر `NULL` اختبارًا.

---

#### `struct kprobe kp`

تحديد الـ hook بـ `symbol_name` بدلًا من عنوان ثابت يجعل الوحدة قابلة للتشغيل على أي kernel دون الحاجة لمعرفة عنوان الدالة مسبقًا؛ الـ kernel يحله في وقت التسجيل عبر `kallsyms`.

---

#### `module_init` و `module_exit`

`register_kprobe` يزرع **breakpoint** (int3 على x86، BRK على ARM64) عند نقطة الدخول للدالة المستهدفة، ويُفعّل الـ callback عند كل ضربة. `unregister_kprobe` في الـ exit إلزامي: إزالة الـ breakpoint قبل تحرير `.text` الخاص بالوحدة يمنع crash في حال استدعاء الدالة المستهدفة بعد `rmmod`.

---

### بناء الوحدة وتشغيلها

```bash
# Makefile بسيط في نفس مجلد kprobe_of_dma.c
obj-m += kprobe_of_dma.o

# بناء
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل
sudo insmod kprobe_of_dma.ko

# مراقبة المخرجات (على نظام يحتوي DMA controllers مثل BeagleBone / RPi)
sudo dmesg -w | grep of_dma

# إزالة
sudo rmmod kprobe_of_dma
```

**مثال مخرجات متوقعة على نظام ARM مدمج:**

```
of_dma kprobe: planted at of_dma_request_slave_channel+0x0/0x60
of_dma: request — node="/soc/serial@4806a000"  channel="tx"
of_dma: request — node="/soc/serial@4806a000"  channel="rx"
of_dma: request — node="/soc/mmc@481d8000"     channel="tx"
of_dma: request — node="/soc/mmc@481d8000"     channel="rx"
of_dma kprobe: removed from of_dma_request_slave_channel+0x0/0x60
```

كل سطر يكشف أي `device_node` (UART, MMC, SPI...) يطلب قناة DMA بالاسم المحدد في Device Tree، مما يُسهّل تشخيص مشاكل تعارض الـ DMA channels أو تتبع ترتيب تهيئة الـ drivers.
