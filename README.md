
```bash
# vscode
$ code

# Dev containers extension: search bar: type:
> Dev containers: reopen in container


# Then open new termainal
$ claude --dangerously-skip-permissions
```


## `OPEN FIRMWARE` ⭐✅
```
OPEN FIRMWARE AND FLATTENED DEVICE TREE
M:	Rob Herring <robh@kernel.org>
M:	Saravana Kannan <saravanak@kernel.org>
L:	devicetree@vger.kernel.org
S:	Maintained
Q:	http://patchwork.kernel.org/project/devicetree/list/
W:	http://www.devicetree.org/
C:	irc://irc.libera.chat/devicetree
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/robh/linux.git
F:	Documentation/ABI/testing/sysfs-firmware-ofw
F:	drivers/of/
F:	include/linux/of*.h
F:	rust/helpers/of.c
F:	rust/kernel/of.rs
F:	scripts/dtc/
F:	tools/testing/selftests/dt/
K:	of_overlay_notifier_
K:	of_overlay_fdt_apply
K:	of_overlay_remove

OPEN FIRMWARE AND FLATTENED DEVICE TREE BINDINGS
M:	Rob Herring <robh@kernel.org>
M:	Krzysztof Kozlowski <krzk+dt@kernel.org>
M:	Conor Dooley <conor+dt@kernel.org>
L:	devicetree@vger.kernel.org
S:	Maintained
Q:	http://patchwork.kernel.org/project/devicetree/list/
C:	irc://irc.libera.chat/devicetree
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/robh/linux.git
F:	Documentation/devicetree/
F:	arch/*/boot/dts/
F:	include/dt-bindings/
```

- [ ] [of.h](include/linux/of.h.md)
- [ ] [of_dma.h](include/linux/of_dma.h.md)
- [ ] [of_net.h](external/linux/include/linux/of_net.h.md)
- [ ] [of_pdt.h](include/linux/of_pdt.h.md)
- [ ] [of_gpio.h](include/linux/of_gpio.h.md)
- [ ] [of_mdio.h](external/linux/include/linux/of_mdio.h.md)
- [ ] [of_platform.h](include/linux/of_platform.h.md)
- [ ] [of_iommu.h](include/linux/of_iommu.h.md)
- [ ] [of_clk.h](include/linux/of_clk.h.md)
- [ ] [of_device.h](include/linux/of_device.h.md)
- [ ] [of_fdt.h](external/linux/include/linux/of_fdt.h.md)
- [ ] [of_irq.h](include/linux/of_irq.h.md)
- [ ] [of_address.h](include/linux/of_address.h.md)
- [ ] [of_graph.h](include/linux/of_graph.h.md)
- [ ] [of_reserved_mem.h](include/linux/of_reserved_mem.h.md)



## `GPIO SUBSYSTEM` ✅⭐
```
M:	Linus Walleij <linusw@kernel.org>
M:	Bartosz Golaszewski <brgl@kernel.org>
L:	linux-gpio@vger.kernel.org
S:	Maintained
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/brgl/linux.git
F:	Documentation/admin-guide/gpio/
	SMA: https://docs.kernel.org/admin-guide/gpio/
F:	Documentation/devicetree/bindings/gpio/
	SMA: Not exist
F:	Documentation/driver-api/gpio/
	SMA: https://docs.kernel.org/driver-api/gpio/
F:	drivers/gpio/
F:	include/dt-bindings/gpio/
F:	include/linux/gpio.h
F:	include/linux/gpio/
F:	include/linux/of_gpio.h
K:	(devm_)?gpio_(request|free|direction|get|set)
K:	GPIOD_FLAGS_BIT_NONEXCLUSIVE
K:	devm_gpiod_unhinge
```

- [ ] [gpio.h](include/linux/gpio.h.md)
- [ ] [of_gpio.h](include/linux/of_gpio.h.md)

- [ ] [aspeed.h](include/linux/aspeed.h.md)
- [ ] [consumer.h](include/linux/consumer.h.md)
- [ ] [driver.h](include/linux/driver.h.md)
- [ ] [machine.h](include/linux/pintrl/machine.h.md)
- [ ] [forwarder.h](include/linux/forwarder.h.md)
- [ ] [generic.h](include/linux/generic.h.md)
- [ ] [gpio-nomadik.h](include/linux/gpio-nomadik.h.md)
- [ ] [gpio-reg.h](include/linux/gpio-reg.h.md)
- [ ] [property.h](include/linux/property.h.md)
- [ ] [regmap.h](include/linux/regmap.h.md)

- [ ] [gpiolib.c](drivers/gpio/gpiolib.c.md)
- [ ] [gpiolib-cdev.h](drivers/gpio/gpiolib-cdev.h.md)
- [ ] [gpiolib-cdev.h](drivers/gpio/gpiolib-cdev.h.md)
- [ ] [gpiolib-of.h](drivers/gpio/gpiolib-of.h.md)
- [ ] [gpiolib-shared.h](drivers/gpio/gpiolib-shared.h.md)
- [ ] [gpiolib-sysfs.h](drivers/gpio/gpiolib-sysfs.h.md)
- [ ] [gpio-regmap.c](drivers/gpio/gpio-regmap.c.md)
- [ ] [gpio-generic.c](drivers/gpio/gpio-generic.c.md)
- [ ] [gpio-mmio.c](drivers/gpio/gpio-mmio.c.md)
- [ ] [gpiolib.h](drivers/gpio/gpiolib.h.md)


## `I2C SUBSYSTEM`

```
I2C SUBSYSTEM
M:	Wolfram Sang <wsa+renesas@sang-engineering.com>
L:	linux-i2c@vger.kernel.org
S:	Maintained
W:	https://i2c.wiki.kernel.org/
Q:	https://patchwork.ozlabs.org/project/linux-i2c/list/
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/wsa/linux.git
F:	Documentation/i2c/
F:	drivers/i2c/*
F:	include/dt-bindings/i2c/i2c.h
F:	include/linux/i2c-dev.h
F:	include/linux/i2c-smbus.h
F:	include/linux/i2c.h
F:	include/uapi/linux/i2c-*.h
F:	include/uapi/linux/i2c.h

I2C SUBSYSTEM [RUST]
M:	Igor Korotin <igor.korotin.linux@gmail.com>
R:	Danilo Krummrich <dakr@kernel.org>
R:	Daniel Almeida <daniel.almeida@collabora.com>
L:	rust-for-linux@vger.kernel.org
S:	Maintained
F:	rust/kernel/i2c.rs
F:	samples/rust/rust_driver_i2c.rs
F:	samples/rust/rust_i2c_client.rs

I2C SUBSYSTEM HOST DRIVERS
M:	Andi Shyti <andi.shyti@kernel.org>
L:	linux-i2c@vger.kernel.org
S:	Maintained
W:	https://i2c.wiki.kernel.org/
Q:	https://patchwork.ozlabs.org/project/linux-i2c/list/
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/andi.shyti/linux.git
F:	Documentation/devicetree/bindings/i2c/
F:	drivers/i2c/algos/
F:	drivers/i2c/busses/
F:	include/dt-bindings/i2c/
```

- [ ] [include/dt-bindings/i2c/i2c.h](include/dt-bindings/i2c/i2c.h.md)
- [ ] [i2c-dev.h](include/linux/i2c-dev.h.md)
- [ ] [i2c-smbus.h](include/linux/i2c-smbus.h.md)
- [ ] [include/linux/i2c.h](include/linux/i2c.h.md)
## `SPI SUBSYSTEM` ✅⭐

```
M:	Mark Brown <broonie@kernel.org>
L:	linux-spi@vger.kernel.org
S:	Maintained
Q:	http://patchwork.kernel.org/project/spi-devel-general/list/
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/broonie/spi.git
F:	Documentation/devicetree/bindings/spi/
F:	Documentation/spi/
F:	drivers/spi/
F:	include/trace/events/spi*
F:	include/linux/spi/
F:	include/uapi/linux/spi/
F:	tools/spi/
```

- [ ] [index.rst](Documentation/spi/index.rst.md)
- [ ] [butterfly.rst](Documentation/spi/butterfly.rst.md)
- [ ] [spidev.rst](Documentation/spi/spidev.rst.md)
- [ ] [spi-summary.rst](Documentation/spi/spi-summary.rst.md)

- [ ] [ad7877.h](include/linux/spi/ad7877.h.md)
- [ ] [eeprom.h](include/linux/spi/eeprom.h.md)
- [ ] [flash.h](include/linux/spi/flash.h.md)
- [ ] [mmc_spi.h](include/linux/spi/mmc_spi.h.md)
- [ ] [spi_gpio.h](include/linux/spi/spi_gpio.h.md)
- [ ] [spi.h](include/linux/spi/spi.h.md)
- [ ] [spi-mem.h](include/linux/spi/spi-mem.h.md)
- [ ] [spi.h](include/uapi/linux/spi/spi.h.md)
- [ ] [spidev.h](include/uapi/linux/spi/spidev.h.md)

- [ ] [spi.c](drivers/spi/spi.c.md)
- [ ] [spi-mem.c](drivers/spi/spi-mem.c.md)
- [ ] [spi-mux.c](drivers/spi/spi-mux.c.md)
- [ ] [spi-offload.c](drivers/spi/spi-offload.c.md)
- [ ] [spidev.c](drivers/spi/spidev.c.md)

## `IRQ SUBSYSTEM`

```
M:	Thomas Gleixner <tglx@kernel.org>
L:	linux-kernel@vger.kernel.org
S:	Maintained
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git irq/core
F:	include/linux/group_cpus.h
F:	include/linux/irq.h
F:	include/linux/irqhandler.h
F:	include/linux/irqnr.h
F:	include/linux/irqreturn.h
F:	kernel/irq/
F:	lib/group_cpus.c
```



## `PCI SUBSYSTEM`

```
```


## `USB SUBSYSTEM`

```
M:	Greg Kroah-Hartman <gregkh@linuxfoundation.org>
L:	linux-usb@vger.kernel.org
S:	Supported
W:	http://www.linux-usb.org
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/gregkh/usb.git
F:	Documentation/devicetree/bindings/usb/
F:	Documentation/usb/
F:	drivers/usb/
F:	include/dt-bindings/usb/
F:	include/linux/usb.h
F:	include/linux/usb/
F:	include/uapi/linux/usb/
```


## `BLOCK LAYER`

```
```


## `SCSI SUBSYSTEM`

```
```


## `IOMMU SUBSYSTEM`
```
```



## `CRYPTO API`

```
```


## `PIN CONTROL SUBSYSTEM` ✅⭐

```
M:	Linus Walleij <linusw@kernel.org>
L:	linux-gpio@vger.kernel.org
S:	Maintained
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/linusw/linux-pinctrl.git
F:	Documentation/devicetree/bindings/pinctrl/
F:	Documentation/driver-api/pin-control.rst
F:	drivers/pinctrl/
F:	include/dt-bindings/pinctrl/
F:	include/linux/pinctrl/
```

- [ ] [pin-control.rst](Documentation/driver-api/pin-control.rst.md)

- [ ] [devinfo.h](include/linux/pintrl/devinfo.h.md)
- [ ] [machine.h](include/linux/pintrl/machine.h.md)
- [ ] [pinconf-generic.h](include/linux/pinctrl/pinconf-generic.h.md)
- [ ] [pinconf.h](include/linux/pintrl/pinconf.h.md)
- [ ] [pinctrl.h](include/linux/pintrl/pinctrl.h.md)
- [ ] [pinctrl-state.h](include/linux/pinctrl/pinctrl-state.h.md)
- [ ] [pinmux.h](include/linux/pinctrl/pinmux.h.md)
- [ ] [pin-control.rst](Documentation/driver-api/pin-control.rst.md)

- [ ] [core.h](drivers/pinctrl/core.h.md) -- [core.c](drivers/pinctrl/core.c.md)
- [ ] [pinctrl-utils.c](drivers/pinctrl/pinctrl-utils.c.md)
- [ ] [pinmux.h](drivers/pinctrl/pinmux.h.md) -- [pinmux.c](drivers/pinctrl/pinmux.c.md)
- [ ] [pinconf.c](drivers/pinctrl/pinconf.c.md)
- [ ] [pinconf-generic.c](drivers/pinctrl/pinconf-generic.c.md)
- [ ] [devicetree.c](drivers/pinctrl/devicetree.c.md)


## `MEMORY MANAGEMENT - CORE`

```
M:	Andrew Morton <akpm@linux-foundation.org>
M:	David Hildenbrand <david@kernel.org>
R:	Lorenzo Stoakes <lorenzo.stoakes@oracle.com>
R:	Liam R. Howlett <Liam.Howlett@oracle.com>
R:	Vlastimil Babka <vbabka@suse.cz>
R:	Mike Rapoport <rppt@kernel.org>
R:	Suren Baghdasaryan <surenb@google.com>
R:	Michal Hocko <mhocko@suse.com>
L:	linux-mm@kvack.org
S:	Maintained
W:	http://www.linux-mm.org
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/akpm/mm
F:	include/linux/gfp.h
F:	include/linux/gfp_types.h
F:	include/linux/highmem.h
F:	include/linux/leafops.h
F:	include/linux/memory.h
F:	include/linux/mm.h
F:	include/linux/mm_*.h
F:	include/linux/mmzone.h
F:	include/linux/mmdebug.h
F:	include/linux/mmu_notifier.h
F:	include/linux/pagewalk.h
F:	include/linux/pgalloc.h
F:	include/linux/pgtable.h
F:	include/linux/ptdump.h
F:	include/linux/vmpressure.h
F:	include/linux/vmstat.h
F:	kernel/fork.c
F:	mm/Kconfig
F:	mm/debug.c
F:	mm/folio-compat.c
F:	mm/highmem.c
F:	mm/init-mm.c
F:	mm/internal.h
F:	mm/maccess.c
F:	mm/memory.c
F:	mm/mmu_notifier.c
F:	mm/mmzone.c
F:	mm/pagewalk.c
F:	mm/pgtable-generic.c
F:	mm/ptdump.c
F:	mm/sparse-vmemmap.c
F:	mm/sparse.c
F:	mm/util.c
F:	mm/vmpressure.c
F:	mm/vmstat.c
N:	include/linux/page[-_]*
```


## `INPUT (KEYBOARD, MOUSE, JOYSTICK, TOUCHSCREEN) DRIVERS`

```
M:	Dmitry Torokhov <dmitry.torokhov@gmail.com>
L:	linux-input@vger.kernel.org
S:	Maintained
Q:	http://patchwork.kernel.org/project/linux-input/list/
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/dtor/input.git
F:	Documentation/devicetree/bindings/input/
F:	Documentation/devicetree/bindings/serio/
F:	Documentation/input/
F:	drivers/input/
F:	include/dt-bindings/input/
F:	include/linux/gameport.h
F:	include/linux/i8042.h
F:	include/linux/input.h
F:	include/linux/input/
F:	include/linux/libps2.h
F:	include/linux/serio.h
F:	include/uapi/linux/gameport.h
F:	include/uapi/linux/input-event-codes.h
F:	include/uapi/linux/input.h
F:	include/uapi/linux/serio.h
F:	include/uapi/linux/uinput.h
```


## `POWER MANAGEMENT CORE`

```
M:	"Rafael J. Wysocki" <rafael@kernel.org>
L:	linux-pm@vger.kernel.org
S:	Supported
B:	https://bugzilla.kernel.org
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/rafael/linux-pm
F:	drivers/base/power/
F:	drivers/powercap/
F:	include/linux/intel_rapl.h
F:	include/linux/pm.h
F:	include/linux/pm_*
F:	include/linux/powercap.h
F:	kernel/configs/nopm.config
```


## `COMMON CLK FRAMEWORK` ✅⭐

```
M:	Michael Turquette <mturquette@baylibre.com>
M:	Stephen Boyd <sboyd@kernel.org>
L:	linux-clk@vger.kernel.org
S:	Maintained
Q:	http://patchwork.kernel.org/project/linux-clk/list/
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/clk/linux.git
F:	Documentation/devicetree/bindings/clock/
F:	drivers/clk/
F:	include/dt-bindings/clock/
F:	include/linux/clk-pr*
F:	include/linux/clk/
F:	include/linux/of_clk.h
F:	scripts/gdb/linux/clk.py
F:	rust/helpers/clk.c
F:	rust/kernel/clk.rs
X:	drivers/clk/clkdev.c
```

- [ ] [clk-devres.c](drivers/clk/clk-devres.c.md)
- [ ] [clk-bulk.c](drivers/clk/clk-bulk.c.md)
- [ ] [clkdev.h](include/linux/clkdev.h.md) - [clkdev.c](drivers/clk/clkdev.c.md)

- [ ] [clk-divider.c](drivers/clk/clk-divider.c.md)
- [ ] [clk-fixed-factor.c](drivers/clk/clk-fixed-factor.c.md)
- [ ] [clk-fixed-rate.c](drivers/clk/clk-fixed-rate.c.md)
- [ ] [clk-gate.c](drivers/clk/clk-gate.c.md)
- [ ] [clk-multiplier.c](drivers/clk/clk-multiplier.c.md)
- [ ] [clk-mux.c](drivers/clk/clk-mux.c.md)
- [ ] [clk-composite.c](drivers/clk/clk-composite.c.md)
- [ ] [clk-gpio.c](drivers/clk/clk-gpio.c.md)
- [ ] [clk-conf.c](drivers/clk/clk-conf.c.md)

- [ ] [clk.h](include/linux/clk.h.md)
- [ ] [clk-provider.h](include/linux/clk-provider.h.md)

- [ ] [of_clk.h](include/linux/of_clk.h.md)
- [ ] [sh_clk.h](include/linux/sh_clk.h.md)



لطيف جداً فى الكيرنال والـ clk framework انك تشوف multi layer of abstractions بيتعمل من الـ functions كتير علشان تستخدم الحاجات دى كيوزر من غير ما تبقى عارف اى تفاصيل عن الهاردوير. بعدين لما تتبع ops توصل فى الآخر انها مجرد function بتعمل write & read لـ bit معين فى register !
لكن كـ OS هو لازم يعمل multi layer of abstraction

#### ايه لزمه الـ clk framework؟
ء
#### ازاى بتعمل test عموماً للـ clk framework؟ وازاى انا عملت test لما عملت port لـ Amlogic؟
ء

- [ ] [syscon.h](include/linux/mfd/syscon.h.md)
- [ ] [auxiliary_bus.h](include/linux/auxiliary_bus.h.md) - [auxiliary.c](drivers/base/auxiliary.c.md)
- [ ] [platform_device.h](include/linux/platform_device.h.md)
- [ ] [platform_profile.h](include/linux/platform_profile.h.md)


## `VOLTAGE AND CURRENT REGULATOR FRAMEWORK`

```
M:	Liam Girdwood <lgirdwood@gmail.com>
M:	Mark Brown <broonie@kernel.org>
L:	linux-kernel@vger.kernel.org
S:	Supported
W:	http://www.slimlogic.co.uk/?p=48
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/broonie/regulator.git
F:	Documentation/devicetree/bindings/regulator/
F:	Documentation/power/regulator/
F:	drivers/regulator/
F:	rust/kernel/regulator.rs
F:	include/dt-bindings/regulator/
F:	include/linux/regulator/
F:	include/uapi/regulator/
K:	regulator_get_optional
```


## `TTY LAYER AND SERIAL DRIVERS`

```
M:	Greg Kroah-Hartman <gregkh@linuxfoundation.org>
M:	Jiri Slaby <jirislaby@kernel.org>
L:	linux-kernel@vger.kernel.org
L:	linux-serial@vger.kernel.org
S:	Supported
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/gregkh/tty.git
F:	Documentation/devicetree/bindings/serial/
F:	Documentation/driver-api/serial/
F:	drivers/tty/
F:	include/linux/selection.h
F:	include/linux/serial.h
F:	include/linux/serial_core.h
F:	include/linux/sysrq.h
F:	include/linux/tty*.h
F:	include/linux/vt.h
F:	include/linux/vt_*.h
F:	include/uapi/linux/serial.h
F:	include/uapi/linux/serial_core.h
F:	include/uapi/linux/tty.h
```


## `SOUND`

```
M:	Jaroslav Kysela <perex@perex.cz>
M:	Takashi Iwai <tiwai@suse.com>
L:	linux-sound@vger.kernel.org
S:	Maintained
W:	http://www.alsa-project.org/
Q:	http://patchwork.kernel.org/project/alsa-devel/list/
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/tiwai/sound.git
F:	Documentation/sound/
F:	include/sound/
F:	include/uapi/sound/
F:	sound/
F:	tools/testing/selftests/alsa
```


## `MEDIA CONTROLLER FRAMEWORK`

```
```


## `DMA GENERIC OFFLOAD ENGINE SUBSYSTEM`

```
```


## `MULTIMEDIA CARD (MMC), SECURE DIGITAL (SD) AND SDIO SUBSYSTEM`

```
M:	Ulf Hansson <ulf.hansson@linaro.org>
L:	linux-mmc@vger.kernel.org
S:	Maintained
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/ulfh/mmc.git
F:	Documentation/devicetree/bindings/mmc/
F:	drivers/mmc/
F:	include/linux/mmc/
F:	include/uapi/linux/mmc/
```

- [ ] [card.h](include/linux/mmc/card.h.md)
- [ ] [core.h](include/linux/mmc/core.h.md)
- [ ] [host.h](include/linux/mmc/host.h.md)
- [ ] [mmc.h](include/linux/mmc/mmc.h.md)
- [ ] [pm.h](include/linux/mmc/pm.h.md)
- [ ] [sd.h](include/linux/mmc/sd.h.md)
- [ ] [sdio.h](include/linux/mmc/sdio.h.md) -- [sdio_ids.h](include/linux/mmc/sdio_ids.h.md) -- [sdio_func.h](include/linux/mmc/sdio_func.h.md) -- [sd_uhs2.h](include/linux/mmc/sd_uhs2.h.md) -- [slot-gpio.h](include/linux/mmc/slot-gpio.h.md)

## `SPI NOR SUBSYSTEM`

```
M:	Tudor Ambarus <tudor.ambarus@linaro.org>
M:	Pratyush Yadav <pratyush@kernel.org>
M:	Michael Walle <mwalle@kernel.org>
L:	linux-mtd@lists.infradead.org
S:	Maintained
W:	http://www.linux-mtd.infradead.org/
Q:	http://patchwork.ozlabs.org/project/linux-mtd/list/
C:	irc://irc.oftc.net/mtd
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/mtd/linux.git spi-nor/next
F:	Documentation/devicetree/bindings/mtd/jedec,spi-nor.yaml
F:	drivers/mtd/spi-nor/
F:	include/linux/mtd/spi-nor.h
```


## `REMOTE PROCESSOR (REMOTEPROC) SUBSYSTEM`

```
M:	Bjorn Andersson <andersson@kernel.org>
M:	Mathieu Poirier <mathieu.poirier@linaro.org>
L:	linux-remoteproc@vger.kernel.org
S:	Maintained
T:	git https://git.kernel.org/pub/scm/linux/kernel/git/remoteproc/linux.git rproc-next
F:	Documentation/ABI/testing/sysfs-class-remoteproc
F:	Documentation/devicetree/bindings/remoteproc/
F:	Documentation/staging/remoteproc.rst
F:	drivers/remoteproc/
F:	include/linux/remoteproc.h
F:	include/linux/remoteproc/
```


## `REMOTE PROCESSOR MESSAGING (RPMSG) SUBSYSTEM`

```
M:	Bjorn Andersson <andersson@kernel.org>
M:	Mathieu Poirier <mathieu.poirier@linaro.org>
L:	linux-remoteproc@vger.kernel.org
S:	Maintained
T:	git https://git.kernel.org/pub/scm/linux/kernel/git/remoteproc/linux.git rpmsg-next
F:	Documentation/ABI/testing/sysfs-bus-rpmsg
F:	Documentation/staging/rpmsg.rst
F:	drivers/rpmsg/
F:	include/linux/rpmsg.h
F:	include/linux/rpmsg/
F:	include/uapi/linux/rpmsg.h
F:	samples/rpmsg/
```



## `CPU FREQUENCY SCALING FRAMEWORK`

```
M:	"Rafael J. Wysocki" <rafael@kernel.org>
M:	Viresh Kumar <viresh.kumar@linaro.org>
L:	linux-pm@vger.kernel.org
S:	Maintained
B:	https://bugzilla.kernel.org
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/rafael/linux-pm.git
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/vireshk/pm.git (For ARM Updates)
F:	Documentation/admin-guide/pm/cpufreq.rst
F:	Documentation/admin-guide/pm/intel_pstate.rst
F:	Documentation/cpu-freq/
F:	Documentation/devicetree/bindings/cpufreq/
F:	drivers/cpufreq/
F:	include/linux/cpufreq.h
F:	include/linux/sched/cpufreq.h
F:	kernel/sched/cpufreq*.c
F:	rust/kernel/cpufreq.rs
F:	tools/testing/selftests/cpufreq/
```


## `THERMAL`

```
M:	Rafael J. Wysocki <rafael@kernel.org>
M:	Daniel Lezcano <daniel.lezcano@linaro.org>
R:	Zhang Rui <rui.zhang@intel.com>
R:	Lukasz Luba <lukasz.luba@arm.com>
L:	linux-pm@vger.kernel.org
S:	Supported
Q:	https://patchwork.kernel.org/project/linux-pm/list/
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/rafael/linux-pm.git thermal
F:	Documentation/ABI/testing/sysfs-class-thermal
F:	Documentation/admin-guide/thermal/
F:	Documentation/devicetree/bindings/thermal/
F:	Documentation/driver-api/thermal/
F:	drivers/thermal/
F:	include/dt-bindings/thermal/
F:	include/linux/cpu_cooling.h
F:	include/linux/thermal.h
F:	include/uapi/linux/thermal.h
F:	tools/lib/thermal/
F:	tools/thermal/

THERMAL DRIVER FOR AMLOGIC SOCS
M:	Guillaume La Roque <glaroque@baylibre.com>
L:	linux-pm@vger.kernel.org
L:	linux-amlogic@lists.infradead.org
S:	Supported
W:	http://linux-meson.com/
F:	Documentation/devicetree/bindings/thermal/amlogic,thermal.yaml
F:	drivers/thermal/amlogic_thermal.c
```


## `REAL TIME CLOCK (RTC) SUBSYSTEM`

```
M:	Alexandre Belloni <alexandre.belloni@bootlin.com>
L:	linux-rtc@vger.kernel.org
S:	Maintained
Q:	http://patchwork.ozlabs.org/project/rtc-linux/list/
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/abelloni/linux.git
F:	Documentation/admin-guide/rtc.rst
F:	Documentation/devicetree/bindings/rtc/
F:	drivers/rtc/
F:	include/linux/rtc.h
F:	include/linux/rtc/
F:	include/uapi/linux/rtc.h
F:	tools/testing/selftests/rtc/
```


## `RESET CONTROLLER FRAMEWORK`

```
M:	Philipp Zabel <p.zabel@pengutronix.de>
S:	Maintained
T:	git https://git.pengutronix.de/git/pza/linux.git
F:	Documentation/devicetree/bindings/reset/
F:	
F:	drivers/reset/
F:	include/dt-bindings/reset/
F:	include/linux/reset-controller.h
F:	include/linux/reset.h
F:	include/linux/reset/
K:	\b(?:devm_|of_)?reset_control(?:ler_[a-z]+|_[a-z_]+)?\b
```
- [ ] Documentation/driver-api/reset.rst
- [ ] [reset.rst](Documentation/driver-api/reset.rst.md)

- [ ] [reset-controller.h](include/linux/reset-controller.h.md)
- [ ] [reset.h](include/linux/reset.h.md)
- [ ] [reset-simple.h](include/linux/reset/reset-simple.h.md)
- [ ] [sunxi.h](include/linux/reset/sunxi.h.md)

- [ ] [core.c](drivers/reset/core.c.md)

- [ ] [reset-gpio.c](drivers/reset/reset-gpio.c.md)

## `GENERIC PHY FRAMEWORK`

```
M:	Vinod Koul <vkoul@kernel.org>
R:	Neil Armstrong <neil.armstrong@linaro.org>
L:	linux-phy@lists.infradead.org
S:	Supported
Q:	https://patchwork.kernel.org/project/linux-phy/list/
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/phy/linux-phy.git
F:	Documentation/devicetree/bindings/phy/
F:	drivers/phy/
F:	include/dt-bindings/phy/
F:	include/linux/phy/
```


## `HARDWARE SPINLOCK CORE`

```
M:	Bjorn Andersson <andersson@kernel.org>
R:	Baolin Wang <baolin.wang7@gmail.com>
L:	linux-remoteproc@vger.kernel.org
S:	Maintained
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/andersson/remoteproc.git hwspinlock-next
F:	Documentation/devicetree/bindings/hwlock/
F:	Documentation/locking/hwspinlock.rst
F:	drivers/hwspinlock/
F:	include/linux/hwspinlock.h
```


## `MAILBOX API`

```
M:	Jassi Brar <jassisinghbrar@gmail.com>
L:	linux-kernel@vger.kernel.org
S:	Maintained
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/jassibrar/mailbox.git for-next
F:	Documentation/devicetree/bindings/mailbox/
F:	drivers/mailbox/
F:	include/dt-bindings/mailbox/
F:	include/linux/mailbox_client.h
F:	include/linux/mailbox_controller.h
```


## `TEE SUBSYSTEM`

```
M:	Jens Wiklander <jens.wiklander@linaro.org>
R:	Sumit Garg <sumit.garg@kernel.org>
L:	op-tee@lists.trustedfirmware.org (moderated for non-subscribers)
S:	Maintained
F:	Documentation/ABI/testing/sysfs-class-tee
F:	Documentation/driver-api/tee.rst
F:	Documentation/tee/
F:	Documentation/userspace-api/tee.rst
F:	drivers/tee/
F:	include/linux/tee_core.h
F:	include/linux/tee_drv.h
F:	include/uapi/linux/tee.h
```


## `CAN NETWORK LAYER`

```
```


## `WATCHDOG DEVICE DRIVERS`

```
M:	Wim Van Sebroeck <wim@linux-watchdog.org>
M:	Guenter Roeck <linux@roeck-us.net>
L:	linux-watchdog@vger.kernel.org
S:	Maintained
W:	http://www.linux-watchdog.org/
T:	git git://www.linux-watchdog.org/linux-watchdog.git
F:	Documentation/devicetree/bindings/watchdog/
F:	Documentation/watchdog/
F:	drivers/watchdog/
F:	include/linux/watchdog.h
F:	include/trace/events/watchdog.h
F:	include/uapi/linux/watchdog.h
```


## `INTERCONNECT API`

```
M:	Georgi Djakov <djakov@kernel.org>
L:	linux-pm@vger.kernel.org
S:	Maintained
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/djakov/icc.git
F:	Documentation/devicetree/bindings/interconnect/
F:	Documentation/driver-api/interconnect.rst
F:	drivers/interconnect/
F:	include/dt-bindings/interconnect/
F:	include/linux/interconnect-clk.h
F:	include/linux/interconnect-provider.h
F:	include/linux/interconnect.h
```


## `PWM SUBSYSTEM`

```
PWM SUBSYSTEM
M:	Uwe Kleine-König <ukleinek@kernel.org>
L:	linux-pwm@vger.kernel.org
S:	Maintained
Q:	https://patchwork.ozlabs.org/project/linux-pwm/list/
T:	git https://git.kernel.org/pub/scm/linux/kernel/git/ukleinek/linux.git
F:	Documentation/devicetree/bindings/pwm/
F:	Documentation/driver-api/pwm.rst
F:	drivers/pwm/
F:	include/dt-bindings/pwm/
F:	include/linux/pwm.h
K:	pwm_(config|apply_might_sleep|apply_atomic|ops)
K:	(devm_)?pwmchip_(add|alloc|remove)
K:	pwm_(round|get|set)_waveform

PWM SUBSYSTEM BINDINGS [RUST]
M:	Michal Wilczynski <m.wilczynski@samsung.com>
L:	linux-pwm@vger.kernel.org
L:	rust-for-linux@vger.kernel.org
S:	Maintained
F:	rust/helpers/pwm.c
F:	rust/kernel/pwm.rs

PWM SUBSYSTEM DRIVERS [RUST]
R:	Michal Wilczynski <m.wilczynski@samsung.com>
F:	drivers/pwm/*.rs
```



## `FPGA MANAGER FRAMEWORK`

```
```


## `IEEE 802.15.4 SUBSYSTEM`

```
```


## `NVMEM FRAMEWORK`

```
```


## `NAND FLASH SUBSYSTEM`

```
```


## `AUXILIARY BUS DRIVER`

```
M:	Greg Kroah-Hartman <gregkh@linuxfoundation.org>
M:	"Rafael J. Wysocki" <rafael@kernel.org>
M:	Danilo Krummrich <dakr@kernel.org>
R:	Dave Ertman <david.m.ertman@intel.com>
R:	Ira Weiny <ira.weiny@intel.com>
R:	Leon Romanovsky <leon@kernel.org>
L:	driver-core@lists.linux.dev
S:	Supported
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/driver-core/driver-core.git
F:	Documentation/driver-api/auxiliary_bus.rst
F:	drivers/base/auxiliary.c
F:	include/linux/auxiliary_bus.h
F:	rust/helpers/auxiliary.c
F:	rust/kernel/auxiliary.rs
F:	samples/rust/rust_driver_auxiliary.rs
```

## `BITMAP API`

```
M:	Yury Norov <yury.norov@gmail.com>
R:	Rasmus Villemoes <linux@rasmusvillemoes.dk>
S:	Maintained
F:	include/linux/bitfield.h
F:	include/linux/bitmap-str.h
F:	include/linux/bitmap.h
F:	include/linux/bits.h
F:	include/linux/cpumask.h
F:	include/linux/cpumask_types.h
F:	include/linux/find.h
F:	include/linux/hw_bitfield.h
F:	include/linux/nodemask.h
F:	include/linux/nodemask_types.h
F:	include/uapi/linux/bits.h
F:	include/vdso/bits.h
F:	lib/bitmap-str.c
F:	lib/bitmap.c
F:	lib/cpumask.c
F:	lib/find_bit.c
F:	lib/find_bit_benchmark.c
F:	lib/test_bitmap.c
F:	lib/tests/cpumask_kunit.c
F:	tools/include/linux/bitfield.h
F:	tools/include/linux/bitmap.h
F:	tools/include/linux/bits.h
F:	tools/include/linux/find.h
F:	tools/include/uapi/linux/bits.h
F:	tools/include/vdso/bits.h
F:	tools/lib/bitmap.c
F:	tools/lib/find_bit.c
```

## `ATOMIC INFRASTRUCTURE`

```
M:	Will Deacon <will@kernel.org>
M:	Peter Zijlstra <peterz@infradead.org>
M:	Boqun Feng <boqun@kernel.org>
R:	Mark Rutland <mark.rutland@arm.com>
R:	Gary Guo <gary@garyguo.net>
L:	linux-kernel@vger.kernel.org
S:	Maintained
F:	Documentation/atomic_*.txt
F:	arch/*/include/asm/atomic*.h
F:	include/*/atomic*.h
F:	include/linux/refcount.h
F:	scripts/atomic/
F:	rust/kernel/sync/atomic.rs
F:	rust/kernel/sync/atomic/
F:	rust/kernel/sync/refcount.rs
```

## `BITOPS API`

```
M:	Yury Norov <yury.norov@gmail.com>
R:	Rasmus Villemoes <linux@rasmusvillemoes.dk>
S:	Maintained
F:	arch/*/include/asm/bitops.h
F:	arch/*/include/asm/bitops_32.h
F:	arch/*/include/asm/bitops_64.h
F:	arch/*/lib/bitops.c
F:	include/asm-generic/bitops
F:	include/asm-generic/bitops.h
F:	include/linux/bitops.h
F:	include/linux/count_zeros.h
F:	lib/hweight.c
F:	lib/test_bitops.c
F:	lib/tests/bitops_kunit.c
F:	tools/*/bitops*
```

## `BLOCK LAYER`

```
M:	Jens Axboe <axboe@kernel.dk>
L:	linux-block@vger.kernel.org
S:	Maintained
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/axboe/linux.git
F:	Documentation/ABI/stable/sysfs-block
F:	Documentation/block/
F:	block/
F:	drivers/block/
F:	include/linux/bio.h
F:	include/linux/blk*
F:	include/uapi/linux/blk*
F:	include/uapi/linux/ioprio.h
F:	kernel/trace/blktrace.c
F:	lib/sbitmap.c
```

## `CPU FREQUENCY SCALING FRAMEWORK`

```
M:	"Rafael J. Wysocki" <rafael@kernel.org>
M:	Viresh Kumar <viresh.kumar@linaro.org>
L:	linux-pm@vger.kernel.org
S:	Maintained
B:	https://bugzilla.kernel.org
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/rafael/linux-pm.git
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/vireshk/pm.git (For ARM Updates)
F:	Documentation/admin-guide/pm/cpufreq.rst
F:	Documentation/admin-guide/pm/intel_pstate.rst
F:	Documentation/cpu-freq/
F:	Documentation/devicetree/bindings/cpufreq/
F:	drivers/cpufreq/
F:	include/linux/cpufreq.h
F:	include/linux/sched/cpufreq.h
F:	kernel/sched/cpufreq*.c
F:	rust/kernel/cpufreq.rs
F:	tools/testing/selftests/cpufreq/
```

## `CPU HOTPLUG`

```
M:	Thomas Gleixner <tglx@kernel.org>
M:	Peter Zijlstra <peterz@infradead.org>
L:	linux-kernel@vger.kernel.org
S:	Maintained
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git smp/core
F:	include/linux/cpu.h
F:	include/linux/cpuhotplug.h
F:	include/linux/smpboot.h
F:	kernel/cpu.c
F:	kernel/smpboot.*
F:	rust/helpers/cpu.c
F:	rust/kernel/cpu.rs
```

## `DEVICE RESOURCE MANAGEMENT HELPERS`

```
M:	Hans de Goede <hansg@kernel.org>
R:	Matti Vaittinen <mazziesaccount@gmail.com>
S:	Maintained
F:	include/linux/devm-helpers.h
```

## `DRIVER CORE, KOBJECTS, DEBUGFS AND SYSFS` ⭐✅

```
M:	Greg Kroah-Hartman <gregkh@linuxfoundation.org>
M:	"Rafael J. Wysocki" <rafael@kernel.org>
M:	Danilo Krummrich <dakr@kernel.org>
L:	driver-core@lists.linux.dev
S:	Supported
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/driver-core/driver-core.git
F:	Documentation/core-api/kobject.rst
F:	Documentation/driver-api/driver-model/
F:	drivers/base/
F:	fs/debugfs/
F:	fs/sysfs/
F:	include/linux/device/
F:	include/linux/debugfs.h
F:	include/linux/device.h
F:	include/linux/fwnode.h
F:	include/linux/kobj*
F:	include/linux/property.h
F:	include/linux/sysfs.h
F:	lib/kobj*
F:	rust/kernel/debugfs.rs
F:	rust/kernel/debugfs/
F:	rust/kernel/device.rs
F:	rust/kernel/device/
F:	rust/kernel/device_id.rs
F:	rust/kernel/devres.rs
F:	rust/kernel/driver.rs
F:	rust/kernel/faux.rs
F:	rust/kernel/platform.rs
F:	rust/kernel/soc.rs
F:	samples/rust/rust_debugfs.rs
F:	samples/rust/rust_debugfs_scoped.rs
F:	samples/rust/rust_driver_platform.rs
F:	samples/rust/rust_driver_faux.rs
F:	samples/rust/rust_soc.rs
```

- Documentation/core-api/kobject.rst
- Documentation/driver-api/driver-model/
- [ ] [debugfs.h](include/linux/debugfs.h.md)
- [ ] [device.h](include/linux/device.h.md)
- [ ] [fwnode.h](include/linux/fwnode.h.md)
- [ ] [kobject.h](include/linux/kobject.h.md)
- [ ] [property.h](include/linux/property.h.md)
- [ ] [sysfs.h](include/linux/sysfs.h.md)

- [ ] [bus.h](include/linux/device/bus.h.md)
- [ ] [class.h](include/linux/device/class.h.md)
- [ ] [devres.h](include/linux/device/devres.h.md)
- [ ] [driver.h](include/linux/device/driver.h.md)
- [ ] [faux.h](include/linux/device/faux.h.md)

## `ETHERNET PHY LIBRARY`

```
M:	Andrew Lunn <andrew@lunn.ch>
M:	Heiner Kallweit <hkallweit1@gmail.com>
R:	Russell King <linux@armlinux.org.uk>
L:	netdev@vger.kernel.org
S:	Maintained
F:	Documentation/ABI/testing/sysfs-class-net-phydev
F:	Documentation/devicetree/bindings/net/ethernet-connector.yaml
F:	Documentation/devicetree/bindings/net/ethernet-phy.yaml
F:	Documentation/devicetree/bindings/net/mdio*
F:	Documentation/devicetree/bindings/net/qca,ar803x.yaml
F:	Documentation/networking/phy-port.rst
F:	Documentation/networking/phy.rst
F:	drivers/net/mdio/
F:	drivers/net/mdio/acpi_mdio.c
F:	drivers/net/mdio/fwnode_mdio.c
F:	drivers/net/mdio/of_mdio.c
F:	drivers/net/pcs/
F:	drivers/net/phy/
F:	include/dt-bindings/net/qca-ar803x.h
F:	include/linux/*mdio*.h
F:	include/linux/linkmode.h
F:	include/linux/mdio/*.h
F:	include/linux/mii.h
F:	include/linux/of_net.h
F:	include/linux/phy.h
F:	include/linux/phy_fixed.h
F:	include/linux/phy_link_topology.h
F:	include/linux/phylib_stubs.h
F:	include/linux/platform_data/mdio-bcm-unimac.h
F:	include/linux/platform_data/mdio-gpio.h
F:	include/net/phy/
F:	include/trace/events/mdio.h
F:	include/uapi/linux/mdio.h
F:	include/uapi/linux/mii.h
F:	net/core/of_net.c
```

## `LOCKING PRIMITIVES`

```
M:	Peter Zijlstra <peterz@infradead.org>
M:	Ingo Molnar <mingo@redhat.com>
M:	Will Deacon <will@kernel.org>
M:	Boqun Feng <boqun@kernel.org> (LOCKDEP & RUST)
R:	Waiman Long <longman@redhat.com>
L:	linux-kernel@vger.kernel.org
S:	Maintained
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git locking/core
F:	Documentation/locking/
F:	arch/*/include/asm/spinlock*.h
F:	include/linux/local_lock*.h
F:	include/linux/lockdep*.h
F:	include/linux/mutex*.h
F:	include/linux/rwlock*.h
F:	include/linux/rwsem*.h
F:	include/linux/seqlock.h
F:	include/linux/spinlock*.h
F:	kernel/locking/
F:	lib/locking*.[ch]
F:	rust/helpers/mutex.c
F:	rust/helpers/spinlock.c
F:	rust/kernel/sync/lock.rs
F:	rust/kernel/sync/lock/
F:	rust/kernel/sync/locked_by.rs
X:	kernel/locking/locktorture.c
```

## `SECURITY SUBSYSTEM`

```
M:	Paul Moore <paul@paul-moore.com>
M:	James Morris <jmorris@namei.org>
M:	"Serge E. Hallyn" <serge@hallyn.com>
L:	linux-security-module@vger.kernel.org
S:	Supported
Q:	https://patchwork.kernel.org/project/linux-security-module/list
B:	mailto:linux-security-module@vger.kernel.org
P:	https://github.com/LinuxSecurityModule/kernel/blob/main/README.md
T:	git https://git.kernel.org/pub/scm/linux/kernel/git/pcmoore/lsm.git
F:	include/linux/lsm/
F:	include/linux/lsm_audit.h
F:	include/linux/lsm_hook_defs.h
F:	include/linux/lsm_hooks.h
F:	include/linux/security.h
F:	include/uapi/linux/lsm.h
F:	security/
F:	tools/testing/selftests/lsm/
F:	rust/kernel/security.rs
X:	security/selinux/
K:	\bsecurity_[a-z_0-9]\+\b
```

- [ ] [lsm_audit.h](include/linux/lsm_audit.h.md)
- [ ] [lsm_hook_defs.h](include/linux/lsm_hook_defs.h.md)
- [ ] [lsm_hooks.h](include/linux/lsm_hooks.h.md)
- [ ] [security.h](include/linux/security.h.md)
- [ ] [lsm.h](include/uapi/linux/lsm.h.md)

- [ ] [apparmor.h](include/linux/lsm/apparmor.h.md)
- [ ] [bpf.h](include/linux/lsm/bpf.h.md)
- [ ] [selinux.h](include/linux/lsm/selinux.h.md)
- [ ] [smack.h](include/linux/lsm/smack.h.md)

## `XARRAY` ✅

```
M:	Matthew Wilcox <willy@infradead.org>
L:	linux-fsdevel@vger.kernel.org
L:	linux-mm@kvack.org
S:	Supported
F:	Documentation/core-api/idr.rst
F:	Documentation/core-api/xarray.rst
F:	include/linux/idr.h
F:	include/linux/xarray.h
F:	lib/idr.c
F:	lib/test_xarray.c
F:	lib/xarray.c
F:	tools/testing/radix-tree
```

- [ ] [idr.rst](Documentation/core-api/idr.rst.md)
- [ ] [xarray.rst](Documentation/core-api/xarray.rst.md)
- [ ] [idr.h](include/linux/idr.h.md)
- [ ] [xarray.h](include/linux/xarray.h.md)

# TODO
- [ ] https://docs.kernel.org/core-api/kernel-api.html
- [ ] Linux notifications subsystem: include/linux/notifier.h
- [ ] https://docs.kernel.org/core-api/index.html
- [ ] https://origin.kernel.org/doc/html/latest/driver-api/driver-model/
- [ ] lwn
- [ ] linux commits messages
- [ ] whole mailing lists
- [ ] kernelnewbies
- [ ] elinux.org
