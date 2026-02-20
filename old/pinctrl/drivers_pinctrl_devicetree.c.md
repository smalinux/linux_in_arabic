# شرح `devicetree.c` — الـ Device Tree Integration

هذا الملف هو **المترجم** بين صيغة الـ Device Tree وبين الـ pinctrl mapping table. يحوّل الـ `pinctrl-0`, `pinctrl-1`... properties في الـ DTS إلى `pinctrl_map` entries قابلة للتفعيل.

---

## أولاً: `struct pinctrl_dt_map` — تتبع الـ Maps المُنشأة من DT

```c
struct pinctrl_dt_map {
    struct list_head   node;      // linked list داخل struct pinctrl
    struct pinctrl_dev *pctldev;  // من أنشأ هذه الـ map
    struct pinctrl_map *map;      // المصفوفة نفسها
    unsigned int       num_maps;  // عددها
};
```

**لماذا نحتاج هذه الـ struct؟**

الـ maps المُنشأة من DT مختلفة عن الـ static maps المُسجَّلة بـ `pinctrl_register_mappings()` — هي **مؤقتة** ويجب تحريرها عند تدمير الـ `pinctrl` handle. هذه الـ struct تتتبعها في قائمة `p->dt_maps` لتحريرها لاحقاً.

---

## ثانياً: دورة حياة الـ DT Maps — التحرير

### `dt_free_map(pctldev, map, num_maps)`

```c
static void dt_free_map(struct pinctrl_dev *pctldev,
                        struct pinctrl_map *map, unsigned int num_maps)
{
    // 1. حرر الـ dev_name لكل entry (مُخصَّص بـ kstrdup_const)
    for (i = 0; i < num_maps; ++i) {
        kfree_const(map[i].dev_name);
        map[i].dev_name = NULL;
    }

    // 2. لو عندنا controller → استدعي dt_free_map الخاصة به
    if (pctldev) {
        ops->dt_free_map(pctldev, map, num_maps);
    } else {
        // DUMMY_STATE ليس له controller → حرر مباشرة
        kfree(map);
    }
}
```

لاحظ التصميم: `dev_name` يُحرر هنا دائماً (لأن هذا الملف هو من خصّصه بـ `kstrdup_const`)، لكن الـ map array نفسها يُحررها الـ driver الخاص به عبر `ops->dt_free_map`.

### `pinctrl_dt_free_maps(p)`

```c
void pinctrl_dt_free_maps(struct pinctrl *p)
{
    list_for_each_entry_safe(dt_map, n1, &p->dt_maps, node) {
        pinctrl_unregister_mappings(dt_map->map);  // أزلها من الـ global table
        list_del(&dt_map->node);                   // أزلها من القائمة المحلية
        dt_free_map(dt_map->pctldev, dt_map->map, dt_map->num_maps);
        kfree(dt_map);                             // حرر الـ wrapper
    }

    of_node_put(p->dev->of_node);  // أفرج عن الـ DT node reference
}
```

يُستدعى عند `devm_pinctrl_put()` أو إزالة الـ device. يمشي على القائمة ويُحرر كل شيء بالترتيب الصحيح.

---

## ثالثاً: `dt_remember_or_free_map()` — حفظ الـ Maps

```c
static int dt_remember_or_free_map(struct pinctrl *p, const char *statename,
                                   struct pinctrl_dev *pctldev,
                                   struct pinctrl_map *map,
                                   unsigned int num_maps)
{
    // 1. أكمل الحقول المشتركة لكل entry
    for (i = 0; i < num_maps; i++) {
        devname = kstrdup_const(dev_name(p->dev), GFP_KERNEL);
        map[i].dev_name      = devname;   // اسم الـ device المالك
        map[i].name          = statename; // اسم الـ state
        if (pctldev)
            map[i].ctrl_dev_name = dev_name(pctldev->dev);
    }

    // 2. أنشئ wrapper وأضفه للقائمة
    dt_map->pctldev  = pctldev;
    dt_map->map      = map;
    dt_map->num_maps = num_maps;
    list_add_tail(&dt_map->node, &p->dt_maps);

    // 3. سجّل في الـ global mapping table
    return pinctrl_register_mappings(map, num_maps);
}
```

لاحظ: الـ driver يُنشئ الـ map بدون `dev_name` و`name` و`ctrl_dev_name` — هذه الملف هو من يُكملها. هذا تقسيم مسؤوليات واضح: الـ driver يعرف الـ hardware، هذا الملف يعرف الـ device context.

---

## رابعاً: `pinctrl_dt_to_map()` — الـ Entry Point الرئيسي

هذه الدالة هي قلب الملف. تُستدعى من `pinctrl_get()` لكل device يملك `of_node`.

```c
int pinctrl_dt_to_map(struct pinctrl *p, struct pinctrl_dev *pctldev)
{
    struct device_node *np = p->dev->of_node;

    if (!np) return 0;  // ليس DT device

    of_node_get(np);  // احجز reference

    // للـ DT المثالي:
    // pinctrl-0 = <&state0_node1 &state0_node2>;
    // pinctrl-1 = <&state1_node>;
    // pinctrl-names = "default", "sleep";

    for (state = 0; ; state++) {
        // ابحث عن "pinctrl-0"، "pinctrl-1"، ...
        propname = kasprintf(GFP_KERNEL, "pinctrl-%d", state);
        prop = of_find_property(np, propname, &size);
        if (!prop) break;  // انتهت الـ states

        // احصل على اسم الـ state من "pinctrl-names"
        ret = of_property_read_string_index(np, "pinctrl-names",
                                            state, &statename);
        if (ret < 0)
            statename = prop->name + strlen("pinctrl-");
            // لو لا يوجد اسم → استخدم الرقم "0", "1", ...

        // كل property هي قائمة phandles
        list = prop->value;
        size /= sizeof(*list);

        for (config = 0; config < size; config++) {
            phandle = be32_to_cpup(list++);
            np_config = of_find_node_by_phandle(phandle);

            ret = dt_to_map_one_config(p, pctldev, statename, np_config);
        }

        // لو الـ property فارغة → أنشئ DUMMY_STATE
        if (!size)
            dt_remember_dummy_state(p, statename);
    }
}
```

```
DTS مثال:
    uart0 {
        pinctrl-names = "default", "sleep";
        pinctrl-0 = <&uart0_pins_default>;    ← state 0
        pinctrl-1 = <&uart0_pins_sleep>;      ← state 1
    };

التسلسل:
    state=0 → propname="pinctrl-0" → موجود
              statename="default"  (من pinctrl-names[0])
              phandle → &uart0_pins_default → dt_to_map_one_config()

    state=1 → propname="pinctrl-1" → موجود
              statename="sleep"    (من pinctrl-names[1])
              phandle → &uart0_pins_sleep → dt_to_map_one_config()

    state=2 → propname="pinctrl-2" → غير موجود → break
```

### تفصيل الـ statename fallback

```c
if (ret < 0)
    statename = prop->name + strlen("pinctrl-");
```

لو لم يوجد `pinctrl-names`:

- `prop->name` = `"pinctrl-0"` → `statename` = `"0"`
- `prop->name` = `"pinctrl-1"` → `statename` = `"1"`

ذكي: لا يخصص memory جديدة، فقط pointer arithmetic في الـ string الموجودة أصلاً.

---

## خامساً: `dt_to_map_one_config()` — معالجة Node واحد

```c
static int dt_to_map_one_config(struct pinctrl *p,
                                struct pinctrl_dev *hog_pctldev,
                                const char *statename,
                                struct device_node *np_config)
{
    // 1. ابحث عن الـ pin controller المسؤول عن هذا الـ node
    np_pctldev = of_node_get(np_config);
    for (;;) {
        // تحقق من pinctrl-use-default
        allow_default |= of_property_read_bool(np_pctldev,
                                               "pinctrl-use-default");

        np_pctldev = of_get_next_parent(np_pctldev);  // اصعد في الـ tree

        if (!np_pctldev || of_node_is_root(np_pctldev)) {
            // وصلنا للـ root ولم نجد controller
            if (IS_ENABLED(CONFIG_MODULES) && !allow_default)
                ret = -EPROBE_DEFER;  // ربما الـ module لم يُحمَّل بعد
            return ret;
        }

        // هل هذا hog؟ (device يُعرّف نفسه)
        if (hog_pctldev && (np_pctldev == p->dev->of_node)) {
            pctldev = hog_pctldev;
            break;
        }

        // هل هذا الـ node مسجَّل كـ pin controller؟
        pctldev = get_pinctrl_dev_from_of_node(np_pctldev);
        if (pctldev) break;

        // هل نحن ندور في حلقة (hog loop)؟
        if (np_pctldev == p->dev->of_node)
            return 1;  // أخبر الـ caller بالـ circular reference
    }

    // 2. استدعي dt_node_to_map الخاصة بالـ driver
    ret = ops->dt_node_to_map(pctldev, np_config, &map, &num_maps);

    // 3. احفظ الـ maps
    return dt_remember_or_free_map(p, statename, pctldev, map, num_maps);
}
```

### كيف يجد الـ Pin Controller؟

```
DTS structure:
    soc {
        pinctrl: pinctrl@11000 {    ← هذا هو الـ controller
            uart0_pins: uart0-pins {
                pins = "GPIO0_A0";
                ← هذا هو np_config
            };
        };
    };

    uart0 {
        pinctrl-0 = <&uart0_pins>;  ← phandle إلى uart0_pins
    };

البحث:
    np_config = uart0_pins node
    أول iteration: parent = pinctrl node
    get_pinctrl_dev_from_of_node(pinctrl node) → وجد! ← break
```

### `pinctrl-use-default` Property

لو وجد هذا الـ property في أي ancestor أثناء الصعود، يُوقف الـ `EPROBE_DEFER`. مفيد لـ platforms التي تحمّل الـ bootloader الـ pinctrl config وتريد الـ kernel يكمل بدون انتظار الـ driver.

---

## سادساً: `dt_remember_dummy_state()`

```c
static int dt_remember_dummy_state(struct pinctrl *p, const char *statename)
{
    map = kzalloc(sizeof(*map), GFP_KERNEL);
    map->type = PIN_MAP_TYPE_DUMMY_STATE;

    // pctldev = NULL (لا يوجد controller لـ DUMMY)
    return dt_remember_or_free_map(p, statename, NULL, map, 1);
}
```

يُستدعى لو property الـ state موجودة لكن فارغة:

```dts
pinctrl-0 = <>;  // ← فارغة
pinctrl-names = "default";
```

هذا يقول: "أعلم أن الـ driver يبحث عن default state، لكن هذه الـ platform لا تحتاج pinctrl. السماح بالمتابعة بدون error."

---

## سابعاً: دوال الـ Index/Args Parsing

هذه الدوال تُستخدم من `pinctrl-single.c` لقراءة lists من الـ DT.

### `pinctrl_find_cells_size(np)`

```c
static int pinctrl_find_cells_size(const struct device_node *np)
{
    // ابحث عن #pinctrl-cells في parent أو grandparent
    error = of_property_read_u32(np->parent, "#pinctrl-cells", &cells_size);
    if (error)
        error = of_property_read_u32(np->parent->parent,
                                     "#pinctrl-cells", &cells_size);
}
```

`#pinctrl-cells` تُخبر كم u32 تأخذ كل entry في الـ list (إضافة للـ index). مثلاً:

```dts
pinctrl: pinctrl@0 {
    #pinctrl-cells = <1>;  // ← كل entry = index + 1 value
};

uart0_pins: uart0-pins {
    pinctrl-single,pins = 
        0x100 0x02   // offset + value (cell size = 1)
        0x104 0x02
    >;
};
```

### `pinctrl_get_list_and_count(np, list_name, list, cells_size, nr_elements)`

```c
static int pinctrl_get_list_and_count(...)
{
    *list = of_get_property(np, list_name, &size);

    *cells_size = pinctrl_find_cells_size(np);

    // كل element = 1 (index) + cells_size
    *nr_elements = (size / sizeof(**list)) / (*cells_size + 1);
}
```

```
مثال: cells_size=1، list = {0x100, 0x02, 0x104, 0x02}
  size = 4 * 4 = 16 bytes
  nr_elements = (16/4) / (1+1) = 4/2 = 2 elements
```

### `pinctrl_count_index_with_args(np, list_name)`

عدّاد بسيط — يُرجع عدد العناصر في الـ list:

```c
int pinctrl_count_index_with_args(const struct device_node *np,
                                   const char *list_name)
{
    pinctrl_get_list_and_count(np, list_name, &list, &nr_cells, &size);
    return size;
}
```

يُستخدمه `pinctrl-single.c`:

```c
rows = pinctrl_count_index_with_args(np, "pinctrl-single,pins");
```

### `pinctrl_copy_args(np, list, index, nr_cells, nr_elem, out_args)`

```c
static int pinctrl_copy_args(...)
{
    out_args->np = np;
    out_args->args_count = nr_cells + 1;

    list += index * (nr_cells + 1);  // اقفز للـ element المطلوب

    for (i = 0; i < nr_cells + 1; i++)
        out_args->args[i] = be32_to_cpup(list++);
}
```

```
list = {0x100, 0x02, 0x104, 0x06}  (2 elements، cells_size=1)

index=0: list += 0 → args = {0x100, 0x02}
index=1: list += 2 → args = {0x104, 0x06}
```

### `pinctrl_parse_index_with_args(np, list_name, index, out_args)`

يجمع `get_list_and_count` و`copy_args` معاً في دالة واحدة:

```c
int pinctrl_parse_index_with_args(np, list_name, index, out_args)
{
    pinctrl_get_list_and_count(np, list_name, &list, &nr_cells, &nr_elem);
    pinctrl_copy_args(np, list, index, nr_cells, nr_elem, out_args);
}
```

يُستخدمه `pinctrl-single.c` هكذا:

```c
for (i = 0; i < rows; i++) {
    pinctrl_parse_index_with_args(np, "pinctrl-single,pins", i, &spec);
    offset = spec.args[0];
    val    = spec.args[1];
}
```

---

## الصورة الكاملة — دورة حياة DT Parsing

```
devm_pinctrl_get(dev)
        │
        ▼
pinctrl_get(dev)
        │
        ▼
pinctrl_dt_to_map(p, NULL)
        │
        ├─ for each "pinctrl-N" property:
        │       │
        │       ├─ get statename from "pinctrl-names"
        │       │
        │       └─ for each phandle in property:
        │               │
        │               ▼
        │           dt_to_map_one_config()
        │               │
        │               ├─ اصعد في DT tree لإيجاد pin controller
        │               │
        │               ├─ استدعي ops->dt_node_to_map()
        │               │   (ينتج pinctrl_map[] من الـ DT node)
        │               │
        │               └─ dt_remember_or_free_map()
        │                   ├─ أكمل dev_name و name و ctrl_dev_name
        │                   ├─ أضف للـ p->dt_maps list
        │                   └─ pinctrl_register_mappings()
        │
        ▼
pinctrl_lookup_state(p, "default")
    → يجد الـ entries في الـ global mapping table
        │
        ▼
pinctrl_select_state(p, state)
    → يُفعّل الـ hardware

عند devm_pinctrl_put():
        │
        ▼
pinctrl_dt_free_maps(p)
    → يُلغي التسجيل ويُحرر كل شيء
```

الخلاصة: هذا الملف يُخفي تعقيد قراءة الـ DT عن باقي الـ subsystem — الـ core يرى فقط `pinctrl_map` entries كأنها static mappings، ولا يعرف من أين جاءت.