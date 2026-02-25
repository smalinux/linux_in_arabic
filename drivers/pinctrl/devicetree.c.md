## Phase 1: الصورة الكبيرة ببساطة

### إيه هو الملف ده؟

`devicetree.c` و `devicetree.h` هما الجسر اللي بيخلي الـ **pin control subsystem** يفهم الـ **Device Tree** (DT). باختصار: لو عندك chip فيها pins (زي Raspberry Pi أو Snapdragon)، الـ DT بيوصف إزاي كل pin يتوصل بأي وظيفة — UART، SPI، GPIO، إلخ. الملفين دول بيقروا الوصف ده ويحولوه لـ mapping table جوه الـ kernel.

### ليه بيوجد؟

قبل الـ Device Tree، كانت إعدادات الـ pins مكتوبة hardcoded في board files (ملفات C) لكل board على حدة — فوضى. الـ DT جه عشان يفصل الـ hardware description عن الـ code. الملف ده بيعمل الترجمة بين اللغتين:

- **DT side**: خصائص زي `pinctrl-0`, `pinctrl-names`, phandles بتشاور على nodes
- **Kernel side**: `struct pinctrl_map[]` — الشكل الداخلي اللي الـ core بيشتغل بيه

### مثال من الواقع

خد Raspberry Pi 4 (BCM2711). في الـ DT:

```dts
/* Device يطلب pin state */
uart0: serial@7e201000 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart0_pins>;   /* phandle للـ config node */
};

/* Pin controller يعرّف الـ config */
uart0_pins: uart0 {
    brcm,pins = <14 15>;
    brcm,function = <4>;  /* ALT0 = TXD/RXD */
};
```

لما `uart0` يتـ probe، الـ `pinctrl_dt_to_map()` بيمشي على `pinctrl-0`، يلاقي الـ phandle `uart0_pins`، يصعد لأبوه عشان يلاقي الـ pin controller، وبعدين يكلم الـ BCM2835 driver عشان يحول الـ node لـ `pinctrl_map` entries.

### التشبيه

تخيل مطعم كبير:
- **الـ DT** = قائمة الطلبات (menu) مكتوبة بلغة إنجليزي
- **`devicetree.c`** = الترجمان اللي بياخد الطلب ويحوله لأوردر داخلي
- **الـ pin controller driver** = الشيف اللي عارف يعمل الأكل فعلاً
- **`pinctrl_map`** = الأوردر الداخلي المكتوب على ورقة المطبخ

### Maintainer

حسب `MAINTAINERS`:

```
PIN CONTROL SUBSYSTEM
M:  Linus Walleij <linusw@kernel.org>
L:  linux-gpio@vger.kernel.org
S:  Maintained
T:  git git://git.kernel.org/pub/scm/linux/kernel/git/linusw/linux-pinctrl.git
F:  drivers/pinctrl/
F:  include/linux/pinctrl/
F:  Documentation/driver-api/pin-control.rst
```

### ملفات الـ subsystem

| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/core.c` | قلب الـ subsystem — تسجيل controllers، إدارة states |
| `drivers/pinctrl/devicetree.c` | **الملف الحالي** — ترجمة DT لـ mappings |
| `drivers/pinctrl/pinmux.c` | إدارة الـ mux (توجيه الـ pin لوظيفة معينة) |
| `drivers/pinctrl/pinconf.c` | إدارة الـ config (pull-up، drive strength، إلخ) |
| `drivers/pinctrl/pinconf-generic.c` | generic config parser للـ DT properties |
| `drivers/pinctrl/pinctrl-generic.c` | generic groups/functions implementation |
| `include/linux/pinctrl/pinctrl.h` | الـ public API + `struct pinctrl_ops` |
| `include/linux/pinctrl/machine.h` | `struct pinctrl_map` و map types |
| `include/linux/pinctrl/consumer.h` | API للـ consumer devices (`pinctrl_get`, `pinctrl_select_state`) |

---

## Phase 2: شرح الـ pinctrl Device Tree Framework

### المشكلة

الـ pinctrl core (في `core.c`) يشتغل على **mapping tables** — مصفوفات من `struct pinctrl_map` بتقول: "الـ device X، في الـ state Y، يستخدم الـ pin group Z بالـ function W". المشكلة إن في الأنظمة المبنية على DT (ARM، RISC-V، معظم embedded)، الـ mappings دي مش موجودة statically في C code — هي موجودة في الـ DT blob اللي بييجي من الـ bootloader وقت الـ runtime.

في غياب `devicetree.c`، كل device كان لازم يسجل الـ maps يدوياً عبر `pinctrl_register_mappings()` — مستحيل على أنظمة DT-based لأن الـ board info متاحة فقط في الـ DT.

### الحل

`devicetree.c` بيعمل **lazy parsing**: لما أي device يطلب الـ pinctrl handle بتاعه (عبر `pinctrl_get()` في core)، الـ core بيكلم `pinctrl_dt_to_map()` عشان يحول الـ DT properties للـ device ده لـ mapping table entries ويسجلها dynamically.

### Architecture Diagram

```
Device Probe
    │
    ▼
pinctrl_get(dev)         ← core.c
    │
    ├─► dev->of_node موجود؟
    │       │
    │       YES
    │       ▼
    │   pinctrl_dt_to_map(p, pctldev)        ← devicetree.c
    │       │
    │       │  loop: pinctrl-0, pinctrl-1, ...
    │       ▼
    │   of_find_property(np, "pinctrl-N")
    │       │
    │       │  لكل phandle في الـ property
    │       ▼
    │   dt_to_map_one_config()
    │       │
    │       │  1. امشي على الـ parent nodes
    │       │     لحد ما تلاقي pin controller
    │       │     (get_pinctrl_dev_from_of_node)
    │       │
    │       │  2. كلّم driver hook:
    │       ▼
    │   ops->dt_node_to_map(pctldev, np_config,
    │                        &map, &num_maps)
    │       │           ▲
    │       │           │ كل driver بيعمله بنفسه
    │       │           │ (BCM, Qualcomm, Rockchip, ...)
    │       │
    │       ▼
    │   dt_remember_or_free_map()
    │       │
    │       ├─► كمّل الـ map entries (dev_name, state name, ctrl_dev_name)
    │       ├─► حط الـ map في p->dt_maps list
    │       └─► pinctrl_register_mappings(map, num_maps)
    │                   │
    │                   ▼
    │           pinctrl_maps list (global)
    │
    ▼
pinctrl_lookup_state()   ← core.c يكمل الشغل
```

### Core Abstractions

#### 1. `struct pinctrl_dt_map` (private to devicetree.c)

```c
struct pinctrl_dt_map {
    struct list_head node;      /* linked into p->dt_maps */
    struct pinctrl_dev *pctldev; /* من أيه controller الـ maps دي؟ */
    struct pinctrl_map *map;    /* الـ entries نفسها */
    unsigned int num_maps;
};
```

**لماذا موجودة؟** عشان نعرف نعمل cleanup صح. لما device يتـ deregister، `pinctrl_dt_free_maps()` بتمشي على الـ list دي وتفرج عن كل حاجة — وبتكلم `ops->dt_free_map` عشان الـ driver يفرج عن memory خصته.

#### 2. `pinctrl_dt_to_map()` — نقطة الدخول الرئيسية

```c
int pinctrl_dt_to_map(struct pinctrl *p, struct pinctrl_dev *pctldev)
```

- بتلف على `pinctrl-0`, `pinctrl-1`, ... حتى ما تلاقيش property
- لكل state، بتقرأ اسمه من `pinctrl-names` أو بتستخدم الرقم مباشرة
- لكل phandle في الـ state، بتكلم `dt_to_map_one_config()`
- لو الـ state فاضي (size=0)، بتعمل **dummy state** عشان الـ core يعرف الـ state موجود

#### 3. `dt_to_map_one_config()` — اللب

```c
static int dt_to_map_one_config(struct pinctrl *p,
                                struct pinctrl_dev *hog_pctldev,
                                const char *statename,
                                struct device_node *np_config)
```

**خطوتين رئيسيتين:**

**أولاً: إيجاد الـ pin controller** — بيطلع للأعلى في شجرة الـ DT:

```c
np_pctldev = of_node_get(np_config);
for (;;) {
    np_pctldev = of_get_next_parent(np_pctldev);
    pctldev = get_pinctrl_dev_from_of_node(np_pctldev);
    if (pctldev) break;
}
```

الـ config node ممكن يكون على أي مستوى داخل الـ pin controller node — الـ loop دي بتضمن إننا نوصل للـ controller صح بغض النظر عن العمق.

**ثانياً: تفويض الـ parsing للـ driver:**

```c
ret = ops->dt_node_to_map(pctldev, np_config, &map, &num_maps);
```

الـ `devicetree.c` **مش بيعرف** إزاي يفسر `brcm,pins` أو `qcom,pin-function` — ده شغل الـ driver. الـ framework بس بيوفر الـ scaffolding.

#### 4. Hog Detection

**Pin hogging** = الـ pin controller نفسه بيحجز pins لنفسه (مش لـ consumer device تانية). الـ code بيتعامل معاه specially:

```c
/* If we're creating a hog we can use the passed pctldev */
if (hog_pctldev && (np_pctldev == p->dev->of_node)) {
    pctldev = hog_pctldev;
    break;
}
/* Do not defer probing of hogs (circular loop) */
if (np_pctldev == p->dev->of_node) {
    of_node_put(np_pctldev);
    return 1;  /* caller catches this */
}
```

لو مش hog ولقى إن الـ config بتبص على نفس الـ controller node اللي لسه ما اتـ registerش، بيرجع `1` (مش error) عشان الـ caller يعرف يتخطى من غير ما يـ defer.

#### 5. `pinctrl-use-default` Property

```c
allow_default = of_property_read_bool(np_pctldev, "pinctrl-use-default");
```

لو الـ pin controller node مش موجود بعد (لسه ما اتـ probeش) وفيه `pinctrl-use-default`، بدل ما الـ code يـ return `-EPROBE_DEFER`، بيكمل. ده بيتحاشى defer loops في بعض الـ embedded platforms.

#### 6. `pinctrl_count_index_with_args` / `pinctrl_parse_index_with_args`

دول لـ **pinctrl arrays** — نوع تاني من الـ DT bindings بيستخدموه بعض الـ drivers (مش الـ standard `pinctrl-N` pattern):

```c
/* Format: <index arg1 arg2 ...> * N entries */
/* #pinctrl-cells يحدد عدد الـ args لكل entry */

int pinctrl_count_index_with_args(const struct device_node *np,
                                  const char *list_name);

int pinctrl_parse_index_with_args(const struct device_node *np,
                                  const char *list_name, int index,
                                  struct of_phandle_args *out_args);
```

الـ `#pinctrl-cells` property بتتقرأ من parent أو grandparent، وبتحدد كام u32 بيجي بعد الـ index في كل entry.

### ما بيعمله الـ Framework مقابل ما بيتركه للـ Driver

| المسؤولية | `devicetree.c` | Driver (`dt_node_to_map`) |
|-----------|:--------------:|:------------------------:|
| قراءة `pinctrl-N` properties | نعم | لا |
| إيجاد الـ pin controller من الـ DT tree | نعم | لا |
| تعبئة `dev_name`، `name`، `ctrl_dev_name` | نعم | لا |
| تسجيل الـ maps في الـ global list | نعم | لا |
| Cleanup عند الـ device removal | نعم | جزئياً (via `dt_free_map` hook) |
| تفسير vendor-specific DT properties | لا | نعم |
| تخصيص `struct pinctrl_map[]` | لا | نعم |
| تحديد الـ map type (MUX/CONFIG/DUMMY) | لا | نعم |

### الـ `CONFIG_OF` Guard في `devicetree.h`

```c
#ifdef CONFIG_OF
/* real implementations */
void pinctrl_dt_free_maps(struct pinctrl *p);
int pinctrl_dt_to_map(struct pinctrl *p, struct pinctrl_dev *pctldev);
/* ... */
#else
/* stub implementations returning 0 or -ENODEV */
static inline int pinctrl_dt_to_map(...) { return 0; }
/* ... */
#endif
```

ده بيخلي الـ `core.c` يكلم الـ functions دي بدون `#ifdef` في كل مكان. على platforms من غير DT (زي x86)، كل حاجة بتتحول لـ no-ops في compile time.

### Data Flow Summary

```
DT blob (in memory)
    │
    │  of_find_property("pinctrl-0")
    ▼
phandle list ──► of_find_node_by_phandle()
                        │
                        ▼
                  np_config (DT node)
                        │
                        │  dt_to_map_one_config()
                        │  walk parents ──► pctldev
                        │
                        ▼
                  ops->dt_node_to_map()    [driver code]
                        │
                        ▼
                  struct pinctrl_map[]
                        │
                  dt_remember_or_free_map()
                        │
                  ┌─────┴──────────────────────┐
                  │                            │
                  ▼                            ▼
           p->dt_maps list         pinctrl_maps (global)
           (for cleanup)           (for lookup by core)
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### الـ Enums والـ Flags المهمة

#### `pinctrl_map_type` — أنواع الـ mapping entries

| القيمة | المعنى |
|--------|--------|
| `PIN_MAP_TYPE_INVALID` | entry فاسد، مش بيتستخدم |
| `PIN_MAP_TYPE_DUMMY_STATE` | state موجود في DT بس مفيش config nodes جواه |
| `PIN_MAP_TYPE_MUX_GROUP` | إعداد pinmux لـ group كاملة |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | إعداد config (pull/drive) لـ group |
| `PIN_MAP_TYPE_CONFIGS_PIN` | إعداد config لـ pin منفرد |

#### `PINFUNCTION_FLAG_GPIO`

| الـ Flag | القيمة | الاستخدام |
|----------|--------|-----------|
| `PINFUNCTION_FLAG_GPIO` | `BIT(0)` | بيميز الـ function إنها GPIO function، بيتحدد في `pinfunction.flags` |

---

### الـ Structs المهمة

#### 1. `struct pinctrl_dt_map` — الـ chunk اللي بيتعمل من DT

```c
struct pinctrl_dt_map {
    struct list_head    node;      /* ربط في قايمة pinctrl->dt_maps */
    struct pinctrl_dev *pctldev;   /* الـ controller اللي عمل الـ alloc */
    struct pinctrl_map *map;       /* مصفوفة الـ mapping entries */
    unsigned int        num_maps;  /* عدد الـ entries في المصفوفة */
};
```

**الغرض:** بيحفظ chunk من الـ mapping table اللي تم parse-ها من DT لـ device معين. لما device بيتحذف، بيتمشي على القايمة ده ويحرر كل حاجة.

**الـ Connections:**
- متربط في `struct pinctrl` عن طريق `pinctrl->dt_maps` (list_head)
- بيشاور على `struct pinctrl_dev` اللي مسؤول عن تحرير الـ map
- الـ `map` هي مصفوفة من `struct pinctrl_map` اللي بيتسجلوا في global registry

---

#### 2. `struct pinctrl` — الـ per-device state holder

```c
struct pinctrl {
    struct list_head      node;     /* في global pinctrl list */
    struct device        *dev;      /* الـ device صاحب الـ handle */
    struct list_head      states;   /* قايمة الـ states (default, sleep, ...) */
    struct pinctrl_state *state;    /* الـ state الحالي المفعّل */
    struct list_head      dt_maps;  /* قايمة pinctrl_dt_map chunks */
    struct kref           users;    /* reference count */
};
```

**الغرض:** هو الـ handle اللي بيمثل device واحد في الـ pinctrl subsystem. بيشيل كل states الـ device وكل الـ DT maps اللي اتحولت.

---

#### 3. `struct pinctrl_dev` — الـ pin controller device

```c
struct pinctrl_dev {
    struct list_head         node;           /* في global controllers list */
    const struct pinctrl_desc *desc;         /* وصف الـ controller */
    struct radix_tree_root   pin_desc_tree;  /* شجرة الـ pin descriptors */
    struct list_head         gpio_ranges;    /* GPIO ranges */
    struct device           *dev;            /* الـ device الفعلي */
    struct module           *owner;
    void                    *driver_data;
    struct pinctrl          *p;              /* للـ hogging */
    struct pinctrl_state    *hog_default;
    struct pinctrl_state    *hog_sleep;
    struct mutex             mutex;          /* per-controller lock */
};
```

**الغرض:** يمثل pin controller واحد (زي Tegra pinctrl). بيشيل كل المعلومات عن الـ pins والـ groups والـ functions اللي بيديرها.

---

#### 4. `struct pinctrl_ops` — الـ vtable بتاع الـ controller driver

```c
struct pinctrl_ops {
    int         (*get_groups_count)(struct pinctrl_dev *);
    const char *(*get_group_name)(struct pinctrl_dev *, unsigned int);
    int         (*get_group_pins)(struct pinctrl_dev *, unsigned int,
                                  const unsigned int **, unsigned int *);
    void        (*pin_dbg_show)(struct pinctrl_dev *, struct seq_file *,
                                unsigned int);
    /* DT-specific callbacks */
    int         (*dt_node_to_map)(struct pinctrl_dev *,
                                  struct device_node *np_config,
                                  struct pinctrl_map **, unsigned int *);
    void        (*dt_free_map)(struct pinctrl_dev *,
                               struct pinctrl_map *, unsigned int);
};
```

**الغرض:** الـ virtual function table اللي كل driver بيوفرها. `dt_node_to_map` هو الـ callback الأساسي في الـ DT integration — بياخد DT node ويحوله لـ mapping entries. `dt_free_map` مسؤول عن تحرير اللي اتعمل.

---

#### 5. `struct pinctrl_map` — entry واحدة في الـ mapping table

بيتعرف في `include/linux/pinctrl/machine.h`. بيشيل:
- `dev_name`: اسم الـ device صاحب الـ state
- `name`: اسم الـ state (مثلاً "default")
- `ctrl_dev_name`: اسم الـ pin controller
- `type`: نوع الـ entry (`PIN_MAP_TYPE_*`)
- `data`: union بيشيل إما `mux` أو `configs`

---

#### 6. `struct of_phandle_args` — نتيجة parse الـ phandle list

```c
struct of_phandle_args {
    struct device_node *np;        /* الـ node */
    int                 args_count;/* عدد الـ arguments */
    uint32_t            args[MAX_PHANDLE_ARGS]; /* القيم */
};
```

**الغرض:** بتستخدمه `pinctrl_parse_index_with_args` عشان ترجع بيانات entry معينة من pinctrl array في DT.

---

### ASCII Diagram — علاقة الـ Structs

```
  Device (consumer)
       |
       | of_node  →  DT Node
       |              |
       |         pinctrl-0 = <&cfg_node>
       |         pinctrl-names = "default"
       v
  struct pinctrl  (per consumer device)
  ┌─────────────────────────────┐
  │ dev ──→ struct device       │
  │ states → [pinctrl_state]    │
  │ state  → (current active)   │
  │ dt_maps → ─────────────────────────────────┐
  │ users (kref)                │              │
  └─────────────────────────────┘              │
                                               ▼
                                  struct pinctrl_dt_map
                                  ┌──────────────────────┐
                                  │ node (list_head)     │
                                  │ pctldev ─────────────────┐
                                  │ map ─→ [pinctrl_map]  │  │
                                  │ num_maps              │  │
                                  └──────────────────────┘  │
                                                             ▼
                                              struct pinctrl_dev
                                  ┌──────────────────────────────┐
                                  │ desc ─→ pinctrl_desc         │
                                  │         └─→ pctlops          │
                                  │              ├ dt_node_to_map│
                                  │              └ dt_free_map   │
                                  │ pin_desc_tree (radix)        │
                                  │ mutex                        │
                                  └──────────────────────────────┘

  Global Registry:
  pinctrl_maps (list) ←── pinctrl_register_mappings()
  ┌────────────────┐
  │ pinctrl_maps   │
  │   ├ maps[]     │  ←── entries created by dt_node_to_map()
  │   └ num_maps   │
  └────────────────┘
```

---

### Lifecycle Diagram — من DT لـ active state

```
pinctrl_get(dev)
    │
    ├─→ pinctrl_dt_to_map(p, NULL)
    │       │
    │       ├─ for each pinctrl-N property:
    │       │     ├─ get statename from pinctrl-names
    │       │     ├─ for each phandle in list:
    │       │     │     └─→ dt_to_map_one_config()
    │       │     │              ├─ walk parents → find pctldev
    │       │     │              ├─ ops->dt_node_to_map()   [driver callback]
    │       │     │              └─→ dt_remember_or_free_map()
    │       │     │                      ├─ fill dev_name, statename
    │       │     │                      ├─ kzalloc(pinctrl_dt_map)
    │       │     │                      ├─ list_add_tail → p->dt_maps
    │       │     │                      └─ pinctrl_register_mappings()
    │       │     └─ (if empty list) → dt_remember_dummy_state()
    │       └─ return 0
    │
    └─→ create_pinctrl_state() for each registered mapping
            → p->states list populated

pinctrl_select_state(p, "default")
    │
    └─→ apply settings from pinctrl_state->settings
            (mux + configs written to hardware)

device_del() / pinctrl_put()
    │
    └─→ pinctrl_dt_free_maps(p)
            ├─ list_for_each_entry_safe on p->dt_maps
            ├─ pinctrl_unregister_mappings()
            ├─ dt_free_map() → ops->dt_free_map() or kfree()
            ├─ kfree(dt_map)
            └─ of_node_put(p->dev->of_node)
```

---

### Call Flow — `pinctrl_dt_to_map` بالتفصيل

```
pinctrl_dt_to_map(p, pctldev)
│
├── of_node_get(np)                    // pin ref on DT node
│
├── for state = 0, 1, 2, ...
│     │
│     ├── kasprintf("pinctrl-%d", state)
│     ├── of_find_property(np, propname)
│     │     └── if not found && state==0 → -ENODEV
│     │         if not found && state>0  → break (done)
│     │
│     ├── of_property_read_string_index("pinctrl-names", state)
│     │     └── if not found → use numeric string from prop->name
│     │
│     ├── for each phandle in prop->value:
│     │     ├── of_find_node_by_phandle(phandle) → np_config
│     │     ├── dt_to_map_one_config(p, pctldev, statename, np_config)
│     │     │     ├── walk parent chain → find pctldev
│     │     │     │     ├── check "pinctrl-use-default" property
│     │     │     │     └── get_pinctrl_dev_from_of_node()
│     │     │     ├── ops->dt_node_to_map() → map[], num_maps
│     │     │     └── dt_remember_or_free_map()
│     │     │           ├── fill map[i].dev_name, .name, .ctrl_dev_name
│     │     │           ├── kzalloc(pinctrl_dt_map)
│     │     │           ├── list_add_tail → p->dt_maps
│     │     │           └── pinctrl_register_mappings(map, num_maps)
│     │     └── of_node_put(np_config)
│     │
│     └── if size==0 → dt_remember_dummy_state(p, statename)
│
└── return 0  (or goto err → pinctrl_dt_free_maps)
```

---

### Locking Strategy

| الموقف | الـ Lock المستخدم |
|--------|------------------|
| تسجيل الـ maps في global registry | `pinctrl_maps_mutex` (في `pinctrl_register_mappings`) |
| عمليات الـ controller نفسه | `pctldev->mutex` |
| parse الـ DT وبناء الـ dt_maps | لا يوجد lock — بيتعمل في probe context قبل ما الـ device يبقى accessible |
| تحرير الـ dt_maps | بيتعمل في device teardown — mutex مش محتاج لأن الـ device مش accessible |
| الـ `of_node` reference | `of_node_get/put` — atomic refcount |

**ملاحظة:** الـ `pinctrl_dt_map` list (موجودة في `pinctrl->dt_maps`) مش محمية بـ explicit mutex لأنها بتتبنى في وقت الـ probe فقط وبتتحرر في وقت الـ remove فقط — مفيش concurrent access.

---

## Phase 4: شرح الـ Functions

### جدول API الكامل

| الـ Function | النطاق | الـ Export |
|-------------|--------|-----------|
| `pinctrl_dt_to_map` | internal | لا |
| `pinctrl_dt_free_maps` | internal | لا |
| `of_pinctrl_get` | public | `EXPORT_SYMBOL_GPL` |
| `pinctrl_count_index_with_args` | public | `EXPORT_SYMBOL_GPL` |
| `pinctrl_parse_index_with_args` | public | `EXPORT_SYMBOL_GPL` |
| `dt_to_map_one_config` | static | لا |
| `dt_remember_or_free_map` | static | لا |
| `dt_remember_dummy_state` | static | لا |
| `dt_free_map` | static | لا |
| `pinctrl_find_cells_size` | static | لا |
| `pinctrl_get_list_and_count` | static | لا |
| `pinctrl_copy_args` | static | لا |

---

### Group 1: الـ Core DT Parsing Functions

---

#### `pinctrl_dt_to_map`

```c
int pinctrl_dt_to_map(struct pinctrl *p, struct pinctrl_dev *pctldev)
```

الـ function الرئيسية اللي بتحول بيانات الـ device tree لـ mapping table entries. بتمشي على كل `pinctrl-N` property في الـ DT node بتاعة الـ device، وبتحوّل كل phandle فيها لـ map entries عن طريق الـ driver callback. لو `pctldev` مش NULL معناه إننا بنعمل "hog" للـ controller نفسه.

**Parameters:**

| الـ Parameter | النوع | الوصف |
|--------------|-------|-------|
| `p` | `struct pinctrl *` | الـ handle بتاع الـ consumer device |
| `pctldev` | `struct pinctrl_dev *` | لو الـ device هو الـ controller نفسه (hog)، غير كده NULL |

**Return:** `0` نجاح، `-ENODEV` لو مفيش `pinctrl-0`، `-ENOMEM` لو الـ allocation فشل، `-EINVAL` لو phandle غلط، `-EPROBE_DEFER` لو الـ controller لسه ما اتجهزش (مع CONFIG_MODULES).

**Locking:** مش بياخد lock. بيتعمل في probe context. `pinctrl_register_mappings` اللي بيتكلمها داخلياً هي اللي بتاخد `pinctrl_maps_mutex`.

**Error handling:** عند أي error بيعمل `goto err` اللي بيكلم `pinctrl_dt_free_maps` عشان ينظف كل اللي اتعمل.

**Pseudocode:**
```
function pinctrl_dt_to_map(p, pctldev):
    np = p->dev->of_node
    if np is NULL:
        return 0  // not a DT device

    of_node_get(np)  // take reference

    for state = 0 to infinity:
        propname = "pinctrl-{state}"
        prop = find_property(np, propname)
        if not found:
            if state == 0: return -ENODEV
            else: break

        statename = pinctrl-names[state] OR numeric string

        for each phandle in prop:
            np_config = find_node_by_phandle(phandle)
            ret = dt_to_map_one_config(p, pctldev, statename, np_config)
            if ret == 1: continue  // hog skip
            if ret < 0: goto err

        if no phandles in prop:
            dt_remember_dummy_state(p, statename)

    return 0
err:
    pinctrl_dt_free_maps(p)
    return ret
```

---

#### `pinctrl_dt_free_maps`

```c
void pinctrl_dt_free_maps(struct pinctrl *p)
```

بتحرر كل الـ DT-derived mapping entries اللي اتعملت لـ device معين. بتمشي على `p->dt_maps` وبتلغي تسجيل كل chunk من الـ global registry وبعدين بتحررها.

**Parameters:**

| الـ Parameter | النوع | الوصف |
|--------------|-------|-------|
| `p` | `struct pinctrl *` | الـ handle اللي هيتنظف |

**Return:** void.

**Locking:** `pinctrl_unregister_mappings` بتاخد `pinctrl_maps_mutex` داخلياً. الـ `of_node_put` في الآخر بيحرر الـ reference اللي اتعملت في `pinctrl_dt_to_map`.

**Caller context:** بيتعمل call من `pinctrl_put` أو من error path في `pinctrl_dt_to_map` نفسها.

**ملاحظة مهمة:** بيستخدم `list_for_each_entry_safe` لأنه بيعمل `list_del` جوا الـ loop.

```c
// نمط الاستخدام الآمن
list_for_each_entry_safe(dt_map, n1, &p->dt_maps, node) {
    pinctrl_unregister_mappings(dt_map->map);  // remove from global list
    list_del(&dt_map->node);                    // detach from p->dt_maps
    dt_free_map(dt_map->pctldev, dt_map->map, dt_map->num_maps);
    kfree(dt_map);
}
of_node_put(p->dev->of_node);  // release DT node ref
```

---

### Group 2: الـ Internal Helper Functions

---

#### `dt_to_map_one_config`

```c
static int dt_to_map_one_config(struct pinctrl *p,
                                struct pinctrl_dev *hog_pctldev,
                                const char *statename,
                                struct device_node *np_config)
```

بتاخد DT config node واحد، بتلاقي الـ pin controller اللي بيملكه بالمشي على الـ parent chain، وبتكلم `dt_node_to_map` callback بتاعة الـ driver. لو الـ config node تبع الـ controller نفسه (hog) وملقتش `pctldev` جاهز بترجع 1 عشان الـ caller يعمل skip.

**Parameters:**

| الـ Parameter | النوع | الوصف |
|--------------|-------|-------|
| `p` | `struct pinctrl *` | الـ consumer handle |
| `hog_pctldev` | `struct pinctrl_dev *` | لو bنعمل hog، ده هو الـ pctldev، غير كده NULL |
| `statename` | `const char *` | اسم الـ state (مثلاً "default") |
| `np_config` | `struct device_node *` | الـ DT node اللي فيه الـ pin config |

**Return:** `0` نجاح، `1` skip (hog circular)، `< 0` error.

**Logic خاص:** بيمشي على الـ parent nodes عشان يلاقي الـ pin controller. في كل خطوة بيتحقق من property `pinctrl-use-default` — لو موجودة، مش هيعمل probe defer حتى لو ملقاش controller. لو `np_pctldev` وصل لـ root node من غير ما يلاقي controller، بيرجع `-ENODEV` أو `-EPROBE_DEFER`.

```
np_config → parent → parent → ... → pinctrl_dev node
                ↑
         check each level for "pinctrl-use-default"
         and get_pinctrl_dev_from_of_node()
```

---

#### `dt_remember_or_free_map`

```c
static int dt_remember_or_free_map(struct pinctrl *p, const char *statename,
                                   struct pinctrl_dev *pctldev,
                                   struct pinctrl_map *map,
                                   unsigned int num_maps)
```

بتاخد الـ map array اللي عملها الـ driver، بتملا الـ common fields فيها (dev_name, statename, ctrl_dev_name)، بتعمل `pinctrl_dt_map` جديدة وبتضيفها لـ `p->dt_maps`، وبعدين بتسجلها في الـ global registry. لو حاجة فشلت بتحرر كل حاجة.

**Parameters:**

| الـ Parameter | النوع | الوصف |
|--------------|-------|-------|
| `p` | `struct pinctrl *` | الـ consumer |
| `statename` | `const char *` | اسم الـ state |
| `pctldev` | `struct pinctrl_dev *` | الـ controller، أو NULL للـ DUMMY_STATE |
| `map` | `struct pinctrl_map *` | المصفوفة اللي عملها الـ driver |
| `num_maps` | `unsigned int` | عدد الـ entries |

**Return:** `0` نجاح، `-ENOMEM` فشل.

**ملاحظة:** بيستخدم `kstrdup_const` مش `kstrdup` عشان `dev_name()` ممكن يرجع pointer لـ const string في الـ kernel، و`kstrdup_const` بتتجنب الـ copy لو كانت كده.

**Pseudocode:**
```
function dt_remember_or_free_map(p, statename, pctldev, map, num_maps):
    for i in 0..num_maps:
        devname = kstrdup_const(dev_name(p->dev))
        if !devname: goto err_free_map
        map[i].dev_name = devname
        map[i].name = statename
        if pctldev: map[i].ctrl_dev_name = dev_name(pctldev->dev)

    dt_map = kzalloc(sizeof(*dt_map))
    if !dt_map: goto err_free_map

    dt_map->pctldev = pctldev
    dt_map->map = map
    dt_map->num_maps = num_maps
    list_add_tail(&dt_map->node, &p->dt_maps)

    return pinctrl_register_mappings(map, num_maps)

err_free_map:
    dt_free_map(pctldev, map, num_maps)
    return -ENOMEM
```

---

#### `dt_free_map`

```c
static void dt_free_map(struct pinctrl_dev *pctldev,
                        struct pinctrl_map *map, unsigned int num_maps)
```

بتحرر الـ map array. أول حاجة بتحرر `dev_name` في كل entry عن طريق `kfree_const` (لأنها اتعملت بـ `kstrdup_const`). بعدين لو فيه `pctldev` بتكلم `ops->dt_free_map` عشان الـ driver يحرر الـ array نفسها (لأن الـ driver هو اللي عملها). لو مفيش `pctldev` (حالة DUMMY_STATE) بتعمل `kfree` مباشرة.

**Parameters:**

| الـ Parameter | النوع | الوصف |
|--------------|-------|-------|
| `pctldev` | `struct pinctrl_dev *` | الـ controller، أو NULL للـ DUMMY_STATE |
| `map` | `struct pinctrl_map *` | المصفوفة المراد تحريرها |
| `num_maps` | `unsigned int` | عدد الـ entries |

**Return:** void.

**تفاصيل مهمة:** الـ `dev_name` field في كل entry اتعملت بـ `kstrdup_const` من الـ core، فالـ core هي اللي بتحررها. أما الـ map array نفسها وأي fields تانية جوا الـ entries، دي مسؤولية الـ driver.

---

#### `dt_remember_dummy_state`

```c
static int dt_remember_dummy_state(struct pinctrl *p, const char *statename)
```

لما `pinctrl-N` property موجودة بس فاضية (مفيش phandles)، بتعمل map entry واحدة من نوع `PIN_MAP_TYPE_DUMMY_STATE`. الهدف هو إن الـ state يتعرف ويتسجل حتى لو مفيش hardware configuration تبعها.

**Parameters:**

| الـ Parameter | النوع | الوصف |
|--------------|-------|-------|
| `p` | `struct pinctrl *` | الـ consumer |
| `statename` | `const char *` | اسم الـ state الفاضي |

**Return:** `0` نجاح، `-ENOMEM` فشل الـ allocation.

**Use case:** بيتستخدم لما device ليه state اسمه "sleep" مثلاً بس مش محتاج يعمل أي hardware changes فيه — بس محتاج الـ state يتعرف عليه الـ framework.

---

### Group 3: الـ Index/Args Parsing Functions

---

#### `pinctrl_count_index_with_args`

```c
int pinctrl_count_index_with_args(const struct device_node *np,
                                  const char *list_name)
```

بتحسب عدد العناصر في pinctrl property list اللي كل element فيها عبارة عن index + N cells من arguments. شايلة `#pinctrl-cells` من parent أو grandparent عشان تعرف حجم كل element.

**Parameters:**

| الـ Parameter | النوع | الوصف |
|--------------|-------|-------|
| `np` | `const struct device_node *` | الـ node اللي فيه الـ property |
| `list_name` | `const char *` | اسم الـ property |

**Return:** عدد العناصر (>= 0) نجاح، `-ENOENT` لو الـ property أو `#pinctrl-cells` مش موجودين.

**Export:** `EXPORT_SYMBOL_GPL` — بيتستخدم من drivers خارج الـ pinctrl core.

**مثال DT:**
```
// في الـ DT node
some-prop = <0x1 0x2 0x3  0x4 0x5 0x6>;
// #pinctrl-cells = <2> في parent
// يعني عندنا 2 عناصر: [0x1, 0x2, 0x3] و [0x4, 0x5, 0x6]
// index(1) + 2 cells = 3 u32 per element
// total = 6 / 3 = 2 elements
```

---

#### `pinctrl_parse_index_with_args`

```c
int pinctrl_parse_index_with_args(const struct device_node *np,
                                  const char *list_name, int index,
                                  struct of_phandle_args *out_args)
```

بتاخد property list وبتجيب element معين منها بالـ index وبتحطه في `out_args`. العنصر بيتكون من (index_within_controller + N cells).

**Parameters:**

| الـ Parameter | النوع | الوصف |
|--------------|-------|-------|
| `np` | `const struct device_node *` | الـ node صاحب الـ property |
| `list_name` | `const char *` | اسم الـ property |
| `index` | `int` | رقم العنصر المطلوب (zero-based) |
| `out_args` | `struct of_phandle_args *` | الـ output |

**Return:** `0` نجاح، `-ENOENT` لو الـ property أو `#pinctrl-cells` مش موجودين، `-EINVAL` لو الـ index أكبر من عدد العناصر.

**Export:** `EXPORT_SYMBOL_GPL`.

**ملاحظة:** لو `nr_cells` == 0 بترجع 0 مباشرة (خطأ في الـ bindings لكن مش بتعاقب).

---

#### `pinctrl_find_cells_size` (static)

```c
static int pinctrl_find_cells_size(const struct device_node *np)
```

بتدور على `#pinctrl-cells` property في `np->parent` أو `np->parent->parent`. بتحاول المستويين عشان تعمل flexible binding — الـ `#pinctrl-cells` ممكن تكون في الـ pin controller node (parent) أو في container node (grandparent).

**Return:** قيمة `#pinctrl-cells` (>= 0)، أو `-ENOENT` لو ملقتهاش في أي مستوى.

---

#### `pinctrl_get_list_and_count` (static)

```c
static int pinctrl_get_list_and_count(const struct device_node *np,
                                      const char *list_name,
                                      const __be32 **list,
                                      int *cells_size,
                                      int *nr_elements)
```

helper بتجيب الـ raw property data وبتحسب عدد العناصر فيها. بتملا `*list` بـ pointer للبيانات، و`*cells_size` بعدد الـ cells لكل element، و`*nr_elements` بالعدد الكلي للعناصر.

**Parameters:**

| الـ Parameter | النوع | الوصف |
|--------------|-------|-------|
| `np` | `const struct device_node *` | الـ node |
| `list_name` | `const char *` | اسم الـ property |
| `list` | `const __be32 **` | output: pointer للبيانات الـ raw |
| `cells_size` | `int *` | output: عدد الـ cells لكل element |
| `nr_elements` | `int *` | output: عدد العناصر |

**Return:** `0` نجاح، `-ENOENT` فشل.

**الحسابات:**
```
total_u32 = size_bytes / sizeof(u32)
// كل element = 1 (index) + cells_size (args)
nr_elements = total_u32 / (cells_size + 1)
```

---

#### `pinctrl_copy_args` (static)

```c
static int pinctrl_copy_args(const struct device_node *np,
                             const __be32 *list,
                             int index, int nr_cells, int nr_elem,
                             struct of_phandle_args *out_args)
```

بتنسخ بيانات element معين من الـ raw list لـ `out_args`. بتعمل bounds check، بتحسب offset الـ element في الـ list، وبتنسخ `nr_cells + 1` قيم (الـ +1 هو الـ index الأول في كل element).

**Parameters:**

| الـ Parameter | النوع | الوصف |
|--------------|-------|-------|
| `np` | `const struct device_node *` | الـ node (بيتحط في `out_args->np`) |
| `list` | `const __be32 *` | أول element في الـ list |
| `index` | `int` | العنصر المطلوب |
| `nr_cells` | `int` | عدد الـ cells لكل element |
| `nr_elem` | `int` | العدد الكلي للعناصر |
| `out_args` | `struct of_phandle_args *` | الـ output |

**Return:** `0` نجاح، `-EINVAL` لو `index >= nr_elem`.

**Address calculation:**
```c
// skip to the right entry
list += index * (nr_cells + 1);
// then copy nr_cells+1 values (index + args)
out_args->args_count = nr_cells + 1;
```

---

### Group 4: الـ Public Lookup Function

---

#### `of_pinctrl_get`

```c
struct pinctrl_dev *of_pinctrl_get(struct device_node *np)
```

wrapper بسيط حول `get_pinctrl_dev_from_of_node`. بتاخد DT node وبترجع الـ `pinctrl_dev` اللي بيديره. بتستخدمها الـ drivers اللي محتاجة تلاقي pin controller بالـ DT node بتاعه.

**Parameters:**

| الـ Parameter | النوع | الوصف |
|--------------|-------|-------|
| `np` | `struct device_node *` | الـ DT node بتاعة الـ pin controller |

**Return:** `struct pinctrl_dev *` لو لقيه، أو `NULL`.

**Export:** `EXPORT_SYMBOL_GPL` — بيتستخدم من GPIO drivers وغيرها اللي محتاجة تتكلم مع pin controller مباشرة.

**Caller context:** أي context، مش بتاخد lock، `get_pinctrl_dev_from_of_node` بتاخد `pinctrl_maps_mutex` داخلياً لو محتاج.
## Phase 5: دليل الـ Debugging الشامل

### debugfs Entries

الـ pinctrl subsystem بيعمل expose لـ runtime state عن طريق `debugfs` — mount بيكون على `/sys/kernel/debug/`.

```
/sys/kernel/debug/pinctrl/
├── pinctrl-devices          ← كل pin controllers مسجّلين
├── pinctrl-handles          ← كل pinctrl handles مفتوحة
├── pinctrl-maps             ← mapping table كاملة (ناتج pinctrl_register_mappings)
└── <controller-name>/
    ├── pins                 ← كل الـ pins وأسمائها
    ├── groups               ← الـ pin groups
    ├── pinmux-functions     ← الـ mux functions
    ├── pinmux-pins          ← مين بيستخدم مين
    └── pinconf-pins         ← bias/drive-strength لكل pin
```

**Shell commands جاهزة للنسخ:**

```bash
# شوف كل pin controllers موجودين
cat /sys/kernel/debug/pinctrl/pinctrl-devices

# شوف mapping table كاملة (ناتج dt_remember_or_free_map)
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# شوف كل الـ handles المفتوحة (من pinctrl_get)
cat /sys/kernel/debug/pinctrl/pinctrl-handles

# dump كل pins لـ controller معين (مثلاً RK3562)
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pins

# شوف mux assignments الحالية
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinmux-pins

# شوف pin configuration (pull-up/down, drive strength)
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinconf-pins

# شوف الـ groups المتاحة
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/groups

# شوف الـ functions المتاحة
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinmux-functions
```

---

### sysfs Entries

الـ pinctrl subsystem نفسه ما بيعمل expose كتير عبر sysfs المباشر، بس الـ GPIO subsystem المرتبط بيعمل:

```bash
# شوف GPIO lines وحالتها
cat /sys/kernel/debug/gpio

# device attributes للـ pin controller كـ platform device
ls /sys/bus/platform/devices/<pinctrl-dev>/

# شوف of_node المرتبط بالـ device
cat /sys/bus/platform/devices/<pinctrl-dev>/of_node

# تتبع اسم الـ device اللي بتشتغل عليه (ناتج dev_name() في الكود)
cat /sys/bus/platform/devices/<pinctrl-dev>/uevent
```

---

### ftrace Usage

**الـ function tracer** هو أقوى أداة لتتبع `pinctrl_dt_to_map` و `dt_to_map_one_config` وقت probe:

```bash
# Mount tracefs لو مش موجود
mount -t tracefs nodev /sys/kernel/tracing

# تتبع دالة pinctrl_dt_to_map بالكامل
cd /sys/kernel/tracing
echo 0 > tracing_on
echo function > current_tracer
echo pinctrl_dt_to_map > set_ftrace_filter
echo 1 > tracing_on

# شغّل الـ driver (rebind مثلاً)
echo <dev-name> > /sys/bus/platform/drivers/<driver>/bind

echo 0 > tracing_on
cat trace

# تتبع كل دوال pinctrl دفعة واحدة (function_graph أوضح)
echo function_graph > current_tracer
echo 'pinctrl_*' > set_ftrace_filter
echo 'dt_*' >> set_ftrace_filter
echo 1 > tracing_on

# تتبع مع stack trace عند كل call
echo 1 > options/func_stack_trace
```

**مثال ناتج function_graph لـ pinctrl_dt_to_map:**

```
 3) | pinctrl_dt_to_map() {
 3) | dt_to_map_one_config() {
 3) |   get_pinctrl_dev_from_of_node();
 3) |   ops->dt_node_to_map();       /* driver-specific */
 3) |   dt_remember_or_free_map() {
 3) |     pinctrl_register_mappings();
 3)   }
 3) }
```

---

### Dynamic Debug

**dynamic_debug** بيخلي `dev_dbg()` يشتغل بدون recompile — مفيد جداً لأن الكود في `pinctrl_dt_to_map` بيستخدم `dev_dbg` لما مفيش `of_node`:

```bash
# فعّل كل رسائل debug في ملف devicetree.c
echo 'file drivers/pinctrl/devicetree.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لكل ملفات pinctrl
echo 'file drivers/pinctrl/* +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع طباعة اسم الـ function والـ line number
echo 'file drivers/pinctrl/devicetree.c +pflm' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug entries الحالية لـ pinctrl
grep pinctrl /sys/kernel/debug/dynamic_debug/control

# ألغِ التفعيل
echo 'file drivers/pinctrl/devicetree.c -p' > /sys/kernel/debug/dynamic_debug/control
```

الرسالة اللي بتظهر عند تفعيل dynamic_debug:
```
[    2.345] pinctrl-dev: no of_node; not parsing pinctrl DT
```
هي من السطر:
```c
dev_dbg(p->dev, "no of_node; not parsing pinctrl DT\n");
```

---

### CONFIG_DEBUG_* Options

| Option | الأثر | متى تستخدمها |
|--------|-------|--------------|
| `CONFIG_DEBUG_FS` | يفعّل كل debugfs entries المذكورة | دايماً وقت debug |
| `CONFIG_PINCTRL_AMD=y` + `CONFIG_DEBUG_FS` | مثال: AMD pinctrl debugfs | SoC-specific |
| `CONFIG_OF` | يفعّل كل كود `devicetree.c` أصلاً | مطلوب للـ DT |
| `CONFIG_PROVE_LOCKING` | يكتشف lock ordering bugs في pinctrl_mutex | kernel dev |
| `CONFIG_KASAN` | يكتشف heap bugs في `kzalloc` / `kfree` calls | memory bugs |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `dev_dbg()` runtime | أهم خيار |
| `CONFIG_DEBUG_OBJECTS` | يتتبع lifecycle الـ objects | use-after-free |
| `CONFIG_KALLSYMS` | يعمل resolve لأسماء الدوال في stack traces | stack traces |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(DEBUG_FS|DYNAMIC_DEBUG|PINCTRL|OF)'
```

---

### Error Messages Table

| الرسالة | الدالة | السبب | الحل |
|---------|---------|-------|------|
| `prop %s index %i invalid phandle` | `pinctrl_dt_to_map` | phandle في `pinctrl-0` بيشير لـ node مش موجود | تحقق من DT بـ `dtc -I dtb -O dts` |
| `pctldev %s doesn't support DT` | `dt_to_map_one_config` | الـ driver ما بيعرّف `dt_node_to_map` في `pctlops` | الـ driver ناقص implement |
| `there is not valid maps for state %s` | `dt_to_map_one_config` | الـ pinctrl node فاضي أو فيه typo | راجع محتوى الـ pinctrl node في DT |
| `no of_node; not parsing pinctrl DT` | `pinctrl_dt_to_map` | الـ device اتعمل instantiate بدون DT | طبيعي لـ platform devices غير-DT |
| `-EPROBE_DEFER` (implicit) | `dt_to_map_one_config` | الـ pin controller لسه ما بـ probe | الـ driver هيتعمل retry تلقائياً |

---

### dump_stack Points

في إيه موقف تحط `dump_stack()` يدوياً؟

```c
/* في dt_to_map_one_config — لو عايز تشوف مين استدعى pinctrl_get */
if (!pctldev) {
    dev_err(p->dev, "DEBUG: pctldev not found\n");
    dump_stack();  /* <-- يطبع call chain كاملة */
}

/* في dt_remember_or_free_map — لتتبع من أين بييجي كل map */
pr_debug("registering %u maps for state '%s'\n", num_maps, statename);
dump_stack();
```

```bash
# شوف dump_stack output في dmesg
dmesg | grep -A 20 "Call Trace"

# أو بـ kernel log level مرتفع
dmesg -l debug | tail -100
```

---

### Register Dump Techniques

لـ pinctrl controllers اللي بتتعامل مع MMIO registers:

```bash
# قراءة register مباشرة عبر /dev/mem (بحذر!)
devmem2 0xFE740000 w   # RK3562 GPIO0 base مثلاً

# أو عبر iomem في debugfs
cat /proc/iomem | grep -i pinctrl
cat /proc/iomem | grep -i gpio

# بعض الـ SoCs بتعمل expose registers عبر debugfs خاص
# مثال STM32:
cat /sys/kernel/debug/pinctrl/soc:pin-controller/pinconf-pins
```

في الكود، الـ `pinctrl_dev` بيخزن `base` address — الـ driver نفسه مسؤول عن dump، مش `devicetree.c`.

---

### DT Debugging Techniques

```bash
# 1. compile DTB وافحص الـ output بعيون بشرية
dtc -I dts -O dts -o /tmp/out.dts /path/to/board.dts
grep -A5 "pinctrl-0" /tmp/out.dts

# 2. شوف الـ DTB اللي kernel بيستخدمه فعلاً
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -B2 -A10 "pinctrl"

# 3. تحقق من الـ properties المقروءة فعلاً
find /proc/device-tree -name "pinctrl*" | xargs ls -la

# 4. شوف الـ phandles
find /proc/device-tree -name "pinctrl-0" | xargs xxd

# 5. تحقق من of_node للـ device المشكلة
ls /sys/bus/platform/devices/<dev>/of_node/
cat /sys/bus/platform/devices/<dev>/of_node/pinctrl-names

# 6. تتبع probe order
dmesg | grep -E "(pinctrl|EPROBE_DEFER)" | head -30

# 7. تحقق من #pinctrl-cells (مهم لـ pinctrl_find_cells_size)
find /proc/device-tree -name "#pinctrl-cells" | xargs cat | xxd
```

---

## Phase 6: سيناريوهات من الحياة العملية

### Scenario 1: RK3562 — UART Debug Port مش بيشتغل

#### العنوان
RK3562 UART2 Debug Console لا تعطي output بعد bootloader

#### السياق
منتج: Industrial gateway بـ Rockchip RK3562 (Cortex-A53 quad-core). الـ bootloader شغال تمام، بس بعد ما kernel يبدأ يـ boot مفيش serial output.

#### المشكلة
```dts
/* Board DTS - خطأ شائع */
&uart2 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&uart2_xfer>;  /* phandle موجود؟ */
};
```

الـ kernel log (عبر JTAG):
```
[    1.234] pinctrl-rockchip fe770000.pinctrl: prop pinctrl-0 index 0 invalid phandle
[    1.235] uart2: probe failed: -EINVAL
```

#### التحليل
الكود في `pinctrl_dt_to_map`:
```c
np_config = of_find_node_by_phandle(phandle);
if (!np_config) {
    dev_err(p->dev, "prop %s index %i invalid phandle\n",
        prop->name, config);
    ret = -EINVAL;
    goto err;
}
```
الـ `phandle` بيشير لـ `uart2_xfer` اللي مش معرّف في الـ pinctrl node. السبب: نسيان include ملف `.dtsi` الصحيح لـ RK3562.

#### الحل
```bash
# افحص الـ compiled DTB
dtc -I dtb -O dts /boot/rk3562-board.dtb | grep -B5 "uart2_xfer"
# نتيجة: مش موجود!

# الحل في DTS
#include "rk3562-pinctrl.dtsi"   /* كان ناقص */
```

```dts
/* rk3562-pinctrl.dtsi - يجب include */
&pinctrl {
    uart2 {
        uart2_xfer: uart2-xfer {  /* الـ node الصح */
            rockchip,pins = <4 RK_PB0 2 &pcfg_pull_up>,
                            <4 RK_PB1 2 &pcfg_pull_none>;
        };
    };
};
```

#### الدرس المستفاد
دايماً افحص الـ compiled DTB فعلاً، مش الـ source. الـ phandle errors في `pinctrl_dt_to_map` بتعني غالباً missing include أو typo في اسم الـ label.

---

### Scenario 2: STM32MP1 — SPI Flash مش بيتعرف

#### العنوان
STM32MP1 QSPI Flash يظهر في probe لكن transfers بتفشل

#### السياق
منتج: Embedded HMI Panel بـ STMicroelectronics STM32MP157C. الـ QSPI flash مفروض يشتغل بـ 4-bit mode (Quad SPI) بس بيشتغل بـ 1-bit فقط.

#### المشكلة
```dts
&qspi {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&qspi_clk_pins_a &qspi_bk1_pins_a>;
    /* qspi_bk1_pins_a بتعرّف CLK, CS, IO0, IO1 بس */
    /* IO2, IO3 ناقصين! */
};
```

الـ device probe بنجاح لأن `dt_to_map_one_config` نجح، بس الـ QSPI controller ما عندوش IO2/IO3 مُعرَّفين فـ hardware.

#### التحليل
```bash
cat /sys/kernel/debug/pinctrl/soc:pin-controller@50002000/pinmux-pins | grep qspi
# Output:
# pin 84 (PG12): qspi (GPIO UNCLAIMED)   <-- IO0
# pin 85 (PG13): qspi (GPIO UNCLAIMED)   <-- IO1
# IO2, IO3 مش ظاهرين خالص
```

الـ `dt_to_map_one_config` بتـ parse بنجاح الـ DT nodes الموجودة، بس الـ hardware بيحتاج 4 data lines.

#### الحل
```dts
/* stm32mp157c-hmi.dts */
&qspi {
    pinctrl-names = "default", "sleep";
    /* أضف qspi_bk1_pins_b اللي بيشمل IO2 و IO3 */
    pinctrl-0 = <&qspi_clk_pins_a
                 &qspi_bk1_pins_a
                 &qspi_bk1_pins_b>;  /* IO2, IO3 */
};
```

```bash
# تحقق بعد الإصلاح
cat /sys/kernel/debug/pinctrl/soc:pin-controller@50002000/pinmux-pins | grep qspi
# الـ 4 lines يظهروا كلهم
```

#### الدرس المستفاد
نجاح `pinctrl_dt_to_map` لا يعني صحة الـ configuration. ممكن pins ناقصة ومفيش error — لازم تتحقق من `pinmux-pins` في debugfs يدوياً.

---

### Scenario 3: NXP i.MX8MQ — Ethernet لا يعمل بعد suspend/resume

#### العنوان
i.MX8MQ Ethernet (FEC) يفقد connectivity بعد system suspend

#### السياق
منتج: Automotive infotainment unit بـ NXP i.MX8MQ. الـ Ethernet شغال تمام، بس بعد أول suspend/resume cycle يوقف.

#### المشكلة
```dts
&fec1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_fec1>;
    /* "sleep" state ناقص! */
};
```

عند suspend، الـ pinctrl subsystem بيدور على `sleep` state — لما ما يلقيهاش، بيفضل في `default` state، والـ pins بتفضل active، وبسبب power sequencing للـ PHY ممكن يحصل corruption.

#### التحليل
```bash
# شوف الـ states المتاحة
cat /sys/kernel/debug/pinctrl/pinctrl-handles | grep fec
# Output:
# device: 30be0000.ethernet  current state: default
# state: default
# (sleep state مش موجودة)

# في dmesg وقت resume
dmesg | grep -i "fec\|pinctrl\|resume"
# [  45.123] fec 30be0000.ethernet: no pinctrl state for "sleep"
```

الـ `pinctrl_dt_to_map` بيبني الـ mapping table من الـ DT، ولأن `pinctrl-1` (sleep) مش موجود في DT، الـ state مش بتتبنى.

#### الحل
```dts
/* imx8mq-auto.dts */
&iomuxc {
    pinctrl_fec1: fec1grp {
        fsl,pins = <
            MX8MQ_IOMUXC_ENET_MDC__ENET_MDC    0x3
            /* ... باقي الـ pins ... */
        >;
    };
    pinctrl_fec1_sleep: fec1grp-sleep {
        fsl,pins = <
            MX8MQ_IOMUXC_ENET_MDC__ENET_MDC    0x0  /* high-Z */
            /* ... كل الـ pins بـ 0 drive ... */
        >;
    };
};

&fec1 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&pinctrl_fec1>;
    pinctrl-1 = <&pinctrl_fec1_sleep>;  /* أضف sleep state */
};
```

#### الدرس المستفاد
لأي peripheral بيدعم suspend/resume، `sleep` pinctrl state مش optional — هي ضرورية. الـ `pinctrl_dt_to_map` بتـ iterate على `pinctrl-0`, `pinctrl-1`, ... — أول property مش موجودة بتوقف الـ loop.

---

### Scenario 4: TI AM62x — I2C PMIC لا يستجيب

#### العنوان
AM62x SK-EVM: PMIC (TPS65219) على I2C0 لا يستجيب أثناء early boot

#### السياق
منتج: Industrial IoT gateway بـ Texas Instruments AM625 (AM62x family). الـ PMIC مسؤول عن power rails، وعدم استجابته بيمنع SoC من تشغيل محيطات تانية.

#### المشكلة
```
[    0.892] i2c i2c-0: can't use slave addr 0x48: in use
[    0.893] tps65219 0-0048: probe failed: -EBUSY
```

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-maps | grep i2c0
# الـ I2C0 pins متعيّنة لـ device تاني!
```

#### التحليل
في الـ AM62x DT، الـ `main_i2c0_pins_default` اتعمل assign لـ I2C0 وكمان غلطاً لـ GPIO expander device:

```dts
/* خطأ في board DTS */
&gpio_expander {
    pinctrl-names = "default";
    pinctrl-0 = <&main_i2c0_pins_default>;  /* نفس pins الـ I2C! */
};
```

الـ `dt_to_map_one_config` نجح للاتنين، الـ `pinctrl_register_mappings` قبلهم، بس وقت `pinctrl_select_state` الثاني رفض لأن الـ pins `owned` بالفعل.

#### الحل
```bash
# اكتشف التعارض
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinmux-pins | grep "i2c0"
# pin 0: i2c0 (GPIO UNCLAIMED) -- conflict!

# الحل: GPIO expander على I2C1 وليس I2C0
```

```dts
/* am625-iot-gateway.dts - بعد الإصلاح */
&gpio_expander {
    reg = <0x20>;
    /* أزل pinctrl-0 الخاطئ */
};
```

#### الدرس المستفاد
`pinctrl_register_mappings` ما بترفضش duplicate assignments وقت التسجيل — الـ conflict بيظهر فقط وقت `select_state`. استخدم `pinmux-pins` في debugfs لاكتشاف التعارضات قبل الـ production.

---

### Scenario 5: Allwinner H616 — USB OTG لا يكتشف الـ Device Mode

#### العنوان
Orange Pi Zero 2 (H616): USB OTG بيشتغل كـ host دايماً، مش بيتحول لـ device mode

#### السياق
منتج: Embedded media player بـ Allwinner H616. الـ USB port مفروض يشتغل كـ OTG — host أو device حسب الـ ID pin.

#### المشكلة
```dts
/* h616-orangepi-zero2.dts */
&usb_otg {
    pinctrl-names = "default";
    pinctrl-0 = <&usb_otg_pins>;
    dr_mode = "otg";
};
```

```bash
dmesg | grep otg
# [    3.456] musb-hdrc musb-hdrc.1: MUSB HDRC host driver
# بيشتغل كـ host فقط رغم dr_mode = "otg"
```

#### التحليل
```bash
cat /sys/kernel/debug/pinctrl/pio/pinconf-pins | grep "ID\|otg"
# PL10 (USB-ID): bias-pull-up -- ده المشكلة!
```

الـ `usb_otg_pins` بتـ configure الـ ID pin بـ pull-up دايماً، يعني الـ controller بيشوف ID=HIGH = host mode دايماً.

`dt_to_map_one_config` → `ops->dt_node_to_map` (sunxi driver) → بتقرأ الـ `bias-pull-up` من الـ DT وبتطبقها.

#### الحل
```dts
&pio {
    usb_otg_id_pin: usb-otg-id-pin {
        pins = "PL10";
        function = "gpio_in";
        bias-disable;  /* بدل pull-up — يسمح بقراءة الـ ID pin فعلاً */
    };
};

&usb_otg {
    pinctrl-0 = <&usb_otg_pins &usb_otg_id_pin>;
};
```

```bash
# تحقق
cat /sys/kernel/debug/pinctrl/pio/pinconf-pins | grep "PL10"
# PL10: bias-disable  ✓

# الآن OTG بيشتغل صح
dmesg | grep otg
# [    3.456] musb-hdrc: OTG mode enabled
```

#### الدرس المستفاد
الـ pinctrl configuration خطأ بيأثر على الـ hardware behavior بطرق غير مباشرة. `pinconf-pins` في debugfs يكشف الـ actual configuration المطبّقة — ليس بس الـ mux، لكن كمان الـ bias وalـ drive strength.

---

## Phase 7: مصادر ومراجع

### LWN.net Articles

| المقال | الرابط | السنة |
|--------|--------|-------|
| The pin control subsystem | [lwn.net/Articles/468759](https://lwn.net/Articles/468759/) | 2011 |
| drivers: create a pin control subsystem | [lwn.net/Articles/463335](https://lwn.net/Articles/463335/) | 2011 |
| Documentation/pinctrl.txt | [lwn.net/Articles/465077](https://lwn.net/Articles/465077/) | 2011 |
| pinctrl: add a generic pin config interface | [lwn.net/Articles/468770](https://lwn.net/Articles/468770/) | 2011 |
| drivers: create a pin control subsystem v8 | [lwn.net/Articles/460768](https://lwn.net/Articles/460768/) | 2011 |
| pin controller subsystem v7 | [lwn.net/Articles/459190](https://lwn.net/Articles/459190/) | 2011 |
| An alternative device-tree source language | [lwn.net/Articles/730217](https://lwn.net/Articles/730217/) | 2017 |

---

### Kernel Documentation Paths

```
Documentation/driver-api/pin-control.rst    ← الدليل الرئيسي للـ subsystem
Documentation/devicetree/bindings/pinctrl/  ← DT bindings لكل SoCs
Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt
drivers/pinctrl/devicetree.c                ← الملف الرئيسي موضوع الدراسة
drivers/pinctrl/devicetree.h
drivers/pinctrl/core.c                      ← core subsystem
drivers/pinctrl/core.h
include/linux/pinctrl/pinctrl.h             ← public API
include/linux/pinctrl/machine.h             ← pinctrl_map definitions
```

**Online Kernel Docs:**
- [docs.kernel.org — PIN CONTROL subsystem](https://docs.kernel.org/driver-api/pin-control.html)
- [kernel.org — pinctrl-bindings.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt)
- [kernel.org — pinctrl.txt (old)](https://www.kernel.org/doc/Documentation/pinctrl.txt)
- [mjmwired.net — fsl,imx-pinctrl.txt](https://mjmwired.net/kernel/Documentation/devicetree/bindings/pinctrl/fsl,imx-pinctrl.txt)

---

### KernelNewbies Resources

- [Linux_3.8_DriverArch — pinctrl additions](https://kernelnewbies.org/Linux_3.8_DriverArch)
- [Linux_3.9_DriverArch — pinctrl additions](https://kernelnewbies.org/Linux_3.9_DriverArch)
- [KernelNewbies Documents](https://kernelnewbies.org/Documents)

---

### SoC-Specific References

| SoC | الرابط |
|-----|--------|
| STM32MP1 | [wiki.st.com — Pinctrl DT configuration](https://wiki.st.com/stm32mpu/wiki/Pinctrl_device_tree_configuration) |
| i.MX pinctrl | [mjmwired.net — fsl,imx-pinctrl](https://mjmwired.net/kernel/Documentation/devicetree/bindings/pinctrl/fsl,imx-pinctrl.txt) |
| General DT | [elinux.org — Device Tree Reference](https://elinux.org/Device_Tree_Reference) |
| Tutorial | [blog.modest-destiny.com — Linux DT Pinctrl Tutorial](https://blog.modest-destiny.com/posts/linux-device-tree-pinctrl-tutorial/) |
| embedded.com | [Linux Device Driver: Pin Control Subsystem](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) |

---

### Recommended Books

| الكتاب | المؤلف | الأهمية |
|--------|--------|---------|
| **Linux Device Drivers, 3rd Ed. (LDD3)** | Corbet, Rubini, Kroah-Hartman | الأساس — فصول device model و sysfs |
| **Linux Kernel Development, 3rd Ed.** | Robert Love | فهم kernel internals — memory، locking |
| **Embedded Linux Primer, 2nd Ed.** | Christopher Hallinan | DT وplatform drivers من منظور embedded |
| **Mastering Embedded Linux Programming** | Frank Vasquez & Chris Simmonds | pinctrl عملي في Yocto/Buildroot context |
| **The Linux Programming Interface** | Michael Kerrisk | userspace interaction مع sysfs/debugfs |

---

### Search Terms مفيدة

```
pinctrl_dt_to_map site:lore.kernel.org
pinctrl dt_node_to_map driver implementation
"pinctrl-0" "pinctrl-names" device tree binding
pinctrl EPROBE_DEFER of_node phandle
pinctrl_register_mappings kernel
debugfs pinctrl pinmux-pins
of_pinctrl_get EXPORT_SYMBOL_GPL
pinctrl_count_index_with_args usage
#pinctrl-cells device tree
```

---

## Phase 8: Writing simple module

### الهدف
عمل **kprobe** على دالة `of_pinctrl_get` — الدالة الوحيدة الـ exported بـ `EXPORT_SYMBOL_GPL` في `devicetree.c`. الـ kprobe بيسمحلنا نتتبع كل caller بيطلب الـ pin controller من DT node، ونطبع الـ `device_node` name.

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe module: trace every call to of_pinctrl_get()
 *
 * of_pinctrl_get() is the only EXPORT_SYMBOL_GPL in devicetree.c.
 * It wraps get_pinctrl_dev_from_of_node(np) and is called by
 * drivers that need to look up a pinctrl controller by DT node.
 */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/of.h>       /* struct device_node, of_node_full_name() */
#include <linux/printk.h>

/* Forward declaration of the probed function signature */
/* struct pinctrl_dev *of_pinctrl_get(struct device_node *np); */

/* pre-handler: runs BEFORE of_pinctrl_get executes */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * على x86_64: أول argument في rdi
     * على ARM64:  أول argument في x0
     * استخدام regs_get_kernel_argument() بيجعل الكود portable
     */
    struct device_node *np =
        (struct device_node *)regs_get_kernel_argument(regs, 0);

    /* np ممكن يكون NULL — نتحقق قبل استخدامه */
    if (np)
        pr_info("[pinctrl-probe] of_pinctrl_get called for node: %s\n",
                of_node_full_name(np));
    else
        pr_info("[pinctrl-probe] of_pinctrl_get called with NULL node\n");

    return 0; /* 0 = استمر في تنفيذ الدالة الأصلية */
}

/* post-handler: runs AFTER of_pinctrl_get returns */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * على x86_64: return value في rax
     * نطبعه كـ pointer — لو NULL يعني الـ controller مش موجود
     */
    pr_info("[pinctrl-probe] of_pinctrl_get returned: %px\n",
            (void *)regs_return_value(regs));
}

/* تعريف الـ kprobe struct */
static struct kprobe kp = {
    .symbol_name = "of_pinctrl_get",  /* اسم الدالة المراد probe-ها */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

static int __init pinctrl_probe_init(void)
{
    int ret;

    /* سجّل الـ kprobe في الكيرنل */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[pinctrl-probe] register_kprobe failed: %d\n", ret);
        /*
         * أسباب الفشل الشائعة:
         *   -EINVAL : الدالة غير موجودة أو kprobes مش مفعّل
         *   -ENOENT : CONFIG_KPROBES=n أو الدالة inline
         */
        return ret;
    }

    pr_info("[pinctrl-probe] kprobe registered at %px (of_pinctrl_get)\n",
            kp.addr);
    return 0;
}

static void __exit pinctrl_probe_exit(void)
{
    /* إلغاء تسجيل الـ kprobe عند rmmod */
    unregister_kprobe(&kp);
    pr_info("[pinctrl-probe] kprobe unregistered\n");
}

module_init(pinctrl_probe_init);
module_exit(pinctrl_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc study");
MODULE_DESCRIPTION("kprobe tracer for of_pinctrl_get in pinctrl/devicetree.c");
MODULE_VERSION("1.0");
```

---

### Makefile للـ Module

```makefile
# Makefile
obj-m := pinctrl_probe.o

KDIR  ?= /lib/modules/$(shell uname -r)/build
PWD   := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

| الجزء | الشرح |
|-------|-------|
| `kp.symbol_name = "of_pinctrl_get"` | الكيرنل بيحل الـ symbol لـ address تلقائياً عبر `kallsyms` |
| `handler_pre` | بيتشغّل قبل دخول `of_pinctrl_get` — بيقرأ الـ `device_node *np` من الـ registers |
| `regs_get_kernel_argument(regs, 0)` | portable API بيجيب أول argument بغض النظر عن الـ architecture |
| `of_node_full_name(np)` | بيرجّع المسار الكامل للـ DT node (مثلاً `/soc/pinctrl@fe770000`) |
| `handler_post` | بيتشغّل بعد return — بيطبع الـ `pinctrl_dev *` المرجوع |
| `regs_return_value(regs)` | portable API لقراءة الـ return value |
| `register_kprobe` / `unregister_kprobe` | lifecycle الـ probe — لازم unregister في `module_exit` |
| `MODULE_LICENSE("GPL")` | مطلوب لأن `of_pinctrl_get` exported بـ `EXPORT_SYMBOL_GPL` |

---

### استخدام الـ Module

```bash
# بناء الـ module
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
insmod pinctrl_probe.ko

# شوف الـ output (طبع عند كل driver probe يستخدم of_pinctrl_get)
dmesg | grep pinctrl-probe

# مثال output وقت driver probe:
# [   12.345] [pinctrl-probe] kprobe registered at ffffffffc0123456 (of_pinctrl_get)
# [   13.001] [pinctrl-probe] of_pinctrl_get called for node: /soc/pinctrl@fe770000
# [   13.002] [pinctrl-probe] of_pinctrl_get returned: ffff888003a12800

# إزالة الـ module
rmmod pinctrl_probe
dmesg | grep pinctrl-probe | tail -1
# [   20.000] [pinctrl-probe] kprobe unregistered
```

**متطلبات الـ kernel config:**
```
CONFIG_KPROBES=y
CONFIG_KALLSYMS=y
CONFIG_DEBUG_FS=y          # اختياري لكن مفيد
CONFIG_DYNAMIC_DEBUG=y     # اختياري
```
