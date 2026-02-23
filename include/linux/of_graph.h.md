## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

**`include/linux/of_graph.h`** هو واجهة برمجية (API header) تنتمي إلى نظام **Open Firmware / Flattened Device Tree** داخل نواة Linux، وتحديداً إلى قسم **OF Graph** — وهو آلية لوصف **الروابط الفيزيائية بين الأجهزة** في Device Tree بطريقة منظمة وقابلة للتنقل.

**المشرف على هذا الـ subsystem:** Rob Herring و Saravana Kannan
**القائمة البريدية:** `devicetree@vger.kernel.org`
**الملفات المنتمية:** `drivers/of/` و `include/linux/of*.h`

---

### المشكلة التي يحلها — قصة واقعية

تخيّل لوحة مدمجة (مثل Raspberry Pi أو Samsung Exynos) فيها:

- **كاميرا OV5640** (حساس صور)
- **وحدة DCMI** (Digital Camera Interface — تستقبل الصور)
- **معالج DSI** (يرسل الصورة للشاشة)
- **شاشة HDMI أو LCD**

كل هذه الأجهزة موصولة ببعضها بكابلات أو buses فيزيائية. لكن النواة لا تعرف من أين تبدأ لتصف هذه الروابط.

**قبل OF Graph** كانت المشكلة:
- كل driver يكتب كوداً خاصاً به لاكتشاف الجهاز المتصل به.
- لا يوجد معيار موحّد لوصف "هذا الـ sensor موصول بهذا الـ ISP من الـ port رقم 0".
- الكود ممتلئ بـ hardcoded names وأرقام.

**الحل — OF Graph:**
تُعرَّف الروابط مرة واحدة في **Device Tree Source (DTS)**، ثم أي driver يستطيع أن يسأل: "من المتصل بي على port رقم X?" ويحصل على إجابة برمجية جاهزة.

---

### القصة كاملة — كيف تعمل الأشياء

#### 1. في ملف DTS (وصف الهاردوير)

```
/* حساس الكاميرا */
ov5640: camera@3c {
    port {
        ov5640_0: endpoint {
            remote-endpoint = <&dcmi_0>;   /* موصول بـ DCMI */
        };
    };
};

/* وحدة استقبال الكاميرا */
dcmi: dcmi@50050000 {
    port {
        dcmi_0: endpoint {
            remote-endpoint = <&ov5640_0>; /* موصول بالكاميرا */
        };
    };
};
```

هذا مأخوذ فعلياً من ملف `arch/arm/boot/dts/st/stm32mp157c-ev1.dts`.

#### 2. في الكود (يستخدم of_graph.h)

```c
/* driver يريد أن يعرف من المتصل به */
struct device_node *ep = of_graph_get_endpoint_by_regs(dev->of_node, 0, -1);
struct device_node *remote_dev = of_graph_get_remote_port_parent(ep);
/* الآن remote_dev يشير إلى الجهاز الآخر في الـ Device Tree */
```

---

### هيكل OF Graph — بصرياً

```
device_node (كاميرا)
└── ports
    └── port@0          ← port (منفذ فيزيائي)
        └── endpoint@0  ← endpoint (طرف الوصلة)
            └── remote-endpoint → endpoint@0 في DCMI

device_node (DCMI)
└── port
    └── endpoint@0      ← endpoint (الطرف الآخر)
        └── remote-endpoint → endpoint@0 في الكاميرا
```

الـ **port** = المنفذ الفيزيائي (مثل: منفذ MIPI CSI-2).
الـ **endpoint** = الوصلة المحددة على هذا المنفذ (لو في منفذ واحد أكثر من اتصال).
الـ **remote-endpoint** = مؤشر (phandle) للطرف الآخر.

---

### ماذا يوفر هذا الـ Header تحديداً؟

#### البنية الأساسية

```c
struct of_endpoint {
    unsigned int port;              /* رقم الـ port (من خاصية reg) */
    unsigned int id;                /* رقم الـ endpoint */
    const struct device_node *local_node; /* إشارة لـ node الـ endpoint */
};
```

#### دوال القراءة والتنقل

| الدالة | الوظيفة |
|---|---|
| `of_graph_is_present()` | هل يوجد graph أصلاً في هذا الـ node؟ |
| `of_graph_parse_endpoint()` | اقرأ بيانات endpoint معين |
| `of_graph_get_port_by_id()` | ابحث عن port برقمه |
| `of_graph_get_next_endpoint()` | انتقل للـ endpoint التالي |
| `of_graph_get_next_port()` | انتقل للـ port التالي |
| `of_graph_get_remote_endpoint()` | ما هو الـ endpoint المتصل بهذا؟ |
| `of_graph_get_remote_port_parent()` | ما هو الجهاز المتصل؟ (الأهم) |
| `of_graph_get_remote_node()` | ابحث عن الجهاز بعيد باستخدام port/endpoint رقم |
| `of_graph_get_endpoint_count()` | كم عدد الـ endpoints في الجهاز؟ |
| `of_graph_get_port_count()` | كم عدد الـ ports في الجهاز؟ |

#### ماكروات التكرار

```c
/* المرور على كل endpoint في الجهاز */
for_each_endpoint_of_node(parent, child) { ... }

/* المرور على كل port */
for_each_of_graph_port(parent, child) { ... }

/* المرور على كل endpoint داخل port */
for_each_of_graph_port_endpoint(parent, child) { ... }
```

---

### أين يُستخدم هذا في الواقع؟

**الـ subsystems التي تستخدم `of_graph.h` بكثرة:**

- **Media (V4L2/Camera):** كاميرات MIPI CSI-2، ISP، DCMI — مثل `drivers/media/platform/st/stm32/stm32-dcmi.c`
- **DRM/GPU:** اتصال LTDC ↔ DSI ↔ شاشة — مثل `drivers/gpu/drm/omapdrm/`
- **PHY drivers:** تصف من يستخدم هذا الـ PHY — مثل Qualcomm USB/PCIe PHYs
- **Sound (ASoC):** graph card، اتصال codecs بـ CPUs صوت
- **Networking:** Ethernet switches مع ports متعددة

---

### الملفات المكونة لهذا الـ Subsystem

#### Header الرئيسي
| الملف | الوظيفة |
|---|---|
| `include/linux/of_graph.h` | **هذا الملف** — API و structs |
| `include/linux/of.h` | الـ core: تعريف `device_node`، `property`، `of_phandle_args` |

#### التنفيذ
| الملف | الوظيفة |
|---|---|
| `drivers/of/property.c` | **تنفيذ كل دوال `of_graph_*`** |
| `drivers/of/base.c` | دوال OF الأساسية (traversal، property reading) |
| `drivers/of/device.c` | ربط OF nodes بـ Linux devices |

#### Headers مكملة
| الملف | الوظيفة |
|---|---|
| `include/linux/of_platform.h` | تحويل OF nodes إلى platform devices |
| `include/linux/of_device.h` | match tables للـ drivers |
| `include/linux/of_address.h` | قراءة عناوين الذاكرة من DTS |
| `include/linux/of_irq.h` | قراءة الـ interrupts من DTS |

#### أمثلة DTS (هاردوير فعلي)
| الملف | الوظيفة |
|---|---|
| `arch/arm/boot/dts/st/stm32mp157c-ev1.dts` | مثال حقيقي: كاميرا + DCMI + DSI + LCD |
| `arch/*/boot/dts/` | آلاف الأمثلة لأجهزة مختلفة |

#### التوثيق
| الملف | الوظيفة |
|---|---|
| `Documentation/devicetree/bindings/` | قواعد bindings الرسمية |
| `Documentation/firmware-guide/acpi/dsd/graph.rst` | graph على ACPI |

---

### لماذا هذا مهم؟

بدون OF Graph، كل driver للكاميرا يحتاج أن يعرف مسبقاً اسم الجهاز الآخر، أو يبحث بطرق مخصصة. مع OF Graph:

1. **قابلية النقل:** نفس الـ driver يعمل على أي board بمجرد تعديل DTS.
2. **الاكتشاف التلقائي:** drivers تكتشف جيرانها وقت التشغيل.
3. **معيار موحّد:** كل الـ subsystems تتحدث نفس اللغة.
4. **دعم الـ probe order:** النواة تعرف من يحتاج من، وتُشغّل الـ drivers بالترتيب الصحيح.
## Phase 2: شرح الـ OF Graph Framework

### المشكلة التي يحلها هذا الـ Subsystem

في أنظمة embedded المبنية على ARM، يوجد دائماً سلسلة من الأجهزة المترابطة لنقل البيانات المرئية أو الصوتية:

```
Camera Sensor → MIPI CSI-2 Controller → ISP → Display Controller → DSI Bridge → LCD Panel
```

كل جهاز في هذه السلسلة هو **driver** مستقل في kernel. المشكلة: كيف يعرف الـ `ISP driver` من أين تأتي بياناته؟ وكيف يعرف الـ `Display Controller driver` إلى أين يرسل مخرجاته؟

قبل وجود OF Graph، كانت الحلول هكذا:
- **Hard-coding** في الـ driver مباشرةً — مستحيل على منصات متعددة.
- **Platform data** يُمرر عبر `board files` — طريقة قديمة تُولّد مئات الملفات غير قابلة للصيانة.
- **Custom DTS properties** لكل vendor — فوضى تامة بلا standard.

**النتيجة:** لا يوجد طريقة معيارية للتعبير عن "هذا المنفذ (port) من هذا الجهاز متصل بذلك المنفذ من ذلك الجهاز" داخل Device Tree.

---

### الحل: OF Graph Binding

**الـ OF Graph** هو standard لتمثيل الاتصالات بين الأجهزة (hardware topology) داخل Device Tree Source (DTS). يُعرِّف هيكلاً هرمياً من ثلاثة مستويات:

```
device-node
  └── ports          (اختياري، إذا كان الجهاز له أكثر من port)
        └── port@N   (منفذ رقم N، مع reg = <N>)
              └── endpoint@M  (نقطة اتصال رقم M، مع remote-endpoint = <&phandle>)
```

**الـ `remote-endpoint`** هو phandle يشير إلى الـ endpoint الآخر في الجهاز الثاني — هذا هو جوهر الربط.

---

### المثال التوضيحي: كاميرا متصلة بـ ISP على SoC من Samsung Exynos

```dts
/* ملف DTS المبسط */

/* جهاز sensor الكاميرا */
ov5640: camera@3c {
    compatible = "ovti,ov5640";
    reg = <0x3c>;

    port {
        /* endpoint واحد — المخرج من الكاميرا */
        ov5640_out: endpoint {
            remote-endpoint = <&csi_in>;   /* يشير إلى ISP */
            data-lanes = <1 2>;
        };
    };
};

/* وحدة MIPI CSI-2 داخل الـ SoC */
csi2: csi@11880000 {
    compatible = "samsung,exynos5250-csis";
    reg = <0x11880000 0x1000>;

    ports {
        /* port@0 = المدخل من الكاميرا */
        port@0 {
            reg = <0>;
            csi_in: endpoint {
                remote-endpoint = <&ov5640_out>;  /* يعود للكاميرا */
                data-lanes = <1 2>;
            };
        };

        /* port@1 = المخرج إلى الـ ISP */
        port@1 {
            reg = <1>;
            csi_out: endpoint {
                remote-endpoint = <&isp_in>;
            };
        };
    };
};
```

هنا يتضح: الـ `ov5640_out` يشير إلى `csi_in` والعكس صحيح. هذه **علاقة ثنائية الاتجاه** داخل DTS.

---

### التشبيه الواقعي: شبكة التوصيل الكهربائي في مصنع

تخيل مصنعاً فيه آلات (أجهزة) وبينها **موصلات (cables)**:

| مفهوم الكهرباء | مقابله في OF Graph |
|---|---|
| الآلة (machine) | الـ `device node` في DTS |
| مقبس الآلة (socket) | الـ `port` |
| طرف الكابل (connector) | الـ `endpoint` |
| الكابل نفسه (cable) | الـ `remote-endpoint` phandle (وجهه الآخر) |
| مخطط التوصيل (wiring diagram) | ملف DTS كامل |
| مهندس يقرأ المخطط | الـ `of_graph_*` API functions |
| رقم المقبس على الآلة | الـ `reg` property على الـ `port` |
| رقم الكابل المتصل بالمقبس | الـ `reg` property على الـ `endpoint` |

**التعمق في التشبيه:**

عندما يريد مهندس الصيانة معرفة "ما الآلة المتصلة بمخرج رقم 2 من آلة A؟"، يفتح المخطط، يجد مقبس رقم 2، يتبع الكابل، يصل إلى الآلة B. بالمثل، الـ driver يستدعي:

```c
/* ابحث عن الـ endpoint المقابل عبر of_graph API */
struct device_node *remote = of_graph_get_remote_endpoint(local_endpoint);
struct device_node *remote_device = of_graph_get_port_parent(remote);
```

---

### البنية المعمارية الكاملة

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                     Device Tree (DTB/DTS)                       │
 │                                                                 │
 │  ┌──────────────┐    remote-endpoint    ┌──────────────────┐   │
 │  │  camera@3c   │◄─────────────────────►│  csi@11880000    │   │
 │  │  └─ port     │                       │  └─ ports        │   │
 │  │     └─endpt  │                       │     ├─ port@0    │   │
 │  └──────────────┘                       │     │  └─ endpt  │   │
 │                                         │     └─ port@1    │   │
 │                                         │        └─ endpt  │   │
 │                                         └──────────────────┘   │
 └─────────────────────────────────────────────────────────────────┘
                               │
                    OF core parses DTB at boot
                               │
                               ▼
 ┌─────────────────────────────────────────────────────────────────┐
 │               Kernel Device Tree in Memory                      │
 │                                                                 │
 │    struct device_node  ◄──── struct property (linked list)     │
 │    ┌──────────────┐                                             │
 │    │  name        │  "camera@3c"                                │
 │    │  full_name   │  "/i2c@13860000/camera@3c"                  │
 │    │  phandle     │  0x42                                       │
 │    │  *properties │──► name → compatible → reg → port → ...    │
 │    │  *parent     │──► i2c@13860000 node                        │
 │    │  *child      │──► port node                                │
 │    │  *sibling    │──► next device on same bus                  │
 │    └──────────────┘                                             │
 └─────────────────────────────────────────────────────────────────┘
                               │
                    of_graph_* API (of_graph.h)
                               │
         ┌─────────────────────┼──────────────────────┐
         ▼                     ▼                      ▼
 ┌───────────────┐   ┌──────────────────┐   ┌────────────────────┐
 │  V4L2 / Media │   │   DRM / Display  │   │   ASoC / Audio     │
 │  Subsystem    │   │   Subsystem      │   │   Subsystem        │
 │               │   │                 │   │                    │
 │  Camera driver│   │  Display driver  │   │  Codec driver      │
 │  ISP driver   │   │  Bridge driver   │   │  DAI driver        │
 └───────────────┘   └──────────────────┘   └────────────────────┘
```

---

### الـ Core Abstraction: الـ `struct of_endpoint`

الـ abstraction الجوهرية هي **نقطة الاتصال المُعرَّفة بـ (port_id, endpoint_id)**:

```c
struct of_endpoint {
    unsigned int port;       /* رقم الـ port (قيمة reg الخاصة بالـ port node) */
    unsigned int id;         /* رقم الـ endpoint داخل الـ port */
    const struct device_node *local_node; /* مؤشر إلى الـ endpoint node في DT */
};
```

هذه البنية تُلخّص "أنا endpoint رقم `id` داخل port رقم `port` من جهاز معين". يتم ملؤها عبر:

```c
/* يقرأ reg من الـ endpoint node ومن الـ port الأب */
int of_graph_parse_endpoint(const struct device_node *node,
                            struct of_endpoint *endpoint);
```

**العلاقة بين الـ structs:**

```
struct device_node (camera@3c)
         │
         │  [child]
         ▼
struct device_node (port)           ← port node: reg = <0>
         │
         │  [child]
         ▼
struct device_node (endpoint)       ← endpoint node: reg = <0>
         │                                            remote-endpoint = <&csi_in>
         │  of_graph_parse_endpoint()
         ▼
struct of_endpoint {
    .port       = 0,               ← من reg الـ port الأب
    .id         = 0,               ← من reg الـ endpoint نفسه
    .local_node = endpoint_node,   ← مؤشر مباشر
}
```

---

### الـ API Functions: ماذا يملك هذا الـ Subsystem وماذا يفوّض؟

#### ما يملكه OF Graph ويُنفّذه بنفسه:

| الدالة | ما تفعله |
|---|---|
| `of_graph_parse_endpoint()` | تملأ `struct of_endpoint` من الـ DT node |
| `of_graph_get_next_endpoint()` | تُكرر على جميع endpoints في device node |
| `of_graph_get_next_port()` | تُكرر على جميع ports في device أو ports node |
| `of_graph_get_next_port_endpoint()` | تُكرر على endpoints داخل port محدد |
| `of_graph_get_port_by_id()` | تجلب port محدد بـ reg id |
| `of_graph_get_endpoint_by_regs()` | تجلب endpoint بـ (port_reg, endpoint_reg) |
| `of_graph_get_remote_endpoint()` | تتبع الـ `remote-endpoint` phandle وتعيد الـ node المقابل |
| `of_graph_get_remote_port_parent()` | تعيد الـ device node الأب للطرف الآخر من الاتصال |
| `of_graph_get_remote_port()` | تعيد الـ port node في الطرف الآخر |
| `of_graph_get_remote_node()` | تجد الـ device الآخر عبر (port, endpoint) |
| `of_graph_get_port_parent()` | من الـ port أو endpoint، يصعد للـ device الأب (يتجاوز `ports` node إذا وُجد) |
| `of_graph_is_present()` | يتحقق من وجود OF graph binding في node معين |
| `of_graph_get_endpoint_count()` | يعد عدد الـ endpoints |
| `of_graph_get_port_count()` | يعد عدد الـ ports |

#### ما يُفوَّض للـ drivers:

- **تفسير معنى الاتصال**: هل هذا MIPI أم LVDS أم HDMI؟ — شأن driver.
- **التفاوض على الـ format والـ timing**: مثل V4L2 `try_fmt` / `set_fmt`.
- **إنشاء الـ media links** بين entities في الـ media graph — هذا شأن الـ V4L2/MC subsystem.
- **إدارة الـ power** عند الربط — شأن الـ driver.
- **التحقق من التوافق** بين جانبَي الاتصال (data lanes, clock mode).

---

### الـ Macros للتكرار (Iteration Macros)

#### `for_each_endpoint_of_node`

```c
/* تكرار على كل endpoints في جهاز ما */
#define for_each_endpoint_of_node(parent, child) \
    for (child = of_graph_get_next_endpoint(parent, NULL); child != NULL; \
         child = of_graph_get_next_endpoint(parent, child))
```

**ملاحظة مهمة:** هذا الـ macro لا يستخدم `__free(device_node)` — أي إذا كسرت الحلقة بـ `break`، **يجب** استدعاء `of_node_put(child)` يدوياً لتحرير الـ reference count.

#### `for_each_of_graph_port` و `for_each_of_graph_port_endpoint`

```c
/* تكرار على ports — يستخدم cleanup attribute التلقائي */
#define for_each_of_graph_port(parent, child)           \
    for (struct device_node *child __free(device_node) = \
             of_graph_get_next_port(parent, NULL);        \
         child != NULL;                                   \
         child = of_graph_get_next_port(parent, child))
```

هذان الـ macros الجديدان يستخدمان **`__free(device_node)`** المُعرَّف بـ `DEFINE_FREE(device_node, ...)` في `of.h`:

```c
/* من linux/of.h */
DEFINE_FREE(device_node, struct device_node *, if (_T) of_node_put(_T))
```

هذا يعني أن الـ `child` سيُحرَّر تلقائياً عند انتهاء scope الحلقة — **scoped resource management** بدون `goto cleanup`.

إذا أردت الاحتفاظ بـ `child` بعد الحلقة، استخدم `no_free_ptr(child)` أو `return_ptr(child)` لإلغاء الـ cleanup.

---

### مثال عملي: كيف يستخدمها driver فعلي

فيما يلي مثال مبسط من نمط V4L2 subdev driver لكاميرا:

```c
static int camera_parse_endpoint(struct camera_dev *cam,
                                  struct device_node *node)
{
    struct of_endpoint endpoint;
    struct device_node *remote_ep, *remote_dev;
    int ret;

    /* اقرأ بيانات الـ endpoint المحلي */
    ret = of_graph_parse_endpoint(node, &endpoint);
    if (ret)
        return ret;

    pr_info("Camera: port=%u, endpoint=%u\n", endpoint.port, endpoint.id);

    /* اجلب الطرف الآخر من الاتصال */
    remote_ep = of_graph_get_remote_endpoint(node);
    if (!remote_ep) {
        dev_err(cam->dev, "No remote endpoint found\n");
        return -ENODEV;
    }

    /* اجلب الـ device الكامل (أب الـ port) من الطرف الآخر */
    remote_dev = of_graph_get_port_parent(remote_ep);
    of_node_put(remote_ep);  /* حرر ref الـ endpoint */

    if (!remote_dev)
        return -ENODEV;

    /* الآن remote_dev يشير إلى الـ ISP device node */
    pr_info("Connected to: %s\n", remote_dev->full_name);

    of_node_put(remote_dev);
    return 0;
}

static int camera_probe(struct i2c_client *client)
{
    struct camera_dev *cam = /* ... */;
    struct device_node *np = client->dev.of_node;
    struct device_node *ep;

    /* تكرار على جميع endpoints في الكاميرا */
    for_each_endpoint_of_node(np, ep) {
        int ret = camera_parse_endpoint(cam, ep);
        if (ret) {
            of_node_put(ep);  /* إلزامي عند break */
            return ret;
        }
    }
    return 0;
}
```

---

### أين يقع OF Graph في الـ Kernel Stack؟

```
┌─────────────────────────────────────────────────────────────────────┐
│                     User Space                                      │
│   libcamera / GStreamer / ffmpeg                                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  ioctl / V4L2 API
┌──────────────────────────────▼──────────────────────────────────────┐
│                  Kernel Subsystems (Consumers)                      │
│                                                                     │
│   ┌─────────────┐  ┌──────────────────┐  ┌────────────────────┐    │
│   │  V4L2/MC    │  │   DRM/KMS        │  │   ASoC             │    │
│   │  (cameras,  │  │   (display,      │  │   (audio codec     │    │
│   │   ISPs)     │  │    bridges)      │  │    DAI links)      │    │
│   └──────┬──────┘  └────────┬─────────┘  └─────────┬──────────┘    │
│          │                  │                       │               │
│          └──────────────────┴───────────────────────┘               │
│                             │                                       │
│                   of_graph_* API calls                              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│              OF Graph Layer  (include/linux/of_graph.h)             │
│                                                                     │
│   of_graph_get_remote_endpoint()   of_graph_parse_endpoint()        │
│   of_graph_get_port_parent()       of_graph_get_next_endpoint()     │
│   of_graph_get_remote_port_parent() ...                             │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  يستخدم OF core API
┌──────────────────────────────▼──────────────────────────────────────┐
│              OF Core  (include/linux/of.h)                          │
│                                                                     │
│   struct device_node   struct property                              │
│   of_find_node_by_phandle()   of_get_next_child()                   │
│   of_get_property()   of_node_get() / of_node_put()                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│          Device Tree Blob (DTB) — parsed at boot time               │
│          /proc/device-tree  —  sysfs representation                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

### نقطة دقيقة: `get_port_parent` و مشكلة الـ `ports` node

بعض الأجهزة تستخدم `ports` node وسيطة (اختيارية حسب الـ binding):

```dts
/* بدون ports node: */
device@0 { port { endpoint { ... }; }; };

/* مع ports node: */
device@1 { ports { port@0 { endpoint { ... }; }; }; };
```

الدالة `of_graph_get_port_parent()` تتعامل مع كلتا الحالتين بذكاء — إذا كان أبو الـ `port` هو `ports`، تصعد مستوى إضافياً للوصول إلى الـ device الفعلي. هذا يُخفي تفاصيل الـ DTS structure عن الـ driver.

```c
/* السلوك الداخلي المبسط: */
struct device_node *of_graph_get_port_parent(struct device_node *node)
{
    struct device_node *np;

    np = of_get_parent(node);                  /* port → ports أو device */
    if (!np) return NULL;

    /* إذا الأب اسمه "ports"، اصعد مرة أخرى */
    if (of_node_name_eq(np, "ports")) {
        struct device_node *tmp = of_get_parent(np);
        of_node_put(np);
        np = tmp;
    }
    return np;                                  /* هذا هو الـ device الفعلي */
}
```

---

### ملخص: ما يملكه OF Graph وما يُفوّضه

| الجانب | OF Graph يملكه | يُفوّضه للـ driver/subsystem |
|---|---|---|
| **هيكل DT** | تعريف port/endpoint hierarchy | اختيار الـ properties الإضافية (lanes, format) |
| **التنقل** | كل دوال `of_graph_get_*` | منطق اتخاذ القرار بعد الحصول على الـ node |
| **Reference counting** | `of_node_get/put` عبر OF core | استدعاء `of_node_put` في الوقت الصحيح |
| **الربط المنطقي** | لا شيء — يُعيد فقط `device_node*` | إنشاء V4L2 links أو DRM encoder chains |
| **التحقق من التوافق** | لا شيء | driver يقرأ properties ويتحقق |
| **Power management** | لا شيء | driver يتحكم في الـ power عند connect/disconnect |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Config Options والـ Macros — Cheatsheet

| الرمز / الخيار | النوع | الوظيفة |
|---|---|---|
| `CONFIG_OF` | Kconfig | يُفعّل كامل الـ OF graph API — بدونه كل الدوال تُرجع NULL/0/false |
| `CONFIG_OF_DYNAMIC` | Kconfig | يُفعّل reference counting حقيقي عبر `of_node_get/put` |
| `CONFIG_OF_KOBJ` | Kconfig | يُدمج `kobject` داخل `device_node` لنشره في sysfs |
| `OF_DYNAMIC` (flag=1) | Node flag | الـ node وخصائصه مُخصَّصة بـ `kmalloc` |
| `OF_DETACHED` (flag=2) | Node flag | الـ node مفصول عن شجرة الـ device tree |
| `OF_POPULATED` (flag=3) | Node flag | تم إنشاء الـ device المقابل مسبقاً |
| `OF_POPULATED_BUS` (flag=4) | Node flag | تم إنشاء platform bus للـ children |
| `OF_OVERLAY` (flag=5) | Node flag | مُخصَّص لـ overlay |
| `OF_OVERLAY_FREE_CSET` (flag=6) | Node flag | في حالة تحرير overlay cset |
| `DEFINE_FREE(device_node, ...)` | Cleanup macro | يُسجّل `of_node_put` كـ auto-cleanup عند خروج المتغير من النطاق |

---

### الـ Iteration Macros — Cheatsheet

| الـ Macro | الاستخدام | ملاحظة مهمة |
|---|---|---|
| `for_each_endpoint_of_node(parent, child)` | التكرار على جميع الـ endpoints تحت device node | عند `break` يجب استدعاء `of_node_put(child)` يدوياً |
| `for_each_of_graph_port(parent, child)` | التكرار على الـ ports تحت device/ports node | يستخدم `__free(device_node)` — عند `break` استخدم `no_free_ptr(child)` |
| `for_each_of_graph_port_endpoint(parent, child)` | التكرار على الـ endpoints داخل port واحد | نفس قاعدة الـ `__free` |

---

### 1. الـ Structs المهمة

#### 1.1 `struct of_endpoint`

**الغرض:** يُمثّل بيانات **نقطة اتصال (endpoint)** محللّة من الـ device tree، وهي النقطة التي يتصل فيها جهازان (مثلاً: camera sensor ↔ CSI controller).

```c
struct of_endpoint {
    unsigned int port;              /* رقم الـ port (قيمة reg property في عقدة port) */
    unsigned int id;                /* رقم الـ endpoint داخل الـ port (قيمة reg property) */
    const struct device_node *local_node; /* مؤشر لعقدة الـ device tree للـ endpoint نفسه */
};
```

| الحقل | النوع | الدور |
|---|---|---|
| `port` | `unsigned int` | يُعرّف الـ port الذي ينتمي إليه الـ endpoint — مأخوذ من `reg` property في عقدة `port@N` |
| `id` | `unsigned int` | يُعرّف الـ endpoint داخل الـ port — مأخوذ من `reg` property في عقدة `endpoint@N` |
| `local_node` | `const struct device_node *` | مؤشر مباشر لعقدة الـ endpoint في الـ device tree — لا تملكه (لا تستدعي `of_node_put`) |

**مثال واقعي:** في كاميرا مدمجة مع SoC:
```
/ {
    camera: ov5640@3c {
        port@0 {               /* port = 0 */
            reg = <0>;
            cam_out: endpoint@0 {   /* id = 0 */
                reg = <0>;
                remote-endpoint = <&csi_in>;
            };
        };
    };
};
```
بعد `of_graph_parse_endpoint(cam_out_node, &ep)`:
- `ep.port = 0`
- `ep.id = 0`
- `ep.local_node = cam_out_node`

---

#### 1.2 `struct device_node` (من `include/linux/of.h`)

**الغرض:** يُمثّل **عقدة واحدة** في شجرة الـ device tree المحمّلة في الذاكرة. هو الوحدة الأساسية التي تتعامل معها كل دوال الـ OF graph.

```c
struct device_node {
    const char *name;           /* اسم العقدة (آخر مكوّن من full_name) */
    phandle phandle;            /* معرّف عددي فريد للإشارة إليها من عقد أخرى */
    const char *full_name;      /* المسار الكامل مثل /soc/camera@3c/port@0/endpoint@0 */
    struct fwnode_handle fwnode;/* واجهة موحّدة للـ firmware nodes (ACPI/DT) */

    struct property *properties; /* قائمة مرتبطة من الخصائص (reg, compatible, ...) */
    struct property *deadprops;  /* خصائص محذوفة (لأغراض overlay) */
    struct device_node *parent;  /* العقدة الأب */
    struct device_node *child;   /* أول عقدة ابن */
    struct device_node *sibling; /* العقدة الأخ (التالية في نفس المستوى) */
    unsigned long _flags;        /* أعلام OF_DYNAMIC, OF_POPULATED, ... */
    void *data;                  /* بيانات خاصة بالمنصة */
};
```

| الحقل | الأهمية في OF graph |
|---|---|
| `full_name` | يُحدد موقع العقدة في التسلسل الهرمي (device → ports → port@N → endpoint@N) |
| `parent` | يُستخدم للتنقل للأعلى: endpoint → port → ports/device |
| `child/sibling` | يُستخدم للتكرار عبر الـ ports والـ endpoints |
| `properties` | تحتوي على `reg` (للـ port/endpoint id)، `remote-endpoint` (phandle للطرف الآخر) |
| `phandle` | يُستخدم في `remote-endpoint = <&other_ep>` للربط بين الأجهزة |

---

#### 1.3 `struct property` (من `include/linux/of.h`)

**الغرض:** يُمثّل **خاصية واحدة** داخل عقدة device tree (مثل `reg`, `remote-endpoint`, `compatible`).

```c
struct property {
    char  *name;           /* اسم الخاصية مثل "reg" أو "remote-endpoint" */
    int    length;         /* الحجم بالبايت */
    void  *value;          /* القيمة الخام (big-endian) */
    struct property *next; /* الخاصية التالية في القائمة المرتبطة */
};
```

**أهميتها في OF graph:** الدالة `of_graph_parse_endpoint` تقرأ `reg` property لاستخراج `port` و`id`، والدالة `of_graph_get_remote_endpoint` تقرأ `remote-endpoint` property (phandle) للوصول للطرف الآخر.

---

### 2. مخطط العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────────┐
│                     device_node (SoC/camera device)             │
│  name="ov5640"  full_name="/camera@3c"  phandle=0x10           │
│  properties → [compatible] → [reg] → ...                        │
│  child ──────────────────────────────────────────────────────┐  │
└──────────────────────────────────────────────────────────────│──┘
                                                               │
                                                               ▼
┌─────────────────────────────────────────────────────────────────┐
│               device_node ("ports" or "port@0")                 │
│  name="port"   full_name="/camera@3c/port@0"                    │
│  properties → [reg=0]                                           │
│  parent ──→ camera device_node                                  │
│  child ──────────────────────────────────────────────────────┐  │
└──────────────────────────────────────────────────────────────│──┘
                                                               │
                                                               ▼
┌─────────────────────────────────────────────────────────────────┐
│               device_node (endpoint@0)                          │
│  name="endpoint"  full_name="/camera@3c/port@0/endpoint@0"      │
│  properties → [reg=0] → [remote-endpoint=phandle(0x20)]        │
│  parent ──→ port@0 device_node                                  │
│  phandle=0x15                                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │ remote-endpoint phandle
                             │ (of_graph_get_remote_endpoint)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│               device_node (endpoint@0 of CSI controller)        │
│  name="endpoint"  full_name="/soc/csi@30a90000/port@0/ep@0"    │
│  phandle=0x20                                                   │
│  properties → [reg=0] → [remote-endpoint=phandle(0x15)]        │
└─────────────────────────────────────────────────────────────────┘

     ┌──────────────────────────────────┐
     │        struct of_endpoint        │
     │  port = 0                        │  ←── مُملأ بـ of_graph_parse_endpoint()
     │  id   = 0                        │
     │  local_node ──→ endpoint@0 node  │
     └──────────────────────────────────┘
```

---

### 3. مخطط التسلسل الهرمي في الـ Device Tree

```
device node (e.g., camera, display, ISP)
│
├── port@0                    ← of_graph_get_port_by_id(dev, 0)
│   ├── endpoint@0            ← of_graph_get_endpoint_by_regs(dev, 0, 0)
│   │     └── remote-endpoint ──────────────────────────────────┐
│   └── endpoint@1                                               │
│                                                                │
└── port@1                                                       │
    └── endpoint@0                                               │
                                                                 │
other-device node                                                │
│                                                                │
└── port@0                                                       │
    └── endpoint@0  ←─────────────────────────────────────────── ┘
          └── remote-endpoint ──→ (back to camera endpoint@0)

      ↑ هذا هو مفهوم OF graph: ربط الأجهزة عبر phandles متبادلة
```

**ملاحظة:** قد يوجد `ports` node وسيط عندما يحتوي الجهاز على ports متعددة — الدوال تتعامل مع الحالتين بشفافية.

---

### 4. مخطط دورة الحياة (Lifecycle)

```
[Boot / DTB parsing]
      │
      ▼
of_flat_dt_unflatten_tree()
  → ينشئ شجرة device_node في الذاكرة
  → يُحدد phandles لجميع العقد
      │
      ▼
[Driver Probe]
      │
      ├─► of_graph_is_present(dev_node)
      │       └── يتحقق من وجود port أو ports node تحت الجهاز
      │
      ├─► of_graph_get_port_by_id(dev_node, port_id)
      │       └── يُرجع device_node للـ port (مع ref count +1)
      │
      ├─► of_graph_get_next_endpoint(dev_node, NULL)
      │       └── يُرجع أول endpoint (مع ref count +1)
      │
      ├─► of_graph_parse_endpoint(ep_node, &ep_struct)
      │       └── يملأ struct of_endpoint بـ port, id, local_node
      │
      ├─► of_graph_get_remote_endpoint(ep_node)
      │       └── يتبع remote-endpoint phandle → يُرجع endpoint الطرف الآخر
      │
      ├─► of_graph_get_remote_port_parent(ep_node)
      │       └── يُرجع device node للجهاز البعيد (الأكثر استخداماً)
      │
      └─► of_node_put(node)  ← يجب استدعاؤه لكل node مُسترجع
              └── يُحرر الـ ref count (أو no-op إذا !CONFIG_OF_DYNAMIC)

[Driver Remove / Cleanup]
      │
      ▼
      تحرير جميع الـ device_node references بـ of_node_put()
      لا يوجد "unregister" — الـ graph structure ثابتة في الـ DTB
```

---

### 5. مخططات تدفق الاستدعاء (Call Flow Diagrams)

#### 5.1 الحصول على الجهاز البعيد المتصل بـ endpoint محدد

```
driver: of_graph_get_remote_port_parent(local_ep_node)
  │
  ├─► of_graph_get_remote_endpoint(local_ep_node)
  │       │
  │       └─► of_parse_phandle(node, "remote-endpoint", 0)
  │               └── يقرأ phandle من property "remote-endpoint"
  │               └── يبحث عن device_node بهذا الـ phandle
  │               └── يُرجع remote_ep_node (ref+1)
  │
  └─► of_graph_get_port_parent(remote_ep_node)
          │
          ├─► of_get_parent(remote_ep_node)   → port_node
          ├─► of_get_parent(port_node)         → ports_node أو device_node
          │
          └── إذا كان اسم الأب "ports":
                  of_get_parent(ports_node)    → device_node (النتيجة النهائية)
              إذا لا:
                  يُرجع ports_node مباشرة (هو device_node)
```

#### 5.2 التكرار على جميع الـ endpoints لجهاز ما

```
for_each_endpoint_of_node(dev_node, ep)
  │
  └─► of_graph_get_next_endpoint(dev_node, prev_ep)
          │
          ├── [أول استدعاء: prev=NULL]
          │       └─► of_get_next_child(dev_node, NULL)
          │               ├── يبحث عن عقدة بـ name="port" أو "ports"
          │               └── إذا وجد "ports": ينزل داخله أولاً
          │
          ├── [داخل port node]
          │       └─► of_get_next_child(port_node, NULL)
          │               └── يُرجع أول endpoint
          │
          └── [الاستدعاءات التالية]
                  └── يتنقل عبر siblings للـ endpoints
                      ثم ينتقل للـ port التالي عند الانتهاء
                      حتى يستنفد جميع الـ ports
```

#### 5.3 البحث عن endpoint بأرقام محددة

```
driver: of_graph_get_endpoint_by_regs(dev_node, port_reg=0, ep_reg=1)
  │
  └─► for_each_endpoint_of_node(dev_node, ep_node):
          │
          └─► of_graph_parse_endpoint(ep_node, &ep)
                  │
                  ├── يقرأ "reg" من ep_node        → ep.id
                  ├── يقرأ "reg" من ep_node->parent → ep.port
                  └── ep.local_node = ep_node
                  │
                  └── إذا ep.port==port_reg && ep.id==ep_reg:
                          return ep_node  ← وُجد!
                      إذا port_reg==-1: يتجاهل مقارنة الـ port
                      إذا ep_reg==-1:   يتجاهل مقارنة الـ endpoint
```

#### 5.4 التنقل الكامل لإيجاد الجهاز البعيد

```
dev_A endpoint_node
      │
      │  of_graph_get_remote_endpoint()
      ▼
dev_B endpoint_node  (ref+1)
      │
      │  of_graph_get_remote_port_parent()
      │    └── of_get_parent() × 1 أو 2
      ▼
dev_B device_node  (ref+1)
      │
      │  المستخدم يستدعي:
      │  platform_device = of_find_device_by_node(dev_B_node)
      ▼
struct platform_device *  ← الجهاز الحقيقي
```

---

### 6. استراتيجية الـ Locking

#### 6.1 آليات الحماية

الـ OF graph API لا تُعرّف locks خاصة بها، بل تعتمد على:

| الآلية | يحمي | متى يُستخدم |
|---|---|---|
| `of_node_get()` / `of_node_put()` | دورة حياة الـ `device_node` في الذاكرة | عند كل استرجاع لـ node يجب الإمساك بـ reference |
| `__free(device_node)` | تلقائي عبر cleanup.h | في الـ `for_each_of_graph_port*` macros |
| `of_mutex` (داخلي في of.c) | قراءة/تعديل شجرة الـ DT | يُقفل داخلياً في `of_find_node_by_phandle` |
| `devtree_lock` (spinlock) | في بعض المنصات | للوصول المتزامن لشجرة الـ DT من interrupts |

#### 6.2 قواعد الـ Reference Counting

```
┌─────────────────────────────────────────────────────────┐
│  كل دالة تُرجع device_node* تزيد refcount بمقدار 1     │
│  المستدعي مسؤول عن استدعاء of_node_put() عند الانتهاء  │
└─────────────────────────────────────────────────────────┘

صحيح:
    struct device_node *np = of_graph_get_remote_port_parent(ep);
    if (np) {
        /* استخدام np */
        of_node_put(np);  /* ← ضروري */
    }

صحيح (C cleanup):
    struct device_node *np __free(device_node) =
        of_graph_get_remote_port_parent(ep);
    /* يُحرر تلقائياً عند الخروج من النطاق */

خطأ شائع:
    /* break من for_each_endpoint_of_node بدون of_node_put */
    for_each_endpoint_of_node(dev, ep) {
        if (found_it)
            break;  /* ← تسريب ref! */
    }
    /* الصحيح: of_node_put(ep) قبل break */
```

#### 6.3 ترتيب الـ Locks (Lock Ordering)

```
لا توجد nested locks في OF graph API نفسها.
الترتيب العام عند التكامل مع subsystems أخرى:

  device_lock(dev)          ← أعلى مستوى
    └── of_mutex / devtree_lock   ← مستوى DT
          └── refcount only       ← OF graph level
```

**تحذير:** لا تستدعِ دوال OF graph وأنت تحمل lock يمكن أن يُحتاج فيه الـ DT lock — خطر deadlock. استرجع الـ device_node قبل الدخول في critical section.

---

### 7. ملخص الدوال مع أدوارها

| الدالة | المدخل | المخرج | الملاحظة |
|---|---|---|---|
| `of_graph_is_present(node)` | device node | `bool` | يتحقق من وجود port/ports child |
| `of_graph_parse_endpoint(node, ep)` | endpoint node | `int` (0 نجاح) | يملأ `struct of_endpoint` |
| `of_graph_get_endpoint_count(np)` | device node | `unsigned int` | عدد الـ endpoints الكلي |
| `of_graph_get_port_count(np)` | device node | `unsigned int` | عدد الـ ports |
| `of_graph_get_port_by_id(node, id)` | device node + port id | `device_node*` | ref+1 |
| `of_graph_get_next_endpoint(parent, prev)` | device node + prev ep | `device_node*` | ref+1، يُستخدم في `for_each_endpoint_of_node` |
| `of_graph_get_next_port(parent, port)` | device/ports node | `device_node*` | ref+1 |
| `of_graph_get_next_port_endpoint(port, prev)` | port node | `device_node*` | ref+1 |
| `of_graph_get_endpoint_by_regs(parent, port_reg, reg)` | device node + أرقام | `device_node*` | ref+1، -1 يعني "أي قيمة" |
| `of_graph_get_remote_endpoint(node)` | endpoint node | `device_node*` | يتبع `remote-endpoint` phandle |
| `of_graph_get_port_parent(node)` | endpoint/port node | `device_node*` | يصعد للـ device مع تجاوز "ports" |
| `of_graph_get_remote_port_parent(node)` | endpoint node | `device_node*` | الدالة الأكثر استخداماً — device الطرف الآخر |
| `of_graph_get_remote_port(node)` | endpoint node | `device_node*` | port الطرف الآخر |
| `of_graph_get_remote_node(node, port, ep)` | endpoint node + أرقام | `device_node*` | device الطرف الآخر محدد بأرقام |
## Phase 4: شرح الـ Functions

---

### ملخص — Cheatsheet للـ API

#### الـ Data Structures

| النوع | الاسم | الوصف |
|-------|-------|-------|
| `struct` | `of_endpoint` | يمثّل endpoint واحد في الـ OF graph، يحمل `port`، `id`، و`local_node` |
| `struct` | `device_node` | عقدة الـ Device Tree (من `of.h`)، هي الوحدة الأساسية لكل عمليات الـ OF graph |

#### الـ Macros / Iterators

| الاسم | الغرض |
|-------|-------|
| `for_each_endpoint_of_node` | يمرّ على كل endpoint في device node |
| `for_each_of_graph_port` | يمرّ على كل port في device أو ports node — يستخدم `__free` |
| `for_each_of_graph_port_endpoint` | يمرّ على كل endpoint داخل port واحد — يستخدم `__free` |

#### الـ Functions — جدول سريع

| الدالة | الإدخال | المُخرج | الغرض |
|--------|---------|---------|-------|
| `of_graph_is_present` | `device_node *` | `bool` | هل يوجد OF graph في الـ node؟ |
| `of_graph_parse_endpoint` | `device_node *`, `of_endpoint *` | `int` | قراءة بيانات endpoint من الـ DT |
| `of_graph_get_endpoint_count` | `device_node *` | `unsigned int` | عدد الـ endpoints |
| `of_graph_get_port_count` | `device_node *` | `unsigned int` | عدد الـ ports |
| `of_graph_get_port_by_id` | `device_node *`, `u32 id` | `device_node *` | جلب port بـ reg معين |
| `of_graph_get_next_endpoint` | `device_node *parent`, `device_node *prev` | `device_node *` | التالي في التسلسل |
| `of_graph_get_next_port` | `device_node *parent`, `device_node *port` | `device_node *` | التالي port |
| `of_graph_get_next_port_endpoint` | `device_node *port`, `device_node *prev` | `device_node *` | التالي endpoint داخل port |
| `of_graph_get_endpoint_by_regs` | `device_node *`, `int port_reg`, `int reg` | `device_node *` | endpoint بـ port/reg محددَين |
| `of_graph_get_remote_endpoint` | `device_node *` | `device_node *` | الـ endpoint الطرف الآخر |
| `of_graph_get_port_parent` | `device_node *` | `device_node *` | الـ device المالك للـ port |
| `of_graph_get_remote_port_parent` | `device_node *` | `device_node *` | الـ device المالك للطرف الآخر |
| `of_graph_get_remote_port` | `device_node *` | `device_node *` | الـ port للطرف الآخر |
| `of_graph_get_remote_node` | `device_node *`, `u32 port`, `u32 endpoint` | `device_node *` | الـ device المتصل عبر port+endpoint |

---

### الفئة الأولى: الـ Presence & Parsing

هذه الدوال تتحقق من وجود الـ OF graph وتفسّر بيانات الـ endpoint الخام من شجرة الـ Device Tree.

---

#### `of_graph_is_present`

```c
bool of_graph_is_present(const struct device_node *node);
```

تتحقق من أن الـ `node` تحتوي على بنية OF graph صالحة — أي أن لديها على الأقل node واحدة من نوع `port` أو `ports`. تُستدعى عادةً من `probe()` قبل أي محاولة traverse للـ graph.

**Parameters:**
- `node` — الـ `device_node` الجذر للـ device المراد فحصه.

**Return:**
- **`true`** — يوجد graph binding.
- **`false`** — لا يوجد، أو `CONFIG_OF` غير مُفعَّل.

**Key details:**
- لا تحتاج reference count، لا تستدعي `of_node_get`.
- آمنة للاستدعاء في أي سياق لا يستلزم sleep.

**Who calls it:**
الـ drivers التي تدعم إما OF graph أو طريقة إعداد بديلة (platform data مثلاً)، تستخدمها كـ guard أول.

---

#### `of_graph_parse_endpoint`

```c
int of_graph_parse_endpoint(const struct device_node *node,
                            struct of_endpoint *endpoint);
```

تقرأ خصائص الـ `reg` من الـ endpoint node وتملأ هيكل `of_endpoint`. تحدّد `port` و`id` بقراءة `reg` من الـ endpoint ومن أبيه (`port` node).

**Parameters:**
- `node` — الـ `device_node` للـ endpoint نفسه (ليس الـ port أو الـ device).
- `endpoint` — مؤشر للهيكل الذي سيُملأ بالبيانات.

**Return:**
- **`0`** — نجاح.
- **سالب** — كود خطأ (`-EINVAL` إذا كانت البنية غير صحيحة).

**Key details:**
- يضع `endpoint->local_node = node` بدون زيادة reference count — المستدعي يملك reference الـ node أصلاً.
- يقرأ `reg` من الـ endpoint node لتحديد `id`.
- يقرأ `reg` من الـ parent (port node) لتحديد `port`.

**Pseudocode:**
```
parse_endpoint(node, ep):
    port_node = node->parent          // الـ port node
    ep->local_node = node
    ep->port = read_reg(port_node)    // reg في الـ port
    ep->id   = read_reg(node)         // reg في الـ endpoint
    return 0
```

**Who calls it:**
الـ drivers التي تحتاج معرفة رقم الـ port والـ endpoint لتوجيه الـ data path (مثل كاميرا V4L2 أو DSI controller).

---

### الفئة الثانية: الـ Counting

---

#### `of_graph_get_endpoint_count`

```c
unsigned int of_graph_get_endpoint_count(const struct device_node *np);
```

تحصي إجمالي الـ endpoints في جميع الـ ports داخل الـ `np`. تُستخدم لتخصيص arrays أو التحقق من topology قبل traversal كامل.

**Parameters:**
- `np` — الـ device node الجذر.

**Return:**
- عدد صحيح غير سالب يمثّل مجموع الـ endpoints.

**Key details:**
- تُكمل iteration كاملة داخلياً — تكلفتها `O(n)` حيث `n` = عدد الـ endpoints.
- لا تُعيد references مفتوحة.

---

#### `of_graph_get_port_count`

```c
unsigned int of_graph_get_port_count(struct device_node *np);
```

تحصي عدد الـ `port` nodes مباشرةً تحت `np` أو تحت `np/ports`.

**Parameters:**
- `np` — الـ device node أو الـ `ports` container node.

**Return:**
- عدد الـ ports.

**Who calls it:**
الـ drivers متعددة الـ ports (multi-lane CSI2، HDMI splitters) لمعرفة عدد الـ data paths المتاحة.

---

### الفئة الثالثة: الـ Lookup بـ ID

---

#### `of_graph_get_port_by_id`

```c
struct device_node *of_graph_get_port_by_id(struct device_node *node, u32 id);
```

تُعيد أول `port` node تحت `node` يكون لها خاصية `reg` مساوية لـ `id`.

**Parameters:**
- `node` — الـ device node الجذر.
- `id` — قيمة `reg` المطلوبة.

**Return:**
- مؤشر `device_node` مع **زيادة reference count** — يجب استدعاء `of_node_put()` عند الانتهاء.
- `NULL` إذا لم يُعثر على port بهذا الـ id.

**Key details:**
- يبحث أيضاً تحت node من نوع `ports` إذا وُجدت.
- **يزيد refcount** — خطر memory leak إذا أُغفل `of_node_put`.

---

#### `of_graph_get_endpoint_by_regs`

```c
struct device_node *of_graph_get_endpoint_by_regs(
        const struct device_node *parent, int port_reg, int reg);
```

تجمع بين البحث بـ port id وبـ endpoint id في خطوة واحدة. قيمة `-1` في أي من المعاملَين تعني "أي قيمة".

**Parameters:**
- `parent` — الـ device node الجذر.
- `port_reg` — رقم الـ port (`reg`)، أو `-1` لعدم التقييد.
- `reg` — رقم الـ endpoint (`reg`)، أو `-1` لعدم التقييد.

**Return:**
- `device_node *` مع زيادة refcount، أو `NULL`.

**Key details:**
- استدعاء `of_node_put()` إلزامي على النتيجة.
- الاستخدام الشائع: `of_graph_get_endpoint_by_regs(dev->of_node, 0, 0)` للحصول على endpoint الأول.

**Who calls it:**
الـ drivers التي تعمل مع topology ثابتة ومعروفة (single CSI port مثلاً).

---

### الفئة الرابعة: الـ Iteration / Traversal

هذه الدوال هي engine الـ OF graph traversal. كل دالة منها تُعيد الـ node التالي في التسلسل مع الاحتفاظ بـ reference count صحيح.

---

#### `of_graph_get_next_endpoint`

```c
struct device_node *of_graph_get_next_endpoint(
        const struct device_node *parent,
        struct device_node *previous);
```

تُعيد الـ endpoint التالي في جميع الـ ports تحت `parent`. إذا كان `previous` هو `NULL`، تبدأ من البداية.

**Parameters:**
- `parent` — الـ device node الجذر الحاوي للـ ports.
- `previous` — آخر endpoint أُعيد، أو `NULL` للبداية. **تستهلك reference** الـ `previous` داخلياً (تستدعي `of_node_put` عليه).

**Return:**
- الـ endpoint التالي مع زيادة refcount، أو `NULL` عند النهاية.

**Key details:**
- يتنقل بين الـ ports تلقائياً عند وصوله لنهاية port.
- الـ macro `for_each_endpoint_of_node` يبني عليها مباشرةً.
- عند **كسر الـ loop** يدوياً، يجب استدعاء `of_node_put(child)` يدوياً.

**Pseudocode:**
```
get_next_endpoint(parent, prev):
    if prev == NULL:
        port = first_port(parent)
    else:
        port = prev->parent   // الـ port المالك
        next_ep = next_sibling_endpoint(prev)
        of_node_put(prev)
        if next_ep:
            return next_ep     // endpoint آخر في نفس الـ port
        port = next_port(parent, port)

    while port:
        ep = first_endpoint(port)
        if ep: return ep
        port = next_port(parent, port)
    return NULL
```

---

#### `of_graph_get_next_port`

```c
struct device_node *of_graph_get_next_port(
        const struct device_node *parent,
        struct device_node *port);
```

تُعيد الـ `port` node التالية تحت `parent`. مشابهة لـ `of_graph_get_next_endpoint` لكن تتوقف عند مستوى الـ port لا الـ endpoint.

**Parameters:**
- `parent` — الـ device node أو `ports` container.
- `port` — آخر port أُعيد، أو `NULL` للبداية.

**Return:**
- `device_node *` للـ port التالية مع refcount، أو `NULL`.

**Key details:**
- يستخدمه `for_each_of_graph_port` مع `__free(device_node)` للـ automatic cleanup.
- إذا أردت إبقاء الـ pointer بعد الـ loop، استخدم `no_free_ptr()` أو `return_ptr()`.

---

#### `of_graph_get_next_port_endpoint`

```c
struct device_node *of_graph_get_next_port_endpoint(
        const struct device_node *port,
        struct device_node *prev);
```

تُعيد الـ endpoint التالي **داخل port واحد فقط** — بخلاف `of_graph_get_next_endpoint` التي تعبر بين الـ ports.

**Parameters:**
- `port` — الـ `port` node المحدد (ليس الـ device node الجذر).
- `prev` — آخر endpoint أُعيد، أو `NULL` للبداية.

**Return:**
- `device_node *` للـ endpoint التالي، أو `NULL`.

**Key details:**
- يُستخدم مع `for_each_of_graph_port_endpoint`.
- مفيد عند الحاجة لمعالجة endpoints port بعد port بشكل منفصل.

---

### الفئة الخامسة: الـ Remote Node Resolution

هذه الدوال الأكثر أهمية عملياً — تسير عبر الـ `remote-endpoint` phandle للوصول إلى الطرف الآخر من الاتصال.

---

#### `of_graph_get_remote_endpoint`

```c
struct device_node *of_graph_get_remote_endpoint(
        const struct device_node *node);
```

تقرأ خاصية `remote-endpoint` من الـ `node` وتُعيد الـ `device_node` الذي يشير إليه الـ phandle — أي الـ endpoint الطرف الآخر.

**Parameters:**
- `node` — الـ endpoint المحلي.

**Return:**
- `device_node *` للـ remote endpoint مع زيادة refcount، أو `NULL`.

**Key details:**
- تحصل على refcount جديد عبر `of_parse_phandle` داخلياً.
- **يجب** استدعاء `of_node_put()` على النتيجة.
- لا تصعد إلى مستوى الـ device — تبقى عند مستوى الـ endpoint.

**مثال عملي:**
```c
/* من endpoint محلي، اعثر على endpoint الطرف الآخر */
struct device_node *remote_ep =
        of_graph_get_remote_endpoint(local_ep);
if (remote_ep) {
    /* استخدام remote_ep */
    of_node_put(remote_ep);
}
```

---

#### `of_graph_get_port_parent`

```c
struct device_node *of_graph_get_port_parent(struct device_node *node);
```

تصعد من مستوى الـ endpoint أو الـ port للوصول إلى الـ `device_node` المالك للـ device الحقيقي. تتخطى nodes من نوع `port` و`ports` للوصول للجذر الصحيح.

**Parameters:**
- `node` — يمكن أن يكون endpoint node أو port node.

**Return:**
- `device_node *` للـ device مع زيادة refcount، أو `NULL`.

**Key details:**
- **تستهلك refcount** الـ `node` المُمرَّر (تستدعي `of_node_put` عليه) — انتبه لعدم استخدامه بعدها.
- تتحقق من اسم الـ node لتحديد إن كانت `port` أو `ports` وتصعد وفقاً لذلك.

**Pseudocode:**
```
get_port_parent(node):
    while node:
        if node->name == "port":
            node = parent(node)   // ارتفع فوق port
        elif node->name == "ports":
            node = parent(node)   // ارتفع فوق ports
        else:
            return node           // وصلنا للـ device
    return NULL
```

---

#### `of_graph_get_remote_port_parent`

```c
struct device_node *of_graph_get_remote_port_parent(
        const struct device_node *node);
```

تجمع بين `of_graph_get_remote_endpoint` و`of_graph_get_port_parent` في خطوة واحدة. تُعيد مباشرةً الـ `device_node` للـ device المتصل بالطرف الآخر.

**Parameters:**
- `node` — الـ endpoint المحلي.

**Return:**
- `device_node *` للـ remote device مع زيادة refcount، أو `NULL`.

**Key details:**
- الأكثر استخداماً في الـ drivers لأنها تعطي الـ device مباشرةً دون خطوات وسيطة.
- **يجب** `of_node_put()` على النتيجة.

**مثال عملي:**
```c
/* من كاميرا CSI2: اعثر على الـ sensor المتصل */
struct device_node *sensor_node =
        of_graph_get_remote_port_parent(csi_endpoint);
if (sensor_node) {
    struct i2c_client *sensor = of_find_i2c_device_by_node(sensor_node);
    of_node_put(sensor_node);
}
```

---

#### `of_graph_get_remote_port`

```c
struct device_node *of_graph_get_remote_port(
        const struct device_node *node);
```

مشابهة لـ `of_graph_get_remote_port_parent` لكنها تتوقف عند مستوى الـ **port** لا الـ device — مفيدة عند الحاجة لمعرفة رقم الـ port في الـ remote device.

**Parameters:**
- `node` — الـ endpoint المحلي.

**Return:**
- `device_node *` للـ remote port مع refcount، أو `NULL`.

**Key details:**
- يمكن الحصول على رقم الـ port بعدها بقراءة `reg` من النتيجة.
- **يجب** `of_node_put()` على النتيجة.

---

#### `of_graph_get_remote_node`

```c
struct device_node *of_graph_get_remote_node(
        const struct device_node *node,
        u32 port, u32 endpoint);
```

الدالة الأشمل والأعلى مستوى — تبدأ من device node محلي، تحدد endpoint محلياً بـ `port` و`endpoint`، ثم تُعيد الـ device المتصل عبر ذلك الـ endpoint.

**Parameters:**
- `node` — الـ device node المحلي (جذر الـ device).
- `port` — رقم الـ port المحلي (`reg` property).
- `endpoint` — رقم الـ endpoint داخل الـ port (`reg` property).

**Return:**
- `device_node *` للـ remote device مع refcount، أو `NULL`.

**Key details:**
- تجمع داخلياً: `get_endpoint_by_regs` → `get_remote_endpoint` → `get_port_parent`.
- مثالية لـ drivers التي تعرف بالضبط الـ topology المتوقعة.
- **يجب** `of_node_put()` على النتيجة.

**Pseudocode:**
```
get_remote_node(node, port, endpoint):
    local_ep = get_endpoint_by_regs(node, port, endpoint)
    if !local_ep: return NULL

    remote_ep = get_remote_endpoint(local_ep)
    of_node_put(local_ep)
    if !remote_ep: return NULL

    remote_dev = get_port_parent(remote_ep)
    // get_port_parent تستهلك remote_ep
    return remote_dev
```

**مثال عملي:**
```c
/* اعثر على الـ display متصل بـ port 0, endpoint 0 للـ GPU */
struct device_node *display =
        of_graph_get_remote_node(gpu->dev.of_node, 0, 0);
if (display) {
    /* تسجيل الاتصال */
    of_node_put(display);
}
```

---

### الفئة السادسة: الـ Iteration Macros

---

#### `for_each_endpoint_of_node`

```c
#define for_each_endpoint_of_node(parent, child) \
    for (child = of_graph_get_next_endpoint(parent, NULL); child != NULL; \
         child = of_graph_get_next_endpoint(parent, child))
```

**الـ macro الكلاسيكي** للمرور على كل الـ endpoints في device. يستخدم النمط التقليدي حيث `of_graph_get_next_endpoint` تتولى إدارة الـ refcount داخلياً في كل دورة.

**Key details:**
- عند `break` من الـ loop: **يجب** استدعاء `of_node_put(child)` يدوياً — لأن `__free` غير مستخدم هنا.
- لا يدعم الـ automatic cleanup — أقدم من `for_each_of_graph_port`.

**مثال:**
```c
struct device_node *ep;
for_each_endpoint_of_node(dev->of_node, ep) {
    struct of_endpoint endpoint;
    of_graph_parse_endpoint(ep, &endpoint);
    /* معالجة الـ endpoint */
    /* لا حاجة لـ of_node_put هنا — المـ macro يتولى ذلك */
}
/* إذا احتجت break: */
for_each_endpoint_of_node(dev->of_node, ep) {
    if (condition) {
        of_node_put(ep); /* إلزامي */
        break;
    }
}
```

---

#### `for_each_of_graph_port`

```c
#define for_each_of_graph_port(parent, child)           \
    for (struct device_node *child __free(device_node) = \
             of_graph_get_next_port(parent, NULL);       \
         child != NULL;                                  \
         child = of_graph_get_next_port(parent, child))
```

**الـ macro الحديث** للمرور على الـ ports. يستخدم `__free(device_node)` من `linux/cleanup.h` للـ automatic `of_node_put` عند خروج الـ variable من scope.

**Key details:**
- `child` مُعرَّف داخل الـ loop — C99 scoping.
- عند `break` مع الحاجة للاحتفاظ بـ `child`: استخدم `no_free_ptr(child)` أو `return_ptr(child)` لنقل الملكية وإلغاء الـ auto-free.
- أحدث وأأمن من `for_each_endpoint_of_node`.

---

#### `for_each_of_graph_port_endpoint`

```c
#define for_each_of_graph_port_endpoint(parent, child)           \
    for (struct device_node *child __free(device_node) =          \
             of_graph_get_next_port_endpoint(parent, NULL);       \
         child != NULL;                                           \
         child = of_graph_get_next_port_endpoint(parent, child))
```

للمرور على الـ endpoints داخل **port واحد** مع automatic cleanup. نفس semantics الـ `__free` كـ `for_each_of_graph_port`.

**Key details:**
- `parent` هنا هو الـ **port node** — ليس الـ device node الجذر.

---

### ملاحظات عامة حول إدارة الـ Reference Count

| السيناريو | الإجراء المطلوب |
|-----------|----------------|
| نتيجة أي دالة `of_graph_get_*` | `of_node_put()` إلزامي |
| `for_each_endpoint_of_node` — إكمال طبيعي | المـ macro يدير التنظيف |
| `for_each_endpoint_of_node` — `break` | `of_node_put(child)` يدوياً |
| `for_each_of_graph_port` — `break` | `__free` يتولى التنظيف تلقائياً |
| `for_each_of_graph_port` — الاحتفاظ بـ pointer بعد الـ loop | `no_free_ptr(child)` أو `return_ptr(child)` |
| `of_graph_get_port_parent` | **تستهلك** الـ node المُمرَّرة — لا تستخدمها بعدها |

---

### مثال متكامل: اكتشاف الـ Topology كاملاً

```c
void discover_graph_topology(struct device *dev)
{
    struct device_node *np = dev->of_node;

    /* تحقق أولاً من وجود graph */
    if (!of_graph_is_present(np))
        return;

    /* المرور على كل الـ ports */
    for_each_of_graph_port(np, port) {
        struct of_endpoint ep_data;

        /* المرور على كل الـ endpoints في كل port */
        for_each_of_graph_port_endpoint(port, ep) {
            of_graph_parse_endpoint(ep, &ep_data);
            dev_info(dev, "port %u, endpoint %u\n",
                     ep_data.port, ep_data.id);

            /* اعثر على الـ remote device */
            struct device_node *remote =
                    of_graph_get_remote_port_parent(ep);
            if (remote) {
                dev_info(dev, "  connected to: %s\n",
                         remote->full_name);
                of_node_put(remote);
            }
        }
    }
}
```
## Phase 5: دليل الـ Debugging الشامل

### نظرة عامة

الـ **OF graph** (Open Firmware Graph) هو نظام يصف توصيلات الأجهزة في الـ Device Tree من خلال مفهوم الـ **ports** و الـ **endpoints**. يُستخدم بشكل واسع في subsystems مثل V4L2 Media، DRM/KMS، و MIPI CSI/DSI. الـ debugging هنا يتمحور حول التحقق من صحة بنية الـ DT وترتيب الـ `phandle` references بين الأجهزة.

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ OF graph لا يملك debugfs خاص به مباشرة، لكن يمكن الوصول إلى معلوماته عبر:

```bash
# عرض شجرة الـ Device Tree الكاملة من debugfs
mount -t debugfs none /sys/kernel/debug
ls /sys/kernel/debug/
cat /sys/kernel/debug/of_reserved_mem  # للذاكرة المحجوزة إن وُجدت
```

الـ **media subsystem** (أكثر مستخدم لـ OF graph) يعرض topology عبر:

```bash
# عرض الـ media graph (يعتمد على OF graph endpoints)
cat /sys/kernel/debug/media*/graph_dot
# ثم تحويله:
dot -Tpng /sys/kernel/debug/media0/graph_dot -o graph.png
```

لرؤية جميع الـ device nodes الحية:

```bash
ls /sys/kernel/debug/device_component/
# كل جهاز يظهر مكوناته المرتبطة عبر OF graph
```

---

#### 2. مدخلات الـ sysfs

```bash
# عرض كل node في الـ DT كـ directory في sysfs
ls /sys/firmware/devicetree/base/

# للبحث عن nodes التي تحتوي على ports/endpoints
find /sys/firmware/devicetree/base/ -name "port*" -o -name "endpoint*" 2>/dev/null

# قراءة خاصية reg لـ endpoint معين
xxd /sys/firmware/devicetree/base/soc/camera@XX/port@0/endpoint@0/reg

# عرض الـ remote-endpoint phandle
xxd /sys/firmware/devicetree/base/soc/camera@XX/port@0/endpoint@0/remote-endpoint

# عرض اسم الجهاز المرتبط
cat /sys/firmware/devicetree/base/soc/camera@XX/port@0/endpoint@0/remote-endpoint/../../../name
```

جدول مسارات sysfs المهمة:

| المسار | المحتوى |
|--------|---------|
| `/sys/firmware/devicetree/base/<node>/port@N/` | بيانات port رقم N |
| `/sys/firmware/devicetree/base/<node>/port@N/endpoint@M/` | بيانات endpoint M في port N |
| `/sys/firmware/devicetree/base/<node>/port@N/endpoint@M/remote-endpoint` | phandle للـ endpoint البعيد |
| `/sys/firmware/devicetree/base/<node>/ports/` | إن وُجد nodes wrapper |

---

#### 3. الـ ftrace — Tracepoints والـ Events

تفعيل تتبع دوال OF graph:

```bash
# تفعيل function tracing لدوال of_graph
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo function > current_tracer
echo 'of_graph_*' > set_ftrace_filter
echo 1 > tracing_on

# تشغيل العملية المشكوك بها (مثلاً modprobe للـ driver)
modprobe my_camera_driver

echo 0 > tracing_on
cat trace | head -100
```

تفعيل tracepoints الخاصة بالـ OF:

```bash
# عرض كل events متاحة لـ OF
ls /sys/kernel/debug/tracing/events/of/

# تفعيل جميع events الخاصة بالـ OF
echo 1 > /sys/kernel/debug/tracing/events/of/enable

# مثال على output يُظهر of_graph_get_next_endpoint:
#   <...>-1234  [000] .... 123.456: of_graph_get_next_endpoint <-v4l2_async_register_subdev
```

لـ media graph specifically:

```bash
ls /sys/kernel/debug/tracing/events/v4l2/
echo 1 > /sys/kernel/debug/tracing/events/v4l2/enable
```

---

#### 4. الـ printk والـ dynamic debug

تفعيل رسائل debug لـ OF graph subsystem:

```bash
# تفعيل dynamic debug لملفات OF graph
echo 'file of_graph.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/of/base.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل لـ module معين يستخدم OF graph
echo 'module my_camera_driver +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع stack trace لكل رسالة
echo 'file of_graph.c +ps' > /sys/kernel/debug/dynamic_debug/control

# عرض ما تم تفعيله حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep of_graph

# عبر kernel command line عند الإقلاع
# في /etc/default/grub أو bootloader:
# dyndbg="file of_graph.c +p"
```

مستوى الـ printk المؤقت:

```bash
# رفع مستوى الـ log لرؤية KERN_DEBUG
echo 8 > /proc/sys/kernel/printk
# أو
dmesg -n 8
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| الخيار | الوظيفة | تأثير الأداء |
|--------|---------|-------------|
| `CONFIG_OF` | تفعيل OF أساساً | لا شيء |
| `CONFIG_OF_DYNAMIC` | دعم تعديل DT في runtime | منخفض |
| `CONFIG_OF_DEBUG` | رسائل debug إضافية لـ OF | منخفض |
| `CONFIG_DEBUG_DRIVER` | طباعة معلومات الـ driver binding | منخفض |
| `CONFIG_FTRACE` | تفعيل function tracing | متوسط |
| `CONFIG_DYNAMIC_DEBUG` | dynamic debug للـ pr_debug | منخفض |
| `CONFIG_OF_UNITTEST` | وحدات اختبار لـ OF | لا شيء في production |
| `CONFIG_MEDIA_CONTROLLER` | graph الخاص بـ V4L2/DRM | لا شيء |
| `CONFIG_VIDEO_V4L2_SUBDEV_API` | API للـ subdevices عبر OF graph | لا شيء |
| `CONFIG_DEBUG_FS` | تفعيل debugfs | منخفض |

للتحقق من الخيارات المفعّلة في الكرنل الحالي:

```bash
grep -E 'CONFIG_OF|CONFIG_DEBUG_DRIVER|CONFIG_OF_DYNAMIC' /boot/config-$(uname -r)
# أو
zcat /proc/config.gz | grep -E 'CONFIG_OF|CONFIG_DEBUG'
```

---

#### 6. أدوات خاصة بالـ Subsystem

**أداة `dtc` لفحص الـ DT:**

```bash
# تحويل DT binary إلى نص قابل للقراءة
dtc -I dtb -O dts /sys/firmware/fdt > /tmp/current.dts

# البحث عن كل ports/endpoints
grep -n -A5 'remote-endpoint' /tmp/current.dts

# التحقق من صحة بنية OF graph
grep -n 'endpoint@\|port@\|remote-endpoint\|reg = ' /tmp/current.dts
```

**أداة `media-ctl` للـ V4L2 media graph:**

```bash
# عرض topology كاملة مبنية على OF graph
media-ctl -d /dev/media0 -p

# عرض كـ dot graph
media-ctl -d /dev/media0 --print-dot | dot -Tpng -o media_graph.png

# إعداد format عبر link (يعتمد على OF graph للـ routing)
media-ctl -d /dev/media0 -V '"sensor 0-0010":0 [fmt:SGRBG10/640x480]'
```

**أداة `v4l2-ctl`:**

```bash
# عرض معلومات الجهاز المرتبطة بـ OF graph
v4l2-ctl --device=/dev/video0 --info
v4l2-ctl --list-devices
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|-------------|--------|------|
| `OF: graph: no port node found in <node>` | الـ node لا يحتوي على `port@N` | أضف `port@0 { ... }` في DT |
| `OF: graph: endpoint <node> has no 'remote-endpoint' property` | مفقود `remote-endpoint = <&phandle>` | أضف الـ phandle الصحيح |
| `OF: graph: invalid phandle` | الـ phandle يشير إلى node غير موجود | تحقق من صحة الـ label المستخدم |
| `could not get #address-cells` | ports node لا يحتوي `#address-cells` | أضف `#address-cells = <1>; #size-cells = <0>;` |
| `v4l2-async: can't find subdev node <node>` | الـ subdev لم يُسجَّل بعد | تحقق من ترتيب الـ probe/deferred probe |
| `no device found for node <node>` | `of_graph_get_port_parent()` فشل | تحقق من الـ phandle chain |
| `endpoint->remote_np is NULL` | `of_graph_get_remote_endpoint()` أعاد NULL | تحقق من `remote-endpoint` property |
| `failed to get remote device node` | `of_graph_get_remote_port_parent()` فشل | الجهاز البعيد غير موجود في DT |
| `-EPROBE_DEFER` في dmesg | الجهاز ينتظر جهاز آخر (عبر OF graph) | طبيعي — تأكد من تسجيل الجهاز الآخر |
| `of_graph_get_endpoint_count returned 0` | لا endpoints مكتشفة | خطأ في بنية DT (port/endpoint غير صحيح) |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

في driver يستخدم OF graph، ضع هذه النقاط للتشخيص:

```c
static int my_driver_probe(struct platform_device *pdev)
{
    struct device_node *ep, *remote;
    struct of_endpoint endpoint;

    /* نقطة 1: تحقق من وجود OF graph أصلاً */
    if (!of_graph_is_present(pdev->dev.of_node)) {
        dev_warn(&pdev->dev, "no OF graph found\n");
        /* WARN هنا مفيد إن كان OF graph إلزامياً */
        WARN_ON(!of_graph_is_present(pdev->dev.of_node));
        return -ENODEV;
    }

    ep = of_graph_get_next_endpoint(pdev->dev.of_node, NULL);

    /* نقطة 2: تحقق من parse endpoint */
    if (ep) {
        int ret = of_graph_parse_endpoint(ep, &endpoint);
        if (WARN_ON(ret < 0)) {
            /* dump_stack هنا يُظهر من استدعى probe */
            dump_stack();
            of_node_put(ep);
            return ret;
        }

        dev_dbg(&pdev->dev, "port=%u id=%u\n",
                endpoint.port, endpoint.id);

        /* نقطة 3: التحقق من الـ remote endpoint */
        remote = of_graph_get_remote_endpoint(ep);
        if (WARN_ON(!remote)) {
            dev_err(&pdev->dev,
                    "endpoint %pOF has no remote\n", ep);
            of_node_put(ep);
            return -EINVAL;
        }

        /* نقطة 4: التحقق من الجهاز البعيد */
        struct device_node *remote_parent =
            of_graph_get_remote_port_parent(ep);
        if (WARN_ON(!remote_parent)) {
            dump_stack(); /* من المفترض ألا يحدث هذا */
            of_node_put(remote);
            of_node_put(ep);
            return -EINVAL;
        }

        of_node_put(remote_parent);
        of_node_put(remote);
        of_node_put(ep);
    }
    return 0;
}
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ Hardware مع الـ Kernel

```bash
# التحقق من أن الكرنل يرى نفس بنية DT كما هو في الـ hardware
# قراءة الـ DTB المُحمَّل فعلياً:
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A 20 'csi\|mipi\|camera\|isp'

# مقارنة عدد الـ endpoints المكتشفة مع ما هو موجود فيزيائياً:
# إذا كانت الكاميرا CSI لها 4 lanes فيجب أن يكون في DT: data-lanes = <1 2 3 4>
grep -A5 'data-lanes' /tmp/current.dts
```

للتحقق من حالة الـ clocks المرتبطة بـ endpoints:

```bash
# فحص حالة الـ clocks
cat /sys/kernel/debug/clk/clk_summary | grep -i 'csi\|mipi\|isp'

# التحقق من حالة الـ regulators
cat /sys/kernel/debug/regulator/regulator_summary
```

---

#### 2. قراءة الـ Registers (Register Dump)

```bash
# قراءة register واحد (عنوان MIPI CSI controller مثلاً)
# devmem2 أداة userspace تقرأ /dev/mem
devmem2 0xFE080000 w   # قراءة 4 bytes من عنوان 0xFE080000

# قراءة نطاق من الـ registers
for addr in $(seq 0xFE080000 4 0xFE0800FF); do
    printf "0x%08X: " $addr
    devmem2 $addr w 2>/dev/null | grep 'Read at'
done

# أو عبر /dev/mem مباشرة (يحتاج CONFIG_DEVMEM=y)
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 0x100, offset=0xFE080000)
    for i in range(0, 0x100, 4):
        val = struct.unpack('<I', m[i:i+4])[0]
        print(f'0x{0xFE080000+i:08X}: 0x{val:08X}')
"

# عبر io utility (من package i2c-tools أو busybox)
io -4 -r 0xFE080000
```

> **تحذير**: تعطيل CONFIG_STRICT_DEVMEM قد يكون مطلوباً في بعض kernels. استخدم `iomem=relaxed` في kernel cmdline للـ development.

---

#### 3. نصائح Logic Analyzer / Oscilloscope

عند تشخيص مشاكل OF graph على مستوى الـ hardware الفيزيائي:

**لـ MIPI CSI-2 (الأكثر شيوعاً مع OF graph):**

```
المسارات المهمة للقياس:
┌─────────────────────────────────────────────────┐
│  Camera Sensor ──[MIPI D-PHY]──> CSI-2 Receiver │
│                                                  │
│  Signals:                                        │
│    CLK+/CLK-  : Clock lane (differential)        │
│    D0+/D0-    : Data lane 0 (differential)       │
│    D1+/D1-    : Data lane 1 (إن وُجد)            │
│                                                  │
│  Logic Analyzer: Protocol mode MIPI CSI-2        │
│  Oscilloscope: LP/HS transition timing           │
│    LP-11 → LP-01 → LP-00 → HS-0 → data          │
└─────────────────────────────────────────────────┘
```

نقاط القياس والتفسير:

| الإشارة | الحالة الطبيعية | حالة الخطأ |
|---------|----------------|-----------|
| CLK lane في LP mode | 1.2V differential | 0V → clock لم يبدأ |
| HS burst timing | تتوافق مع `link-frequencies` في DT | jitter عالٍ → تحقق من DT |
| data-lanes count | يتطابق مع `data-lanes = <...>` | signal غير موجود → DT خاطئ |
| VSYNC/HSYNC | تتوافق مع timing في DT | غائب → sensor لم يُبرمج |

**للـ I2C (لبرمجة الـ sensor عبر OF graph):**

```bash
# تحقق من waveform الـ I2C (400KHz أو 1MHz)
# Oscilloscope: قس SDA و SCL
# Logic Analyzer: Protocol I2C، ابحث عن NAK
i2cdetect -y -r 1   # bus 1 مثلاً
i2cdump -y 1 0x3C   # sensor address 0x3C
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في dmesg

| المشكلة الفيزيائية | النمط في dmesg | التشخيص |
|-------------------|----------------|---------|
| كابل MIPI مقلوب / غير موصول | `csi: failed to lock clock lane` | Logic analyzer على CLK lane |
| جهد الـ VDD للـ sensor خاطئ | `i2c: transaction failed` → ثم `of_graph: endpoint has no remote device` | قس الجهد بـ multimeter |
| عدم تطابق عدد الـ data lanes | `csi: lane count mismatch: dt=4 hw=2` | راجع `data-lanes` في DT |
| الـ sensor لم يُبرمج | `v4l2: no valid signal detected` | Logic analyzer على I2C |
| مشكلة في الـ clock source | `clk: failed to set rate for csi_clk` | `cat /sys/kernel/debug/clk/clk_summary` |
| reference clock خاطئ | `sensor: invalid mclk frequency` | راجع `clock-frequency` في DT |

---

#### 5. تشخيص الـ Device Tree — التحقق من التطابق مع الـ Hardware

**أداة `dtc` للتحقق من بنية OF graph:**

```bash
# تحليل ملف DTS قبل compile
dtc -I dts -O dtb -o /tmp/test.dtb my_board.dts 2>&1
# أي خطأ هنا يعني بنية OF graph غير صحيحة

# التحقق من أن كل remote-endpoint يشير إلى endpoint حقيقي
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | \
    grep -B2 -A2 'remote-endpoint'
```

**مثال: بنية OF graph صحيحة في DTS:**

```dts
/* Camera sensor node */
&i2c1 {
    camera: sensor@3c {
        compatible = "ovti,ov5640";
        reg = <0x3c>;

        port {  /* OF graph: port@0 ضمني */
            cam_ep: endpoint {
                remote-endpoint = <&csi_ep>;  /* phandle للـ CSI */
                data-lanes = <1 2>;           /* 2-lane MIPI */
                clock-lanes = <0>;
            };
        };
    };
};

/* CSI-2 receiver node */
&csi {
    ports {
        #address-cells = <1>;
        #size-cells = <0>;

        port@0 {
            reg = <0>;
            csi_ep: endpoint {
                remote-endpoint = <&cam_ep>;  /* phandle للـ camera */
                data-lanes = <1 2>;           /* يجب أن يتطابق */
                clock-lanes = <0>;
            };
        };
    };
};
```

**نقاط التحقق في DT:**

```bash
# 1. تحقق من أن remote-endpoint bidirectional (كل طرف يشير للآخر)
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | \
    grep -A1 'remote-endpoint' | grep phandle

# 2. تحقق من تطابق data-lanes في الطرفين
grep -A10 'endpoint' /tmp/current.dts | grep 'data-lanes'

# 3. تحقق من reg values تبدأ من 0
grep -B5 'endpoint' /tmp/current.dts | grep 'reg = '

# 4. تشغيل of_unittest (إن كان مفعلاً)
dmesg | grep 'OF: selftest'
# يجب أن تظهر: "OF: selftest: Ran X tests; 0 failures"
```

---

### Practical Commands — أوامر جاهزة للنسخ

#### مجموعة فحص سريع (Quick Health Check)

```bash
#!/bin/bash
# of_graph_debug.sh — فحص سريع لـ OF graph

echo "=== OF Graph Quick Debug ==="
echo ""

echo "--- 1. Device Tree: ports and endpoints ---"
find /sys/firmware/devicetree/base/ \
    -name "endpoint*" -o -name "port@*" 2>/dev/null | sort

echo ""
echo "--- 2. Kernel log: OF graph messages ---"
dmesg | grep -iE 'of.*graph|of.*endpoint|of.*port|remote.endpoint' | tail -20

echo ""
echo "--- 3. Dynamic debug: enable OF graph ---"
echo 'file drivers/of/property.c +p' > /sys/kernel/debug/dynamic_debug/control 2>/dev/null
echo 'file of_graph.c +p' > /sys/kernel/debug/dynamic_debug/control 2>/dev/null

echo ""
echo "--- 4. Media devices (if applicable) ---"
ls /dev/media* 2>/dev/null && media-ctl -p 2>/dev/null | head -40

echo ""
echo "--- 5. V4L2 devices ---"
v4l2-ctl --list-devices 2>/dev/null
```

#### فحص endpoint محدد

```bash
# بدل <node-path> بالمسار الفعلي
NODE_PATH="/sys/firmware/devicetree/base/soc/csi@0"

echo "=== Endpoints in $NODE_PATH ==="
find "$NODE_PATH" -name "endpoint*" | while read ep; do
    echo "Endpoint: $ep"
    echo -n "  reg: "; xxd "$ep/reg" 2>/dev/null | head -1
    echo -n "  remote-endpoint phandle: "
    xxd "$ep/remote-endpoint" 2>/dev/null | head -1
    echo ""
done
```

#### تتبع دوال OF graph بـ ftrace

```bash
#!/bin/bash
# تشغيل ftrace لتتبع of_graph functions

TRACE=/sys/kernel/debug/tracing

echo 0 > $TRACE/tracing_on
echo > $TRACE/trace
echo function > $TRACE/current_tracer
echo 'of_graph_*:traceon' > $TRACE/set_ftrace_filter
echo 1 > $TRACE/tracing_on

echo "Probing driver..."
modprobe $1  # اسم الـ module كـ argument

sleep 2
echo 0 > $TRACE/tracing_on

echo "=== OF Graph Function Calls ==="
grep 'of_graph' $TRACE/trace | head -50
```

```bash
# تشغيله:
bash ftrace_of_graph.sh my_camera_driver
```

#### مثال على الـ output وتفسيره

```
# مثال على نتيجة: media-ctl -p
Media controller API version 5.15.0

Media device information
------------------------
driver          my-isp
model           ISP Camera
serial
bus info        platform:fe080000.isp

Device topology
- entity 1: ov5640 1-003c (1 pad, 1 link)    ← sensor (OF node: i2c1/camera@3c)
            type V4L2 subdev subtype Sensor flags 0
            device node name /dev/v4l-subdev0
        pad0: Source                           ← port@0/endpoint@0 في DT
                -> "csi 0" 0:0 [ENABLED]      ← remote-endpoint يشير إلى CSI

- entity 2: csi 0 (2 pads, 2 links)          ← CSI controller (OF node: csi@fe080000)
            type V4L2 subdev subtype Unknown flags 0
            device node name /dev/v4l-subdev1
        pad0: Sink                             ← port@0/endpoint@0
                [fmt:SGRBG10/1920x1080]
                <- "ov5640 1-003c" 0:0 [ENABLED]
        pad1: Source                           ← port@1/endpoint@0
                -> "isp-video" 0:0 [ENABLED]
```

**تفسير الـ output:**
- `[ENABLED]` = الـ link نشط → OF graph تم parse بنجاح
- `[IMMUTABLE]` = الـ link ثابت (مُعرَّف في DT)
- عدم ظهور link = فشل في `of_graph_get_remote_endpoint()` → خطأ في DT

#### أمر شامل للتشخيص

```bash
# تشخيص شامل في سطر واحد
dmesg --level=err,warn | grep -iE 'of_graph|endpoint|remote.endpoint|phandle' && \
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | \
    awk '/endpoint/{found=1} found{print; if(/}/) {found=0; count++; if(count>20) exit}}'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: شاشة HDMI لا تعمل على بورد Rockchip RK3562

#### العنوان
**HDMI output dead** بعد نقل Device Tree من مشروع قديم إلى BSP جديد

#### السياق
فريق يبني **Android TV box** بمعالج **RK3562**. الـ display pipeline تمر عبر: `vop` (Video Output Processor) → `hdmi` → connector. المطوّر نقل ملف `.dts` من مشروع RK3566 قديم وعدّل أسماء الـ nodes فقط. الـ boot يكتمل لكن الشاشة تبقى سوداء تماماً.

#### المشكلة
الـ `drm` driver يستدعي `of_graph_get_remote_node()` للبحث عن الـ HDMI node المرتبط بالـ VOP، لكن الدالة تعيد `NULL` دائماً.

#### التحليل
الدالة المعنية في `of_graph.h`:

```c
struct device_node *of_graph_get_remote_node(const struct device_node *node,
                                             u32 port, u32 endpoint);
```

الـ driver يستدعيها هكذا:

```c
/* داخل driver RK drm */
struct device_node *hdmi_node =
    of_graph_get_remote_node(vop_node, 1, 0);
/* port=1 يعني منفذ HDMI، endpoint=0 */
```

عند فحص الـ DTS المنقول:

```dts
/* خطأ: port@1 مفقود، endpoint يشير لـ port@0 فقط */
&vop {
    ports {
        port@0 {
            reg = <0>;
            vop_out_rgb: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&rgb_in>;
            };
        };
        /* port@1 مفقود تماماً! */
    };
};
```

`of_graph_get_remote_node()` داخلياً تستدعي `of_graph_get_endpoint_by_regs(node, 1, 0)` التي تمشي عبر `for_each_endpoint_of_node` بحثاً عن endpoint بـ `port_reg=1` و`reg=0`، فلا تجده.

#### الحل

```dts
&vop {
    ports {
        port@0 {
            reg = <0>;
            vop_out_rgb: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&rgb_in>;
            };
        };
        /* إضافة port@1 الصحيح */
        port@1 {
            reg = <1>;
            vop_out_hdmi: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&hdmi_in_vop>;
            };
        };
    };
};

&hdmi {
    ports {
        port@0 {
            reg = <0>;
            hdmi_in_vop: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&vop_out_hdmi>;
            };
        };
    };
};
```

**أوامر التحقق:**

```bash
# تحقق من وجود endpoint بالأرقام الصحيحة
dtc -I dtb -O dts /sys/firmware/fdt | grep -A5 "port@1"

# تتبع الـ graph من kernel
cat /sys/kernel/debug/of_graph/...
```

#### الدرس المستفاد
**الـ `port` و `endpoint` أرقام (`reg`) ليست اختيارية** — `of_graph_get_endpoint_by_regs()` تبحث بالأرقام بدقة. نقل DTS بين SoCs مختلفة يتطلب مراجعة كاملة لبنية الـ `ports` وليس فقط أسماء الـ nodes.

---

### السيناريو 2: crash عشوائي في driver كاميرا على STM32MP1

#### العنوان
**kernel panic** بسبب استخدام `for_each_endpoint_of_node` دون `of_node_put` عند الخروج المبكر

#### السياق
منتج **IoT sensor** يحمل **STM32MP1** متصل بكاميرا **OV5640** عبر MIPI-CSI2. الـ driver يمشي على endpoints للبحث عن الكاميرا المرتبطة. النظام يعمل لساعات ثم يُصدر `BUG: Bad page state` عشوائياً.

#### المشكلة
**memory leak** في `device_node` reference count يؤدي لـ use-after-free.

#### التحليل
الـ macro في `of_graph.h`:

```c
#define for_each_endpoint_of_node(parent, child) \
    for (child = of_graph_get_next_endpoint(parent, NULL); child != NULL; \
         child = of_graph_get_next_endpoint(parent, child))
```

التعليق يوضح صراحةً:
> *"When breaking out of the loop, `of_node_put(child)` has to be called manually."*

الكود المعطوب:

```c
struct device_node *ep;
for_each_endpoint_of_node(csi_node, ep) {
    struct device_node *remote =
        of_graph_get_remote_port_parent(ep);
    if (remote && of_device_is_compatible(remote, "ovti,ov5640")) {
        sensor_node = remote;
        break; /* خروج مبكر دون of_node_put(ep) ! */
    }
    of_node_put(remote);
}
```

كل `break` يترك `ep` بـ refcount مرتفع، مما يمنع تحرير الـ `device_node` من الـ `of_node_cache`، وبعد آلاف iterations يحدث corruption.

#### الحل

```c
struct device_node *ep;
for_each_endpoint_of_node(csi_node, ep) {
    struct device_node *remote =
        of_graph_get_remote_port_parent(ep);
    if (remote && of_device_is_compatible(remote, "ovti,ov5640")) {
        sensor_node = remote;
        of_node_put(ep); /* إطلاق ep قبل الخروج */
        break;
    }
    of_node_put(remote);
}
```

**أو الأفضل:** استخدام الـ macros الجديدة التي تدعم `__free(device_node)` الـ cleanup أوتوماتيكياً:

```c
/* for_each_of_graph_port و for_each_of_graph_port_endpoint */
/* تستخدم __free() — لا حاجة لـ manual put عند break */
for_each_of_graph_port(csi_node, port) {
    for_each_of_graph_port_endpoint(port, ep) {
        struct device_node *remote =
            of_graph_get_remote_port_parent(ep);
        if (remote &&
            of_device_is_compatible(remote, "ovti,ov5640")) {
            sensor_node = no_free_ptr(remote);
            goto found; /* الـ __free() سيُنظّف ep و port */
        }
        of_node_put(remote);
    }
}
found:
```

#### الدرس المستفاد
**الـ `for_each_endpoint_of_node` legacy macro لا تنظّف تلقائياً** عند الخروج المبكر. الـ macros الحديثة (`for_each_of_graph_port`, `for_each_of_graph_port_endpoint`) تستخدم `__free(device_node)` من `linux/cleanup.h` لضمان إطلاق الـ refcount تلقائياً — استخدمها دائماً في الكود الجديد.

---

### السيناريو 3: فشل probe لـ DSI display على i.MX8MM

#### العنوان
**driver لا يجد الـ panel** رغم صحة الـ DTS ظاهرياً — مشكلة في `of_graph_is_present()`

#### السياق
فريق bring-up يعمل على **custom board** بـ **i.MX8MM** و panel **MIPI-DSI** من نوع **Ilitek ILI9881**. الـ `mipi_dsi` host driver يُطبع:

```
mipi_dsi 30a00000.mipi_dsi: no graph found, skipping
```

وينهي الـ probe بدون خطأ — مجرد skip.

#### المشكلة
`of_graph_is_present()` تعيد `false` رغم وجود nodes في الـ DTS.

#### التحليل
الدالة:

```c
bool of_graph_is_present(const struct device_node *node);
```

تبحث عن وجود `port` أو `ports` child مباشر أو عبر `ports` container. الـ DTS المكتوب خطأ:

```dts
/* خطأ: ports موجودة لكن داخل label خاطئ */
&mipi_dsi {
    status = "okay";
    /* المطوّر كتب port مباشرة بدون reg */
    port {
        mipi_dsi_out: endpoint {
            remote-endpoint = <&panel_in>;
        };
    };
};
```

المشكلة: `port` بدون `reg` property. `of_graph_is_present()` تبحث عن nodes اسمها `port` أو `ports`، وهي موجودة، لكن `of_graph_get_next_endpoint()` تعتمد على `reg` للـ iteration، فتفشل عند محاولة parse الـ endpoint.

**الفحص:**

```bash
# فحص وجود reg في port node
cat /sys/firmware/devicetree/base/mipi_dsi@30a00000/port/reg
# xxd يُظهر: لا شيء
```

#### الحل

```dts
&mipi_dsi {
    status = "okay";
    ports {
        #address-cells = <1>;
        #size-cells = <0>;
        port@1 {          /* port output عادة reg=1 لـ i.MX DSI */
            reg = <1>;
            mipi_dsi_out: endpoint {
                remote-endpoint = <&panel_in>;
            };
        };
    };
};

&panel {
    ports {
        #address-cells = <1>;
        #size-cells = <0>;
        port@0 {
            reg = <0>;
            panel_in: endpoint {
                remote-endpoint = <&mipi_dsi_out>;
            };
        };
    };
};
```

**التحقق:**

```bash
# بعد تحميل dtb الجديد
of_graph_get_endpoint_count() يجب أن يُرجع > 0
# أو من sysfs:
ls /sys/firmware/devicetree/base/mipi_dsi@30a00000/ports/
```

#### الدرس المستفاد
**`of_graph_is_present()` تتحقق من وجود البنية فقط** — لا تتحقق من صحة الـ `reg` values. الفشل الحقيقي يظهر لاحقاً في `of_graph_parse_endpoint()`. دائماً أضف `#address-cells` و`#size-cells` داخل `ports` وأعطِ كل `port` و`endpoint` قيمة `reg` صريحة.

---

### السيناريو 4: تعارض بين مسارين صوتيين على AM62x

#### العنوان
**صوت HDMI يتداخل مع McASP** — driver يربط endpoint خاطئ بسبب `port_reg` مكرر

#### السياق
**industrial gateway** بـ **TI AM62x** يحتاج مسارين صوتيين مستقلين:
- **McASP0** → I2S → codec خارجي (للمصنع)
- **HDMI audio** → عبر display subsystem (للمراقبة)

الـ ALSA driver يبدأ ويُسجّل كلا الـ sound cards، لكن عند تشغيل صوت على HDMI يصدر الصوت من الـ codec الخارجي فقط.

#### المشكلة
كلا الـ audio endpoints يملكان `port_reg=0` و`reg=0`، فـ `of_graph_get_endpoint_by_regs()` تُرجع أول endpoint تجده.

#### التحليل

```c
struct device_node *of_graph_get_endpoint_by_regs(
    const struct device_node *parent, int port_reg, int reg);
```

الدالة تمشي عبر `for_each_endpoint_of_node` وتقارن:
```c
of_graph_parse_endpoint(endpoint, &ep);
if ((port_reg == -1 || ep.port == port_reg) &&
    (reg == -1 || ep.id == reg))
    return endpoint;
```

الـ DTS المعطوب:

```dts
&audio_subsystem {
    ports {
        #address-cells = <1>;
        #size-cells = <0>;
        /* كلاهما port@0 — خطأ! */
        port@0 {
            reg = <0>;
            mcasp_ep: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&codec_in>;
            };
        };
        port@0 { /* مكرر! يتجاهله OF parser */
            reg = <0>;
            hdmi_audio_ep: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&hdmi_audio_in>;
            };
        };
    };
};
```

الـ OF parser يتجاهل الـ `port@0` الثاني لأن اسمه مكرر، فيبقى فقط endpoint واحد مرئي.

#### الحل

```dts
&audio_subsystem {
    ports {
        #address-cells = <1>;
        #size-cells = <0>;
        port@0 {   /* McASP — port 0 */
            reg = <0>;
            mcasp_ep: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&codec_in>;
            };
        };
        port@1 {   /* HDMI audio — port 1 */
            reg = <1>;
            hdmi_audio_ep: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&hdmi_audio_in>;
            };
        };
    };
};
```

**في الـ driver:**

```c
/* تحديد كل endpoint بـ port_reg مختلف */
struct device_node *mcasp_ep =
    of_graph_get_endpoint_by_regs(audio_node, 0, 0);
struct device_node *hdmi_ep =
    of_graph_get_endpoint_by_regs(audio_node, 1, 0);
```

**فحص عدد الـ ports:**

```bash
# يجب أن يُرجع 2
od -t u4 /sys/firmware/devicetree/base/.../ports/\#address-cells
```

#### الدرس المستفاد
**`port@N` يجب أن تكون أرقامه فريدة** ضمن نفس الـ `ports` container — الـ OF tree لا يسمح بـ duplicate node names. `of_graph_get_endpoint_by_regs()` تعمل بالأرقام الفعلية (`ep.port`) لا بترتيب الظهور. استخدم `of_graph_get_port_count()` للتحقق من عدد الـ ports المكتشفة فعلاً.

---

### السيناريو 5: MIPI-CSI كاميرا ثانية لا تعمل على Allwinner H616

#### العنوان
**dual camera system** — الكاميرا الثانية invisible للـ V4L2 framework

#### السياق
**automotive ECU** بـ **Allwinner H616** يتطلب كاميرتين MIPI-CSI متزامنتين: كاميرا أمامية (**OV2718**) وخلفية (**OV2718** أيضاً) لنظام ADAS. الـ `sun6i-csi` driver يكتشف الكاميرا الأمامية ويُسجّلها، لكن الخلفية لا تظهر في `/dev/video*`.

#### المشكلة
الـ driver يستخدم `of_graph_get_next_endpoint()` بشكل خاطئ — يفترض endpoint واحد فقط.

#### التحليل

```c
struct device_node *of_graph_get_next_endpoint(
    const struct device_node *parent,
    struct device_node *previous);
```

هذه الدالة تدعم الـ iteration: عند تمرير `NULL` كـ `previous` تُرجع أول endpoint، وعند تمرير endpoint سابق تُرجع التالي.

كود الـ driver المعطوب:

```c
static int sun6i_csi_probe(struct platform_device *pdev)
{
    struct device_node *ep_node;

    /* يأخذ أول endpoint فقط ويتجاهل الباقي */
    ep_node = of_graph_get_next_endpoint(dev->of_node, NULL);
    if (!ep_node) {
        dev_err(dev, "no endpoint found\n");
        return -ENODEV;
    }
    sun6i_register_sensor(ep_node);
    of_node_put(ep_node);
    /* انتهى — الكاميرا الثانية لم تُعالج */
}
```

الـ DTS يحتوي كلا الكاميرتين:

```dts
&csi {
    ports {
        #address-cells = <1>;
        #size-cells = <0>;
        port@0 {
            reg = <0>;
            csi_ep0: endpoint@0 {  /* OV2718 أمامي */
                reg = <0>;
                remote-endpoint = <&cam0_out>;
            };
            csi_ep1: endpoint@1 {  /* OV2718 خلفي */
                reg = <1>;
                remote-endpoint = <&cam1_out>;
            };
        };
    };
};
```

`of_graph_get_endpoint_count()` تُرجع `2` لكن الـ driver لا يستخدمها.

#### الحل

```c
static int sun6i_csi_probe(struct platform_device *pdev)
{
    struct device_node *ep_node;
    unsigned int count;

    /* فحص عدد الـ endpoints أولاً */
    count = of_graph_get_endpoint_count(dev->of_node);
    dev_info(dev, "found %u camera endpoint(s)\n", count);

    /* iteration صحيح على جميع الـ endpoints */
    for_each_endpoint_of_node(dev->of_node, ep_node) {
        struct of_endpoint ep;
        int ret;

        ret = of_graph_parse_endpoint(ep_node, &ep);
        if (ret) {
            dev_warn(dev, "failed to parse endpoint\n");
            continue; /* of_node_put يحدث في الدورة التالية */
        }

        dev_info(dev, "registering sensor at port=%u, ep=%u\n",
                 ep.port, ep.id);

        ret = sun6i_register_sensor(ep_node);
        if (ret)
            dev_err(dev, "sensor %u registration failed\n", ep.id);
        /* لا of_node_put هنا — الـ macro تتولاه في الدورة التالية */
    }
    return 0;
}
```

**التحقق من الـ graph:**

```bash
# عرض جميع endpoints المكتشفة
find /sys/firmware/devicetree/base/ -name "endpoint*" | \
    xargs -I{} sh -c 'echo "=== {} ===" && ls {}'

# فحص remote-endpoint pointers
cat /sys/firmware/devicetree/base/csi/.../port@0/endpoint@0/remote-endpoint | xxd
```

**اختبار V4L2:**

```bash
v4l2-ctl --list-devices
# يجب أن يظهر جهازان: /dev/video0 و /dev/video1
```

#### الدرس المستفاد
**`of_graph_get_endpoint_count()` يجب أن تكون أول استدعاء في driver يدعم multi-sensor** — إذا أرجعت قيمة > 1 يتحول الكود لـ iteration كامل بـ `for_each_endpoint_of_node`. افتراض endpoint وحيد هو خطأ شائع في bring-up يُخفي نصف الـ hardware. أيضاً، `of_graph_parse_endpoint()` ضرورية لاستخراج `port` و`id` الفعليين لتمييز الكاميرات عن بعضها.
## Phase 7: مصادر ومراجع

### توثيق النواة الرسمي

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| **DeviceTree Kernel API** | [docs.kernel.org/devicetree/kernel-api.html](https://docs.kernel.org/devicetree/kernel-api.html) | التوثيق الرسمي لكامل API الخاص بـ `of_graph`، بما فيه `of_graph_parse_endpoint` و`of_graph_get_next_endpoint` وسائر الدوال |
| **graph.txt Binding** | [kernel.org/doc/Documentation/devicetree/bindings/graph.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/graph.txt) | المواصفة الرسمية لـ binding الخاص بـ `port` و`endpoint` و`remote-endpoint` في Device Tree |
| **graph.txt (GitHub mirror)** | [github.com/torvalds/linux blob graph.txt](https://github.com/torvalds/linux/blob/master/Documentation/devicetree/bindings/graph.txt) | نسخة من ملف binding للرجوع إليها مباشرةً |
| **Linux and the Devicetree** | [docs.kernel.org/devicetree/usage-model.html](https://docs.kernel.org/devicetree/usage-model.html) | نموذج الاستخدام العام لـ Device Tree في النواة |
| **Open Firmware & Devicetree index** | [docs.kernel.org/devicetree/index.html](https://docs.kernel.org/6.8/devicetree/index.html) | فهرس توثيق Device Tree الكامل |
| **ACPI/DSD Graphs** | [docs.kernel.org/firmware-guide/acpi/dsd/graph.html](https://docs.kernel.org/firmware-guide/acpi/dsd/graph.html) | نفس مفهوم `port/endpoint` لكن عبر ACPI DSD — مقارنة مفيدة مع `of_graph` |
| **Media Controller Core** | [docs.kernel.org/driver-api/media/mc-core.html](https://docs.kernel.org/driver-api/media/mc-core.html) | شرح معمّق لكيفية استخدام `of_graph` ضمن إطار عمل Media Controller |

---

### مقالات LWN.net ذات الصلة

هذه المقالات تغطي السياق التاريخي والتقني الذي ظهرت فيه `of_graph` وعلاقتها بالأنظمة الفرعية المجاورة:

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| **Device tree bindings** | [lwn.net/Articles/572114](https://lwn.net/Articles/572114/) | شرح عام لنظام bindings في Device Tree وكيف تُوثَّق الواجهات |
| **Device trees II: The harder parts** | [lwn.net/Articles/573409](https://lwn.net/Articles/573409/) | تعمّق في التفاصيل الصعبة لـ Device Tree، من بينها توصيف الواجهات بين الأجهزة |
| **Platform devices and device trees** | [lwn.net/Articles/448502](https://lwn.net/Articles/448502/) | كيف تتكامل `platform_device` مع Device Tree |
| **The media controller subsystem** | [lwn.net/Articles/415714](https://lwn.net/Articles/415714/) | المقال المرجعي لفهم لماذا يحتاج النظام إلى `of_graph` أصلاً — توصيف الـ pipeline |
| **Media controller (core and V4L2)** | [lwn.net/Articles/417479](https://lwn.net/Articles/417479/) | تفاصيل تقنية لكيفية ربط كيانات الوسائط عبر Graph |
| **Device-tree schemas** | [lwn.net/Articles/771621](https://lwn.net/Articles/771621/) | الانتقال من `.txt` إلى YAML schema لتوثيق bindings |
| **Device Tree Overlays** | [lwn.net/Articles/616859](https://lwn.net/Articles/616859/) | تعديل Device Tree ديناميكياً بعد boot |

---

### مناقشات Mailing List

الـ `of_graph` API نشأ من نقاشات تتعلق بالـ V4L2 وـ media pipeline. أهم المنافذ للبحث:

- **LKML Archive** — [lkml.org](https://lkml.org/)
- **lore.kernel.org** — البحث عن: `of_graph` أو `port endpoint remote-endpoint device tree`
  - مثال مباشر: [lore.kernel.org](https://lore.kernel.org/linux-media/) — قائمة `linux-media`
- **Patchwork** — متابعة سلاسل patches لـ `of_graph_get_next_endpoint`:
  - [patchwork.kernel.org — rename of_graph_get_next_endpoint](https://patchwork.kernel.org/project/linux-media/patch/87jznq6qk6.wl-kuninori.morimoto.gx@renesas.com/)
  - [patchwork.kernel.org — ASoC audio-graph-card](https://patchwork.kernel.org/project/linux-samsung-soc/patch/87zfwm5bws.wl-kuninori.morimoto.gx@renesas.com/)
- **Fix لـ multiple endpoints per port**: [lkml.kernel.org/lkml/20181211020557](https://lkml.kernel.org/lkml/20181211020557.61783-2-tony@atomide.com/T/)

---

### مشروع Xilinx كمثال تطبيقي مرجعي

الـ `of_graph` API ظهر بوضوح في دعم Xilinx Video IP Core:

- **LWN — Xilinx Video IP core**: [lwn.net/Articles/619447](https://lwn.net/Articles/619447/)
  يُظهر كيف يُبنى pipeline كامل في Device Tree ويُقرأ عبر `of_graph`.

---

### الكود المصدري الأساسي

| الملف | الوصف |
|-------|-------|
| `include/linux/of_graph.h` | ملف الترويسة موضوع الدراسة — يعرّف الـ struct والـ macros والـ API |
| `drivers/of/property.c` | تنفيذ الدوال الفعلي لـ `of_graph_*` ([GitHub](https://github.com/torvalds/linux/blob/master/drivers/of/property.c)) |
| `Documentation/devicetree/bindings/graph.txt` | مواصفة الـ binding للـ port/endpoint |
| `drivers/media/` | المستخدم الرئيسي لـ `of_graph` في النواة |

---

### الموارد التعليمية

#### eLinux.org

| الصفحة | الرابط |
|--------|--------|
| Device Tree Reference | [elinux.org/Device_Tree_Reference](https://elinux.org/Device_Tree_Reference) |
| Device Tree Mysteries | [elinux.org/Device_Tree_Mysteries](https://elinux.org/Device_Tree_Mysteries) |
| Linux Drivers Device Tree Guide | [elinux.org/Linux_Drivers_Device_Tree_Guide](https://elinux.org/Linux_Drivers_Device_Tree_Guide) |
| Device Trees (عام) | [elinux.org/Device_Trees](https://elinux.org/Device_Trees) |

#### Kernel Newbies

| الصفحة | الرابط |
|--------|--------|
| Documents | [kernelnewbies.org/Documents](https://kernelnewbies.org/Documents) |
| KernelGlossary | [kernelnewbies.org/KernelGlossary](https://kernelnewbies.org/KernelGlossary) |
| Linux 5.8 (تغييرات of_graph) | [kernelnewbies.org/Linux_5.8](https://kernelnewbies.org/Linux_5.8) |
| Laurent Pinchart — Virtual Media Controller | [kernelnewbies.org/LaurentPinchart](https://kernelnewbies.org/LaurentPinchart) |

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 14**: Linux Device Model — يشرح `device_node`، الـ reference counting، وبنية الأجهزة العامة.
- **الفصل 3**: يُقدّم أساسيات التعامل مع heirarchy الأجهزة.
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: الأجهزة والوحدات — فهم كيف تُسجَّل الأجهزة وتُكتشف.
- يُرشّح لفهم السياق العام قبل التعمق في `of_graph`.

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 16**: يتناول Open Firmware وDevice Tree في السياق المضمّن.
- مثالي لفهم كيف تُقرأ بيانات `port/endpoint` على أنظمة ARM المضمّنة.

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- يتناول subsystem architecture بعمق، مفيد لفهم كيف تتشابك `of_graph` مع بقية النواة.

---

### مصطلحات البحث الموصى بها

للعثور على مزيد من المعلومات، استخدم هذه المصطلحات في محركات البحث أو في `git log`:

```
of_graph_get_next_endpoint
of_graph_parse_endpoint
device tree port endpoint remote-endpoint linux
linux kernel media pipeline device tree
of_graph V4L2 CSI camera sensor
of_graph DRM display subsystem
linux of_graph_get_remote_node
linux firmware guide acpi dsd graph
linux kernel graph binding yaml schema
```

#### للبحث في كود النواة مباشرةً:

```bash
# البحث عن استخدامات of_graph في drivers
git log --all --oneline -- include/linux/of_graph.h

# من يستخدم of_graph_get_next_endpoint
grep -r "of_graph_get_next_endpoint" drivers/ --include="*.c" -l

# تاريخ ملف الترويسة
git log --follow -p include/linux/of_graph.h
```

---

### جدول مرجعي سريع

| الموضوع | المرجع |
|---------|--------|
| API الكاملة | [docs.kernel.org/devicetree/kernel-api.html](https://docs.kernel.org/devicetree/kernel-api.html) |
| Binding الرسمي | [kernel.org graph.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/graph.txt) |
| Media Controller | [lwn.net/Articles/415714](https://lwn.net/Articles/415714/) |
| تنفيذ الكود | [drivers/of/property.c](https://github.com/torvalds/linux/blob/master/drivers/of/property.c) |
| ACPI مقابل DT | [docs.kernel.org/firmware-guide/acpi/dsd/graph.html](https://docs.kernel.org/firmware-guide/acpi/dsd/graph.html) |
| DT bindings عامة | [lwn.net/Articles/572114](https://lwn.net/Articles/572114/) |
| eLinux Reference | [elinux.org/Device_Tree_Reference](https://elinux.org/Device_Tree_Reference) |
## Phase 8: Writing simple module

### الفكرة

سنستخدم **kprobe** لمراقبة الدالة `of_graph_parse_endpoint`، وهي دالة مُصدَّرة من نواة Linux تقوم بتحليل بيانات **OF graph endpoint** من شجرة الأجهزة (Device Tree). كلّما استدعى أيّ driver هذه الدالة — سواء كان camera subsystem أو display pipeline أو أي مكوّن يعتمد على OF graph — سيطبع الـ module معرّف الـ port ومعرّف الـ endpoint واسم الـ node.

هذا الاختيار مفيد لأن:
- `of_graph_parse_endpoint` تُستدعى في مسارات تهيئة الـ hardware الحقيقية.
- مخرجاتها (`port`, `id`, `local_node`) تعطي معلومة واضحة وقابلة للطباعة.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe module: trace of_graph_parse_endpoint calls
 *
 * Hooks into of_graph_parse_endpoint() and prints the port/endpoint
 * identifiers every time a driver parses an OF graph endpoint.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* kprobe API */
#include <linux/of_graph.h>    /* struct of_endpoint, of_graph_parse_endpoint */
#include <linux/of.h>          /* struct device_node */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("Trace of_graph_parse_endpoint via kprobe");

/* ---------------------------------------------------------------
 * pre_handler — يُستدعى قبيل تنفيذ of_graph_parse_endpoint مباشرةً.
 * regs يحمل حالة المسجّلات عند نقطة الاختراق، ومنها نستخرج المعاملات.
 * --------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64: first argument  → rdi (node)
     *            second argument → rsi (endpoint struct pointer)
     * We only read 'node' here because 'endpoint' is still unfilled
     * at pre-handler time (the function hasn't run yet).
     */
    const struct device_node *node =
        (const struct device_node *)regs->di;   /* 1st arg */

    /* node may be NULL in error paths; guard before dereferencing */
    if (node && node->full_name)
        pr_info("of_graph_probe: parsing endpoint node: %s\n",
                node->full_name);
    else
        pr_info("of_graph_probe: parsing endpoint node: (unknown)\n");

    return 0; /* 0 = continue normal execution */
}

/* ---------------------------------------------------------------
 * post_handler — يُستدعى بعد عودة of_graph_parse_endpoint بنجاح.
 * هنا نقرأ بنية of_endpoint التي ملأتها الدالة لنطبع port و id.
 * --------------------------------------------------------------- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * On x86-64: second argument is still in rsi (pointer to
     * struct of_endpoint that was filled by the callee).
     */
    const struct of_endpoint *ep =
        (const struct of_endpoint *)regs->si;   /* 2nd arg */

    if (!ep)
        return;

    pr_info("of_graph_probe: -> port=%u  endpoint_id=%u\n",
            ep->port, ep->id);
}

/* ---------------------------------------------------------------
 * تعريف الـ kprobe: نحدد اسم الدالة المستهدفة بدلاً من عنوانها
 * حتى يبقى الكود محمولاً بين إصدارات النواة المختلفة.
 * --------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "of_graph_parse_endpoint",
    .pre_handler = handler_pre,
    .post_handler = handler_post,
};

/* ---------------------------------------------------------------
 * module_init: نسجّل الـ kprobe؛ إن فشل التسجيل نُعيد كود الخطأ
 * لكي لا يُحمَّل الـ module وهو غير فعّال.
 * --------------------------------------------------------------- */
static int __init of_graph_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("of_graph_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("of_graph_probe: kprobe planted at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ---------------------------------------------------------------
 * module_exit: إلغاء تسجيل الـ kprobe ضروري لتجنّب استدعاء
 * handler_pre/post بعد تحرير ذاكرة الـ module، مما يسبب kernel panic.
 * --------------------------------------------------------------- */
static void __exit of_graph_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("of_graph_probe: kprobe removed\n");
}

module_init(of_graph_probe_init);
module_exit(of_graph_probe_exit);
```

---

### Makefile للبناء

```makefile
obj-m += of_graph_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل قسم

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | ماكروهات `module_init` / `module_exit` و `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info` / `pr_err` |
| `linux/kprobes.h` | تعريف `struct kprobe` و `register_kprobe` |
| `linux/of_graph.h` | تعريف `struct of_endpoint` اللازم لقراءة نتيجة الدالة |
| `linux/of.h` | تعريف `struct device_node` المستخدمة كأول معامل |

#### الـ `pre_handler`

**الـ** `pre_handler` يعمل قبل أن تُنفَّذ الدالة المستهدفة، لذا يصل إلى المعامل الأول (`node`) فقط — بنية `of_endpoint` لا تزال فارغة. نستخرج مؤشر الـ node من المسجّل `rdi` (اتفاقية x86-64 ABI) ونطبع اسمه الكامل.

#### الـ `post_handler`

**الـ** `post_handler` يعمل بعد عودة الدالة بنجاح، فتكون بنية `of_endpoint` قد امتلأت بقيم `port` و `id` و `local_node`. نقرأها من المسجّل `rsi` (الأرغومنت الثاني الذي لم يتغيّر موقعه على الـ stack frame) ونطبع المعرّفَين.

#### `struct kprobe`

الحقل `symbol_name` يجعل النواة تبحث بنفسها عن عنوان الدالة في جدول الرموز (`kallsyms`)، وهذا أفضل من تمرير عنوان مُشفَّر لأنه يعمل مع أي تجميعة دون تعديل.

#### `module_init` و `module_exit`

- `register_kprobe` تزرع **breakpoint** (أو `ftrace hook` حسب البنية) عند مدخل الدالة.
- `unregister_kprobe` في `module_exit` **إلزامي**: إذا أُزيل الـ module دون إلغاء التسجيل، فإن النواة ستحاول استدعاء `handler_pre` الذي لم يعد موجوداً في الذاكرة فتحدث **kernel panic**.

---

### تشغيل واختبار

```bash
# بناء الوحدة
make

# تحميلها
sudo insmod of_graph_probe.ko

# مراقبة المخرجات (قد تحتاج لتشغيل driver يستخدم OF graph)
sudo dmesg | grep of_graph_probe

# إلغاء التحميل
sudo rmmod of_graph_probe
```

**مثال على مخرج متوقّع** عند تهيئة كاميرا أو شاشة تعتمد على Device Tree graph:

```
[  12.345678] of_graph_probe: kprobe planted at of_graph_parse_endpoint (ffffffffc0a12340)
[  12.401234] of_graph_probe: parsing endpoint node: /soc/camera@fed00000/port@0/endpoint@0
[  12.401289] of_graph_probe: -> port=0  endpoint_id=0
[  12.402100] of_graph_probe: parsing endpoint node: /soc/display@ff900000/port@0/endpoint@0
[  12.402145] of_graph_probe: -> port=0  endpoint_id=0
```
