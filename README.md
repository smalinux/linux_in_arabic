
```bash
# vscode
$ code

# Dev containers extension: search bar: type:
> Dev containers: reopen in container


# Then open new termainal
$ claude --dangerously-skip-permissions
```


## clk framework

- [ ] [clk.h](include/linux/clk.h.md)
- [ ] [clk-provider.h](include/linux/clk-provider.h.md)
- [ ] ----
- [ ] [of_clk.h](include/linux/of_clk.h.md)
- [ ] [sh_clk.h](include/linux/sh_clk.h.md)
- [ ] [clkdev.h](include/linux/clkdev.h.md) - [clkdev.c](drivers/clk/clkdev.c.md)


لطيف جداً فى الكيرنال والـ clk framework انك تشوف multi layer of abstractions بيتعمل من الـ functions كتير علشان تستخدم الحاجات دى كيوزر من غير ما تبقى عارف اى تفاصيل عن الهاردوير. بعدين لما تتبع ops توصل فى الآخر انها مجرد function بتعمل write & read لـ bit معين فى register !
لكن كـ OS هو لازم يعمل multi layer of abstraction

#### ايه لزمه الـ clk framework؟
ء
#### ازاى بتعمل test عموماً للـ clk framework؟ وازاى انا عملت test لما عملت port لـ Amlogic؟
ء

- [ ] [syscon.h](include/linux/mfd/syscon.h.md)
- [ ] [auxiliary_bus.h](include/linux/auxiliary_bus.h.md) - [auxiliary.c](drivers/base/auxiliary.c.md)
- [ ] [platform_device.h](include/linux/platform_device.h.md)
- [ ] platform_profile.h ?



## of - open firmware
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