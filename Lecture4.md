# Isolation & system call entry/exit

## trap机制

用户程序和内核的切换被称为`trap`,主要发生在三种情况下：

1. 用户程序执行系统调用；
2. 用户程序执行时发生`page fault`,运算时出现除以0等错误。
3. 设备触发中断

在`trap`的过程，需要将硬件转化成适合内核运行的状态，最重要的就是寄存器，寄存器的性能是最佳的，所以我们既希望它可以被用户程序使用也可以被内核使用。

<img src="http://cdn.zhengyanchen.cn/img202209211356981.png" alt="截屏2022-09-21 13.56.08" style="zoom:50%;" />

要考虑的寄存器中包含32个用户寄存器（包含栈寄存器`sp`）和一些比较重要的寄存器：

* `stack pointer`栈寄存器，发生`trap`以后，该寄存器将指向内核的一个地址，因为内核需要一个栈来执行内核的C程序。
* `PC`(Program Counter Register),程序计数器。
* `MODE`，mode模式位寄存器。
* ` STVEC`,指向了内核处理`trap`指令的起始地址。
* `SATP`(Supervisor Address  Translation  and Protection)，存放页表的物理地址。
* `SEPC`(Supervisor  Exception Program Counter),在`trap`过程中保存`PC`的值。
* `SSRATCH`, 之后介绍。

在`trap`开始的时候，需要做以下几件事：

1. 保存32个用户寄存器，切换回来的时候将恢复，这样做的效果在于让用户程序意识不到发生了`trap`.
2. 用户程序此时的`PC`也将被保存到`SEPC`。
3. `MODE`改成`supervisor mode`.
4. `SARP`从指向user page table到指向kernel page table.
5. 跳入内核的C程序中。

还有两件事需要注意：

1. 从隔离性和安全性的角度考虑，`trap`的机制不管是硬件还是内核部分都不能依赖用户空间，比如只是保留了寄存器，而不能使用里面的数据。
2. `supervisor mode`较于`user mode`所能做的事情其实很有限，==只有两点== ：
   1. read/write寄存器----`SATP`,`STVEC`,`SEPC`,`SSCRATCH`。
   2. 访问`PTE_U=0`的地址（反过来，不能访问`PTE_U=1`的地址）。

## trap执行流程



<img src="http://cdn.zhengyanchen.cn/img202209220921323.png" alt="截屏2022-09-22 09.21.37" style="zoom:30%;" />

## 
