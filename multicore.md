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
			reg = <0x01>;
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
			reg = <0x02>;
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
			reg = <0x03;
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

出现个报错：

![](./pictures/m-4.png)

查看源代码得知是无法生成对应的sbi message，需要处理hart start的sbi

![](./pictures/m-6.png)



## 多个vcpu的初始化

我太菜了，所以先整两个cpu。

### secondary_main()

main()函数里面做了这些事情：

- 初始化PerCpu()
- 创建vcpu
- 创建vm
- 创建页表
- vm.run
- 目前我所做的是一个vcpu对应一个物理核心，思路是每个物理cpu创建自己的vcpu添加到vm里面。

所以需要把vm变成static的。

```
use lazy_init::LazyInit;

static mut HS_VM: LazyInit<VM<HyperCraftHalImpl, GuestPageTable>> = LazyInit::new();

use core::sync::atomic::{AtomicUsize, Ordering};

static INITED_VCPUS: AtomicUsize = AtomicUsize::new(0);
```

模仿arceos定义INITED_VCPUS来记录已经初始化了的cpu，以同步主核和副核的初始化。

在axruntime里面添加secondary_main()的代码：

```
    #[cfg(feature = "hv")]
    unsafe {
        secondary_main(cpu_id);
    }
```

并在secondary_main()里面完成初始化操作。

### 同步

主核和副核的初始化需要同步，具体为

1. 主核创建vm后副核才能向vm添加vcpu
2. 所有副核初始化完毕后主核才能开始运行vm
3. 所有副核初始化完毕后主核才能运行自己的vcpu

模仿arceos通过`core::hint::spin_loop();`来实现

如副核等待vm初始化的代码：

```
    while let None = unsafe { HS_VM.try_get() } {
        core::hint::spin_loop();
    }
```

其他的同步只需要检测INITED_VCPUS的值就好了。

### vcpu可执行状态

副核的vcpu初始化的时候没有正确的入口地址，正确的入口地址需要在guest调用 sbi_hart_start的时候才会给出，所以需要一个状态来记录vcpu是否可执行。

在vcpu.rs中已经定义了一个：

```
/// The availability of vCPU in a VM.
pub enum VmCpuStatus {
    /// The vCPU is not powered on.
    PoweredOff,
    /// The vCPU is available to be run.
    Runnable,
    /// The vCPU has benn claimed exclusively for running on a (physical) CPU.
    Running,
}
```

将他作为一个字段添加到`VmCpuStatus`中。

可运行状态在主核唤醒副核的时候由主核修改所以vcpu需要加锁。

```
pub struct VmCpus<H: HyperCraftHal> {
    inner: [Once<Mutex<VCpu<H>>>; VM_CPUS_MAX],
    marker: core::marker::PhantomData<H>,
}
```

在vm.run()里面添加部分代码用来阻止vcpu的运行：

```
while !self.is_runnable(vcpu_id) {
    core::hint::spin_loop();
}
```

## hart_start的处理

主核启动之后，linux会调用hart_start()来启动副核，需要根据sbi手册的规定来处理。

```
let vcpu = self.vcpus.get_vcpu(hartid as usize).unwrap();
let mut vcpu = vcpu.lock();
vcpu.start_init(hartid, start_addr, opaque);
vcpu.set_status(crate::VmCpuStatus::Runnable);
gprs.set_reg(GprIndex::A0, 0);
```

start负责为vcpu设置寄存器

```
pub fn start_init(&mut self, hart_id: usize, start_addr: usize, opaque: usize) {
    self.regs.guest_regs.gprs.set_reg(GprIndex::A0, hart_id);
    self.regs.guest_regs.gprs.set_reg(GprIndex::A1, opaque);
    self.regs.guest_regs.sepc = start_addr;
}
```

将`VmCpuStatus`设置为`Runnable`之后，vm.run就可以继续向下执行了。

## 处理IPI

为了正常启动还需要处理IPI。

## send_ipi

首先是要处理发送ipi的sbi，简单调用sbi即可。

```
HyperCallMsg::IPI(ipi) => {
    let IPIFunction::SendIPI {
        hart_mask,
        hart_mask_base,
    } = ipi;
    let sbiret = sbi_rt::send_ipi(hart_mask, hart_mask_base);
    gprs.set_reg(GprIndex::A0, sbiret.error);
    gprs.set_reg(GprIndex::A0, sbiret.value);
}
```

## SupervisorSoft

send_ipi会使得目标核心触发一个SupervisorSoft中断。

但是，按照guest的视角来看，它并不知道HS-MODE发生的事情。

需要处理plic向guest报告一个`VIRTUAL_SUPERVISOR_SOFT`中断。

模仿原本的`handle_irq`写一个`handle_soft_irq`

```
fn handle_soft_irq(&mut self, vcpu_id: usize) {
    let context_id = vcpu_id * 2 + 1;
    // let claim_and_complete_addr = self.plic.base() + 0x0020_0004 + 0x1000 * context_id;
    // let irq = unsafe { core::ptr::read_volatile(claim_and_complete_addr as *const u32) };
    // assert!(irq != 0);
    self.plic.claim_complete[context_id] = 16;
    CSR.hvip
        .read_and_set_bits(traps::interrupt::VIRTUAL_SUPERVISOR_SOFT);
}
```

和原本的`VIRTUAL_SUPERVISOR_EXTERNAL`还是存在差别，评价为，寄了。

这里将irq写死为一个值，因为如果按照原本处理`EXTERNAL`的办法来处理会得到一个0值。

在某些情况下会卡死，寄。
