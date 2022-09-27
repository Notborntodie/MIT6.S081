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

**接下来的将通过在sh程序系统调用`write`来具体阐述`trap`的发生过程**

##  ecall指令之前的状态

来看`sh.asm`里,注意到`write`第一个的地址为`0xdea`：

```asm
0000000000000dea <write>:
.global write
write:
 li a7, SYS_write
     dea:	48c1                	li	a7,16
 ecall
     dec:	00000073          	ecall
 ret
     df0:	8082                	ret
```

于是打开gdb开始调试，并在`0xdea`处设置断点：

```sh
(gdb) b *0xdea
(gdb) c
```

注意此时还没有调用`ecall`,让我们查看此时的各种状态信息：

* 首先查看`pc`,

  <img src="http://cdn.zhengyanchen.cn/img202209231041833.png" alt="截屏2022-09-23 10.41.12" style="zoom: 50%;" />

  这是我们设置断点的位置，需要注意的是`0xdea`这是函数指针类型的。

* 然后查看`satp`,

  <img src="http://cdn.zhengyanchen.cn/img202209231156676.png" alt="截屏2022-09-23 11.56.03" style="zoom:50%;" />

  但这只告诉了我们`satp`存放页表的物理地址，于是在`qemu`的界面键入`ctrl a+c`进入`qemu`的console ,输入`info mem`便可以看到页表信息：

<img src="http://cdn.zhengyanchen.cn/img202209231208652.png" alt="截屏2022-09-23 12.08.54" style="zoom:50%;" />

​		*标志位a（accessed）表示该pte是否被使用过,d(dirty)表示该pte是否被写过*

​		可以看到`shell`的地址分布:

1. `0x0000`到` 0x1000`为`shell`的代码和数据段

2. `0x2000`是guard page

3. `0x3000`是stack page

   需要注意的是`shell`是一个小程序，地址都很低，而页表中两个很高的地址分别为`trapframe page`和`trampoline page`,他们都没有设置`PTE_U`。

* 还可以查看`a0`,`a1`和`a2`寄存器
  * `a0`,文件描述符
  * `a1`,`shell`想要写的字符串的指针，可以看到字符串在栈上。
  * `a2`,写入的字符数

<img src="http://cdn.zhengyanchen.cn/img202209231225168.png" alt="截屏2022-09-23 12.25.18" style="zoom:50%;" />

* 最后可以通过键入查看所有寄存器

  ```shell
  info reg
  ```

  也可以键入查看`write`函数的指令

  ```shell
  x/3i 0xdea
  ```



##  ecall指令后的状态

这里有个大坑，那就是不能使用`gdb-multiarch` 来调试，如果从`ecall`处`si`,并不能跳进内核空间，需要安装`riscv-gnu-toolchain`,（ 如何安装在`配置环境.md`里讲解）并使用

```shell
make CPUS=1 qemu-gdb
```

```shell
riscv32-unknown-elf-gdb kernel/kernel 
```

进行调试















​		
