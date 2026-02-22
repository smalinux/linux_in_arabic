
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