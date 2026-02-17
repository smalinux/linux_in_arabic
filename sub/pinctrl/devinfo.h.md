# شرح `devinfo.h` — ربط الـ pinctrl بالـ Device Core

هذا الملف قصير جداً لكنه **الحلقة المفقودة** التي تشرح كيف يتكامل الـ pinctrl تلقائياً مع كل device في الـ kernel بدون أن يكتب كل driver كود صريح.

---

## أولاً: `struct dev_pin_info` — الـ Pin State Container

هذه الـ struct تُخزَّن **داخل `struct device` مباشرة** في `dev->pins`. أي device في الـ kernel يمكنه حمل هذه المعلومات.

```c
struct dev_pin_info {
    struct pinctrl       *p;             // الـ handle الرئيسي للـ pinctrl
    struct pinctrl_state *default_state; // دائماً موجود (خارج CONFIG_PM)
    struct pinctrl_state *init_state;    // دائماً موجود (خارج CONFIG_PM)
#ifdef CONFIG_PM
    struct pinctrl_state *sleep_state;   // فقط لو PM مُفعَّل
    struct pinctrl_state *idle_state;    // فقط لو PM مُفعَّل
#endif
};
```

```
struct device
┌─────────────────────────────────────────────┐
│  name: "foo-uart.0"                         │
│  driver: ...                                │
│  ...                                        │
│  pins: ──────────────────────────────────┐  │
└──────────────────────────────────────────┼──┘
                                           ▼
                              struct dev_pin_info
                         ┌────────────────────────────┐
                         │ p: → struct pinctrl handle │
                         │ default_state: → "default" │
                         │ init_state:    → "init"    │
                         │ sleep_state:   → "sleep"   │ (لو CONFIG_PM)
                         │ idle_state:    → "idle"    │ (لو CONFIG_PM)
                         └────────────────────────────┘
```

لاحظ أن `sleep_state` و`idle_state` محميتان بـ `#ifdef CONFIG_PM` — لأنهما لا معنى لهما على أنظمة بدون power management.

---

## ثانياً: `pinctrl_init_done()` — إشعار اكتمال الـ Probe

```c
extern int pinctrl_init_done(struct device *dev);
```

تُستدعى من الـ device core **بعد اكتمال `probe()`** لإخبار الـ pinctrl subsystem بأن الـ driver انتهى من التهيئة.

الهدف المحدد: لو الـ pins لا تزال في `"init"` state بعد انتهاء الـ probe، الـ core ينقلها تلقائياً إلى `"default"` state. هذا هو التطبيق الفعلي للسلوك الذي شرحناه في `pinctrl-state.h`.

```
Device core يستدعي probe()
        │
        ▼
probe() ينتهي
        │
        ▼
device core يستدعي pinctrl_init_done(dev)
        │
        ├─ لو pins في "init" state ──► ينقلها إلى "default"
        └─ لو pins في "default" أو غيره ──► لا يفعل شيئاً
```

---

## ثالثاً: `dev_pinctrl()` — accessor بسيط

```c
static inline struct pinctrl *dev_pinctrl(struct device *dev)
{
    if (!dev->pins)
        return NULL;
    return dev->pins->p;
}
```

دالة مساعدة للحصول على الـ `pinctrl handle` من أي `struct device *`. تتحقق أولاً من أن `dev->pins` ليس NULL (أي أن هذا الـ device له pinctrl info أصلاً) قبل الوصول إليه.

---

## رابعاً: الـ Stubs عند `CONFIG_PINCTRL=n`

```c
#else
static inline int pinctrl_init_done(struct device *dev) { return 0; }
static inline struct pinctrl *dev_pinctrl(struct device *dev) { return NULL; }
#endif
```

لو الـ pinctrl subsystem غير مُفعَّل في الـ kernel config، كل الدوال تصبح **no-ops** تُحوَّل إلى لا شيء عند الـ compilation. هذا النمط شائع جداً في الـ kernel لضمان أن الـ driver code يعمل على أي configuration.

---

## الصورة الكاملة — دور هذا الملف في الـ Flow

هذا الملف هو **ما يجعل الـ pinctrl يعمل تلقائياً** بدون كود صريح في الـ driver:

```
kernel/driver.c (device core)
        │
        ├─ really_probe()
        │       │
        │       ├─ pinctrl_bind_pins(dev)    ← يملأ dev->pins
        │       │       │                      (يفعل get/lookup لكل state)
        │       │       ▼
        │       │   dev->pins = kzalloc(struct dev_pin_info)
        │       │   dev->pins->p             = pinctrl_get(dev)
        │       │   dev->pins->default_state = lookup("default")
        │       │   dev->pins->init_state    = lookup("init")
        │       │   dev->pins->sleep_state   = lookup("sleep")
        │       │   dev->pins->idle_state    = lookup("idle")
        │       │
        │       ├─ لو init_state موجود → pinctrl_select_state(init_state)
        │       │   لو لا               → pinctrl_select_state(default_state)
        │       │
        │       ├─ drv->probe(dev)      ← كود الـ driver
        │       │
        │       └─ pinctrl_init_done(dev)
        │               │
        │               └─ لو لا تزال في init → انقل إلى default
        │
        └─ dev_pm_ops.suspend()
                │
                └─ pinctrl_pm_select_sleep_state(dev)
                        │
                        └─ pinctrl_select_state(dev->pins->sleep_state)
```

الـ driver العادي لا يرى أي من هذا — الـ device core يُدير كل شيء عبر `dev->pins` الذي يُعرَّف هنا.