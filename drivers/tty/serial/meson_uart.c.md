## Phase 1: الصورة الكبيرة

# `meson_uart.c`

> **PATH**: `drivers/tty/serial/meson_uart.c`
> **Subsystem**: ARM/Amlogic Meson SoC support — Serial / TTY
> **الوظيفة الأساسية**: درايفر الـ UART الخاص بشرائح Amlogic Meson — يربط الـ hardware registers بـ Linux serial core حتى تشتغل الـ serial ports كـ `/dev/ttyAML0` أو `/dev/ttyS0`

---

### ما هي القصة؟

تخيل إنك اشتريت راوتر أو Android TV box مبني على شريحة Amlogic (زي Amlogic S905 أو S922). الشريحة دي جوّاها hardware component اسمه **UART** (Universal Asynchronous Receiver-Transmitter) — ده زي "فم وأذن" الشريحة، بيستخدمه اللينكس عشان:

- يطبع رسائل الكرنل في البداية (kernel boot log)
- يوفر terminal للـ debugging عبر serial cable
- يتكلم مع أجهزة تانية زي GPS أو Bluetooth modules

المشكلة إن كل شريحة بتتكلم بطريقتها الخاصة مع الـ UART hardware. الـ Amlogic Meson chips عندها registers معينة ببروتوكول خاص بيها.

**الملف ده** هو اللي "يترجم" بين لينكس اللي بيقول "اكتب حرف على الـ serial port" وبين الـ hardware registers الفعلية جوّا شريحة Amlogic.

---

### الصورة الكبيرة — إزاي اللينكس بيتعامل مع الـ UART

```
┌─────────────────────────────────────────┐
│         userspace (تطبيقاتك)            │
│  open("/dev/ttyAML0")  write(fd, ...)   │
└───────────────────┬─────────────────────┘
                    │
┌───────────────────▼─────────────────────┐
│         TTY Layer (tty_core)             │
│   tty_write() → line discipline         │
└───────────────────┬─────────────────────┘
                    │
┌───────────────────▼─────────────────────┐
│     Serial Core (serial_core.c)         │
│   uart_write() → يستدعي uart_ops        │
└───────────────────┬─────────────────────┘
                    │
┌───────────────────▼─────────────────────┐
│    meson_uart.c  ← أنتَ هنا             │
│  meson_uart_start_tx()                  │
│  writel(ch, port->membase + WFIFO)      │
└───────────────────┬─────────────────────┘
                    │
┌───────────────────▼─────────────────────┐
│   Amlogic Meson UART Hardware           │
│   Registers: WFIFO, RFIFO, CONTROL...   │
└─────────────────────────────────────────┘
```

الملف ده بيقعد في الطبقة الأخيرة — بين الـ software وبين الـ hardware مباشرة.

---

### ليه الملف ده موجود أصلاً؟

لأن كل UART chip بيختلف. الـ registers بيختلفوا، طريقة حساب الـ baud rate بتختلف، مكان الـ FIFO بيختلف. الـ Linux serial core بيقدم "عقد" (interface) اسمه `uart_ops` — وأي درايفر لازم يملا الـ ops دي بالكود الخاص بشريحته.

الـ `meson_uart.c` بيملا الـ `uart_ops` بدوال خاصة بشرائح Amlogic Meson.

---

### الـ Hardware اللي بيتعامل معاه

شرائح Amlogic Meson عندها UART hardware بـ 6 registers رئيسية:

| Register | Offset | الوظيفة |
|----------|--------|----------|
| `AML_UART_WFIFO` | `0x00` | اكتب بايت هنا عشان يتبعت |
| `AML_UART_RFIFO` | `0x04` | اقرا البايت اللي وصل |
| `AML_UART_CONTROL` | `0x08` | تفعيل TX/RX، interrupts، parity |
| `AML_UART_STATUS` | `0x0c` | هل الـ FIFO فاضي/ممتلي؟ في أخطاء؟ |
| `AML_UART_MISC` | `0x10` | عتبات الـ IRQ للـ FIFO |
| `AML_UART_REG5` | `0x14` | حساب الـ baud rate |

---

### إيه اللي الملف بيعمله بالظبط؟

**١. إدارة البيانات الصادرة (TX)**

الدالة `meson_uart_start_tx()` بتفضل تكتب bytes من الـ software buffer على الـ `WFIFO` register طالما الـ hardware FIFO مش مليان. لو فضل فيه data، بتفعّل الـ TX interrupt عشان تُكمل لما الـ hardware يفضّي.

**٢. استقبال البيانات الواردة (RX)**

الدالة `meson_receive_chars()` بتقرا bytes من الـ `RFIFO` register وتحطها في الـ TTY flip buffer. بتشيك كمان على أخطاء زي الـ parity error والـ frame error.

**٣. الـ Interrupt Handler**

`meson_uart_interrupt()` هو "القلب" — بيتشغّل لما الـ hardware يبعت interrupt. بيشيك على حالة الـ RX والـ TX ويستدعي الدوال المناسبة.

**٤. إعداد الـ baud rate**

الدالة `meson_uart_change_speed()` بتحسب القيمة الصح اللي تتكتب في `REG5` عشان تضبط السرعة. في حالة الـ 24MHz crystal clock، في طريقتين للقسمة (`/2` أو `/3`) حسب الجيل من الشريحة.

**٥. دعم الـ early console**

قبل ما اللينكس يكمّل الـ boot ويجهّز كل حاجة، في آلية اسمها **earlycon** تخلي الـ UART يشتغل من أول لحظة عشان تشوف رسائل الـ boot. الملف ده بيسجّل نفسه كـ earlycon بـ `OF_EARLYCON_DECLARE`.

**٦. دعم الـ kgdb (Console Polling)**

لو مفعّل `CONFIG_CONSOLE_POLL`، الملف بيوفر دالتين `poll_get_char` و`poll_put_char` اللي بتشتغلوا بدون interrupts، مفيدين لـ kgdb debugging.

---

### SoCs المدعومة

| Compatible String | الشريحة | ملاحظة |
|---|---|---|
| `amlogic,meson6-uart` | Meson6 | قديم |
| `amlogic,meson8-uart` | Meson8 | |
| `amlogic,meson8b-uart` | Meson8b | |
| `amlogic,meson-gx-uart` | GX series | |
| `amlogic,meson-g12a-uart` | G12A | عنده `xtal_div2` |
| `amlogic,meson-s4-uart` | S4 | يستخدم `ttyS` |
| `amlogic,meson-a1-uart` | A1 | يستخدم `ttyS` |

الفرق بين الأجيال هو في طريقة حساب الـ baud rate وفي اسم الـ device node (`ttyAML` vs `ttyS`).

---

### الـ Power Domains — قصة مهمة

في شرائح Amlogic، الـ UART بتيجي في نوعين:
- **Always-On (AO)**: بيشتغل حتى لو الـ SoC في وضع النوم. ده اللي بيستخدمه الـ earlycon.
- **Everything-Else (EE)**: بحتاج clock gating وأقل أولوية.

---

### الملفات المرتبطة اللي المفروض تعرفها

**Core Serial Infrastructure:**
- `drivers/tty/serial/serial_core.c` — الـ framework اللي الدرايفر بيتوسع فيه. بيوفر `uart_register_driver()`, `uart_add_one_port()`, إلخ.
- `include/linux/serial_core.h` — تعريف `uart_port`, `uart_ops`, `uart_driver`
- `include/linux/serial.h` — تعريف `serial_struct` وثوابت الـ TTY

**TTY Layer:**
- `drivers/tty/tty_io.c` — الطبقة العليا من الـ TTY subsystem
- `include/linux/tty.h` — تعريف `tty_struct`
- `include/linux/tty_flip.h` — دوال الـ flip buffer زي `tty_insert_flip_char()`

**Amlogic Platform:**
- `drivers/clk/meson/` — clock drivers اللي بتوفر الـ xtal و pclk و baud clocks
- `arch/arm64/boot/dts/amlogic/` — Device Tree files اللي بتعرّف الـ UART nodes
- `Documentation/devicetree/bindings/serial/amlogic,meson-uart.yaml` — مواصفات الـ DT bindings
- `drivers/tty/serial/Kconfig` — إعدادات التفعيل (`SERIAL_MESON`, `SERIAL_MESON_CONSOLE`)

**Early Console:**
- `drivers/tty/serial/earlycon.c` — آلية الـ earlycon اللي الملف بيستخدمها
## Phase 2: شرح الـ Serial Core Framework

### المشكلة — ليش يوجد هذا الـ Framework أصلاً؟

تخيل عندك شركة كبيرة فيها 50 موظف، وكل موظف يتعامل مع الزبائن بطريقته الخاصة. موظف يرد بالتليفون، موظف يرد بالإيميل، موظف يرد بالواتساب. النتيجة؟ فوضى عارمة.

هذا بالضبط كان وضع الـ serial ports في Linux الأول. كل driver لكل hardware مختلف كان يكتب نفس الكود من الصفر:
- كود إدارة الـ TTY (الـ terminal)
- كود التعامل مع الـ userspace (أوامر `ioctl`, `open`, `close`)
- كود الـ flow control
- كود الـ console (للطباعة أثناء boot)
- كود الإحصاءات والـ sysfs

المشكلة المزدوجة:
1. **تكرار كود ضخم** — كل driver يعيد اختراع العجلة
2. **تعقيد هائل** — كل مبرمج hardware يحتاج يفهم نظام الـ TTY بالكامل قبل ما يقدر يكتب driver

الـ kernel في النهاية احتاج طريقة يفصل فيها **"ما يعرفه الكرنل عن الـ TTY"** عن **"ما يعرفه الـ driver عن الـ hardware"**.

---

### الحل — الـ Serial Core Framework

الـ **serial_core** هو الـ framework المسؤول عن إدارة كل serial/UART ports في Linux. فكرته الأساسية هي تقسيم العمل:

| مَن | مسؤولياته |
|-----|-----------|
| **serial_core** (الـ framework) | TTY layer، ioctl، console، buffer management، locking، statistics |
| **Driver** (مثل `meson_uart.c`) | قراءة/كتابة registers، إعداد baud rate، التعامل مع interrupt الـ hardware |

الـ driver ما يحتاج يعرف شيء عن الـ TTY. يكفيه يملأ struct فيها function pointers وتسلّمها للـ framework.

---

### التشبيه الحقيقي

تخيل مطعم:

- **الـ Serial Core** = المدير والنادل. يستقبل الزبون، يأخذ الطلب، يوصل الأكل، يحسب الفاتورة.
- **الـ Driver** = الطباخ. ما يعرف شيء عن الزبون. كل اللي يعرفه: "طلبوا برجر، سوّيه وسلّمه".

النادل (serial_core) يعرف يتعامل مع أي طباخ (أي hardware) طالما الطباخ يتبع القائمة (الـ `uart_ops` struct).

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    User Space                           │
│   open("/dev/ttyAML0")  read()  write()  ioctl()        │
└────────────────────────┬────────────────────────────────┘
                         │  system calls
┌────────────────────────▼────────────────────────────────┐
│                  VFS Layer (الـ Kernel)                 │
│              character device interface                  │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                   TTY Layer                             │
│   tty_write() → line discipline → tty_port             │
│   tty_read()  ← flip buffer    ← tty_insert_flip_char  │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│              Serial Core (serial_core)                  │
│   uart_register_driver()   uart_add_one_port()          │
│   uart_write_wakeup()      uart_get_baud_rate()         │
│   uart_update_timeout()    uart_console_write()         │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │              struct uart_driver                 │   │
│   │  driver_name, dev_name, nr, cons, state         │   │
│   └──────────────────────┬──────────────────────────┘   │
│                          │  1 driver → N ports           │
│   ┌──────────────────────▼──────────────────────────┐   │
│   │              struct uart_port                   │   │
│   │  membase, irq, uartclk, ops, fifosize           │   │
│   │  icount, flags, state, cons                     │   │
│   └──────────────────────┬──────────────────────────┘   │
│                          │  يستدعي function pointers     │
└──────────────────────────┼──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│             struct uart_ops (الـ Driver Interface)      │
│   startup()     shutdown()    set_termios()             │
│   start_tx()    stop_tx()     stop_rx()                 │
│   tx_empty()    get_mctrl()   set_mctrl()               │
│   request_port() release_port() config_port()           │
│   poll_get_char() poll_put_char()  [kgdb support]       │
└──────────────────────────┬──────────────────────────────┘
                           │  implemented by
┌──────────────────────────▼──────────────────────────────┐
│           meson_uart.c (Hardware Driver)                │
│   يتحدث مباشرة مع registers الـ Amlogic UART chip       │
│   WFIFO, RFIFO, CONTROL, STATUS, MISC, REG5             │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│            Amlogic Meson SoC Hardware                   │
│     Physical UART Peripheral (TX/RX pins)               │
└─────────────────────────────────────────────────────────┘
```

---

### الـ Structs الأساسية وكيف تتشابك

#### 1. الـ `uart_driver` — بطاقة تعريف الـ Driver

```c
struct uart_driver {
    struct module       *owner;       // THIS_MODULE
    const char          *driver_name; // "meson_uart"
    const char          *dev_name;    // "ttyAML" أو "ttyS"
    int                  major;       // major number (kernel يختاره)
    int                  minor;       // minor number بداية
    int                  nr;          // عدد الـ ports المدعومة (12)
    struct console       *cons;       // console struct (لو الـ port يدعم console)

    /* private - الـ serial_core يملأها */
    struct uart_state   *state;       // array بحجم nr
    struct tty_driver   *tty_driver;  // الـ TTY driver المربوط
};
```

هذا الـ struct هو "رخصة القيادة" للـ driver. لما تسجل `uart_register_driver()` يعرف الـ kernel إنك تريد تدير N ports باسم `ttyAML0`, `ttyAML1`, ...

في `meson_uart.c` عندنا **اثنين** من هذا الـ struct:

```c
static struct uart_driver meson_uart_driver_ttyAML; // للـ AO (Always-On) UART
static struct uart_driver meson_uart_driver_ttyS;   // للـ EE (Everything Else) UART
```

ليش اثنين؟ لأن الـ Amlogic SoC فيها نوعين من الـ UART:
- **AO UART**: دائماً شغّال حتى في وضع sleep → `/dev/ttyAML*`
- **EE UART**: الـ UART العادي → `/dev/ttyS*`

---

#### 2. الـ `uart_port` — ممثل كل Port بحد ذاته

```c
struct uart_port {
    spinlock_t          lock;           // حماية من الـ race conditions
    unsigned char __iomem *membase;     // عنوان الـ registers في الذاكرة
    unsigned int        irq;            // رقم الـ interrupt
    unsigned int        uartclk;        // سرعة الـ clock (Hz)
    unsigned int        fifosize;       // حجم الـ TX FIFO
    unsigned char       x_char;         // XON/XOFF character خاص
    resource_size_t     mapbase;        // العنوان الفيزيائي للـ registers
    resource_size_t     mapsize;        // حجم منطقة الـ registers
    struct uart_state   *state;         // pointer للـ state (يشمل الـ tty_port)
    struct uart_icount  icount;         // إحصاءات (tx، rx، errors)
    unsigned int        read_status_mask;  // أي errors نهتم فيها؟
    unsigned int        ignore_status_mask; // أي errors نتجاهل؟
    const struct uart_ops *ops;         // الـ function pointers للـ driver
    void                *private_data;  // بيانات خاصة بالـ hardware
    unsigned int        type;           // PORT_MESON
    unsigned int        line;           // رقم الـ port (0، 1، 2...)
    upf_t               flags;          // UPF_BOOT_AUTOCONF, UPF_LOW_LATENCY...
    struct console      *cons;          // console مرتبط (لو وجد)
};
```

**الـ `uart_port` هو قلب كل شيء**. كل عملية (إرسال، استقبال، تغيير baud rate) تمر من خلاله.

---

#### 3. الـ `uart_ops` — عقد العمل بين الـ Framework والـ Driver

```c
struct uart_ops {
    // ---- إدارة الـ TX ----
    void    (*start_tx)(struct uart_port *);   // ابدأ الإرسال
    void    (*stop_tx)(struct uart_port *);    // أوقف الإرسال
    unsigned int (*tx_empty)(struct uart_port *); // هل الـ TX FIFO فاضي؟

    // ---- إدارة الـ RX ----
    void    (*stop_rx)(struct uart_port *);    // أوقف الاستقبال

    // ---- دورة حياة الـ Port ----
    int     (*startup)(struct uart_port *);   // فتح الـ port (request IRQ)
    void    (*shutdown)(struct uart_port *);  // إغلاق الـ port (free IRQ)

    // ---- الإعدادات ----
    void    (*set_termios)(struct uart_port *, struct ktermios *, const struct ktermios *);
    // baud rate، data bits، parity، stop bits

    // ---- الـ Modem Control ----
    unsigned int (*get_mctrl)(struct uart_port *); // اقرأ CTS/DCD/DSR
    void    (*set_mctrl)(struct uart_port *, unsigned int); // اكتب RTS/DTR

    // ---- إدارة الموارد ----
    int     (*request_port)(struct uart_port *);  // احجز الـ I/O memory
    void    (*release_port)(struct uart_port *);  // حرر الـ I/O memory
    void    (*config_port)(struct uart_port *, int); // detect + configure
    int     (*verify_port)(struct uart_port *, struct serial_struct *); // تحقق

    // ---- kgdb/console polling (اختياري) ----
    void    (*poll_put_char)(struct uart_port *, unsigned char);
    int     (*poll_get_char)(struct uart_port *);
};
```

الـ `meson_uart.c` يملأ هذا الـ struct في:

```c
static const struct uart_ops meson_uart_ops = {
    .set_mctrl    = meson_uart_set_mctrl,
    .get_mctrl    = meson_uart_get_mctrl,
    .tx_empty     = meson_uart_tx_empty,
    .start_tx     = meson_uart_start_tx,
    .stop_tx      = meson_uart_stop_tx,
    .stop_rx      = meson_uart_stop_rx,
    .startup      = meson_uart_startup,
    .shutdown     = meson_uart_shutdown,
    .set_termios  = meson_uart_set_termios,
    .type         = meson_uart_type,
    .config_port  = meson_uart_config_port,
    .request_port = meson_uart_request_port,
    .release_port = meson_uart_release_port,
    .verify_port  = meson_uart_verify_port,
    .poll_get_char = meson_uart_poll_get_char,  // عند تفعيل CONFIG_CONSOLE_POLL
    .poll_put_char = meson_uart_poll_put_char,
};
```

---

#### 4. الـ `uart_state` و `tty_port` — جسر البيانات

```
uart_driver
    └── uart_state[]   (array بحجم nr)
            ├── tty_port        (الـ TTY buffer وإدارة الـ opens)
            │       └── xmit_fifo  (kfifo للـ TX data من userspace)
            ├── uart_port*      (pointer لجهاز الـ hardware)
            └── pm_state        (حالة الـ power management)
```

الـ `xmit_fifo` هي الـ buffer المتوسطة: الـ userspace يكتب فيها، والـ driver يقرأ منها ويرسل للـ hardware.

---

### رحلة البيانات — من الـ User إلى الـ Hardware

#### إرسال بيانات (TX Path)

```
User writes to /dev/ttyAML0
         │
         ▼
    tty_write()        [TTY layer]
         │
         ▼
  tty_port.xmit_fifo  [circular buffer - يخزن البيانات مؤقتاً]
         │
         ▼
  serial_core calls ops->start_tx()
         │
         ▼
  meson_uart_start_tx()
         │  loop: كل ما TX FIFO مش مليان
         ▼
  uart_fifo_get() → يجيب char من xmit_fifo
         │
         ▼
  writel(ch, port->membase + AML_UART_WFIFO)
         │
         ▼
  Amlogic UART Hardware → TX Pin → Serial Line
```

#### استقبال بيانات (RX Path)

```
Serial Line → RX Pin → Amlogic UART Hardware
         │
         ▼
  UART Interrupt fires (RX FIFO not empty)
         │
         ▼
  meson_uart_interrupt()
         │
         ▼
  meson_receive_chars()
         │  loop: كل ما RX FIFO مش فاضي
         ▼
  readl(port->membase + AML_UART_RFIFO) → يقرأ الـ byte
         │
         ▼
  tty_insert_flip_char(tport, ch, flag)  → يحط في flip buffer
         │
         ▼
  tty_flip_buffer_push()  → يرسل لـ TTY layer
         │
         ▼
  Line Discipline → User's read() returns data
```

---

### الـ TTY Flip Buffer — ليش اسمه "Flip"؟

الـ **flip buffer** حيلة ذكية. فيه bufferين:

```
         [interrupt context writes here]
         ┌─────────────────────────────┐
Buffer A │ c1 | c2 | c3 | c4 | ...    │  ← الـ driver يكتب هنا
         └─────────────────────────────┘
         ┌─────────────────────────────┐
Buffer B │ c5 | c6 | c7 | ...         │  ← الـ TTY يقرأ منه
         └─────────────────────────────┘
         [process context reads here]
```

عند `tty_flip_buffer_push()`: الـ bufferين يتبادلان أدوارهم (flip). هذا يمنع الـ race condition بين الـ interrupt handler (اللي يكتب) والـ process (اللي يقرأ).

---

### الـ Console Framework — البورت الخاص

الـ UART الأول عادةً يستخدم كـ **kernel console** — المكان اللي يطبع فيه الكرنل رسائله (dmesg، kernel panics، boot messages).

```
┌────────────────────────────────────────────────┐
│           struct console                       │
│   .name  = "ttyAML"                           │
│   .write = meson_serial_console_write         │  ← يكتب مباشرة بدون buffers
│   .setup = meson_serial_console_setup         │  ← يعدّ الـ baud rate
│   .flags = CON_PRINTBUFFER                    │  ← يطبع ما فاته قبل التسجيل
│   .index = -1                                 │  ← أي port يستخدم
└────────────────────────────────────────────────┘
```

**الفرق بين console write والعادي:**
- الـ console write يعطّل الـ interrupts ويكتب مباشرة poll-style (لأنه قد يُستدعى من kernel panic)
- الـ عادي يستخدم الـ interrupt-driven approach مع buffers

**الـ Early Console** — `earlycon`:

```c
OF_EARLYCON_DECLARE(meson, "amlogic,meson-ao-uart",
                   meson_serial_early_console_setup);
```

هذا للطباعة **قبل** ما kernel يفتح أي driver أصلاً. مفيد لـ debugging أثناء الـ boot الأول.

---

### الـ Clock Framework — إدارة سرعة الـ Baud Rate

```
                  ┌─────────────────────────────┐
                  │   Amlogic Clock Controller  │
                  │                             │
                  │  xtal ──→ 24 MHz            │
                  │  pclk ──→ peripheral clock  │
                  │  baud ──→ baud rate clock   │
                  └──────────┬──────────────────┘
                             │  clk_get_rate(clk_baud)
                             ▼
                    port->uartclk = N Hz
                             │
                             ▼
              meson_uart_change_speed()
                             │
              ┌──────────────┴──────────────┐
              │   uartclk == 24 MHz?        │
              │   نعم → XTAL mode           │  لا → PLL mode
              │   val = 24M/3/baud - 1      │  val = clk/4/baud - 1
              │   + AML_UART_BAUD_XTAL      │
              └─────────────────────────────┘
                             │
                             ▼
              writel(val, port->membase + AML_UART_REG5)
```

الـ 24 MHz من الـ crystal (XTAL) أكثر دقة من الـ PLL، لهذا لو الـ clock هو 24 MHz يستخدم طريقة حساب مختلفة.

---

### الـ Platform Driver — كيف يُكتشف الـ Hardware؟

في الأنظمة المدمجة (embedded) ما في BIOS. الـ kernel يعرف الـ hardware من ملف يسمى **Device Tree (DT)**:

```
Device Tree:
serial@c81004c0 {
    compatible = "amlogic,meson-gx-uart";
    reg = <0xc81004c0 0x18>;         ← عنوان الـ registers
    interrupts = <0 26 1>;           ← رقم الـ interrupt
    clocks = <&xtal>, <&uart0>, <&xtal>;
    clock-names = "xtal", "pclk", "baud";
    fifo-size = <128>;
};
```

الـ kernel يقرأ الـ DT ويبحث عن driver يتوافق مع `compatible`:

```c
static const struct of_device_id meson_uart_dt_match[] = {
    { .compatible = "amlogic,meson6-uart" },
    { .compatible = "amlogic,meson8-uart" },
    { .compatible = "amlogic,meson-gx-uart" },
    { .compatible = "amlogic,meson-g12a-uart",
      .data = &meson_g12a_uart_data },   // ← بيانات خاصة بهذا الـ chip
    { .compatible = "amlogic,meson-a1-uart",
      .data = &meson_a1_uart_data },
    ...
};
```

لما يجد تطابق → يستدعي `meson_uart_probe()` ويمرر له معلومات الـ hardware.

---

### الـ `meson_uart_data` — تعدد الـ Chip Variants

```c
struct meson_uart_data {
    struct uart_driver *uart_driver; // أي driver يستخدم (ttyAML أو ttyS)
    bool has_xtal_div2;              // هل يدعم قسمة XTAL على 2؟
};

// G12A: يدعم XTAL/2، يستخدم ttyAML
static struct meson_uart_data meson_g12a_uart_data = {
    .has_xtal_div2 = true,
};

// A1: لا يدعم XTAL/2، يستخدم ttyS
static struct meson_uart_data meson_a1_uart_data = {
    .uart_driver   = &meson_uart_driver_ttyS,
    .has_xtal_div2 = false,
};

// S4: يدعم XTAL/2، يستخدم ttyS
static struct meson_uart_data meson_s4_uart_data = {
    .uart_driver   = &meson_uart_driver_ttyS,
    .has_xtal_div2 = true,
};
```

بدل ما تكتب driver منفصل لكل chip، تغيّر فقط هذه البيانات الصغيرة.

---

### ملخص — الصورة الكاملة في جملة

**الـ Serial Core framework** يعمل كطبقة وسيطة تأخذ على عاتقها كل تعقيدات نظام الـ TTY، وتعرض للـ hardware driver واجهة بسيطة (الـ `uart_ops`) — الـ driver ما يحتاج يعرف إلا كيف يتحدث مع الـ registers، والـ framework يتكفل بالباقي.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### جدول الـ Registers — Cheatsheet

| Register | Offset | الوظيفة |
|---|---|---|
| `AML_UART_WFIFO` | `0x00` | اكتب بايت هنا تبعت على السلك |
| `AML_UART_RFIFO` | `0x04` | اقرأ من هنا تجيب بايت وصل |
| `AML_UART_CONTROL` | `0x08` | كل إعدادات الـ UART في مكان واحد |
| `AML_UART_STATUS` | `0x0c` | حالة الـ FIFO والأخطاء |
| `AML_UART_MISC` | `0x10` | عتبات الـ IRQ (كم بايت قبل ما يشتغل الـ interrupt) |
| `AML_UART_REG5` | `0x14` | حساب الـ baud rate |

---

### جدول بتات الـ CONTROL Register

| Bit/Mask | الاسم | المعنى |
|---|---|---|
| `BIT(12)` | `AML_UART_TX_EN` | تفعيل الإرسال |
| `BIT(13)` | `AML_UART_RX_EN` | تفعيل الاستقبال |
| `BIT(15)` | `AML_UART_TWO_WIRE_EN` | وضع السلكين فقط (بدون RTS/CTS) |
| `0x03 << 16` | `AML_UART_STOP_BIT_LEN_MASK` | عدد بتات التوقف |
| `0x00 << 16` | `AML_UART_STOP_BIT_1SB` | stop bit واحد |
| `0x01 << 16` | `AML_UART_STOP_BIT_2SB` | stop bit اثنان |
| `BIT(18)` | `AML_UART_PARITY_TYPE` | نوع الـ parity: 0=even, 1=odd |
| `BIT(19)` | `AML_UART_PARITY_EN` | تفعيل الـ parity |
| `BIT(22)` | `AML_UART_TX_RST` | reset الـ TX FIFO |
| `BIT(23)` | `AML_UART_RX_RST` | reset الـ RX FIFO |
| `BIT(24)` | `AML_UART_CLEAR_ERR` | امسح أعلام الأخطاء (يدوي — لا تصفر وحدها) |
| `BIT(27)` | `AML_UART_RX_INT_EN` | interrupt عند وصول بيانات |
| `BIT(28)` | `AML_UART_TX_INT_EN` | interrupt عندما يفرغ الـ TX FIFO |
| `0x03 << 20` | `AML_UART_DATA_LEN_MASK` | طول البيانات |
| `0x00 << 20` | `AML_UART_DATA_LEN_8BIT` | 8 بتات |
| `0x01 << 20` | `AML_UART_DATA_LEN_7BIT` | 7 بتات |
| `0x02 << 20` | `AML_UART_DATA_LEN_6BIT` | 6 بتات |
| `0x03 << 20` | `AML_UART_DATA_LEN_5BIT` | 5 بتات |

---

### جدول بتات الـ STATUS Register

| Bit | الاسم | المعنى |
|---|---|---|
| `BIT(16)` | `AML_UART_PARITY_ERR` | خطأ في الـ parity |
| `BIT(17)` | `AML_UART_FRAME_ERR` | خطأ في الـ framing (stop bit خاطئ) |
| `BIT(18)` | `AML_UART_TX_FIFO_WERR` | فاض الـ RX buffer = overrun |
| `BIT(20)` | `AML_UART_RX_EMPTY` | الـ RX FIFO فاضي (لا يوجد بيانات) |
| `BIT(21)` | `AML_UART_TX_FULL` | الـ TX FIFO ممتلئ |
| `BIT(22)` | `AML_UART_TX_EMPTY` | الـ TX FIFO فاضي تماماً |
| `BIT(25)` | `AML_UART_XMIT_BUSY` | الـ shift register لسا يشتغل |

> **ملاحظة:** `AML_UART_ERR` = `PARITY_ERR | FRAME_ERR | TX_FIFO_WERR` — ماسك جاهز لكشف أي خطأ.

---

### جدول بتات الـ REG5 (Baud Rate)

| Bit/Mask | الاسم | المعنى |
|---|---|---|
| `0x7fffff` | `AML_UART_BAUD_MASK` | قيمة القسمة لحساب الـ baud |
| `BIT(23)` | `AML_UART_BAUD_USE` | استخدم القيمة في REG5 (لازم تشتغل) |
| `BIT(24)` | `AML_UART_BAUD_XTAL` | استخدم crystal clock مش PLL |
| `BIT(27)` | `AML_UART_BAUD_XTAL_DIV2` | قسّم الـ crystal على 2 قبل الحساب |

---

### جدول ثوابت مهمة أخرى

| الثابت | القيمة | الاستخدام |
|---|---|---|
| `AML_UART_PORT_NUM` | 12 | أقصى عدد منافذ |
| `AML_UART_PORT_OFFSET` | 6 | أول index للمنافذ اللي بدون alias |
| `AML_UART_POLL_USEC` | 5 | وقت الانتظار بين محاولات polling |
| `AML_UART_TIMEOUT_USEC` | 10000 | timeout كلي لـ kgdb polling |

---

### الـ UPF flags المستخدمة في الـ Driver

| Flag | المعنى في السياق |
|---|---|
| `UPF_BOOT_AUTOCONF` | اكتشف نوع المنفذ تلقائياً عند البوت |
| `UPF_LOW_LATENCY` | وضع زمن استجابة منخفض |
| `UPF_HARD_FLOW` | دعم RTS/CTS الهاردوير (يُفعَّل عند وجود `uart-has-rtscts` في DT) |

---

### الـ Structs المهمة

#### `struct meson_uart_data`

الغرض: بيانات خاصة بكل نسخة من شرائح Amlogic — تُحفظ في `uart_port->private_data`.

```c
struct meson_uart_data {
    struct uart_driver *uart_driver;  /* أي driver يستخدم: ttyAML أم ttyS */
    bool has_xtal_div2;               /* هل تدعم الشريحة قسمة الـ xtal على 2 */
};
```

| الحقل | الوظيفة |
|---|---|
| `uart_driver` | إذا كان NULL يُستخدم `ttyAML` كافتراضي |
| `has_xtal_div2` | يؤثر على حساب الـ baud عند `uartclk == 24MHz` |

**الاستخدام في الشرائح:**

| الشريحة | `uart_driver` | `has_xtal_div2` |
|---|---|---|
| meson6/8/8b/gx | — (NULL) | false |
| meson-g12a | — (NULL) | **true** |
| meson-a1 | `ttyS` | false |
| meson-s4 | `ttyS` | **true** |

---

#### `struct uart_port`

الغرض: قلب كل شيء — يمثل منفذ UART واحد. يحمل كل ما يتعلق بالهاردوير والحالة.

الحقول المهمة التي يُعبّئها الـ driver:

```c
struct uart_port {
    spinlock_t      lock;          /* القفل الرئيسي للمنفذ */
    unsigned char __iomem *membase;/* عنوان الـ registers بعد ioremap */
    resource_size_t mapbase;       /* عنوان فيزيائي من DT */
    resource_size_t mapsize;       /* حجم منطقة الـ registers */
    unsigned int    irq;           /* رقم الـ interrupt */
    unsigned int    uartclk;       /* تردد الكلوك (يُحدَّد من clk_baud) */
    unsigned int    fifosize;      /* حجم الـ TX FIFO (64 أو 128) */
    unsigned char   x_char;        /* حرف XON/XOFF في الطريق */
    unsigned int    type;          /* PORT_MESON */
    const struct uart_ops *ops;    /* جدول الدوال → meson_uart_ops */
    struct uart_state *state;      /* يحوي tty_port والـ xmit buffer */
    struct uart_icount icount;     /* عدادات tx/rx/errors */
    upf_t           flags;         /* UPF_BOOT_AUTOCONF | UPF_LOW_LATENCY ... */
    unsigned int    read_status_mask;   /* أي أخطاء تُبلَّغ للـ TTY */
    unsigned int    ignore_status_mask; /* أي أخطاء تُتجاهل */
    void            *private_data; /* يشير إلى meson_uart_data */
    struct device   *dev;          /* الجهاز الأب (platform_device) */
    unsigned int    line;          /* رقم المنفذ (pdev->id) */
};
```

---

#### `struct uart_driver`

الغرض: يمثل الـ driver ككل عند تسجيله مع kernel — يربط بين منافذ UART وطبقة الـ TTY.

```c
struct uart_driver {
    struct module   *owner;        /* THIS_MODULE */
    const char      *driver_name;  /* "meson_uart" */
    const char      *dev_name;     /* "ttyAML" أو "ttyS" */
    int              nr;           /* AML_UART_PORT_NUM = 12 */
    struct console  *cons;         /* console إذا كان SERIAL_MESON_CONSOLE مفعّل */
    struct uart_state *state;      /* يُملأ عند uart_register_driver() */
    struct tty_driver *tty_driver; /* يُملأ داخلياً */
};
```

في الـ driver عندنا نسختين:
- **`meson_uart_driver_ttyAML`** — للشرائح القديمة
- **`meson_uart_driver_ttyS`** — لـ A1 وS4 (compatible مع `/dev/ttyS*`)

---

#### `struct uart_ops` — جدول الدوال (meson_uart_ops)

الغرض: نقطة التواصل بين serial_core وبين الهاردوير. كل حقل هو pointer لدالة.

| الحقل في uart_ops | الدالة المُنفَّذة | الوظيفة |
|---|---|---|
| `tx_empty` | `meson_uart_tx_empty` | هل FIFO + shift register فاضيين؟ |
| `set_mctrl` | `meson_uart_set_mctrl` | تعيين RTS/DTR (فارغة — لا modem signals) |
| `get_mctrl` | `meson_uart_get_mctrl` | يُرجع دائماً `TIOCM_CTS` |
| `start_tx` | `meson_uart_start_tx` | ابدأ الإرسال من الـ xmit buffer |
| `stop_tx` | `meson_uart_stop_tx` | أطفئ TX_INT_EN |
| `stop_rx` | `meson_uart_stop_rx` | أطفئ RX_EN |
| `startup` | `meson_uart_startup` | فعّل TX/RX + IRQ، اطلب الـ interrupt |
| `shutdown` | `meson_uart_shutdown` | أوقف كل شيء، حرر الـ interrupt |
| `set_termios` | `meson_uart_set_termios` | غيّر baud/parity/bits/flow |
| `type` | `meson_uart_type` | أرجع "meson_uart" |
| `config_port` | `meson_uart_config_port` | عيّن PORT_MESON واطلب الـ port |
| `request_port` | `meson_uart_request_port` | احجز منطقة الذاكرة و ioremap |
| `release_port` | `meson_uart_release_port` | حرر الـ ioremap والمنطقة |
| `verify_port` | `meson_uart_verify_port` | تحقق من إعدادات serial_struct |
| `poll_put_char` | `meson_uart_poll_put_char` | إرسال حرف لـ kgdb (blocking) |
| `poll_get_char` | `meson_uart_poll_get_char` | استقبال حرف لـ kgdb |

---

#### `struct uart_state` و `struct tty_port`

**`uart_state`** يربط بين `uart_port` وطبقة الـ TTY:

```c
struct uart_state {
    struct tty_port     port;       /* يحوي xmit_fifo (kfifo) */
    enum uart_pm_state  pm_state;
    struct uart_port    *uart_port; /* العودة للـ uart_port */
};
```

في `meson_uart_start_tx`:
```c
struct tty_port *tport = &port->state->port;
// tport->xmit_fifo هو الـ kfifo الذي يحمل البيانات المنتظرة للإرسال
```

---

#### `struct console` — لوحة الـ Console

عند تفعيل `CONFIG_SERIAL_MESON_CONSOLE`، ينشئ الـ macro `MESON_SERIAL_CONSOLE` هيكل console:

```c
struct console meson_serial_console_ttyAML = {
    .name   = "ttyAML",
    .write  = meson_serial_console_write,
    .device = uart_console_device,
    .setup  = meson_serial_console_setup,
    .flags  = CON_PRINTBUFFER,
    .index  = -1,
    .data   = &meson_uart_driver_ttyAML,
};
```

---

### مخطط علاقات الـ Structs

```
platform_device (pdev)
    │
    ├─► device_get_match_data() ──► meson_uart_data
    │                                   │
    │                                   ├── uart_driver  (ttyAML أو ttyS)
    │                                   └── has_xtal_div2
    │
    └─► devm_kzalloc() ──────────► uart_port
                                       │
                                       ├── .ops ──────────► uart_ops
                                       │                    (جدول الدوال)
                                       │
                                       ├── .private_data ─► meson_uart_data
                                       │
                                       ├── .state ─────────► uart_state
                                       │                       │
                                       │                       └── tty_port
                                       │                           (xmit_fifo)
                                       │
                                       ├── .cons ──────────► console (اختياري)
                                       │
                                       └── .dev ───────────► device (pdev->dev)


meson_ports[12]  ◄─── مصفوفة global تحفظ كل uart_port نشط
```

---

### مخطط دورة حياة الـ Driver

```
insmod / module_init
    │
    ▼
module_platform_driver(meson_uart_platform_driver)
    │
    ▼ [kernel يكتشف device في DT]
meson_uart_probe(pdev)
    │
    ├─ 1. احسب id من DT alias أو من أول slot فاضي
    ├─ 2. platform_get_resource() ← عنوان الـ registers
    ├─ 3. platform_get_irq()      ← رقم الـ interrupt
    ├─ 4. of_property_read_u32()  ← fifo-size
    ├─ 5. devm_kzalloc()          ← أنشئ uart_port
    ├─ 6. meson_uart_probe_clocks()
    │       ├─ devm_clk_get_enabled("pclk")
    │       ├─ devm_clk_get_enabled("xtal")
    │       └─ devm_clk_get_enabled("baud") → port->uartclk
    ├─ 7. device_get_match_data() ← meson_uart_data
    ├─ 8. مرة واحدة فقط: uart_register_driver()
    ├─ 9. عبّئ حقول uart_port كلها
    ├─ 10. meson_ports[id] = port
    ├─ 11. meson_uart_request_port() → ioremap
    │       meson_uart_reset()       → TX/RX reset
    │       meson_uart_release_port() → iounmap مؤقت
    └─ 12. uart_add_one_port() ← سجّل المنفذ مع serial_core
                │
                └─► [serial_core يستدعي config_port]
                        └─► meson_uart_config_port()
                                └─► meson_uart_request_port() ← ioremap نهائي


[المستخدم يفتح /dev/ttyAML0]
    │
    ▼
serial_core يستدعي ops->startup()
    │
    ▼
meson_uart_startup(port)
    ├─ أشعل TX_EN + RX_EN
    ├─ أشعل RX_INT_EN + TX_INT_EN
    ├─ اضبط عتبات الـ IRQ في MISC
    └─ request_irq(meson_uart_interrupt)


[المستخدم يغلق /dev/ttyAML0]
    │
    ▼
meson_uart_shutdown(port)
    ├─ free_irq()
    ├─ أطفئ RX_EN
    └─ أطفئ RX_INT_EN + TX_INT_EN


rmmod / meson_uart_remove(pdev)
    ├─ uart_remove_one_port()
    ├─ meson_ports[id] = NULL
    └─ إذا لا يوجد منافذ أخرى: uart_unregister_driver()
```

---

### مخطط تدفق استقبال البيانات (RX Path)

```
البايتات تصل على السلك
    │
    ▼
[Hardware يملأ RX FIFO]
    │
    ▼ [RX_EMPTY = 0 → interrupt يطلع]
meson_uart_interrupt(irq, port)
    │
    ├─ uart_port_lock(port)   ← أخذ الـ spinlock
    │
    ├─ STATUS & AML_UART_RX_EMPTY == 0 ؟
    │       │
    │       ▼ نعم
    │   meson_receive_chars(port)
    │       │
    │       ├─ Loop: اقرأ STATUS
    │       │   ├─ إذا AML_UART_ERR:
    │       │   │   ├─ حدّث icount.overrun/frame/parity
    │       │   │   ├─ اكتب CLEAR_ERR (ثم امسحها يدوياً)
    │       │   │   └─ عيّن flag = TTY_FRAME أو TTY_PARITY
    │       │   ├─ اقرأ ch من RFIFO
    │       │   ├─ إذا FRAME_ERR && ch==0: brk++, flag=TTY_BREAK
    │       │   ├─ uart_prepare_sysrq_char() ← kgdb؟
    │       │   └─ tty_insert_flip_char(tport, ch, flag)
    │       │
    │       └─ tty_flip_buffer_push() ← أرسل للـ line discipline
    │
    └─ uart_unlock_and_check_sysrq(port) ← حرر القفل
```

---

### مخطط تدفق إرسال البيانات (TX Path)

```
userspace يكتب على المنفذ
    │
    ▼
TTY layer يملأ tty_port->xmit_fifo
    │
    ▼
serial_core يستدعي ops->start_tx()
    │
    ▼
meson_uart_start_tx(port)
    │
    ├─ uart_tx_stopped()؟ → stop_tx() والخروج
    │
    ├─ Loop: while TX_FIFO not FULL
    │   ├─ إذا x_char: اكتب WFIFO مباشرة، icount.tx++
    │   └─ uart_fifo_get() → اقرأ بايت من xmit_fifo → اكتب WFIFO
    │
    ├─ إذا xmit_fifo لسا فيه بيانات:
    │       أشعل TX_INT_EN ← سيولّد interrupt عندما يفرغ FIFO
    │
    └─ إذا xmit_fifo < WAKEUP_CHARS (256):
            uart_write_wakeup() ← أيقظ userspace ليرسل المزيد


[TX FIFO يفرغ → interrupt]
    │
    ▼
meson_uart_interrupt()
    │
    ├─ STATUS & AML_UART_TX_FULL == 0
    └─ CONTROL & AML_UART_TX_INT_EN ؟
            └─► meson_uart_start_tx() ← أرسل المزيد
```

---

### مخطط تدفق ضبط الـ Termios

```
userspace يستدعي tcsetattr()
    │
    ▼
serial_core يستدعي ops->set_termios()
    │
    ▼
meson_uart_set_termios(port, termios, old)
    │
    ├─ uart_port_lock_irqsave()
    │
    ├─ اقرأ CONTROL register
    │
    ├─ عدّل DATA_LEN_MASK حسب CSIZE (5/6/7/8 بت)
    │
    ├─ PARENB: فعّل/ألغِ PARITY_EN
    ├─ PARODD: فعّل/ألغِ PARITY_TYPE (odd/even)
    ├─ CSTOPB: STOP_BIT_2SB أم STOP_BIT_1SB
    │
    ├─ CRTSCTS + UPF_HARD_FLOW:
    │   ├─ نعم: ألغِ TWO_WIRE_EN (فعّل RTS/CTS الهاردوير)
    │   └─ لا:  أشعل TWO_WIRE_EN (سلكان فقط)
    │
    ├─ اكتب CONTROL
    │
    ├─ uart_get_baud_rate() ← احسب الـ baud المطلوب
    │
    ├─ meson_uart_change_speed(port, baud)
    │   ├─ انتظر TX_EMPTY
    │   ├─ إذا uartclk==24MHz:
    │   │   ├─ has_xtal_div2: xtal_div=2, أشعل BAUD_XTAL_DIV2
    │   │   └─ مش: xtal_div=3
    │   │   val = DIV_ROUND_CLOSEST(24MHz / xtal_div, baud) - 1
    │   │   val |= BAUD_XTAL
    │   └─ غير ذلك: val = DIV_ROUND_CLOSEST(uartclk/4, baud) - 1
    │   val |= BAUD_USE → اكتب REG5
    │
    ├─ حدّث read_status_mask و ignore_status_mask
    ├─ uart_update_timeout()
    └─ uart_port_unlock_irqrestore()
```

---

### مخطط Early Console

```
Bootloader يمرر "earlycon=meson,..."
    │
    ▼
kernel يستدعي OF_EARLYCON_DECLARE handler
    │
    ▼
meson_serial_early_console_setup(device, opt)
    ├─ تحقق من device->port.membase
    ├─ مeson_uart_enable_tx_engine() ← فعّل TX_EN فقط
    └─ device->con->write = meson_serial_early_console_write
            │
            └─► meson_serial_port_write()
                    └─► uart_console_write()
                            └─► meson_console_putchar()
                                    └─► انتظر TX_FULL=0 → اكتب WFIFO
```

---

### استراتيجية الـ Locking

الـ driver يستخدم قفلاً واحداً فقط: **`port->lock`** وهو `spinlock_t` موجود داخل `uart_port`.

#### القفل يحمي:

| المورد | لماذا؟ |
|---|---|
| قراءة وكتابة الـ registers | لمنع تعارض الـ interrupt مع الكود العادي |
| `port->icount` (العدادات) | تُحدَّث من interrupt handler وأيضاً من سياقات أخرى |
| `port->x_char` | يُقرأ في interrupt handler |
| الـ xmit_fifo (عبر uart_state) | يُقرأ في TX interrupt ويُكتب من userspace |

#### متى يُؤخذ القفل؟

```
الحالة                          │ كيف يُؤخذ
────────────────────────────────┼──────────────────────────────────
interrupt handler               │ uart_port_lock() ← داخل IRQ
startup / shutdown              │ uart_port_lock_irqsave() ← disable IRQs
set_termios                     │ uart_port_lock_irqsave()
console write (normal)          │ uart_port_lock_irqsave()
console write (oops_in_progress)│ uart_port_trylock_irqsave() ← قد يفشل
poll_get/put_char (kgdb)        │ uart_port_lock_irqsave()
meson_uart_reset() في probe     │ لا يوجد قفل ← لا console بعد
```

#### لماذا `trylock` في الـ console؟

عند `oops_in_progress` (kernel panic)، قد يكون القفل محجوزاً بالفعل من IRQ handler معطوب. استخدام `trylock` يمنع الـ deadlock — إذا فشل، تكتب المعلومة بدون قفل وهذا أفضل من التجمد.

#### ترتيب القفل

في هذا الـ driver لا يوجد إلا قفل واحد فلا توجد مشكلة deadlock بين أقفال متعددة. الـ `port->lock` هو السيد الوحيد.

#### الـ `uart_port_lock` vs `spin_lock`

الـ kernel الحديث لف `spin_lock` في `uart_port_lock` لأن بعض consoles تستخدم `nbcon` (non-blocking console). في Meson UART:

```
uart_port_lock(port)
    ├─ spin_lock(&port->lock)
    └─ __uart_port_nbcon_acquire(port)  ← للـ nbcon console إذا وجد
```

هذا يضمن تنسيقاً صحيحاً حتى مع الـ consoles الجديدة.
## Phase 4: شرح الـ Functions

### جدول الـ Functions

| Function | Type | Purpose |
|----------|------|---------|
| `meson_uart_set_mctrl()` | static | ضبط إشارات التحكم بالـ modem — stub فاضية |
| `meson_uart_get_mctrl()` | static | إرجاع حالة إشارات الـ modem — دايمًا CTS |
| `meson_uart_tx_empty()` | static | فحص لو الـ TX FIFO فاضي |
| `meson_uart_stop_tx()` | static | إيقاف interrupt الإرسال |
| `meson_uart_stop_rx()` | static | تعطيل الاستقبال |
| `meson_uart_shutdown()` | static | إغلاق الـ port وتحرير الـ IRQ |
| `meson_uart_start_tx()` | static | بدء إرسال البيانات من الـ FIFO |
| `meson_receive_chars()` | static | استقبال الحروف من الـ RX FIFO ومعالجة الأخطاء |
| `meson_uart_interrupt()` | static | معالج الـ interrupt الرئيسي |
| `meson_uart_type()` | static | إرجاع اسم نوع الـ port |
| `meson_uart_reset()` | static | إعادة تهيئة الـ TX و RX FIFO |
| `meson_uart_startup()` | static | فتح الـ port وتفعيل الـ interrupts |
| `meson_uart_change_speed()` | static | ضبط الـ baud rate في الـ hardware |
| `meson_uart_set_termios()` | static | تطبيق إعدادات الـ terminal الجديدة |
| `meson_uart_verify_port()` | static | التحقق من صحة إعدادات الـ port |
| `meson_uart_release_port()` | static | تحرير الـ IO memory |
| `meson_uart_request_port()` | static | حجز الـ IO memory وعمل ioremap |
| `meson_uart_config_port()` | static | ضبط نوع الـ port |
| `meson_uart_poll_get_char()` | static | قراءة حرف واحد في وضع الـ polling |
| `meson_uart_poll_put_char()` | static | إرسال حرف واحد في وضع الـ polling |
| `meson_uart_enable_tx_engine()` | static | تفعيل محرك الإرسال للـ console |
| `meson_console_putchar()` | static | إرسال حرف واحد عبر الـ console |
| `meson_serial_port_write()` | static | كتابة سلسلة نصوص عبر الـ console مع قفل الـ port |
| `meson_serial_console_write()` | static | نقطة دخول الـ console لكتابة النصوص |
| `meson_serial_console_setup()` | static | إعداد الـ console وضبط المعاملات |
| `meson_serial_early_console_write()` | static | كتابة المبكرة قبل ما الـ driver يتهيأ |
| `meson_serial_early_console_setup()` | static `__init` | إعداد الـ earlycon |
| `meson_uart_probe_clocks()` | static | حجز وتفعيل الـ clocks الثلاثة للـ UART |
| `meson_uart_current()` | static | اختيار الـ uart_driver المناسب |
| `meson_uart_probe()` | static | الـ probe الرئيسي — تهيئة كل الـ port |
| `meson_uart_remove()` | static | إزالة الـ port وتنظيف الموارد |

---

### المجموعة الأولى: إشارات التحكم بالـ Modem (Modem Control)

الـ UART فيه مجموعة إشارات اسمها **modem control signals** زي RTS و CTS و DTR — هي أصلًا من زمن الـ modems القديمة. الـ Meson UART ما يدعمها بشكل كامل، فالـ driver يعمل stubbing أو إرجاع قيم ثابتة.

---

#### `meson_uart_set_mctrl()`

```c
static void meson_uart_set_mctrl(struct uart_port *port, unsigned int mctrl)
{
}
```

**شو يعمل:** لا شيء — جسم الدالة فاضي خالص. الـ Meson UART ما يدعم التحكم في إشارات الـ modem برمجيًا.

- **`port`** — مؤشر لـ struct uart_port — الـ port المطلوب ضبطه.
- **`mctrl`** — بيتات الحالة المطلوبة مثل `TIOCM_RTS` — يتجاهلها الـ driver.
- **Return:** لا شيء.
- **يُستدعى من:** الـ serial core عند فتح أو إغلاق الـ terminal.

---

#### `meson_uart_get_mctrl()`

```c
static unsigned int meson_uart_get_mctrl(struct uart_port *port)
{
    return TIOCM_CTS;
}
```

**شو يعمل:** يُرجع دايمًا `TIOCM_CTS` — يعني يقول للـ serial core "الخط جاهز للإرسال دايمًا". لأن الـ hardware ما يوفر قراءة حقيقية لإشارة الـ CTS، الـ driver يحل المشكلة بإرجاع قيمة ثابتة.

- **`port`** — الـ port المراد قراءة حالته.
- **Return:** `TIOCM_CTS` دايمًا.
- **Locking:** لا قفل — قصير وآمن.
- **يُستدعى من:** الـ serial core عند الحاجة لحالة الـ modem.

---

### المجموعة الثانية: التحكم في الإرسال والاستقبال (TX/RX Control)

هذه المجموعة مسؤولة عن بدء وإيقاف الإرسال والاستقبال على مستوى الـ hardware مباشرة.

---

#### `meson_uart_tx_empty()`

```c
static unsigned int meson_uart_tx_empty(struct uart_port *port)
{
    u32 val;
    val = readl(port->membase + AML_UART_STATUS);
    val &= (AML_UART_TX_EMPTY | AML_UART_XMIT_BUSY);
    return (val == AML_UART_TX_EMPTY) ? TIOCSER_TEMT : 0;
}
```

**شو يعمل:** يقرأ الـ status register ويتحقق من بيتين: `TX_EMPTY` و `XMIT_BUSY`. يُعتبر الـ FIFO فاضيًا فعلًا فقط لو `TX_EMPTY=1` و `XMIT_BUSY=0` في نفس الوقت — لأنه ممكن الـ FIFO يكون فاضي لكن الـ shift register ما زال يُرسل البيت الأخير.

- **`port`** — الـ uart port.
- **Return:** `TIOCSER_TEMT` لو فاضي كليًا، `0` لو ما زال يعمل.
- **Locking:** لا قفل — يُستدعى من سياق متنوع.
- **يُستدعى من:** الـ serial core وأيضًا من `meson_uart_change_speed()`.

---

#### `meson_uart_stop_tx()`

```c
static void meson_uart_stop_tx(struct uart_port *port)
{
    u32 val;
    val = readl(port->membase + AML_UART_CONTROL);
    val &= ~AML_UART_TX_INT_EN;
    writel(val, port->membase + AML_UART_CONTROL);
}
```

**شو يعمل:** يُلغي تفعيل **TX interrupt** فقط — يعني الـ hardware لن يطلب interrupt لما يكون الـ FIFO فاضيًا. هذا يوقف تدفق الإرسال بشكل غير مباشر لأن `meson_uart_start_tx()` يعتمد على هذا الـ interrupt.

- **`port`** — الـ uart port.
- **Return:** لا شيء.
- **Locking:** يُستدعى مع قفل الـ port محجوز مسبقًا.
- **يُستدعى من:** الـ serial core ومن `meson_uart_start_tx()`.

---

#### `meson_uart_stop_rx()`

```c
static void meson_uart_stop_rx(struct uart_port *port)
{
    u32 val;
    val = readl(port->membase + AML_UART_CONTROL);
    val &= ~AML_UART_RX_EN;
    writel(val, port->membase + AML_UART_CONTROL);
}
```

**شو يعمل:** يُعطّل الـ RX receiver كليًا عن طريق مسح بت `AML_UART_RX_EN` في الـ control register. بعد هذا الـ hardware لن يقبل أي بيانات واردة.

- **`port`** — الـ uart port.
- **Return:** لا شيء.
- **Locking:** يُستدعى مع قفل الـ port.
- **يُستدعى من:** الـ serial core عند إغلاق الـ port.

---

#### `meson_uart_start_tx()`

```c
static void meson_uart_start_tx(struct uart_port *port)
{
    struct tty_port *tport = &port->state->port;
    unsigned char ch;
    u32 val;
    ...
}
```

**شو يعمل:** هذه الدالة هي **محرك الإرسال الحقيقي**. تخيلها زي عامل ينقل صناديق من مستودع (الـ software FIFO) إلى شاحنة (الـ hardware TX FIFO) — كل ما الشاحنة عندها مكان، يحمّل صندوق جديد.

**تدفق العمل:**
```
1. لو الـ TX موقوف (uart_tx_stopped) → استدعي stop_tx() وارجع
2. loop: ما دام TX FIFO غير ممتلئ:
   a. لو فيه x_char (حرف عاجل) → ارسله أولًا
   b. خذ حرف من software FIFO بـ uart_fifo_get()
   c. اكتبه في AML_UART_WFIFO
3. لو ما زال في بيانات في الـ software FIFO → فعّل TX_INT_EN
4. لو البيانات أقل من WAKEUP_CHARS → أيقظ الـ process الكاتب
```

- **`port`** — الـ uart port.
- **Return:** لا شيء.
- **Locking:** يُستدعى مع قفل الـ port محجوز.
- **يُستدعى من:** `meson_uart_interrupt()` والـ serial core.
- **جانب مهم:** الـ `x_char` له أولوية على البيانات العادية — يُستخدم لـ XON/XOFF flow control.

---

### المجموعة الثالثة: الاستقبال ومعالجة الأخطاء (RX & Error Handling)

---

#### `meson_receive_chars()`

```c
static void meson_receive_chars(struct uart_port *port)
{
    struct tty_port *tport = &port->state->port;
    char flag;
    u32 ostatus, status, ch, mode;
    ...
}
```

**شو يعمل:** هذه هي دالة **قراءة البيانات الواردة**. تعمل في حلقة تقرأ من الـ `AML_UART_RFIFO` حتى يصبح فارغًا، وفي كل تكرار تفحص الأخطاء وتُعلّم كل حرف بنوعه قبل إرساله للـ TTY layer.

**تدفق العمل بالتفصيل:**
```
do {
    1. اقرأ STATUS register
    2. لو فيه خطأ (PARITY/FRAME/OVERRUN):
       a. زِد عداد الأخطاء في port->icount
       b. اكتب CLEAR_ERR ثم امسحه (لأن الـ hardware ما يمسحه تلقائيًا)
       c. حدد flag = TTY_FRAME أو TTY_PARITY
    3. اقرأ الحرف من AML_UART_RFIFO
    4. لو FRAME_ERR + حرف = 0x00 → هذا BREAK signal
    5. لو الحرف من الـ sysrq → تجاهله (للـ debugger)
    6. لو مش في ignore_status_mask → أرسل الحرف للـ TTY بـ tty_insert_flip_char()
    7. لو OVERRUN → أرسل TTY_OVERRUN للـ TTY
} while (RX FIFO غير فارغ)

8. ادفع البيانات للـ TTY بـ tty_flip_buffer_push()
```

**فهم الـ flags:**
| flag | معناه |
|------|--------|
| `TTY_NORMAL` | حرف عادي بدون مشاكل |
| `TTY_FRAME` | خطأ في إطار البيانات |
| `TTY_PARITY` | خطأ في الـ parity check |
| `TTY_BREAK` | إشارة break — توقف البث |
| `TTY_OVERRUN` | الـ buffer امتلأ وضاع حرف |

- **`port`** — الـ uart port.
- **Return:** لا شيء.
- **Locking:** يُستدعى مع قفل الـ port محجوز من `meson_uart_interrupt()`.
- **جانب مهم:** الـ `CLEAR_ERR` bit يحتاج كتابة يدوية لمسحه — الـ hardware ما يمسحه تلقائيًا.

---

#### `meson_uart_interrupt()`

```c
static irqreturn_t meson_uart_interrupt(int irq, void *dev_id)
{
    struct uart_port *port = (struct uart_port *)dev_id;
    uart_port_lock(port);
    ...
    uart_unlock_and_check_sysrq(port);
    return IRQ_HANDLED;
}
```

**شو يعمل:** هو **قلب الـ driver** — يُستدعى كل ما الـ hardware يطلب attention. تخيله زي مدير يفحص صندوق الوارد والصادر في نفس الوقت.

**تدفق العمل:**
```
1. احجز قفل الـ port (uart_port_lock)
2. لو RX FIFO غير فارغ → استدعي meson_receive_chars()
3. لو TX FIFO غير ممتلئ وكان TX_INT_EN مفعّل → استدعي meson_uart_start_tx()
4. أطلق القفل وافحص sysrq (uart_unlock_and_check_sysrq)
5. أرجع IRQ_HANDLED
```

- **`irq`** — رقم الـ interrupt.
- **`dev_id`** — مؤشر للـ `uart_port` (يُمرر عند `request_irq`).
- **Return:** `IRQ_HANDLED` دايمًا.
- **Locking:** يحجز ويُحرر `port->lock` بنفسه.
- **يُستدعى من:** kernel interrupt subsystem فقط.

---

### المجموعة الرابعة: التهيئة والإغلاق (Startup & Shutdown)

---

#### `meson_uart_reset()`

```c
static void meson_uart_reset(struct uart_port *port)
{
    u32 val;
    val = readl(port->membase + AML_UART_CONTROL);
    val |= (AML_UART_RX_RST | AML_UART_TX_RST | AML_UART_CLEAR_ERR);
    writel(val, port->membase + AML_UART_CONTROL);
    val &= ~(AML_UART_RX_RST | AML_UART_TX_RST | AML_UART_CLEAR_ERR);
    writel(val, port->membase + AML_UART_CONTROL);
}
```

**شو يعمل:** يُعيد تهيئة الـ TX و RX FIFOs عن طريق وضع بيتات الـ reset ثم مسحها — زي ما تضغط زر reset على جهاز، اضغط ثم اتركه. هذا يضمن أن الـ FIFOs فارغة ونظيفة عند بداية الـ driver.

- **`port`** — الـ uart port — يجب أن يكون `port->membase` صالحًا.
- **Return:** لا شيء.
- **Locking:** لا قفل — يُستدعى فقط من `meson_uart_probe()` قبل `request_irq`.
- **جانب مهم:** التعليق في الكود يشرح لماذا لا يوجد قفل — لأن الـ port لم يُسجَّل بعد ولا يوجد console عليه وقتها.

---

#### `meson_uart_startup()`

```c
static int meson_uart_startup(struct uart_port *port)
{
    unsigned long flags;
    u32 val;
    int ret = 0;
    ...
}
```

**شو يعمل:** يُفتح الـ port للاستخدام. يُعادل فتح الحنفية — يمسح الأخطاء، يُشغّل الـ TX و RX، يُفعّل الـ interrupts، ثم يطلب الـ IRQ من الـ kernel.

**تدفق العمل:**
```
1. احجز القفل (uart_port_lock_irqsave)
2. مسح أخطاء قديمة: اكتب CLEAR_ERR ثم امسحه
3. فعّل RX_EN و TX_EN
4. فعّل RX_INT_EN و TX_INT_EN
5. اضبط عتبات الـ interrupt:
   - RX: أرسل interrupt عند استقبال حرف واحد على الأقل
   - TX: أرسل interrupt عندما يكون الـ FIFO نصفه فارغ
6. أطلق القفل
7. request_irq() — اربط meson_uart_interrupt بالـ IRQ
```

- **`port`** — الـ uart port.
- **Return:** `0` للنجاح، أو رمز خطأ من `request_irq()`.
- **Locking:** يستخدم `uart_port_lock_irqsave` للجزء الأول.
- **يُستدعى من:** الـ serial core عند أول `open()` على الـ terminal.

---

#### `meson_uart_shutdown()`

```c
static void meson_uart_shutdown(struct uart_port *port)
{
    unsigned long flags;
    u32 val;

    free_irq(port->irq, port);

    uart_port_lock_irqsave(port, &flags);
    val = readl(port->membase + AML_UART_CONTROL);
    val &= ~AML_UART_RX_EN;
    val &= ~(AML_UART_RX_INT_EN | AML_UART_TX_INT_EN);
    writel(val, port->membase + AML_UART_CONTROL);
    uart_port_unlock_irqrestore(port, flags);
}
```

**شو يعمل:** عكس `meson_uart_startup()` — يُغلق الـ port ويُحرر الموارد. يُحرر الـ IRQ **أولًا** قبل القفل لأنه لو يفعل العكس قد يحدث deadlock مع الـ interrupt handler الذي يحاول حجز نفس القفل.

- **`port`** — الـ uart port.
- **Return:** لا شيء.
- **Locking:** يستخدم `uart_port_lock_irqsave` بعد `free_irq`.
- **جانب مهم:** ترتيب العمليات مهم جدًا — `free_irq` أولًا دايمًا.
- **يُستدعى من:** الـ serial core عند آخر `close()` على الـ terminal.

---

### المجموعة الخامسة: ضبط الـ Baud Rate والـ Termios

---

#### `meson_uart_change_speed()`

```c
static void meson_uart_change_speed(struct uart_port *port, unsigned long baud)
{
    const struct meson_uart_data *private_data = port->private_data;
    u32 val = 0;
    ...
}
```

**شو يعمل:** يحسب قيمة الـ divider ويكتبها في الـ `AML_UART_REG5`. الـ baud rate يُحسب كالتالي:

**معادلة الـ Baud Rate:**

للـ crystal clock (24 MHz):
```
divider = ROUND(uartclk / xtal_div / baud) - 1
```
حيث `xtal_div = 2` للـ G12A وما بعدها، و `3` للأجيال الأقدم.

للـ PLL clock:
```
divider = ROUND(uartclk / 4 / baud) - 1
```

**تدفق العمل:**
```
1. انتظر حتى يفرغ الـ TX FIFO (polling على tx_empty)
2. لو uartclk == 24MHz (crystal):
   a. لو has_xtal_div2 → استخدم div2 وفعّل BAUD_XTAL_DIV2
   b. احسب divider = ROUND(24M / xtal_div / baud) - 1
   c. فعّل BAUD_XTAL
3. لو PLL clock:
   احسب divider = ROUND(uartclk / 4 / baud) - 1
4. فعّل BAUD_USE وأكتب القيمة في REG5
```

- **`port`** — الـ uart port.
- **`baud`** — سرعة الإرسال المطلوبة بالـ bps.
- **Return:** لا شيء.
- **Locking:** لا قفل مباشر — يُستدعى من `meson_uart_set_termios()` الذي يحمل القفل.

---

#### `meson_uart_set_termios()`

```c
static void meson_uart_set_termios(struct uart_port *port,
                                   struct ktermios *termios,
                                   const struct ktermios *old)
```

**شو يعمل:** هذه هي الدالة الأكبر في المجموعة — تُطبق **كل إعدادات الـ serial port** مرة واحدة: طول الكلمة، الـ parity، stop bits، الـ flow control، والـ baud rate.

**تدفق العمل:**
```
1. احجز القفل (uart_port_lock_irqsave)
2. اقرأ control register الحالي
3. اضبط data length حسب CSIZE (5/6/7/8 bits)
4. اضبط parity:
   - PARENB → فعّل PARITY_EN
   - PARODD → فعّل PARITY_TYPE (odd) أو امسحه (even)
5. اضبط stop bits:
   - CSTOPB → 2 stop bits
   - عكسه → 1 stop bit
6. اضبط flow control:
   - CRTSCTS + UPF_HARD_FLOW → امسح TWO_WIRE_EN (يعني فعّل RTS/CTS)
   - عكسه → فعّل TWO_WIRE_EN (2-wire mode)
7. اكتب الإعدادات في control register
8. احسب الـ baud rate المناسب بين 50 و 4000000 bps
9. استدعي meson_uart_change_speed()
10. حدّث read_status_mask و ignore_status_mask
11. استدعي uart_update_timeout()
12. أطلق القفل
```

**الـ status masks:**
- **`read_status_mask`** — الأخطاء التي يهتم بها الـ driver ويُبلغ عنها للـ TTY.
- **`ignore_status_mask`** — الأخطاء التي تُتجاهل تمامًا.

- **`port`** — الـ uart port.
- **`termios`** — الإعدادات الجديدة المطلوبة.
- **`old`** — الإعدادات القديمة للمقارنة.
- **Return:** لا شيء.
- **Locking:** يحجز `uart_port_lock_irqsave` طوال العملية.
- **يُستدعى من:** الـ serial core عند تغيير أي إعداد بـ `tcsetattr()`.

---

### المجموعة السادسة: إدارة الـ Port (Port Management)

---

#### `meson_uart_type()`

```c
static const char *meson_uart_type(struct uart_port *port)
{
    return (port->type == PORT_MESON) ? "meson_uart" : NULL;
}
```

**شو يعمل:** يُرجع اسم نوع الـ port كـ string — يظهر في `/proc/tty/driver/` وفي رسائل الـ kernel.

- **Return:** `"meson_uart"` أو `NULL`.
- **يُستدعى من:** الـ serial core لأغراض التعريف.

---

#### `meson_uart_verify_port()`

```c
static int meson_uart_verify_port(struct uart_port *port,
                                  struct serial_struct *ser)
```

**شو يعمل:** يتحقق من أن الإعدادات المطلوبة منطقية وصحيحة. يُستدعى عند `ioctl(TIOCSSERIAL)`.

- **`port`** — الـ uart port.
- **`ser`** — الإعدادات الجديدة المقترحة.
- **Return:** `0` للنجاح، `-EINVAL` لو:
  - نوع الـ port غلط.
  - رقم الـ IRQ مختلف.
  - الـ baud_base أقل من 9600.

---

#### `meson_uart_request_port()`

```c
static int meson_uart_request_port(struct uart_port *port)
{
    if (!devm_request_mem_region(...))
        return -EBUSY;
    port->membase = devm_ioremap(...);
    if (!port->membase)
        return -ENOMEM;
    return 0;
}
```

**شو يعمل:** يحجز منطقة الـ IO memory ويعمل **ioremap** عليها — أي يربط العنوان الفيزيائي للـ UART registers بعنوان virtual في الـ kernel memory space. بدون هذا لا يمكن قراءة أو كتابة الـ registers.

- **`port`** — الـ uart port، يحمل `port->mapbase` و `port->mapsize`.
- **Return:** `0`، أو `-EBUSY` لو منطقة الـ memory محجوزة، أو `-ENOMEM` لو فشل الـ ioremap.
- **يُستدعى من:** `meson_uart_config_port()` و `meson_uart_probe()`.

---

#### `meson_uart_release_port()`

```c
static void meson_uart_release_port(struct uart_port *port)
{
    devm_iounmap(port->dev, port->membase);
    port->membase = NULL;
    devm_release_mem_region(port->dev, port->mapbase, port->mapsize);
}
```

**شو يعمل:** عكس `meson_uart_request_port()` — يُلغي الـ ioremap ويُحرر حجز منطقة الـ memory. يضع `port->membase = NULL` لمنع أي استخدام بعد التحرير.

- **`port`** — الـ uart port.
- **Return:** لا شيء.
- **يُستدعى من:** الـ serial core وأيضًا من `meson_uart_probe()` بعد الـ reset.

---

#### `meson_uart_config_port()`

```c
static void meson_uart_config_port(struct uart_port *port, int flags)
{
    if (flags & UART_CONFIG_TYPE) {
        port->type = PORT_MESON;
        meson_uart_request_port(port);
    }
}
```

**شو يعمل:** يُضبط نوع الـ port على `PORT_MESON` ويستدعي `meson_uart_request_port()` لعمل الـ ioremap. يُستدعى من الـ serial core عند الـ auto-configuration.

- **`port`** — الـ uart port.
- **`flags`** — عادةً `UART_CONFIG_TYPE`.
- **Return:** لا شيء.

---

### المجموعة السابعة: وضع الـ Polling (للـ kgdb)

هذه الدوال تعمل فقط لو كان `CONFIG_CONSOLE_POLL` مُفعّلًا في الـ kernel. تُستخدم من **kgdb** عند debugging — لأن الـ debugger يحتاج I/O بدون interrupts.

---

#### `meson_uart_poll_get_char()`

```c
static int meson_uart_poll_get_char(struct uart_port *port)
```

**شو يعمل:** يقرأ حرفًا واحدًا بطريقة **polling** — يفحص مباشرة إن كان فيه شيء في الـ RX FIFO.

- **`port`** — الـ uart port.
- **Return:** الحرف المستقبَل (0-255)، أو `NO_POLL_CHAR` لو الـ FIFO فارغ.
- **Locking:** يستخدم `uart_port_lock_irqsave` لأن الـ IRQs قد تكون مفعّلة.

---

#### `meson_uart_poll_put_char()`

```c
static void meson_uart_poll_put_char(struct uart_port *port, unsigned char c)
```

**شو يعمل:** يُرسل حرفًا واحدًا بطريقة **polling** — ينتظر الـ TX FIFO أن يفرغ، يكتب الحرف، ثم ينتظر مرة أخرى حتى يُرسل.

**تدفق العمل:**
```
1. احجز القفل
2. انتظر TX_EMPTY بـ readl_poll_timeout_atomic (max 10ms)
3. لو timeout → اطبع خطأ وانتهِ
4. اكتب الحرف في WFIFO
5. انتظر TX_EMPTY مرة أخرى (تأكد أن الإرسال اكتمل)
6. أطلق القفل
```

- **`port`** — الـ uart port.
- **`c`** — الحرف المراد إرساله.
- **Return:** لا شيء.
- **Timeout:** `AML_UART_POLL_USEC = 5µs` polling interval، `AML_UART_TIMEOUT_USEC = 10000µs` = 10ms حد أقصى.

---

### المجموعة الثامنة: الـ Console

هذه الدوال تعمل فقط مع `CONFIG_SERIAL_MESON_CONSOLE`.

---

#### `meson_uart_enable_tx_engine()`

```c
static void meson_uart_enable_tx_engine(struct uart_port *port)
{
    u32 val;
    val = readl(port->membase + AML_UART_CONTROL);
    val |= AML_UART_TX_EN;
    writel(val, port->membase + AML_UART_CONTROL);
}
```

**شو يعمل:** يُفعّل TX فقط — يُستخدم للـ console قبل ما يكون الـ port مفتوحًا كاملًا. ما يحتاج RX في هذه المرحلة.

---

#### `meson_console_putchar()`

```c
static void meson_console_putchar(struct uart_port *port, unsigned char ch)
```

**شو يعمل:** يُرسل حرفًا واحدًا للـ console بـ **busy wait** — ينتظر حتى يكون الـ TX FIFO غير ممتلئ ثم يكتب الحرف. تخيله زي شخص يقف قدام الطابعة وينتظر.

- **يُستدعى من:** `meson_serial_port_write()` عبر `uart_console_write()`.

---

#### `meson_serial_port_write()`

```c
static void meson_serial_port_write(struct uart_port *port, const char *s,
                                    u_int count)
```

**شو يعمل:** يكتب سلسلة نصية في الـ console مع إدارة حذرة للقفل. إن كان هناك **oops** (kernel crash) جارٍ، يستخدم `trylock` بدل `lock` لتجنب الـ deadlock.

**تدفق العمل:**
```
1. لو oops_in_progress → trylock (ربما يفشل)
   عكسه → lock عادي
2. عطّل TX_INT_EN و RX_INT_EN مؤقتًا
3. استدعي uart_console_write() مع meson_console_putchar
4. استعد الإعدادات الأصلية (أعد تفعيل الـ interrupts)
5. لو كان القفل محجوزًا → أطلقه
```

- **جانب مهم:** تعطيل الـ interrupts أثناء الكتابة يمنع تداخل الـ interrupt handler مع الـ console output.

---

#### `meson_serial_console_write()`

```c
static void meson_serial_console_write(struct console *co, const char *s,
                                       u_int count)
```

**شو يعمل:** نقطة الدخول للـ console — يبحث عن الـ `uart_port` من `meson_ports[]` ثم يُفوّض للـ `meson_serial_port_write()`.

- **`co`** — الـ console struct، `co->index` يُعطي رقم الـ port.
- **`s`** — النص المراد طباعته.
- **`count`** — عدد الحروف.

---

#### `meson_serial_console_setup()`

```c
static int meson_serial_console_setup(struct console *co, char *options)
```

**شو يعمل:** يُعِدّ الـ console عند تسجيله لأول مرة. يُعيّن baud rate وباقي الإعدادات.

**تدفق العمل:**
```
1. تحقق أن index صالح (0 ≤ index < AML_UART_PORT_NUM)
2. تحقق أن الـ port موجود وله membase
3. فعّل TX engine
4. لو options موجودة → parse them بـ uart_parse_options()
5. افرض الإعدادات بـ uart_set_options() (default: 115200/8N1)
```

- **`co`** — الـ console struct.
- **`options`** — سلسلة من نوع `"115200n8"` أو NULL للـ defaults.
- **Return:** `0` للنجاح، أو رمز خطأ.

---

#### `meson_serial_early_console_setup()`

```c
static int __init
meson_serial_early_console_setup(struct earlycon_device *device, const char *opt)
```

**شو يعمل:** يُعِدّ الـ **earlycon** — أي الـ console الذي يعمل في المراحل الأولى جدًا من boot قبل ما أي driver يُحمَّل. يُسجَّل عبر `OF_EARLYCON_DECLARE` لنوعين من الـ hardware.

- **`device`** — الـ earlycon device، يحمل `device->port` و `device->con`.
- **`opt`** — options اختيارية.
- **Return:** `0` للنجاح، `-ENODEV` لو `membase` غير صالح.
- **الـ `__init`:** تعني أن الكود يُحذف من الذاكرة بعد انتهاء الـ boot.

---

### المجموعة التاسعة: الـ Probe والـ Platform Driver

---

#### `meson_uart_probe_clocks()`

```c
static int meson_uart_probe_clocks(struct platform_device *pdev,
                                   struct uart_port *port)
```

**شو يعمل:** يحجز ويُفعّل الـ clocks الثلاثة اللازمة للـ UART:

| Clock | الاسم | الغرض |
|-------|-------|--------|
| `clk_pclk` | `"pclk"` | peripheral clock — clock الـ bus |
| `clk_xtal` | `"xtal"` | crystal oscillator — 24 MHz |
| `clk_baud` | `"baud"` | clock الـ baud rate — المرجع للحسابات |

يستخدم `devm_clk_get_enabled()` الذي يحجز الـ clock **ويُفعّله** في خطوة واحدة، وتُحرر تلقائيًا عند حذف الـ device.

- **`pdev`** — الـ platform device.
- **`port`** — يُحدّث `port->uartclk` بتردد `clk_baud`.
- **Return:** `0` للنجاح، رمز خطأ PTR_ERR لو أي clock فشل.

---

#### `meson_uart_current()`

```c
static struct uart_driver *meson_uart_current(const struct meson_uart_data *pd)
{
    return (pd && pd->uart_driver) ?
        pd->uart_driver : &meson_uart_driver_ttyAML;
}
```

**شو يعمل:** يختار الـ `uart_driver` المناسب. الـ Meson UART يدعم نوعين من أسماء الـ device:
- **`ttyAML`** — الأجهزة القديمة (Meson6, Meson8, GX, G12A).
- **`ttyS`** — الأجهزة الحديثة (A1, S4).

لو الـ private_data فيها `uart_driver` محدد، يستخدمه. عكسه يرجع للـ ttyAML كـ default.

---

#### `meson_uart_probe()`

```c
static int meson_uart_probe(struct platform_device *pdev)
```

**شو يعمل:** هذه أكبر وأهم دالة في الـ driver — يُستدعى لكل UART controller يجده الـ kernel في الـ device tree. تخيلها زي توصيل جهاز جديد والـ OS يُعِدّه.

**تدفق العمل التفصيلي:**
```
1. استخرج ID الـ port:
   a. حاول من device tree alias (serial0, serial1...)
   b. لو ما وُجد → ابحث عن أول slot فاضي من AML_UART_PORT_OFFSET

2. تحقق من صلاحية الـ ID (0 ≤ id < AML_UART_PORT_NUM)

3. احصل على resources:
   - res_mem → IORESOURCE_MEM (عنوان الـ registers)
   - irq → رقم الـ interrupt

4. اقرأ properties من device tree:
   - fifo-size (default: 64 bytes)
   - uart-has-rtscts (boolean)

5. تحقق أن الـ port غير محجوز مسبقًا

6. خصص struct uart_port بـ devm_kzalloc

7. استدعي meson_uart_probe_clocks() لتفعيل الـ clocks

8. احصل على match_data للتعرف على نوع الـ hardware

9. سجّل الـ uart_driver لو لم يُسجَّل بعد (uart_register_driver)

10. اضبط الـ port fields:
    - iotype = UPIO_MEM
    - mapbase, mapsize, irq, flags
    - line, type, ops, fifosize, private_data

11. سجّل الـ port في meson_ports[]
12. احفظ الـ port في pdev->driver_data

13. عمل reset آمن:
    request_port → reset → release_port

14. سجّل الـ port في الـ serial core (uart_add_one_port)
```

- **`pdev`** — الـ platform device من الـ device tree.
- **Return:** `0` للنجاح، رمز خطأ سلبي للفشل.
- **الـ `devm_` prefix:** يعني أن الموارد تُحرر تلقائيًا عند `remove()` — لا حاجة لتنظيف يدوي.

---

#### `meson_uart_remove()`

```c
static void meson_uart_remove(struct platform_device *pdev)
```

**شو يعمل:** عكس `meson_uart_probe()` — يُزيل الـ port ويُنظف. له تفصيل ذكي: لو هذا آخر port متبقٍ، يُلغي تسجيل الـ uart_driver كله.

**تدفق العمل:**
```
1. احصل على port من driver_data
2. احصل على uart_driver المناسب
3. أزل الـ port بـ uart_remove_one_port()
4. امسح meson_ports[pdev->id]
5. افحص كل الـ ports:
   - لو في port ما زال موجود → ارجع (لا تُلغِ الـ driver)
6. لو كل الـ ports اختفت → uart_unregister_driver()
```

- **`pdev`** — الـ platform device.
- **Return:** لا شيء (void منذ kernel 6.11).
- **جانب مهم:** التحقق من باقي الـ ports قبل `unregister_driver` ضروري لأن driver واحد مشترك بين عدة ports.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs المتعلقة بـ meson_uart

الـ `serial_core` بيوفر مدخلات في `/sys/kernel/debug/` للبورتات المسجّلة. مافيش debugfs خاص بـ `meson_uart` نفسه، بس تقدر تستخدم:

```bash
# شوف كل بورتات الـ serial المسجلة
ls /sys/kernel/debug/

# لو فعّلت CONFIG_SERIAL_CORE_DEBUG (مش موجود افتراضيًا)
# بتلاقي إحصاءات الـ UART هنا:
cat /sys/kernel/debug/ttyAML0
cat /sys/kernel/debug/ttyS0
```

> الـ meson_uart مش بيسجّل مدخلات debugfs خاصة بيه، بس الـ `uart_core` بيعرض إحصاءات عبر `/proc/tty/driver/`.

---

#### 2. مدخلات الـ sysfs المفيدة

```bash
# معلومات البورت الأساسية (استبدل ttyAML0 بالبورت المطلوب)
cat /sys/class/tty/ttyAML0/uartclk        # تردد الكلوك بالـ Hz
cat /sys/class/tty/ttyAML0/type           # نوع البورت (PORT_MESON = 35)
cat /sys/class/tty/ttyAML0/line           # رقم الخط
cat /sys/class/tty/ttyAML0/irq            # رقم الـ IRQ
cat /sys/class/tty/ttyAML0/flags          # فلاجات UPF_*
cat /sys/class/tty/ttyAML0/xmit_fifo_size # حجم الـ FIFO

# إحصاءات الاستقبال/الإرسال (icount)
cat /proc/tty/driver/meson_uart           # ملخص لكل البورتات
# أو
cat /proc/tty/driver/ttyAML
cat /proc/tty/driver/ttyS

# معلومات الكلوكات عبر clk framework
ls /sys/kernel/debug/clk/
# ابحث عن كلوك الـ baud الخاص بالـ UART:
cat /sys/kernel/debug/clk/uart_clkc_*/clk_rate
```

**قراءة /proc/tty/driver/meson_uart:**
```
serinfo:1.0 driver revision:
0: uart:meson_uart mmio:0xFF803000 irq:58 tx:1024 rx:2048 CTS|DSR|CD|RI
```
- **tx/rx**: عدد البايتات المرسلة/المستقبلة — لو الأرقام مش بتتغير = مشكلة في الـ interrupt أو الـ FIFO.

---

#### 3. الـ ftrace — Tracepoints وأحداث الـ UART

```bash
# فعّل الـ tracing للـ serial core
cd /sys/kernel/debug/tracing

# فعّل كل أحداث الـ tty
echo 1 > events/tty/enable

# أو أحداث محددة
echo 1 > events/tty/tty_insert_flip_string/enable
echo 1 > events/tty/tty_flip_buffer_push/enable

# تتبع دوال الـ meson_uart بالاسم (function_graph tracer)
echo function_graph > current_tracer
echo 'meson_uart*' > set_ftrace_filter
echo 1 > tracing_on
# ... شغّل العملية اللي تريد تتبعها ...
cat trace | head -100
echo 0 > tracing_on

# تتبع الـ IRQ handler تحديدًا
echo 'meson_uart_interrupt' > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
```

**مثال على output الـ ftrace:**
```
 1)               |  meson_uart_interrupt() {
 1)               |    meson_receive_chars() {
 1)   0.800 us    |      tty_insert_flip_char();
 1)   1.200 us    |      tty_flip_buffer_push();
 1)   2.500 us    |    }
 1)   3.100 us    |  }
```
لو `meson_uart_interrupt` مش بتظهر = مشكلة في الـ IRQ لم يُسجَّل أو لم يُطلَق.

---

#### 4. الـ printk والـ dynamic debug

```bash
# فعّل الـ dynamic debug لـ meson_uart بالكامل
echo 'module meson_uart +p' > /sys/kernel/debug/dynamic_debug/control

# أو لملف محدد
echo 'file drivers/tty/serial/meson_uart.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع معلومات إضافية (اسم الدالة + رقم السطر)
echo 'file drivers/tty/serial/meson_uart.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شاهد الناتج
dmesg -w | grep meson
journalctl -k -f | grep meson

# زيادة مستوى الـ log للـ tty subsystem
echo 8 > /proc/sys/kernel/printk
```

**نقاط printk مفيدة يمكن إضافتها يدويًا أثناء الـ kernel development:**
```c
/* في meson_receive_chars() لتشخيص أخطاء الاستقبال */
if (status & AML_UART_ERR)
    dev_dbg(port->dev, "RX error: status=0x%08x\n", status);

/* في meson_uart_change_speed() لتتبع الـ baud rate */
dev_dbg(port->dev, "baud=%lu uartclk=%u REG5=0x%08x\n",
        baud, port->uartclk, val);

/* في meson_uart_startup() لتأكيد التهيئة */
dev_dbg(port->dev, "CONTROL=0x%08x MISC=0x%08x\n",
        readl(port->membase + AML_UART_CONTROL),
        readl(port->membase + AML_UART_MISC));
```

---

#### 5. خيارات الـ kernel config للـ Debugging

| Config | الوصف |
|--------|-------|
| `CONFIG_SERIAL_MESON` | تفعيل الدريفر نفسه |
| `CONFIG_SERIAL_MESON_CONSOLE` | دعم الـ console على meson UART |
| `CONFIG_CONSOLE_POLL` | دعم الـ kgdb polling |
| `CONFIG_SERIAL_CORE_CONSOLE` | دعم console عام للـ serial |
| `CONFIG_DEBUG_LL` | early lowlevel debug output |
| `CONFIG_EARLY_PRINTK` | طباعة مبكرة قبل بدء الـ console |
| `CONFIG_EARLYCON` | earlycon framework (OF_EARLYCON_DECLARE) |
| `CONFIG_TTY` | الـ TTY layer الأساسي |
| `CONFIG_MAGIC_SYSRQ_SERIAL` | SysRq عبر serial |
| `CONFIG_DYNAMIC_DEBUG` | dynamic debug بالكامل |
| `CONFIG_SERIAL_8250_CONSOLE` | مقارنة مع دريفر آخر لعزل المشكلة |

```bash
# تحقق مما هو مفعّل في الـ kernel الحالي
zcat /proc/config.gz | grep -E 'SERIAL_MESON|EARLYCON|DEBUG_LL'
# أو
grep -E 'SERIAL_MESON|EARLYCON' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ subsystem

**أدوات الـ serial subsystem:**
```bash
# stty - فحص وضبط إعدادات البورت
stty -a -F /dev/ttyAML0
# المخرج المتوقع:
# speed 115200 baud; rows 0; columns 0;
# intr = ^C; ...
# -parenb -parodd cs8 -hupcl -cstopb cread clocal -crtscts

# setserial - معلومات تفصيلية
setserial -g /dev/ttyAML0
# /dev/ttyAML0, UART: unknown, Port: 0x0000, IRQ: 58

# minicom أو picocom للاختبار التفاعلي
picocom -b 115200 /dev/ttyAML0

# فحص الـ IRQ stats
watch -n1 'cat /proc/interrupts | grep -E "meson|ttyAML|ttyS"'

# lsof لمعرفة من يستخدم البورت
lsof /dev/ttyAML0

# فحص حالة الـ port عبر ioctl
python3 -c "
import serial, termios, fcntl
s = serial.Serial('/dev/ttyAML0', 115200)
print('in_waiting:', s.in_waiting)
print('out_waiting:', s.out_waiting)
s.close()
"
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|----------------------|--------|-------|
| `meson_uart: port N already allocated` | البورت رقم N مستخدم بالفعل — تكرار probe | تحقق من الـ Device Tree، تأكد من عدم تكرار `alias serial<N>` |
| `Memory region busy` | عنوان الذاكرة `0xFFxxxxx` محجوز مسبقًا | تحقق من `/proc/iomem`، تأكد من عدم تداخل الـ DT nodes |
| `can't register uart driver` | فشل `uart_register_driver()` — ربما نفد الذاكرة | `dmesg | grep -i memory`، تحقق من `free -m` |
| `Timeout waiting for UART TX EMPTY` | الـ TX FIFO ممتلئ لأكثر من 10ms أثناء الـ poll | مشكلة hardware — تحقق من الـ baud rate وكلوك الـ UART |
| `request_irq failed` | فشل تسجيل الـ IRQ handler | تحقق من `/proc/interrupts` وتعارض الـ IRQ في الـ DT |
| `pclk` / `xtal` / `baud` clock errors | عدم إيجاد الكلوكات في الـ DT | تحقق من `clk-names` في الـ DT node |
| `no membase` (console setup fails) | `port->membase` = NULL أثناء console setup | الـ ioremap فشل أو الـ port غير مُهيأ بعد |

---

#### 8. نقاط إستراتيجية لـ dump_stack() و WARN_ON()

```c
/* في meson_uart_probe() - التحقق من صحة الـ IRQ */
WARN_ON(irq < 0);

/* في meson_uart_startup() - التحقق من الـ membase */
WARN_ON(!port->membase);

/* في meson_receive_chars() - كشف buffer overflow غير متوقع */
WARN_ON_ONCE(status & AML_UART_TX_FIFO_WERR);

/* في meson_uart_change_speed() - قيمة REG5 صفر تعني خطأ في الـ baud */
if (unlikely(val == AML_UART_BAUD_USE)) {
    dev_warn(port->dev, "baud divisor is 0!\n");
    dump_stack();
}

/* في meson_uart_interrupt() - التحقق من صحة الـ port pointer */
WARN_ON(!port || !port->membase);
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ Hardware مع حالة الـ Kernel

```bash
# رقم 1: تأكد أن الـ UART clock فعلاً يعمل
cat /sys/kernel/debug/clk/uart_clkc_*/clk_rate
# المتوقع لـ xtal path: 24000000 Hz
# لو صفر = الكلوك متوقف

# رقم 2: تأكد من عنوان الـ MMIO الصحيح
cat /proc/iomem | grep -i uart
# مثال:
# ff803000-ff80301f : ff803000.serial

# رقم 3: قارن قيم الـ registers الحالية مع ما يتوقعه الـ kernel
# (انظر قسم register dump أدناه)

# رقم 4: تحقق من الـ IRQ
cat /proc/interrupts | grep -i meson
# يجب أن يزداد العداد عند كل بايت مستقبَل
```

---

#### 2. تقنيات الـ Register Dump

الـ meson_uart له 6 registers بدءًا من العنوان الأساسي (مثلاً `0xFF803000`):

| Offset | الاسم | القيمة الافتراضية بعد reset |
|--------|-------|---------------------------|
| 0x00 | WFIFO | write-only |
| 0x04 | RFIFO | read-only |
| 0x08 | CONTROL | — |
| 0x0C | STATUS | — |
| 0x10 | MISC | — |
| 0x14 | REG5 | — |

```bash
# استخدام devmem2 (لازم تثبّته: apt install devmem2 أو busybox devmem)
BASE=0xff803000

devmem2 $((BASE + 0x08)) w   # CONTROL register
devmem2 $((BASE + 0x0c)) w   # STATUS register
devmem2 $((BASE + 0x10)) w   # MISC register
devmem2 $((BASE + 0x14)) w   # REG5 (baud)

# أو عبر /dev/mem (تحتاج CONFIG_DEVMEM=y)
python3 << 'EOF'
import mmap, struct

BASE = 0xff803000
REGS = {
    'CONTROL': 0x08,
    'STATUS':  0x0c,
    'MISC':    0x10,
    'REG5':    0x14,
}

with open('/dev/mem', 'rb') as f:
    mem = mmap.mmap(f.fileno(), 0x20, offset=BASE)
    for name, off in REGS.items():
        mem.seek(off)
        val = struct.unpack('<I', mem.read(4))[0]
        print(f"{name} (0x{BASE+off:08x}): 0x{val:08x}")
    mem.close()
EOF

# استخدام io utility (من package i2c-tools أو gpio-tools)
io -4 -r 0xff803008   # CONTROL
io -4 -r 0xff80300c   # STATUS
```

**تفسير STATUS register (0x0C):**
```
Bit 16 = PARITY_ERR    → خطأ في الـ parity
Bit 17 = FRAME_ERR     → خطأ في الـ framing (baud rate خاطئ؟)
Bit 18 = TX_FIFO_WERR  → overflow في TX FIFO
Bit 20 = RX_EMPTY      → لا يوجد بيانات للقراءة (= 1 عادي)
Bit 21 = TX_FULL       → TX FIFO ممتلئ
Bit 22 = TX_EMPTY      → TX FIFO فاضي (= 1 عادي في الهدوء)
Bit 25 = XMIT_BUSY     → الـ transmitter شغال
```

**تفسير REG5 (0x14) - حساب الـ baud rate:**
```
Bit 23    = BAUD_USE   → يجب أن يكون 1 لتفعيل الـ baud divider
Bit 24    = BAUD_XTAL  → 1 = استخدم الـ XTAL clock (24MHz)
Bit 27    = BAUD_XTAL_DIV2 → تقسيم إضافي على 2 (G12A فقط)
Bits[22:0]= الـ divisor

# الحساب للـ XTAL path بدون DIV2:
# baud = (24_000_000 / 3) / (divisor + 1)
# مثال: divisor = 0x68 = 104
# baud = 8_000_000 / 105 ≈ 76190 (!)
# الصحيح لـ 115200: divisor = round(24M/3/115200) - 1 = 68
```

---

#### 3. نصائح الـ Logic Analyzer / Oscilloscope

```
اتصالات الـ probes:
- CH1: TX pin (مثلاً GPIO_A_14 على Amlogic S905)
- CH2: RX pin
- CH3: RTS (لو موجود)
- CH4: CTS (لو موجود)
- GND: اربطه بـ GND الـ board

إعدادات الـ Logic Analyzer:
- Sample rate: على الأقل 10x الـ baud rate
  → لـ 115200: sample rate = 1.152 MHz على الأقل (استخدم 4 MHz)
- Trigger: falling edge على TX أو RX
- Protocol decoder: UART / async serial
  → Data bits: 8
  → Stop bits: 1
  → Parity: None (افتراضي)
  → Baud: 115200

علامات المشاكل الشائعة على الـ scope:
1. إشارة ثابتة على HIGH = لا يوجد إرسال (TX_EN مش مفعّل؟)
2. إشارة ثابتة على LOW = line stuck (short circuit أو خطأ في الـ init)
3. framing غير صحيح = الـ baud rate أو الـ clock خاطئ
4. noise على الإشارة = مشكلة grounding أو كابل طويل
5. start bit بدون stop bit = FRAME_ERR في الـ kernel
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | ما تراه في dmesg | السبب المحتمل |
|---------|-----------------|---------------|
| لا يوجد إرسال ولا استقبال | لا يزداد tx/rx في `/proc/tty/driver` | TX_EN أو RX_EN مش مفعّل، أو الـ clock متوقف |
| `FRAME_ERR` متكرر | `port->icount.frame` يزداد بسرعة | الـ baud rate غير متطابق مع الطرف الآخر، أو noise في الخط |
| `PARITY_ERR` متكرر | `port->icount.frame` يزداد (bug في الكود: يعدّ parity كـ frame) | parity setting خاطئ في أحد الطرفين |
| `TX_FIFO_WERR` (overrun) | `port->icount.overrun` يزداد | الـ application بطيئ في قراءة الـ RX، أو الـ FIFO صغير |
| Timeout في polling | `Timeout waiting for UART TX EMPTY` | الـ TX لا يُفرَّغ — hardware issue أو الـ baud rate صفر |
| كلوك خاطئ | `probe` يفشل بـ ENOENT | اسم الكلوك في الـ DT لا يتطابق مع pclk/xtal/baud |
| IRQ لا يُطلَق | العداد في `/proc/interrupts` لا يتغير | خطأ في الـ DT interrupt specifier أو الـ GIC |

---

#### 5. الـ Device Tree Debugging

```bash
# رقم 1: تحقق من الـ DT node الفعلي المحمّل
ls /proc/device-tree/soc/serial@*/
# أو
find /proc/device-tree -name "serial*" -o -name "*uart*" 2>/dev/null

# رقم 2: اقرأ خصائص الـ DT
xxd /proc/device-tree/soc/serial@ff803000/compatible
# يجب أن يظهر: amlogic,meson-gx-uart أو ما يشبهه

cat /proc/device-tree/soc/serial@ff803000/clock-names
# يجب: pclk xtal baud (بالترتيب)

xxd /proc/device-tree/soc/serial@ff803000/reg
# يجب أن يطابق BASE الذي تستخدمه مع devmem2

# رقم 3: تحقق من الـ aliases
cat /proc/device-tree/aliases/serial0
# يجب أن يشير للـ node الصحيح

# رقم 4: مقارنة الـ DT المصدر مع ما هو محمّل
# مثال DT node صحيح:
cat << 'EOF'
uart_AO: serial@ff803000 {
    compatible = "amlogic,meson-gx-uart";
    reg = <0x0 0xff803000 0x0 0x18>;  /* 6 regs × 4 bytes = 0x18 */
    interrupts = <GIC_SPI 58 IRQ_TYPE_EDGE_RISING>;
    clocks = <&xtal>, <&clkc CLKID_UART0_AO>, <&xtal>;
    clock-names = "xtal", "pclk", "baud";
    fifo-size = <64>;
    status = "okay";
};
EOF

# رقم 5: تحقق من الـ pinmux (هل الـ pins مضبوطة على UART وليس GPIO؟)
cat /proc/device-tree/soc/serial@ff803000/pinctrl-names
# يجب: default

# رقم 6: live DT debugging عبر dtc
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "serial@ff803000"
```

**مشاكل DT شائعة لـ meson_uart:**

```
مشكلة 1: اسم الكلوك خاطئ
  خاطئ:  clock-names = "clk_uart", "pclk_uart", "uart_baud";
  صحيح:  clock-names = "xtal", "pclk", "baud";
  الأثر: probe يفشل بـ -ENOENT

مشكلة 2: حجم الـ reg خاطئ
  خاطئ:  reg = <0xff803000 0x100>;
  صحيح:  reg = <0xff803000 0x18>;
  الأثر: يعمل لكن يقرأ registers خاطئة

مشكلة 3: compatible غير موجود في meson_uart_dt_match
  خاطئ:  compatible = "amlogic,s905-uart";
  صحيح:  compatible = "amlogic,meson-gx-uart";
  الأثر: الدريفر لا يُربط بالـ device

مشكلة 4: alias خاطئ يسبب تعارض ID
  aliases { serial0 = &uart_AO; serial0 = &uart_EE; }  /* تكرار! */
  الأثر: "port N already allocated" في dmesg
```

---

### Practical Commands

#### مجموعة الأوامر الجاهزة للنسخ

**فحص سريع شامل:**
```bash
#!/bin/bash
# meson_uart_debug.sh - فحص شامل لحالة الـ UART

echo "=== UART Ports ==="
cat /proc/tty/driver/meson_uart 2>/dev/null || \
cat /proc/tty/driver/ttyAML 2>/dev/null || echo "driver not loaded"

echo ""
echo "=== IRQ Status ==="
grep -E "meson|ttyAML|ttyS[0-9]" /proc/interrupts

echo ""
echo "=== MMIO Regions ==="
grep -i uart /proc/iomem

echo ""
echo "=== Clock Rates ==="
find /sys/kernel/debug/clk -name "clk_rate" 2>/dev/null | \
  xargs -I{} sh -c 'echo "{}: $(cat {})"' | grep -i uart

echo ""
echo "=== sysfs port info ==="
for dev in /sys/class/tty/ttyAML* /sys/class/tty/ttyS*; do
    [ -d "$dev" ] || continue
    echo "--- $(basename $dev) ---"
    for attr in uartclk type irq flags; do
        val=$(cat "$dev/$attr" 2>/dev/null)
        [ -n "$val" ] && echo "  $attr: $val"
    done
done
```

**قراءة registers مباشرة:**
```bash
# استبدل BASE بالعنوان الصحيح من /proc/iomem
BASE=0xff803000

echo "Reading meson_uart registers at 0x$(printf '%08x' $BASE):"
for name_off in "WFIFO:0x00" "RFIFO:0x04" "CONTROL:0x08" "STATUS:0x0c" "MISC:0x10" "REG5:0x14"; do
    name=${name_off%%:*}
    off=${name_off##*:}
    addr=$((BASE + off))
    val=$(devmem2 $addr w 2>/dev/null | grep 'Read.*:' | awk '{print $NF}')
    printf "  %-10s (0x%08x): %s\n" "$name" "$addr" "${val:-N/A}"
done
```

**مثال على الـ output وتفسيره:**
```
  CONTROL    (0xff803008): 0x1B003000
    → Bit 28 (TX_INT_EN) = 1  ✓ TX interrupt enabled
    → Bit 27 (RX_INT_EN) = 1  ✓ RX interrupt enabled
    → Bit 13 (RX_EN)     = 1  ✓ Receiver enabled
    → Bit 12 (TX_EN)     = 1  ✓ Transmitter enabled
    → Bits[21:20] = 00        → 8-bit data length
    → Bits[17:16] = 00        → 1 stop bit

  STATUS     (0xff80300c): 0x00500000
    → Bit 22 (TX_EMPTY)  = 1  ✓ TX buffer empty (idle)
    → Bit 20 (RX_EMPTY)  = 1  ✓ No data waiting
    → Bit 18 (FIFO_WERR) = 0  ✓ No overflow
    → Bit 17 (FRAME_ERR) = 0  ✓ No framing errors
    → Bit 16 (PARITY_ERR)= 0  ✓ No parity errors

  REG5       (0xff803014): 0x01800044
    → Bit 24 (BAUD_XTAL) = 1  → using XTAL clock
    → Bit 23 (BAUD_USE)  = 1  ✓ baud divider active
    → Bits[22:0] = 0x44 = 68  → baud = 24M/3/(68+1) = 115942 ≈ 115200 ✓
```

**تفعيل ftrace لتتبع الـ interrupt:**
```bash
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo function > current_tracer
echo 'meson_uart_interrupt meson_receive_chars meson_uart_start_tx' > set_ftrace_filter
echo 1 > tracing_on
sleep 5
echo 0 > tracing_on
grep -c 'meson_uart_interrupt' trace
# لو الناتج صفر أثناء إرسال/استقبال = مشكلة IRQ
cat trace | head -50
```

**اختبار الـ loopback (RX→TX short):**
```bash
# اربط TX بـ RX على المنفذ الفيزيائي أولاً
# ثم:
stty -F /dev/ttyAML0 115200 raw -echo
echo "HELLO" > /dev/ttyAML0 &
timeout 1 cat /dev/ttyAML0
# يجب أن تظهر: HELLO
# لو ماظهرت: مشكلة في الـ TX أو RX أو الـ baud rate
```

**مراقبة الـ icount errors في الوقت الفعلي:**
```bash
# script لمراقبة أخطاء الـ UART
watch -n 0.5 '
echo "=== UART Error Counters ==="
cat /proc/tty/driver/meson_uart 2>/dev/null | \
awk "{
  for(i=1;i<=NF;i++) {
    if(\$i ~ /^(rx|tx|fe|pe|brk|oe):/) printf \"%s\\n\", \$i
  }
}"
'
# fe=frame errors, pe=parity errors, brk=break, oe=overrun
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Android TV Box على Amlogic S905X3 — الـ console لا يظهر أي شيء عند الـ boot

#### العنوان
**الـ earlycon لا يشتغل على TV box بـ Amlogic S905X3 (meson-g12a)**

#### السياق
مهندس firmware بيشتغل على Android TV box بـ SoC هو **Amlogic S905X3** (اللي بيستخدم meson-g12a). المنتج في مرحلة bring-up، والـ bootloader (U-Boot) بيفضي بيشتغل، بس لما kernel بيبدأ، الـ serial console فاضية تماماً — مفيش أي طباعة.

#### المشكلة
الـ kernel command line فيه:
```
earlycon=meson,0xffd23000 console=ttyAML0,115200
```
بس الـ earlycon مش بيطبع حاجة خالص.

#### التحليل
بنتتبع الكود في `meson_uart.c`:

```c
// السطر 645-646: الـ earlycon بيتسجل بـ OF_EARLYCON_DECLARE
OF_EARLYCON_DECLARE(meson, "amlogic,meson-ao-uart", meson_serial_early_console_setup);
OF_EARLYCON_DECLARE(meson, "amlogic,meson-s4-uart", meson_serial_early_console_setup);
```

المشكلة إن الـ `meson-g12a-uart` **مش موجود** في قائمة الـ `OF_EARLYCON_DECLARE`! الـ earlycon setup function:

```c
static int __init
meson_serial_early_console_setup(struct earlycon_device *device, const char *opt)
{
    if (!device->port.membase)
        return -ENODEV;

    meson_uart_enable_tx_engine(&device->port); // بيشغّل TX فقط
    device->con->write = meson_serial_early_console_write;
    return 0;
}
```

الـ `meson_uart_enable_tx_engine` بتعمل إيه؟

```c
static void meson_uart_enable_tx_engine(struct uart_port *port)
{
    u32 val;
    val = readl(port->membase + AML_UART_CONTROL);
    val |= AML_UART_TX_EN;  // بتشغّل TX engine بس
    writel(val, port->membase + AML_UART_CONTROL);
}
```

المشكلة واضحة: **الـ TX_EN بيتحط، لكن لو الـ TX engine أصلاً مش initialized من الـ bootloader، والـ BAUD RATE مش متضبط (REG5 = 0)، مش هيطلع أي بيانات صح على الـ wire.**

علاوة على كده، لو المهندس بعت address غلط في `earlycon=meson,0xffd23000` والـ AO UART الصح على عنوان تاني، الـ `device->port.membase` هيبقى NULL وهيرجع `-ENODEV` بصمت تام.

#### الحل

**أولاً: تحقق من العنوان الصح للـ AO UART في الـ datasheet أو الـ DT:**

```bash
# على الجهاز الشغال (لو فيه shell):
cat /proc/iomem | grep uart
# أو من U-Boot:
# md.l 0xff803000 4   # جرّب العنوانات الشائعة
```

**ثانياً: تأكد إن الـ DT compatible صح:**

```dts
/* في الـ device tree لـ S905X3 */
uart_AO: serial@ffd23000 {
    compatible = "amlogic,meson-g12a-uart",  /* g12a مش ao-uart */
                 "amlogic,meson-ao-uart";
    reg = <0x0 0xffd23000 0x0 0x18>;
    /* ... */
};
```

لازم الـ compatible الثاني `"amlogic,meson-ao-uart"` يكون موجود عشان الـ `OF_EARLYCON_DECLARE` يلاقيه.

**ثالثاً: الكود محتاج إضافة:**

```c
// المفروض يكون في الـ driver:
OF_EARLYCON_DECLARE(meson, "amlogic,meson-g12a-uart", meson_serial_early_console_setup);
```

#### الدرس المستفاد
الـ `OF_EARLYCON_DECLARE` في `meson_uart.c` بيغطي `meson-ao-uart` و`meson-s4-uart` بس. أي SoC جديد من Amlogic لازم يحط compatible string ثاني من القائمة دي، أو يضيف `OF_EARLYCON_DECLARE` جديد في الـ driver. الـ earlycon صامت جداً لما بيفشل — مفيش log، مفيش error.

---

### السيناريو الثاني: Industrial Gateway على Amlogic A113X — الـ baud rate غلط بعد الـ boot

#### العنوان
**الـ UART بيتكلم على 115200 بس الـ Modbus sensor بيستقبل garbage data**

#### السياق
شركة بتبني **industrial gateway** بـ Amlogic A113X (مبني على meson-a1). الـ gateway بيتكلم مع **Modbus RTU sensors** عبر RS-485 على 9600 baud. كل حاجة شغالة على الـ bootloader، بس بعد ما Linux بيبوت، الـ sensor بيقرأ garbage.

#### المشكلة
الـ userspace بيحط الـ baud على 9600 عبر `stty`، لكن الـ sensor مش بيفهم. قياس بـ oscilloscope بيكشف إن الـ baud الفعلي على الـ wire هو ~9615 بدل 9600.

#### التحليل
ندخل على `meson_uart_change_speed`:

```c
static void meson_uart_change_speed(struct uart_port *port, unsigned long baud)
{
    const struct meson_uart_data *private_data = port->private_data;
    u32 val = 0;

    while (!meson_uart_tx_empty(port))
        cpu_relax();

    if (port->uartclk == 24000000) {
        unsigned int xtal_div = 3;

        if (private_data && private_data->has_xtal_div2) {
            xtal_div = 2;
            val |= AML_UART_BAUD_XTAL_DIV2;
        }
        val |= DIV_ROUND_CLOSEST(port->uartclk / xtal_div, baud) - 1;
        val |= AML_UART_BAUD_XTAL;
    } else {
        val =  DIV_ROUND_CLOSEST(port->uartclk / 4, baud) - 1;
    }
    val |= AML_UART_BAUD_USE;
    writel(val, port->membase + AML_UART_REG5);
}
```

الـ A113X في الـ driver data:

```c
static struct meson_uart_data meson_a1_uart_data = {
    .uart_driver = &meson_uart_driver_ttyS,
    .has_xtal_div2 = false,  // xtal_div = 3
};
```

لما `has_xtal_div2 = false`، بيستخدم `xtal_div = 3`:
```
val = DIV_ROUND_CLOSEST(24000000 / 3, 9600) - 1
    = DIV_ROUND_CLOSEST(8000000, 9600) - 1
    = DIV_ROUND_CLOSEST(833.33) - 1
    = 833 - 1
    = 832
```

الـ actual baud = 8000000 / (832 + 1) = 8000000 / 833 = **9604** — ده قريب جداً ومقبول.

**إذاً المشكلة مش في الحساب!** المشكلة إن `port->uartclk` نفسه غلط. نشوف:

```c
static int meson_uart_probe_clocks(struct platform_device *pdev,
                                   struct uart_port *port)
{
    /* ... */
    clk_baud = devm_clk_get_enabled(&pdev->dev, "baud");
    if (IS_ERR(clk_baud))
        return PTR_ERR(clk_baud);

    port->uartclk = clk_get_rate(clk_baud);
    return 0;
}
```

لو الـ `"baud"` clock في الـ DT مش بيرجع 24MHz بالظبط (مثلاً بيرجع 24.014MHz بسبب خطأ في الـ PLL configuration)، الـ formula هتحسب baud خاطئ.

#### الحل

**خطوة 1: تحقق من الـ clock الفعلي:**

```bash
# على الجهاز:
cat /sys/kernel/debug/clk/clk_summary | grep uart
# أو:
cat /sys/kernel/debug/clk/uart_clk/clk_rate
```

**خطوة 2: لو الـ xtal مش بالظبط 24MHz، اضبط الـ DT:**

```dts
&uart_A {
    clocks = <&clkc CLKID_UART0>,
             <&xtal>,
             <&clkc CLKID_UART0>;
    clock-names = "pclk", "xtal", "baud";
    /* تأكد إن xtal = 24000000 بالظبط */
};
```

**خطوة 3: debug مباشر:**

```bash
# اقرأ REG5 مباشر (لو معاك devmem):
devmem 0xFF803014 32  # عنوان REG5
# لو القيمة = 0x01800340 مثلاً:
# BIT23 (BAUD_USE) موجود، BIT24 (BAUD_XTAL) موجود
# الـ divisor = 0x340 & 0x7fffff = 0x340 = 832
```

#### الدرس المستفاد
في الـ Meson UART، الـ baud rate accuracy كلها معتمدة على دقة الـ `"baud"` clock. في industrial protocols زي Modbus وCANopen، حتى 0.5% error ممكن يسبب CRC failures. دايماً تحقق من `clk_get_rate()` فعلياً باستخدام debugfs قبل ما تحكم على الـ driver.

---

### السيناريو الثاني: Custom Board بـ Amlogic S905Y4 — الـ RTS/CTS مش شغالة

#### العنوان
**Flow control بين Bluetooth module والـ SoC بيعمل data corruption تحت load**

#### السياق
مهندس hardware بيصمم **smart speaker** بـ Amlogic S905Y4 (مبني على meson-s4 architecture). الـ Bluetooth module (QCC5125) متوصل بـ UART مع hardware RTS/CTS. تحت load عادي كل شيء تمام، بس لما معدل الـ audio data بيزيد، البيانات بتتخرب.

#### المشكلة
الـ `stty -F /dev/ttyS1` بيقول `crtscts` موجود، لكن فعلياً الـ hardware flow control مش شغال. الـ oscilloscope بيكشف إن RTS line لا بترفع ولا بتنزل.

#### التحليل
نفحص `meson_uart_set_termios`:

```c
if (cflags & CRTSCTS) {
    if (port->flags & UPF_HARD_FLOW)
        val &= ~AML_UART_TWO_WIRE_EN;  // يشغّل الـ 4-wire mode (مع RTS/CTS)
    else
        termios->c_cflag &= ~CRTSCTS;  // يلغي الـ CRTSCTS بصمت!
} else {
    val |= AML_UART_TWO_WIRE_EN;  // 2-wire mode (بدون flow control)
}
```

الكود واضح: لو `UPF_HARD_FLOW` مش موجود في `port->flags`، الـ driver بيلغي `CRTSCTS` من التـ termios بصمت تام — ومفيش error، ومفيش warning!

والـ `UPF_HARD_FLOW` بيتحط فين؟

```c
// في meson_uart_probe:
has_rtscts = of_property_read_bool(pdev->dev.of_node, "uart-has-rtscts");

if (meson_ports[pdev->id]) { /* ... */ }

port->flags = UPF_BOOT_AUTOCONF | UPF_LOW_LATENCY;
if (has_rtscts)
    port->flags |= UPF_HARD_FLOW;  // بيتحط بس لو uart-has-rtscts موجود في DT
```

إذاً الحل في الـ DT.

أيضاً لاحظ إن `meson_uart_get_mctrl` دايماً بيرجع `TIOCM_CTS`:

```c
static unsigned int meson_uart_get_mctrl(struct uart_port *port)
{
    return TIOCM_CTS;  // دايماً يقول CTS active — مش بيقرأ الـ hardware!
}
```

ده معناه إن الـ software بيعتقد CTS دايماً active، حتى لو الـ Bluetooth module بيحاول يوقف الـ data.

#### الحل

**إصلاح الـ Device Tree:**

```dts
&uart_B {
    compatible = "amlogic,meson-s4-uart";
    uart-has-rtscts;  /* هذا السطر الناقص! */
    pinctrl-0 = <&uart_b_pins_with_rtscts>;
    pinctrl-names = "default";
};
```

**تأكد من الـ pinmux كمان:**

```bash
# تحقق إن pins الـ RTS/CTS متوجهة للـ UART مش GPIO:
cat /sys/kernel/debug/pinctrl/pinctrl@ff634000/pinmux-pins | grep -A2 "uart_b"
```

**التحقق بعد الإصلاح:**

```bash
stty -F /dev/ttyS1 crtscts
# أو:
stty -F /dev/ttyS1 -a | grep crtscts
# يجب أن يظهر crtscts بدون '-' قبلها
```

#### الدرس المستفاد
الـ Meson UART driver بيلغي `CRTSCTS` بصمت لو `uart-has-rtscts` مش موجود في الـ DT. ده سلوك خطير في الـ Bluetooth audio applications لأن الـ data corruption بيظهر فقط تحت load عالي — مش في الـ unit tests العادية. دايماً فحص الـ DT properties الـ optional لأنها مش optional في كل hardware setup.

---

### السيناريو الرابع: IoT Sensor Node — الـ UART interrupt مش بيصحى عند الاستقبال

#### العنوان
**الـ RS-232 sensor بيبعت data لكن الـ kernel مش بيقراها (RX interrupt ميجيش)**

#### السياق
فريق IoT بيطور **environmental monitoring node** بـ Amlogic A113L (meson-a1 family). الـ CO2 sensor (Sensirion SCD41) متوصل بـ UART على 9600 baud. الـ sensor بيبعت بيانات كل ثانية، بس الـ userspace read() بتـ block للأبد.

#### المشكلة
```bash
strace cat /dev/ttyS0
# ...
read(3, 0x..., 4096)  = ?   <-- blocked هنا للأبد
```

مفيش data بتوصل رغم إن الـ oscilloscope بيكشف signals صح على الـ RX pin.

#### التحليل
ندخل على `meson_uart_startup`:

```c
static int meson_uart_startup(struct uart_port *port)
{
    unsigned long flags;
    u32 val;
    int ret = 0;

    uart_port_lock_irqsave(port, &flags);

    val = readl(port->membase + AML_UART_CONTROL);
    val |= AML_UART_CLEAR_ERR;
    writel(val, port->membase + AML_UART_CONTROL);
    val &= ~AML_UART_CLEAR_ERR;
    writel(val, port->membase + AML_UART_CONTROL);

    val |= (AML_UART_RX_EN | AML_UART_TX_EN);
    writel(val, port->membase + AML_UART_CONTROL);

    val |= (AML_UART_RX_INT_EN | AML_UART_TX_INT_EN);
    writel(val, port->membase + AML_UART_CONTROL);

    // الـ RECV_IRQ بيتحط على 1 — يعني interrupt بيجي عند استقبال byte واحد
    val = (AML_UART_RECV_IRQ(1) | AML_UART_XMIT_IRQ(port->fifosize / 2));
    writel(val, port->membase + AML_UART_MISC);

    uart_port_unlock_irqrestore(port, flags);

    ret = request_irq(port->irq, meson_uart_interrupt, 0,
                      port->name, port);

    return ret;
}
```

والـ interrupt handler:

```c
static irqreturn_t meson_uart_interrupt(int irq, void *dev_id)
{
    struct uart_port *port = (struct uart_port *)dev_id;

    uart_port_lock(port);

    if (!(readl(port->membase + AML_UART_STATUS) & AML_UART_RX_EMPTY))
        meson_receive_chars(port);

    /* ... TX handling ... */

    uart_unlock_and_check_sysrq(port);

    return IRQ_HANDLED;
}
```

المشكلة المحتملة: لو الـ **IRQ number في الـ DT غلط أو الـ interrupt controller مش متضبط صح**، الـ `request_irq` هيرجع 0 (success) لكن الـ interrupt مش هيوصل أبداً.

تحقق تاني: لو في الـ DT الـ `interrupts` property بتشاور على IRQ خاص بـ UART لكن الـ GIC مش enable له، هيحصل نفس المشكلة.

#### الحل

**خطوة 1: تحقق إن الـ IRQ بيتسجل صح:**

```bash
cat /proc/interrupts | grep meson_uart
# يجب أن يظهر إدخال مع عداد بيزيد
```

**خطوة 2: شوف الـ IRQ numbers المتاحة:**

```bash
cat /proc/interrupts
# دوّر على ttyS0 أو ttyAML0
```

**خطوة 3: لو العداد صفر، جرّب:**

```bash
# أرسل data من الـ sensor وراقب:
watch -n 0.5 'cat /proc/interrupts | grep uart'
```

**خطوة 4: إصلاح الـ DT لو الـ IRQ غلط:**

```dts
&uart_AO {
    compatible = "amlogic,meson-a1-uart";
    interrupts = <GIC_SPI 193 IRQ_TYPE_EDGE_RISING>;  // تحقق من الـ datasheet
    status = "okay";
};
```

**خطوة 5: لو الـ RX_EN مش بيتحط، جرّب force:**

```bash
# لو معاك devmem:
devmem 0xFF803008 32  # قرأ CONTROL register
# BIT13 = RX_EN يجب أن يكون 1
```

#### الدرس المستفاد
الـ `request_irq` بترجع 0 حتى لو الـ IRQ مش هيجي فعلياً. في الـ Meson UART، الـ RX interrupt هو الطريقة الوحيدة لاستقبال data (مفيش DMA في الـ driver ده). دايماً فحص `/proc/interrupts` كأول خطوة debug لأي مشكلة RX.

---

### السيناريو الخامس: Automotive ECU Bring-up — الـ FIFO Overrun بيخرّب بيانات الـ CAN-to-UART Bridge

#### العنوان
**الـ TX_FIFO_WERR errors كتير في automotive application بسبب خطأ في فهم الـ overrun logic**

#### السياق
مهندس embedded بيبني **automotive ECU** بـ Amlogic S905D3G (meson-g12a). الـ ECU بيستقبل CAN frames عبر SPI وبيحولها لـ UART ليبعتها لـ telemetry module. عند السرعة العالية (~500 kbps UART)، الـ kernel log بيظهر:

```
[  45.123] meson_uart: overrun errors
```

وبعض الـ CAN frames بتـ drop.

#### المشكلة
الـ overrun بيحصل رغم إن الـ FIFO size = 64 byte والـ data rate مش عالي جداً.

#### التحليل
نفحص `meson_receive_chars` بعناية:

```c
static void meson_receive_chars(struct uart_port *port)
{
    struct tty_port *tport = &port->state->port;
    char flag;
    u32 ostatus, status, ch, mode;

    do {
        flag = TTY_NORMAL;
        port->icount.rx++;
        ostatus = status = readl(port->membase + AML_UART_STATUS);

        if (status & AML_UART_ERR) {
            if (status & AML_UART_TX_FIFO_WERR)
                port->icount.overrun++;     // عداد الـ overrun
            else if (status & AML_UART_FRAME_ERR)
                port->icount.frame++;
            else if (status & AML_UART_PARITY_ERR)
                port->icount.frame++;       // BUG! المفروض icount.parity

            mode = readl(port->membase + AML_UART_CONTROL);
            mode |= AML_UART_CLEAR_ERR;
            writel(mode, port->membase + AML_UART_CONTROL);

            /* It doesn't clear to 0 automatically */
            mode &= ~AML_UART_CLEAR_ERR;
            writel(mode, port->membase + AML_UART_CONTROL);

            status &= port->read_status_mask;
            if (status & AML_UART_FRAME_ERR)
                flag = TTY_FRAME;
            else if (status & AML_UART_PARITY_ERR)
                flag = TTY_PARITY;
        }

        ch = readl(port->membase + AML_UART_RFIFO);
        ch &= 0xff;

        /* ... */

        if ((status & port->ignore_status_mask) == 0)
            tty_insert_flip_char(tport, ch, flag);

        if (status & AML_UART_TX_FIFO_WERR)
            tty_insert_flip_char(tport, 0, TTY_OVERRUN);  // overrun char

    } while (!(readl(port->membase + AML_UART_STATUS) & AML_UART_RX_EMPTY));

    tty_flip_buffer_push(tport);
}
```

لاحظ إن الـ `AML_UART_TX_FIFO_WERR` اسمه غريب — ليه TX FIFO error بيحصل عند الـ RX؟

ده لأن الـ Amlogic hardware بيستخدم نفس الـ status bit للـ "write to full FIFO" سواء كانت TX أو RX FIFO. يعني لو الـ **RX FIFO امتلأت** وجاء byte جديد، الـ hardware بيحط `TX_FIFO_WERR` — ده تسمية مضللة في الـ hardware نفسه.

**لماذا الـ FIFO بتمتلئ؟**

```c
// في meson_uart_startup:
val = (AML_UART_RECV_IRQ(1) | AML_UART_XMIT_IRQ(port->fifosize / 2));
writel(val, port->membase + AML_UART_MISC);
```

الـ `RECV_IRQ(1)` يعني: "اعملي interrupt لما يكون في الـ FIFO byte واحد على الأقل". ده setting جيد، لكن المشكلة هي الـ **interrupt latency**!

في الـ automotive ECU، لو الـ interrupt handler بيتأخر (بسبب spinlock contention مع SPI driver مثلاً)، الـ FIFO بتتملى قبل ما تُقرأ.

#### الحل

**خطوة 1: قياس الـ interrupt latency:**

```bash
# استخدم ftrace:
echo function > /sys/kernel/debug/tracing/current_tracer
echo meson_uart_interrupt > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل الـ load ...
cat /sys/kernel/debug/tracing/trace | head -50
```

**خطوة 2: رفع الـ RECV_IRQ threshold لتقليل interrupt frequency:**

لاحظ إن الـ `RECV_IRQ(1)` معناه interrupt لكل byte. لو الـ SoC بطيء في الـ interrupt handling، ممكن نرفع الـ threshold:

```c
// بدل RECV_IRQ(1) نستخدم:
val = (AML_UART_RECV_IRQ(16) | AML_UART_XMIT_IRQ(port->fifosize / 2));
// ده هيقلل عدد الـ interrupts لكن يزود الـ latency
// المقايضة حسب الـ application
```

بس ده تعديل في الـ driver نفسه وبيحتاج rebuild.

**خطوة 3: استخدام الـ CPU affinity:**

```bash
# اعرف IRQ number:
cat /proc/interrupts | grep meson_uart
# مثلاً IRQ 54
# حطّ الـ UART interrupt على CPU1 بعيداً عن SPI على CPU0:
echo 2 > /proc/irq/54/smp_affinity  # CPU1 فقط
```

**خطوة 4: تحقق من الـ overrun statistics:**

```bash
cat /proc/tty/driver/ttyS
# أو:
stty -F /dev/ttyS0 -a
# بعد ذلك:
grep overrun /proc/tty/driver/ttyS
```

#### الدرس المستفاد
الـ `AML_UART_TX_FIFO_WERR` اسم مضلل — هو في الواقع بيحصل لما الـ RX FIFO تمتلئ. في الـ automotive applications اللي فيها multiple high-priority drivers (SPI, CAN, UART) بيتشاركوا CPU resources، الـ interrupt latency هي أكبر سبب للـ FIFO overrun. الحل مش دايماً في الـ UART driver نفسه، لكن في الـ system-level IRQ affinity وتقليل الـ spinlock contention.
## Phase 7: مصادر ومراجع

---

### التوثيق الرسمي في الـ Kernel

هذي أهم الملفات في شجرة الـ kernel اللي تساعدك تفهم driver الـ UART بشكل أعمق:

| المسار | الوصف |
|--------|--------|
| `Documentation/driver-api/serial/driver.rst` | الـ Low Level Serial API — شرح كامل لـ `uart_port` و `uart_ops` |
| `Documentation/driver-api/tty/tty_driver.rst` | شرح بنية الـ TTY layer ودور الـ `tty_driver` |
| `Documentation/devicetree/bindings/serial/amlogic,meson-uart.txt` | الـ DT bindings الرسمية للـ Amlogic Meson UART |
| `drivers/tty/serial/serial_core.c` | تنفيذ الـ serial core — قلب الـ subsystem |
| `include/linux/serial_core.h` | تعريفات `uart_port` و `uart_ops` و `uart_driver` |
| `include/linux/serial.h` | الـ structs الأساسية للـ serial port |

---

### مقالات LWN.net

**الـ LWN.net** هو أفضل مرجع لمتابعة تطور الـ kernel وفهم قرارات التصميم.

#### متعلقة بـ UART وـ Serial Core

- [Low Level Serial API — Linux Kernel Documentation](https://static.lwn.net/kerneldoc/driver-api/serial/driver.html)
  شرح رسمي لكل الـ callbacks المطلوبة في `uart_ops`، من `tx_empty` إلى `set_termios`. أي شخص يكتب UART driver لازم يقرأ هذا.

- [TTY Driver and TTY Operations — Linux Kernel Documentation](https://static.lwn.net/kerneldoc/driver-api/tty/tty_driver.html)
  يشرح كيف الـ TTY layer يتعامل مع الـ drivers، وكيف الـ `tty_flip_buffer` يشتغل.

- [TTY Drivers — LDD3 Chapter 18 (PDF)](https://static.lwn.net/images/pdf/LDD3/ch18.pdf)
  الفصل الكامل عن TTY drivers من كتاب Linux Device Drivers الطبعة الثالثة — مجاني ومباشر.

- [The need for TTY slave devices](https://lwn.net/Articles/700489/)
  مقال عن تطور بنية الـ TTY وإضافة مفهوم الـ slave devices، يساعد تفهم لماذا تغيرت الأشياء.

- [UART slave device bus](https://lwn.net/Articles/697534/)
  نقاش في الـ kernel عن كيفية ربط devices فوق الـ UART، مثل Bluetooth و GPS modules.

- [tty: serial: Add mediatek UART driver](https://lwn.net/Articles/608214/)
  مثال عملي لإضافة UART driver لـ SoC مشابه (MediaTek)، يُظهر نفس الأنماط المستخدمة في meson_uart.

- [tty/serial: add support for Xilinx PS UART](https://lwn.net/Articles/440058/)
  مثال آخر لإضافة UART driver لـ embedded SoC، مفيد للمقارنة.

---

### نقاشات الـ Mailing List (lore.kernel.org)

هذي النقاشات الحقيقية بين المطورين حول تطوير `meson_uart.c`:

- [PATCH V2: UART driver compatible with Amlogic Meson S4](https://lore.kernel.org/all/20211229135350.9659-2-yu.tu@amlogic.com/T/)
  سلسلة patches لإضافة دعم شريحة S4، تحتوي على نقاشات تقنية مهمة عن حساب الـ baud rate.

- [PATCH V3: UART driver for Amlogic Meson](https://lore.kernel.org/lkml/20211230102110.3861-1-yu.tu@amlogic.com/t/)
  النسخة الثالثة من نفس السلسلة بعد التعديلات بناءً على ملاحظات الـ reviewers.

- [PATCH V3: dt-bindings: serial: amlogic,meson-uart: support S4](https://lore.kernel.org/linux-arm-kernel/CAFBinCAhbJUevamYqmzrFsVcXoK8Wcntem9VmYUpBL8ktEsVAA@mail.gmail.com/T/)
  نقاش حول الـ Device Tree bindings الجديدة لشريحة S4.

- [serial: meson: make the current driver compatible with S4](https://patchwork.kernel.org/project/linux-amlogic/patch/20211206100200.31914-1-xianwei.zhao@amlogic.com/)
  أول patch يقترح التوافق مع S4.

- [V6,3/5: tty: serial: meson: UART baud rate clock using clock frame](https://patchwork.kernel.org/project/linux-amlogic/patch/20220118030911.12815-4-yu.tu@amlogic.com/)
  تفاصيل تقنية عن استخدام الـ Common Clock Framework لحساب الـ baud rate — مهم جداً لفهم دالة `meson_uart_set_termios`.

---

### الكود المصدري على GitHub

- [drivers/tty/serial/meson_uart.c — torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/tty/serial/meson_uart.c)
  أحدث نسخة من الـ driver في شجرة Linus الرسمية.

---

### توثيق Amlogic Meson على linux-meson.com

- [Kernel Mainlining Progress — linux-meson.com](https://linux-meson.com/mainlining.html)
  موقع غير رسمي لكن ممتاز يتابع تقدم إدخال دعم Amlogic Meson في الـ mainline kernel، يذكر تفاصيل الـ UART support.

---

### صفحات eLinux.org

- [Serial Console — eLinux.org](https://elinux.org/Serial_console)
  دليل عملي لإعداد الـ serial console في أنظمة embedded Linux — مفيد جداً للـ debugging.

- [Amlogic — eLinux.org](https://elinux.org/Amlogic)
  صفحة مرجعية لكل ما يخص Amlogic SoCs في بيئة Linux، تشمل روابط لـ tools وـ resources.

---

### صفحات KernelNewbies.org

- [Linux_6.8 — KernelNewbies](https://kernelnewbies.org/Linux_6.8)
  تغييرات الـ serial/tty subsystem في kernel 6.8، تشمل نقل الـ tty وـ serdev ليكونوا children لـ serial core port device.

- [Linux Kernel Tester's Guide Chapter 3 — KernelNewbies](https://kernelnewbies.org/Linux_Kernel_Tester's_Guide_Chapter3)
  كيفية استخدام الـ serial console لالتقاط رسائل الـ kernel، مفيد جداً للمبتدئين في اختبار الـ drivers.

- [FAQ/WhereDoIBegin — KernelNewbies](https://kernelnewbies.org/FAQ/WhereDoIBegin)
  نقطة بداية ممتازة لمن يريد البدء في تطوير الـ kernel، يشرح استخدام serial console لـ kernel oops.

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
الكتاب الكلاسيكي لتطوير drivers في Linux، متاح مجاناً:

- **الفصل 18 — TTY Drivers**: شرح كامل لبنية الـ TTY، كيف تكتب driver من الصفر، الـ `tty_driver` و `tty_operations`.
- **الفصل 10 — Interrupt Handling**: يشرح كيف الـ interrupt handlers تشتغل — أساسي لفهم `meson_uart_interrupt`.
- **الفصل 13 — USB Drivers** (للمقارنة): نفس الأنماط المستخدمة في كتابة الـ drivers.

رابط الكتاب كاملاً: https://lwn.net/Kernel/LDD3/

#### Linux Kernel Development — Robert Love (الطبعة 3)
- **الفصل 7 — Interrupts and Interrupt Handlers**: فهم كيف الـ IRQ handlers تُسجَّل وتعمل.
- **الفصل 5 — System Calls**: يساعد تفهم كيف الـ user space يتواصل مع الـ driver.
- **الفصل 17 — Devices and Modules**: شرح `platform_device` وـ `platform_driver`.

#### Embedded Linux Primer — Christopher Hallinan (الطبعة 2)
- **الفصل 14 — Kernel Debugging Techniques**: استخدام الـ serial console للـ debugging.
- **الفصل 16 — Bootloaders**: كيف الـ bootloader يُهيئ الـ UART قبل الـ kernel.
- **الفصل 15 — Debugging with KGDB**: استخدام الـ UART كـ debug interface.

---

### مصطلحات للبحث عنها

إذا أردت تعمق أكثر، ابحث عن هذي المصطلحات:

```
uart_port linux kernel
uart_ops callbacks implementation
tty_flip_buffer_push linux
serial_core linux driver
platform_device probe linux
devm_ioremap_resource linux
clk_get_rate linux kernel
console_write linux early console
amlogic meson serial driver
AML_UART register map
termios baud rate calculation linux
```

---

### ملخص سريع للمصادر الأهم

| الأولوية | المصدر | لماذا؟ |
|---------|--------|--------|
| ⭐⭐⭐ | [Low Level Serial API](https://static.lwn.net/kerneldoc/driver-api/serial/driver.html) | الدليل الرسمي للـ uart_ops — إلزامي |
| ⭐⭐⭐ | [LDD3 Chapter 18 PDF](https://static.lwn.net/images/pdf/LDD3/ch18.pdf) | الكتاب الكلاسيكي، مجاني |
| ⭐⭐⭐ | [amlogic,meson-uart.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/serial/amlogic,meson-uart.txt) | الـ DT bindings الرسمية |
| ⭐⭐ | [Mailing list S4 patches](https://lore.kernel.org/all/20211229135350.9659-2-yu.tu@amlogic.com/T/) | نقاشات حقيقية من المطورين |
| ⭐⭐ | [Serial Console — eLinux](https://elinux.org/Serial_console) | دليل عملي للـ debugging |
| ⭐⭐ | [linux-meson.com mainlining](https://linux-meson.com/mainlining.html) | تتبع دعم Meson في الـ kernel |
| ⭐ | [KernelNewbies Linux 6.8](https://kernelnewbies.org/Linux_6.8) | آخر التغييرات في الـ serial subsystem |
## Phase 8: Writing simple module

### الفكرة — ماذا سنعمل؟

سنكتب kernel module يستخدم **kprobe** يربطه بالدالة `meson_uart_set_termios`، وهي الدالة اللي تُستدعى كلما غيّر أي برنامج إعدادات الـ UART (الـ baud rate، عدد الـ bits، الـ parity). تخيّل إن عندك شباك تجسس على "رسالة الضبط" اللي يبعثها النظام للـ UART كلما فتحت terminal أو شغّلت minicom.

الدالة `meson_uart_set_termios` مُعلَنة بشكل static في الملف، لكن رقمها موجود في جدول `meson_uart_ops` الذي يُصدَّر بشكل غير مباشر عبر `uart_add_one_port`. كـ kprobe يمكننا نضع hook على اسمها المرمّز (mangled symbol) أو نستخدم اسم الدالة مباشرة لأن الـ kernel يعرفها عبر kallsyms.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * meson_termios_spy.c
 *
 * Module يراقب كل استدعاء لـ meson_uart_set_termios
 * ويطبع إعدادات الـ UART الجديدة (baud rate, data bits, parity, stop bits).
 *
 * يعمل عبر kprobe — لا يعدّل كود الـ driver الأصلي أبدًا.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* kprobe, register_kprobe */
#include <linux/serial_core.h>  /* uart_port, uart_get_baud_rate */
#include <linux/tty.h>          /* ktermios */

/* ---------------------------------------------------------------
 * pre_handler: يُشغَّل قبل تنفيذ meson_uart_set_termios مباشرةً.
 *
 * توقيع الدالة الأصلية:
 *   void meson_uart_set_termios(struct uart_port *port,
 *                               struct ktermios *termios,
 *                               const struct ktermios *old)
 *
 * على x86_64:
 *   regs->di = port   (أول argument)
 *   regs->si = termios (ثاني argument)
 * على arm64:
 *   regs->regs[0] = port
 *   regs->regs[1] = termios
 *
 * نستخدم helper الـ kprobe لجلب الـ arguments بشكل portable.
 * ----------------------------------------------------------------
 */
static int meson_termios_pre_handler(struct kprobe *p, struct pt_regs *regs)
{
	struct uart_port *port;
	struct ktermios  *termios;
	unsigned int      cflags;
	unsigned int      data_bits;
	const char       *parity_str;
	unsigned int      stop_bits;

	/*
	 * جلب الـ arguments من الـ registers.
	 * على x86_64: di=arg0, si=arg1
	 * على arm64:  regs[0]=arg0, regs[1]=arg1
	 */
#if defined(CONFIG_X86_64)
	port    = (struct uart_port *)regs->di;
	termios = (struct ktermios  *)regs->si;
#elif defined(CONFIG_ARM64)
	port    = (struct uart_port *)regs->regs[0];
	termios = (struct ktermios  *)regs->regs[1];
#else
	/* معمارية غير مدعومة في هذا المثال — نطبع فقط إن الدالة اتُستدعت */
	pr_info("meson_termios_spy: set_termios called (arch unsupported for arg dump)\n");
	return 0;
#endif

	/* تحقق بسيط لتجنب null dereference */
	if (!port || !termios)
		return 0;

	cflags = termios->c_cflag;

	/* استخراج عدد الـ data bits من CSIZE */
	switch (cflags & CSIZE) {
	case CS5: data_bits = 5; break;
	case CS6: data_bits = 6; break;
	case CS7: data_bits = 7; break;
	default:  data_bits = 8; break; /* CS8 */
	}

	/* نوع الـ parity */
	if (!(cflags & PARENB))
		parity_str = "none";
	else if (cflags & PARODD)
		parity_str = "odd";
	else
		parity_str = "even";

	/* عدد الـ stop bits */
	stop_bits = (cflags & CSTOPB) ? 2 : 1;

	/*
	 * uartclk موجودة في port، لكن الـ baud الحقيقي يُحسب بعد هذه الدالة.
	 * نستخرجه من termios بالطريقة نفسها اللي يستخدمها الـ driver:
	 * uart_get_baud_rate تحتاج port lock — هنا نكتفي بـ c_ospeed
	 * لأننا في pre_handler ولا نريد أخذ lock.
	 */
	pr_info("meson_termios_spy: port%d → %u baud | %u%s%u | clk=%u Hz\n",
		port->line,
		termios->c_ospeed,   /* الـ baud rate المطلوب */
		data_bits,
		parity_str,
		stop_bits,
		port->uartclk);

	return 0; /* 0 = استمر في تنفيذ الدالة الأصلية */
}

/* ---------------------------------------------------------------
 * تعريف الـ kprobe — نحدد اسم الدالة كنص
 * الـ kernel سيحوّله لعنوان عبر kallsyms
 * ----------------------------------------------------------------
 */
static struct kprobe meson_termios_kp = {
	.symbol_name    = "meson_uart_set_termios",
	.pre_handler    = meson_termios_pre_handler,
};

/* ---------------------------------------------------------------
 * module_init: نسجّل الـ kprobe عند تحميل الـ module
 * ----------------------------------------------------------------
 */
static int __init meson_termios_spy_init(void)
{
	int ret;

	ret = register_kprobe(&meson_termios_kp);
	if (ret < 0) {
		pr_err("meson_termios_spy: register_kprobe failed: %d\n", ret);
		pr_err("  (تأكد إن CONFIG_KPROBES=y و meson_uart driver محمّل)\n");
		return ret;
	}

	pr_info("meson_termios_spy: planted kprobe at %s (%px)\n",
		meson_termios_kp.symbol_name,
		meson_termios_kp.addr);

	return 0;
}

/* ---------------------------------------------------------------
 * module_exit: نشيل الـ kprobe عند تفريغ الـ module
 * ----------------------------------------------------------------
 */
static void __exit meson_termios_spy_exit(void)
{
	unregister_kprobe(&meson_termios_kp);
	pr_info("meson_termios_spy: kprobe removed\n");
}

module_init(meson_termios_spy_init);
module_exit(meson_termios_spy_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Linux in Arabic Project");
MODULE_DESCRIPTION("kprobe spy on meson_uart_set_termios — prints UART settings on change");
```

---

### شرح كل جزء

#### الـ includes

```c
#include <linux/kprobes.h>
#include <linux/serial_core.h>
#include <linux/tty.h>
```

**الـ kprobes.h** هو قلب الموضوع — يعطينا `struct kprobe` وdالتي `register_kprobe` / `unregister_kprobe`. بدونه ما عندنا أي وسيلة نضع "نقطة مراقبة" على دالة موجودة في الـ kernel. **الـ serial_core.h** يعرّف `struct uart_port` اللي يحتوي على معلومات الـ port زي رقمه وسرعة الـ clock. **الـ tty.h** يعرّف `struct ktermios` اللي فيه الإعدادات الجديدة المطلوبة.

---

#### الـ pre_handler

```c
static int meson_termios_pre_handler(struct kprobe *p, struct pt_regs *regs)
```

هذه الدالة تُشغَّل قبل `meson_uart_set_termios` بميكروثانية واحدة. الـ kernel يمررلها `pt_regs` وهو "صورة" من الـ CPU registers في لحظة الاستدعاء — من هناك نسرق الـ arguments لأنها لا تزال في الـ registers قبل ما تُنسخ للـ stack frame.

الـ `#ifdef` على المعمارية ضروري لأن اتفاقية تمرير الـ arguments (calling convention) مختلفة بين x86_64 وarm64 — على x86_64 أول argument في `rdi` وثاني argument في `rsi`، وعلى arm64 يكونون في `x0` و`x1`.

نستخرج من `c_cflag`:
- **CSIZE**: عدد الـ data bits (5-8)
- **PARENB / PARODD**: نوع الـ parity
- **CSTOPB**: هل عندنا stop bit واحد أو اثنين

ونطبع كل هذا مع رقم الـ port وسرعة الـ clock بـ `pr_info` — النتيجة تظهر في `/var/log/kern.log` أو `dmesg`.

---

#### struct kprobe

```c
static struct kprobe meson_termios_kp = {
	.symbol_name = "meson_uart_set_termios",
	.pre_handler = meson_termios_pre_handler,
};
```

هذا هو "العقد" مع نظام الـ kprobes. نقول: "ابحث عن الدالة باسمها في جدول الـ symbols (kallsyms)، وعندما يصلها أي thread، نفّذ `pre_handler` أولاً". الـ kernel يعرف عنوان الدالة تلقائيًا لأن `CONFIG_KALLSYMS=y` موجود في أي kernel حديث.

---

#### module_init

```c
ret = register_kprobe(&meson_termios_kp);
```

`register_kprobe` تضع **breakpoint** (instruction INT3 على x86 أو BRK على arm64) عند أول byte من الدالة المستهدفة. عندما يصل أي thread لهذا الـ breakpoint، الـ CPU يرفع exception، الـ kernel يعالجه، يستدعي `pre_handler` الخاص بنا، ثم يُكمل تنفيذ الدالة الأصلية. من وجهة نظر المستخدم — كل شيء يحدث بشفافية تامة.

---

#### module_exit

```c
unregister_kprobe(&meson_termios_kp);
```

إزالة الـ kprobe ضرورية جدًا في الـ exit. إذا تركنا الـ breakpoint بعد تفريغ الـ module، أي استدعاء لـ `set_termios` سيؤدي لـ kernel panic لأن `pre_handler` ستشير لكود محذوف من الذاكرة. هذا مثل إخراج شبكة الصيد قبل مغادرة البحر — لازم تجمعها.

---

### كيف تبني وتشغّل

```bash
# Makefile بسيط
obj-m += meson_termios_spy.o

# بناء
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل
sudo insmod meson_termios_spy.ko

# تفعيل تغيير في الـ termios (مثلاً بـ stty على /dev/ttyAML0)
stty -F /dev/ttyAML0 115200

# مشاهدة النتيجة
dmesg | tail -20

# تفريغ
sudo rmmod meson_termios_spy
```

**مثال على الـ output المتوقع في dmesg:**

```
[  142.331207] meson_termios_spy: planted kprobe at meson_uart_set_termios (ffff800008a3c120)
[  143.887442] meson_termios_spy: port0 → 115200 baud | 8none1 | clk=24000000 Hz
[  144.012891] meson_termios_spy: port0 → 9600 baud | 7even1 | clk=24000000 Hz
```

---

### ملاحظة على الـ static symbol

الدالة `meson_uart_set_termios` معلَّنة بـ `static` في الملف الأصلي، لكن الـ kprobes لا يهمه ذلك — يتعامل مع العنوان الفيزيائي مباشرة عبر kallsyms. الـ `static` يخفي الدالة من linker الخارجي فقط، لكن داخل الـ kernel image اسمها موجود في `/proc/kallsyms` (بشرط `CONFIG_KALLSYMS_ALL=y`). تقدر تتحقق بـ:

```bash
sudo grep meson_uart_set_termios /proc/kallsyms
```
