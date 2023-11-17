# 多核支持开发文档

## arceos hypercraft多核启动

过程省略，结论是，主核会去运行hv的main函数，其他核会运行：

```
    #[cfg(not(feature = "multitask"))]
    loop {
        axhal::arch::wait_for_irqs();
    }
```

在riscv中他是：

```
#[inline]
pub fn wait_for_irqs() {
    unsafe { riscv::asm::wfi() }
}

```

貌似在等待中断，后续应该要通过中断来启动其他物理CPU。



## 从dtb获取cpu信息

现在的物理核心数目是被直接写成1的，但是有前辈留下的todo，实现多核支持的花应该从dtb中获取信息吧。

```
// TODO: get cpu info by device tree
        let cpu_nums: usize = 1;
```

查看axruntime的rust_main好像根本没有用到dtb。

![](./pictures/m-1.png)

查找dtb只有这四个地方，也没有传到hv里面，可能要改hv的main函数？

但是guest os的dtb也要对应修改才行。

```
	cpus {
		#address-cells = <0x01>;
		#size-cells = <0x00>;
		timebase-frequency = <0x989680>;

		cpu@0 {
			phandle = <0x01>;
			device_type = "cpu";
			reg = <0x00>;
			status = "okay";
			compatible = "riscv";
			riscv,isa = "rv64ima";
			mmu-type = "riscv,sv39";

			interrupt-controller {
				#interrupt-cells = <0x01>;
				interrupt-controller;
				compatible = "riscv,cpu-intc";
				phandle = <0x02>;
			};
		};

		cpu-map {

			cluster0 {

				core0 {
					cpu = <0x01>;
				};
			};
		};
	};

```

linux.dts看起来貌似只有一个核心。所以guest os的dtb也要对应修改，有根据平台核心数量的不同而更改的办法吗？

（简便起见是不是留着这个todo然后把cpu_nums改成2或者5之类的数字也没问题?



## 修改linux.dtb

为了让guest os感知到多核需要修改linux.dtb。

根据老师的建议修改一下qemu.mk

![](./pictures/m-2.png)

在末尾添加一下 “-machine dumpdtb=qemu.dtb”。

然后`make ARCH=riscv64 A=apps/hv HV=y LOG=info SMP=4 run`添加`SMP=4`，

试着添加一下CPU：

```
	cpus {
		#address-cells = <0x01>;
		#size-cells = <0x00>;
		timebase-frequency = <0x989680>;

		cpu@0 {
			phandle = <0x01>;
			device_type = "cpu";
			reg = <0x00>;
			status = "okay";
			compatible = "riscv";
			riscv,isa = "rv64ima";
			mmu-type = "riscv,sv39";

			interrupt-controller {
				#interrupt-cells = <0x01>;
				interrupt-controller;
				compatible = "riscv,cpu-intc";
				phandle = <0x02>;
			};
		};

		cpu@1 {
			phandle = <0x05>;
			device_type = "cpu";
			reg = <0x00>;
			status = "okay";
			compatible = "riscv";
			riscv,isa = "rv64ima";
			mmu-type = "riscv,sv39";

			interrupt-controller {
				#interrupt-cells = <0x01>;
				interrupt-controller;
				compatible = "riscv,cpu-intc";
				phandle = <0x06>;
			};
		};

		cpu@2 {
			phandle = <0x07>;
			device_type = "cpu";
			reg = <0x00>;
			status = "okay";
			compatible = "riscv";
			riscv,isa = "rv64ima";
			mmu-type = "riscv,sv39";

			interrupt-controller {
				#interrupt-cells = <0x01>;
				interrupt-controller;
				compatible = "riscv,cpu-intc";
				phandle = <0x08>;
			};
		};

		cpu@3 {
			phandle = <0x09>;
			device_type = "cpu";
			reg = <0x00>;
			status = "okay";
			compatible = "riscv";
			riscv,isa = "rv64ima";
			mmu-type = "riscv,sv39";

			interrupt-controller {
				#interrupt-cells = <0x01>;
				interrupt-controller;
				compatible = "riscv,cpu-intc";
				phandle = <0x0A>;
			};
		};



		cpu-map {

			cluster0 {

				core0 {
					cpu = <0x01>;
				};

				core1 {
					cpu = <0x05>;
				};

				core2 {
					cpu = <0x07>;
				};

				core3 {
					cpu = <0x09>;
				};
			};
		};
	};
```

编译获得一堆警告，无视警告继续执行。

![](./pictures/m-3.png)

