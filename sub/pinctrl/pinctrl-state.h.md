# شرح `pinctrl-state.h` — الـ Standard Pin Control States

هذا الملف بسيط جداً — 4 macros فقط تُعرّف أسماء الـ states القياسية — لكن فهمهم العميق مهم جداً للـ power management.

---

## الـ States الأربعة ودورة حياتها

```
        Boot
          │
          ▼
    ┌─────────────┐
    │    "init"   │  ← اختياري، قبل probe() مباشرة
    └──────┬──────┘
           │ بعد probe() تلقائياً
           ▼
    ┌─────────────┐
    │  "default"  │  ← الوضع الطبيعي للتشغيل
    └──────┬──────┘
           │
     ┌─────┴──────┐
     ▼            ▼
┌─────────┐  ┌─────────┐
│ "idle"  │  │ "sleep" │
└────┬────┘  └────┬────┘
     │             │
     └──────┬──────┘
            ▼
         "default"  ← عند الاستيقاظ
```

---

## شرح كل State بالتفصيل

### `PINCTRL_STATE_DEFAULT` = `"default"`

الوضع الطبيعي — الـ pins جاهزة للعمل. يُستخدم في ثلاثة سياقات:

**أولاً:** الـ device core يُفعّله تلقائياً قبل `probe()` (إذا لم يوجد `"init"`).

**ثانياً:** الـ system hogging — الـ pin controller يفعّل هذا الـ state على نفسه فور التسجيل لضبط الـ pins الدائمة (مثل power pins).

**ثالثاً:** عند العودة من السكون في `pm_runtime_resume()` أو `.resume()`.

---

### `PINCTRL_STATE_INIT` = `"init"`

هذا الـ state موجود لحل مشكلة محددة جداً: بعض الـ drivers تسبّب الـ default state **glitch** على الـ pins إذا فُعّلت قبل أن يكتمل الـ probe.

مثال عملي: driver يتحكم في display. لو فعّلت الـ "default" state (التي تُفعّل الـ clock pins) قبل اكتمال الـ initialization، قد يرى الـ display نبضات عشوائية وتظهر وميض (glitch) على الشاشة.

الحل: تُعرّف `"init"` state آمنة (pins في وضع محايد)، وبعد اكتمال `probe()` ينتقل الـ driver تلقائياً إلى `"default"`.

```
بدون "init":  boot → "default" → probe()    ← خطر glitch
مع "init":    boot → "init"   → probe() → "default"  ← آمن
```

**ملاحظة:** إذا انتهى `probe()` والـ pins لا تزال في `"init"` state، الـ core ينقلها تلقائياً إلى `"default"`.

---

### `PINCTRL_STATE_IDLE` = `"idle"`

وضع وسط بين التشغيل الكامل والنوم العميق. النظام هادئ لكن لم يدخل sleep كامل بعد — مثلاً: بعض الـ clocks مُوقفة لكن الـ power لا يزال موجوداً.

يُفعَّل من `pm_runtime_suspend()` أو `pm_runtime_idle()`.

الهدف: توفير الطاقة دون الدخول في دورة suspend/resume كاملة وثقيلة.

---

### `PINCTRL_STATE_SLEEP` = `"sleep"`

أعمق وضع نوم — يُفعَّل من `.suspend()` العادي عند دخول النظام في أعمق حالات الـ sleep.

هنا عادةً تُحوَّل الـ pins لأوضاع تحافظ على الطاقة: pull-down على الـ outputs غير المستخدمة، أو input عالي المقاومة، أو ما يُسمى في بعض الـ datasheets بـ "GPIO mode" كما شرحنا سابقاً.

---

## كيف تُستخدم في الكود

```c
// في الـ driver:
foo_suspend() {
    pinctrl_pm_select_sleep_state(dev);
}

foo_resume() {
    pinctrl_pm_select_default_state(dev);
}

// أو بشكل صريح:
struct pinctrl_state *sleep_state = 
    pinctrl_lookup_state(pinctrl, PINCTRL_STATE_SLEEP);
pinctrl_select_state(pinctrl, sleep_state);
```

---

## ملاحظة مهمة

هذه مجرد أسماء **اتفاقية** — الـ kernel لا يُجبرك على استخدامها. يمكنك تعريف states بأسماء خاصة مثل `"pos-A"` و`"pos-B"` للـ runtime pinmuxing كما رأينا. لكن هذه الأربعة هي التي يفهمها الـ device core ونظام الـ PM تلقائياً دون أي كود إضافي.