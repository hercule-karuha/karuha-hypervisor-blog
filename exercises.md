# hypercraft RISC-V练习

## 练习1
1. 在arceos中运行linux
2. （1）阅读[The RISC-V Instruction Set Manual Volume II: Privileged Architecture](https://drive.google.com/file/d/1EMip5dZlnypTk7pt4WWUKmtjUKTOkBqh/view) Chapter 8(8.1)Privilege Modes，回答RISCV引入虚拟化扩展后，其特权模式有哪些新的变化，各模式之间存在什么关系与区别。（2）请问Hypercraft是哪一种hypervisor。



1.运行成功

![e-1](./pictures/e-1.png)

2.

（1）S-mode变为了HS-mode，U-mode变为VS-mode和VU-mode。其中gust OS运行在VS-mode上。HS-mode运行hypervisor或者OS，与硬件交流和原本的S-mode差不多。需要向上面的VS-mode提供SBI。

（2）属于type-2的hypervisor，运行在arceos上面，由hv app使用。