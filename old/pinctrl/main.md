# Linux Pinctrl Subsystem — ELI5 Guide

> A concise, comprehensive explanation of the Linux `pinctrl` subsystem for embedded Linux engineers.

---

## Table of Contents

1. [The Problem It Solves](https://claude.ai/chat/pinctrl-overview.md#the-problem-it-solves)
2. [The Three Jobs of Pinctrl](https://claude.ai/chat/pinctrl-concepts.md)
3. [Key Concepts & Vocabulary](https://claude.ai/chat/pinctrl-concepts.md#key-concepts)
4. [Architecture & Code Flow](https://claude.ai/chat/pinctrl-architecture.md)
5. [Device Tree Usage](https://claude.ai/chat/pinctrl-devicetree.md)
6. [Driver Development](https://claude.ai/chat/pinctrl-driver.md)
7. [Debugging](https://claude.ai/chat/pinctrl-debug.md)

---

## The Problem It Solves

Modern SoCs (like your RK3562 or STM32MP1) have **hundreds of physical pins**. Each pin can do multiple things:

```
Pin 42 ──► UART2_TX
       ──► I2C1_SDA
       ──► PWM3
       ──► GPIO2_A2   ← default fallback
```

The hardware has internal **multiplexers** (mux registers) that switch a pin between these roles. Without a subsystem managing this, every driver would need to know the SoC's exact register layout — a maintenance nightmare.

**Pinctrl is the single kernel subsystem that:**

- Owns all pin hardware on a given SoC
- Lets peripheral drivers claim the pins they need
- Programs the mux and configuration registers transparently

---

## Bird's Eye View

```
  ┌─────────────────────────────────────────────────────────┐
  │                      Your Driver                         │
  │       devm_pinctrl_get_select_default(dev)               │
  └──────────────────────────┬──────────────────────────────┘
                             │  "I need my default pin state"
                             ▼
  ┌─────────────────────────────────────────────────────────┐
  │                   Pinctrl Core                           │
  │   (drivers/pinctrl/core.c)                               │
  │   Looks up pin group → function → state mapping          │
  └──────────────────────────┬──────────────────────────────┘
                             │  calls ops->set_mux() etc.
                             ▼
  ┌─────────────────────────────────────────────────────────┐
  │              SoC Pinctrl Driver                          │
  │   (e.g. drivers/pinctrl/rockchip/pinctrl-rockchip.c)    │
  │   Writes actual mux + config registers on the hardware   │
  └─────────────────────────────────────────────────────────┘
```

---

## Files in This Guide

|File|Contents|
|---|---|
|`pinctrl-overview.md`|This file — big picture|
|`pinctrl-concepts.md`|Vocabulary, the 3 jobs, mental models|
|`pinctrl-architecture.md`|Kernel internals, structs, call flow|
|`pinctrl-devicetree.md`|DT bindings, writing pin states|
|`pinctrl-driver.md`|Writing an SoC pinctrl driver|
|`pinctrl-debug.md`|Debugfs, common issues, tips|
# Pinctrl — Concepts & Vocabulary

---

## The Three Jobs of Pinctrl

### Job 1 — Pin Muxing (`pinmux`)

> _"Which hardware function does this pin serve?"_

Each pin connects to an internal mux register. Writing a value to that register routes the pin's signal to a specific peripheral.

```
         ┌────────────┐
Pin 42 ──►  MUX REG   ├──► UART2_TX   (value = 0x1)
         │  0x...04c  ├──► I2C1_SDA   (value = 0x2)
         │            ├──► PWM3       (value = 0x3)
         │            └──► GPIO       (value = 0x0)
         └────────────┘
```

Pinctrl writes these mux registers **on behalf of** the peripheral driver. The UART driver never touches a mux register directly.

---

### Job 2 — Pin Configuration (`pinconf`)

> _"How should this pin behave electrically?"_

Separate from muxing, each pin has electrical properties:

|Setting|Options|
|---|---|
|Pull resistor|pull-up / pull-down / none|
|Drive strength|2mA / 4mA / 8mA / 12mA (SoC-specific)|
|Input/output type|push-pull / open-drain|
|Schmitt trigger|enable / disable|
|Slew rate|fast / slow|

These map to `PIN_CONFIG_*` constants in `include/linux/pinctrl/pinconf-generic.h`.

---

### Job 3 — GPIO Fallback

> _"When no peripheral wants it, it's a plain GPIO."_

The pinctrl and GPIO subsystems are tightly coupled. A pin not claimed by any peripheral remains under GPIO control. The `gpiochip` for each bank is registered **by the same pinctrl driver**.

```
Peripheral driver probes → claims pin via pinctrl → pin leaves GPIO pool
Peripheral driver removes → releases pin → pin returns to GPIO pool
```

---

## Key Vocabulary

|Term|ELI5 meaning|
|---|---|
|**pin**|One physical pad/ball on the SoC package|
|**pin number**|Kernel-internal ID for that pad (not the package ball number)|
|**pin group**|A named set of pins that together implement one function (e.g. `uart2-pins` = TX + RX + RTS + CTS)|
|**function**|The peripheral role a group can be muxed to (e.g. `uart2`, `i2c1`, `spi0`)|
|**state**|A named snapshot of mux + config for a device (e.g. `default`, `sleep`, `idle`)|
|**consumer**|Any driver that calls `devm_pinctrl_get()` to request pin states|
|**provider**|The SoC pinctrl driver that owns the hardware registers|
|**hog**|A pin state activated at boot by the pinctrl driver itself, without a consumer driver|

---

## Mental Model: States

Think of pin states like **power profiles** for a device's pins:

```
UART node
  ├── state "default"  → TX/RX muxed, pull-up on RX, 8mA drive
  ├── state "sleep"    → pins set to GPIO input, pull-down (save power)
  └── state "idle"     → TX/RX muxed but lower drive strength
```

The driver switches states by calling:

```c
pinctrl_select_state(pinctrl, state_sleep);
```

The kernel automatically selects `default` on driver probe and `sleep` on runtime suspend (if `pinctrl-pm` is used).

---

## The Ownership Rule

> **A pin can only be owned by one consumer at a time.**

If two drivers try to claim the same pin, the second one gets `-EBUSY`. This is enforced by the pinctrl core's pin request tracking.

```
Driver A claims Pin 42 for UART2 ✓
Driver B tries to claim Pin 42 for I2C1 ✗  → -EBUSY
```

# Pinctrl — Architecture & Kernel Internals

---

## Key Data Structures

```
struct pinctrl_dev          ← one per SoC pinctrl driver instance
  ├── struct pinctrl_desc   ← describes all pins, groups, functions, ops
  │     ├── struct pinctrl_pin_desc[]   ← array of all pins (name + number)
  │     ├── struct pinctrl_ops          ← get_groups, get_group_pins, dt_node_to_map
  │     ├── struct pinmux_ops           ← get_functions, set_mux, gpio_request_enable
  │     └── struct pinconf_ops          ← pin_config_get, pin_config_set
  ├── struct pinctrl_gpio_range[]       ← maps GPIO ranges to this controller
  └── radix_tree of pin requests        ← tracks which pins are claimed
```

---

## Call Flow: Driver Probe

```
1. uart_probe()
   │
2. └─► devm_pinctrl_get(dev)
         │  Pinctrl core finds this dev's node in DT
         │  Parses all pinctrl-N / pinctrl-names properties
         │  Calls provider's .dt_node_to_map() for each state
         │  Returns struct pinctrl* handle
         │
3.    └─► pinctrl_select_state(p, "default")
            │  Iterates all map entries for this state
            │  For each entry:
            │    pinmux_ops->set_mux(pctldev, func_selector, group_selector)
            │    pinconf_ops->pin_config_set(pctldev, pin, configs, nconfigs)
            │
4.          └─► SoC driver writes registers
                  e.g. rockchip_set_mux(bank, pin, mux_val)
                       rockchip_set_drive(bank, pin, drive_val)
```

---

## The `pinctrl_ops` Interface

```c
struct pinctrl_ops {
    /* How many groups does this SoC have? */
    int (*get_groups_count)(struct pinctrl_dev *pctldev);

    /* Name of group N */
    const char *(*get_group_name)(struct pinctrl_dev *pctldev, unsigned selector);

    /* Which pins are in group N? */
    int (*get_group_pins)(struct pinctrl_dev *pctldev, unsigned selector,
                          const unsigned **pins, unsigned *num_pins);

    /* Parse a DT node into a pinctrl_map array */
    int (*dt_node_to_map)(struct pinctrl_dev *pctldev,
                          struct device_node *np_config,
                          struct pinctrl_map **map, unsigned *num_maps);

    void (*dt_free_map)(struct pinctrl_dev *pctldev,
                        struct pinctrl_map *map, unsigned num_maps);
};
```

---

## The `pinmux_ops` Interface

```c
struct pinmux_ops {
    int  (*get_functions_count)(struct pinctrl_dev *pctldev);
    const char *(*get_function_name)(struct pinctrl_dev *pctldev, unsigned selector);
    int  (*get_function_groups)(struct pinctrl_dev *pctldev, unsigned selector,
                                const char * const **groups, unsigned *num_groups);

    /* THE most important one — programs the mux register */
    int  (*set_mux)(struct pinctrl_dev *pctldev,
                    unsigned func_selector, unsigned group_selector);

    /* Called when a GPIO consumer requests a pin */
    int  (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                                struct pinctrl_gpio_range *range, unsigned offset);
    void (*gpio_disable_free)(struct pinctrl_dev *pctldev,
                              struct pinctrl_gpio_range *range, unsigned offset);
};
```

---

## The `pinctrl_map` — The Bridge

The DT parser (`.dt_node_to_map`) converts DT nodes into an array of `pinctrl_map` entries. Each entry is one of:

|Type|Meaning|
|---|---|
|`PIN_MAP_TYPE_MUX_GROUP`|Set this group to this function|
|`PIN_MAP_TYPE_CONFIGS_PIN`|Apply config settings to one pin|
|`PIN_MAP_TYPE_CONFIGS_GROUP`|Apply config settings to all pins in a group|

```c
/* Example map entry for UART2 default state */
{
    .dev_name = "ff000000.serial",
    .name     = PINCTRL_STATE_DEFAULT,
    .type     = PIN_MAP_TYPE_MUX_GROUP,
    .ctrl_dev_name = "pinctrl",
    .data.mux = {
        .group    = "uart2m0-pins",
        .function = "uart2",
    }
}
```

---

## Registration (SoC Driver Side)

```c
/* In your SoC's pinctrl driver probe function */
static const struct pinctrl_desc my_pinctrl_desc = {
    .name    = "my-pinctrl",
    .pins    = my_pins,          /* array of pinctrl_pin_desc */
    .npins   = ARRAY_SIZE(my_pins),
    .pctlops = &my_pctrl_ops,
    .pmxops  = &my_pmx_ops,
    .confops = &my_conf_ops,
    .owner   = THIS_MODULE,
};

pctldev = devm_pinctrl_register(dev, &my_pinctrl_desc, priv);
```

---

## Generic Pin Config

Instead of writing custom `pinconf_ops` for common settings, most drivers use the **generic pinconf** framework:

```c
#include <linux/pinctrl/pinconf-generic.h>

/* In your .dt_node_to_map, parse generic properties: */
pinconf_generic_parse_dt_config(np, pctldev, &configs, &nconfigs);

/* In your pinconf_ops->pin_config_set: */
for (i = 0; i < num_configs; i++) {
    param = pinconf_to_config_param(configs[i]);
    arg   = pinconf_to_config_argument(configs[i]);

    switch (param) {
    case PIN_CONFIG_BIAS_PULL_UP:   /* set pull-up register */ break;
    case PIN_CONFIG_DRIVE_STRENGTH: /* set drive register */   break;
    /* ... */
    }
}
```

# Pinctrl — Device Tree Usage

---

## Two Sides of the DT

There are **two separate places** you write pinctrl DT nodes:

|Where|What it does|
|---|---|
|**Provider node** (SoC dtsi)|Defines all available pin groups and their configs|
|**Consumer node** (board dts)|References those groups for specific devices|

---

## Provider Side — Defining Pin Groups

This lives in the SoC's `.dtsi` file (or your board's pinctrl node):

```dts
/* RK3562 example */
&pinctrl {

    uart2 {
        /* Group name: "uart2m0-pins" */
        uart2m0_pins: uart2m0-pins {
            rockchip,pins =
                /* pin  mux  pinconfig */
                <1 RK_PB4 2 &pcfg_pull_up>,   /* UART2_RX */
                <1 RK_PB5 2 &pcfg_pull_none>;  /* UART2_TX */
        };
    };

    i2c1 {
        i2c1m0_pins: i2c1m0-pins {
            rockchip,pins =
                <0 RK_PB6 1 &pcfg_pull_up>,   /* I2C1_SDA */
                <0 RK_PB7 1 &pcfg_pull_up>;   /* I2C1_SCL */
        };
    };

    /* Reusable config templates */
    pcfg_pull_up: pcfg-pull-up {
        bias-pull-up;
    };

    pcfg_pull_none: pcfg-pull-none {
        bias-disable;
    };

    pcfg_drive_8ma: pcfg-drive-8ma {
        drive-strength = <8>;
    };
};
```

---

## Consumer Side — Using Pin Groups

This lives in your board `.dts` or device node:

```dts
&uart2 {
    status = "okay";

    /* State name → phandle to pin group */
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart2m0_pins>;           /* "default" state */
    pinctrl-1 = <&uart2m0_sleep_pins>;     /* "sleep" state */
};
```

### The Naming Convention

```
pinctrl-names = "default", "sleep", "idle", "mystate", ...
                    │          │
                    │          └── pinctrl-1 = <&...>
                    └── pinctrl-0 = <&...>
```

- `pinctrl-0` maps to the first name in `pinctrl-names`
- `pinctrl-1` maps to the second, and so on
- `"default"` is automatically selected at driver probe
- `"sleep"` is automatically selected on runtime suspend (if `pinctrl-pm` is wired up)

---

## Common Generic Config Properties

These are standardized across all SoCs (from `pinctrl/pinctrl-bindings.txt`):

```dts
my_pin_config: my-pin-config {
    bias-disable;               /* no pull resistor */
    bias-pull-up;               /* pull-up */
    bias-pull-down;             /* pull-down */
    bias-pull-up = <50000>;     /* pull-up, 50kΩ (if SoC supports it) */

    drive-strength = <4>;       /* 4 mA */
    drive-open-drain;           /* open-drain output */
    drive-push-pull;            /* push-pull output */

    input-enable;               /* force input mode */
    output-enable;              /* force output mode */
    output-high;                /* set output high */
    output-low;                 /* set output low */

    input-schmitt-enable;       /* Schmitt trigger */
    slew-rate = <0>;            /* slow slew rate */
};
```

---

## Pin Hogging

A **hog** activates a pin state at pinctrl driver probe time, _before_ any consumer driver loads. Useful for pins that have no driver (e.g. a hardware enable line, an LED).

```dts
&pinctrl {
    /* This group is "hogged" — activated immediately at boot */
    my_hog: my-hog {
        pinctrl-single,pins = <...>;
        pinctrl-names = "default";  /* required for hog */
    };
};
```

Alternatively with the standard hog mechanism:

```dts
&pinctrl {
    my_hog_group: my-hog-group {
        mux {
            groups = "my_pins";
            function = "gpio";
        };
        conf {
            pins = "PIN_A0";
            output-high;
        };
    };
};

/* In the pinctrl node itself, add: */
pinctrl-0 = <&my_hog_group>;
pinctrl-names = "default";
```

---

## Multiple Groups in One State

A single state can activate **multiple pin groups**:

```dts
&spi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&spi0_clk_pins
                 &spi0_mosi_pins
                 &spi0_miso_pins
                 &spi0_cs0_pin>;
};
```

All four groups are applied atomically when `"default"` is selected.

# Pinctrl — Writing an SoC Driver

> Minimal skeleton for bringing up a new SoC's pinctrl driver.

---

## Minimal Driver Skeleton

```c
#include <linux/pinctrl/pinctrl.h>
#include <linux/pinctrl/pinmux.h>
#include <linux/pinctrl/pinconf.h>
#include <linux/pinctrl/pinconf-generic.h>

/* ── 1. Describe every pin ─────────────────────────────────────── */

static const struct pinctrl_pin_desc my_pins[] = {
    PINCTRL_PIN(0,  "PA0"),
    PINCTRL_PIN(1,  "PA1"),
    /* ... */
    PINCTRL_PIN(63, "PB31"),
};

/* ── 2. Describe pin groups ─────────────────────────────────────── */

static const unsigned int uart2_pins[] = { 10, 11 };  /* PA10=TX, PA11=RX */
static const unsigned int i2c1_pins[]  = { 20, 21 };

static const struct my_group_desc {
    const char       *name;
    const unsigned   *pins;
    unsigned          npins;
} my_groups[] = {
    { "uart2-grp", uart2_pins, ARRAY_SIZE(uart2_pins) },
    { "i2c1-grp",  i2c1_pins,  ARRAY_SIZE(i2c1_pins)  },
};

/* ── 3. Describe functions ──────────────────────────────────────── */

static const char * const uart2_groups[] = { "uart2-grp" };
static const char * const i2c1_groups[]  = { "i2c1-grp"  };

static const struct my_func_desc {
    const char        *name;
    const char *const *groups;
    unsigned           ngroups;
} my_functions[] = {
    { "uart2", uart2_groups, ARRAY_SIZE(uart2_groups) },
    { "i2c1",  i2c1_groups,  ARRAY_SIZE(i2c1_groups)  },
};

/* ── 4. Implement pinctrl_ops ───────────────────────────────────── */

static int my_get_groups_count(struct pinctrl_dev *pctldev)
{
    return ARRAY_SIZE(my_groups);
}

static const char *my_get_group_name(struct pinctrl_dev *pctldev, unsigned sel)
{
    return my_groups[sel].name;
}

static int my_get_group_pins(struct pinctrl_dev *pctldev, unsigned sel,
                              const unsigned **pins, unsigned *npins)
{
    *pins  = my_groups[sel].pins;
    *npins = my_groups[sel].npins;
    return 0;
}

static const struct pinctrl_ops my_pctrl_ops = {
    .get_groups_count  = my_get_groups_count,
    .get_group_name    = my_get_group_name,
    .get_group_pins    = my_get_group_pins,
    .dt_node_to_map    = pinconf_generic_dt_node_to_map_all,  /* use generic */
    .dt_free_map       = pinconf_generic_dt_free_map,
};

/* ── 5. Implement pinmux_ops ────────────────────────────────────── */

static int my_get_functions_count(struct pinctrl_dev *pctldev)
{
    return ARRAY_SIZE(my_functions);
}

static const char *my_get_function_name(struct pinctrl_dev *pctldev, unsigned sel)
{
    return my_functions[sel].name;
}

static int my_get_function_groups(struct pinctrl_dev *pctldev, unsigned sel,
                                   const char *const **groups, unsigned *ngroups)
{
    *groups  = my_functions[sel].groups;
    *ngroups = my_functions[sel].ngroups;
    return 0;
}

static int my_set_mux(struct pinctrl_dev *pctldev,
                      unsigned func_sel, unsigned grp_sel)
{
    struct my_priv *priv = pinctrl_dev_get_drvdata(pctldev);
    const struct my_group_desc *grp = &my_groups[grp_sel];
    int i;

    for (i = 0; i < grp->npins; i++) {
        unsigned pin = grp->pins[i];
        /* Write mux value into SoC register */
        my_write_mux_reg(priv, pin, func_sel);
    }
    return 0;
}

static const struct pinmux_ops my_pmx_ops = {
    .get_functions_count = my_get_functions_count,
    .get_function_name   = my_get_function_name,
    .get_function_groups = my_get_function_groups,
    .set_mux             = my_set_mux,
    .strict              = true,  /* disallow GPIO + peripheral sharing */
};

/* ── 6. Implement pinconf_ops ───────────────────────────────────── */

static int my_pin_config_set(struct pinctrl_dev *pctldev, unsigned pin,
                              unsigned long *configs, unsigned nconfigs)
{
    struct my_priv *priv = pinctrl_dev_get_drvdata(pctldev);
    int i;

    for (i = 0; i < nconfigs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        u32 arg = pinconf_to_config_argument(configs[i]);

        switch (param) {
        case PIN_CONFIG_BIAS_PULL_UP:
            my_set_pull(priv, pin, MY_PULL_UP);
            break;
        case PIN_CONFIG_BIAS_PULL_DOWN:
            my_set_pull(priv, pin, MY_PULL_DOWN);
            break;
        case PIN_CONFIG_BIAS_DISABLE:
            my_set_pull(priv, pin, MY_PULL_NONE);
            break;
        case PIN_CONFIG_DRIVE_STRENGTH:
            my_set_drive(priv, pin, arg);
            break;
        default:
            return -ENOTSUPP;
        }
    }
    return 0;
}

static const struct pinconf_ops my_conf_ops = {
    .is_generic      = true,
    .pin_config_get  = my_pin_config_get,
    .pin_config_set  = my_pin_config_set,
};

/* ── 7. Register ────────────────────────────────────────────────── */

static const struct pinctrl_desc my_pinctrl_desc = {
    .name    = "my-pinctrl",
    .pins    = my_pins,
    .npins   = ARRAY_SIZE(my_pins),
    .pctlops = &my_pctrl_ops,
    .pmxops  = &my_pmx_ops,
    .confops = &my_conf_ops,
    .owner   = THIS_MODULE,
};

static int my_pinctrl_probe(struct platform_device *pdev)
{
    struct my_priv *priv;
    struct pinctrl_dev *pctldev;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    priv->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(priv->base))
        return PTR_ERR(priv->base);

    pctldev = devm_pinctrl_register(&pdev->dev, &my_pinctrl_desc, priv);
    return PTR_ERR_OR_ZERO(pctldev);
}
```

---

## Common Patterns

### Using `pinctrl-single` for Simple SoCs

If your SoC has a single register per pin (1 reg = mux + config), you may be able to reuse the existing `pinctrl-single` driver instead of writing your own:

```dts
pinctrl: pinctrl@40010000 {
    compatible = "pinctrl-single";
    reg = <0x40010000 0x200>;
    #pinctrl-cells = <2>;
    pinctrl-single,register-width = <32>;
    pinctrl-single,function-mask = <0xf>;
};
```

### GPIO Range Registration

```c
static struct pinctrl_gpio_range my_gpio_range = {
    .name  = "my-gpio",
    .id    = 0,
    .base  = 0,       /* first GPIO number in kernel */
    .pin_base = 0,    /* first pin number in pinctrl */
    .npins = 64,
    .gc    = &my_gpiochip,
};

pinctrl_add_gpio_range(pctldev, &my_gpio_range);
```

# Pinctrl — Debugging

---

## Debugfs Interface

Mount debugfs and explore pinctrl state at runtime:

```bash
mount -t debugfs none /sys/kernel/debug   # usually auto-mounted

ls /sys/kernel/debug/pinctrl/
# pinctrl-handles   ← active consumer handles and their states
# <controller-name>/
#   pins            ← all pins and their current state
#   pingroups       ← all groups and their pins
#   pinmux-functions
#   pinmux-pins     ← which pins are muxed to what
#   gpio-ranges
```

### Most Useful Commands

```bash
# See ALL pins and their mux state
cat /sys/kernel/debug/pinctrl/<ctrl>/pins

# See which consumer owns which pins
cat /sys/kernel/debug/pinctrl/pinctrl-handles

# See pinmux assignments
cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-pins

# Example output:
# pin 42 (PA10): uart2 [REQUESTED] function uart2 group uart2-grp
# pin 43 (PA11): uart2 [REQUESTED] function uart2 group uart2-grp
# pin 20 (PA20): (MUX UNCLAIMED) (GPIO UNCLAIMED)
```

---

## Common Errors and Fixes

### `pinctrl: pin PA10 already requested by uart2; cannot claim for spi0`

**Cause:** Two devices are trying to use the same pin.  
**Fix:** Check your DT — two nodes reference the same pin in their `pinctrl-0`. One of them is wrong.

---

### `pinctrl: could not find group uart2-grp`

**Cause:** The consumer references a group that doesn't exist in the provider.  
**Fix:** Check the phandle target — the group name in the provider node doesn't match what the consumer is pointing to, or the provider's `dt_node_to_map` failed silently.

---

### `deferred probe` because of pinctrl

**Cause:** The consumer device probed before the pinctrl provider was registered.  
**Fix:** This is normal and self-correcting — the driver framework will retry. If it loops forever, the pinctrl driver itself may have failed to probe. Check `dmesg | grep pinctrl`.

---

### Pin stuck in wrong mux after suspend/resume

**Cause:** Sleep state not defined, or `pinctrl-pm` not connected.  
**Fix:** Add a `"sleep"` state to the consumer's DT node, or ensure the driver calls `pinctrl_pm_select_sleep_state()` in its `.suspend()` callback.

---

### Pin config not applied (wrong pull, wrong drive)

**Cause:** `pinconf_ops->pin_config_set` returns `-ENOTSUPP` for the requested config param.  
**Fix:** Add handling for that `PIN_CONFIG_*` param in your driver's `pin_config_set`.

---

## Useful Kernel Config Options

```
CONFIG_PINCTRL=y                # Core subsystem
CONFIG_PINMUX=y                 # Mux support
CONFIG_PINCONF=y                # Config support
CONFIG_GENERIC_PINCONF=y        # Generic config framework
CONFIG_DEBUG_PINCTRL=y          # Extra debug output
CONFIG_PINCTRL_ROCKCHIP=y       # RK3562/RK3588 driver
CONFIG_PINCTRL_STM32=y          # STM32MP1 driver
```

---

## Quick Reference: Consumer API

```c
/* Get a handle (call once in probe) */
struct pinctrl *p = devm_pinctrl_get(dev);

/* Get a state handle */
struct pinctrl_state *s = pinctrl_lookup_state(p, "default");

/* Select a state */
int ret = pinctrl_select_state(p, s);

/* All-in-one: get + select default */
int ret = pinctrl_pm_select_default_state(dev);  /* uses PM helper */

/* Or the classic one-liner in probe */
ret = devm_pinctrl_get_select_default(dev);  /* get + select "default" */
```

---

## Boot-Time Log Hints

```
# Good — pinctrl driver registered successfully
[    0.123] my-pinctrl 40010000.pinctrl: registered 64 pins

# Good — consumer claimed its pins
[    1.456] pinctrl core: init pin 42 PA10

# Bad — provider missing
[    1.457] uart2: error -517 (EPROBE_DEFER) getting pinctrl

# Bad — conflict
[    1.458] pinctrl: pin PA10 already requested
```