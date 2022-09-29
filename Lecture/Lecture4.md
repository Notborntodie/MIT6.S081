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

进行调试。`si`进入，`pc`从一个很低的地址到了很高的地址，这实际进入了`trampoline page`。

<img src="http://cdn.zhengyanchen.cn/img202209270901864.png" alt="截屏2022-09-27 09.01.22" style="zoom:50%;" />

此时`ecall`已经完成了这几件事

1. 更改 `mode`寄存器，完成从user mode到supervisor mode的转换，（`trampoline page`的`PTE_U`并没有设置，这保证了`trap`机制的安全）。
2. 将用户空间`pc`的值保存在`sepc`寄存器里（打印此时`sepc`的值，可以看到保存正是前面`ecall`指令的地址）。

<img src="http://cdn.zhengyanchen.cn/img202209270920280.png" alt="截屏2022-09-27 09.19.57" style="zoom:33%;" />

3. 跳转到`stvec`寄存器指向的指令，这正是`trampoline page`的开始地址（这里是 `trap`开始的位置）。

<img src="http://cdn.zhengyanchen.cn/img202209270923258.png" alt="截屏2022-09-27 09.23.43" style="zoom: 33%;" />

这都是硬件完成的，这似乎做的太少了，`ecall`并没有完成以下的操作：

1. 没有更改`satp`,这意味着此时依然在使用用户页表，某种程度上，这也是为什么每一个用户页表都需要设置到`trampoline page`的映射却不设置`PTE_U`的原因。（验证一下，在`qemu`的控制台打印页表，和`ecall`之前完全一样）。		![截屏2022-09-27 09.33.37](http://cdn.zhengyanchen.cn/img202209270933602.png)
2. 没有保存或者使用32个用户寄存器，尤其是没有改变`sp`寄存器，所以栈寄存器依然指向用户栈而非内核栈。

以上`ecall`没做的都是软件需要完成的。在本节中，教授还讲解了`risc-v`为什么要这么设计而不去做这些软件的工作。

* **设计原则**：`risc-v`的设计原则是尽量少地完成必要的工作。
* **灵活性**：不做这些工作的结果就是给OS的设计者提供最大程度的自由。
  1. 比如如果用户页表和内核页表拥有共同的映射，那么系统调用时，OS甚至可以设计在部分场景下不切换页表。
  2. 在`trap`的过程中，某些寄存器可以不用保存，`ecall`不保存寄存器的设计可以让OS选择性地保存寄存器。
  3. 在某些简单的系统调用中，甚至不需要栈,那么`sp`都不需要切换。

## uservec

我们现在要做`ecall`以上未做的几件事

观察`trampoline.S`汇编代码

1. ```assembly
   csrrw a0,sscratch,a0
   ```

   `csrrw`指令的作用是交换`a0`和`sscratch`的寄存器内容，交换后`a0`保存的地址如下，这是`trapframe`的地址。

   <img src="http://cdn.zhengyanchen.cn/img202209271039698.png" alt="截屏2022-09-27 10.39.25" style="zoom: 33%;" />

   我们可以在`pro.h`里看到`trapframe`的结构，

   *在这里打断一下，那就是`sccratch`是什么时候保存的trapframe地址的？*

   1. 机器启动后都是由内核跳入用户空间，而这一过程都是由`sret`指令完成的，此时`a0`保存了`trapframe`地址。<img src="http://cdn.zhengyanchen.cn/img202209271131572.png" alt="截屏2022-09-27 11.31.49" style="zoom:33%;" />

   2. 那么`a0`是如果获得的？看`trap.c`里`usertrapret`函数，传入函数`fn`的第一个参数保存在`a0` 上。

      ![截屏2022-09-27 11.33.40](http://cdn.zhengyanchen.cn/img202209271133341.png)

      ==这里`fn`和强制转化为函数指针的用法将在后面的部分解释。==

   ```c
   struct trapframe {
     /*   0 */ uint64 kernel_satp;   // kernel page table
     /*   8 */ uint64 kernel_sp;     // top of process's kernel stack
     /*  16 */ uint64 kernel_trap;   // usertrap()
     /*  24 */ uint64 epc;           // saved user program counter
     /*  32 */ uint64 kernel_hartid; // saved kernel tp
     /*  40 */ uint64 ra;
     /*  48 */ uint64 sp;
     /*  56 */ uint64 gp;
     /*  64 */ uint64 tp;
     /*  72 */ uint64 t0;
     /*  80 */ uint64 t1;
     /*  88 */ uint64 t2;
     /*  96 */ uint64 s0;
     /* 104 */ uint64 s1;
     /* 112 */ uint64 a0;
     /* 120 */ uint64 a1;
     /* 128 */ uint64 a2;
     /* 136 */ uint64 a3;
     /* 144 */ uint64 a4;
     /* 152 */ uint64 a5;
     /* 160 */ uint64 a6;
     /* 168 */ uint64 a7;
     /* 176 */ uint64 s2;
     /* 184 */ uint64 s3;
     /* 192 */ uint64 s4;
     /* 200 */ uint64 s5;
     /* 208 */ uint64 s6;
     /* 216 */ uint64 s7;
     /* 224 */ uint64 s8;
     /* 232 */ uint64 s9;
     /* 240 */ uint64 s10;
     /* 248 */ uint64 s11;
     /* 256 */ uint64 t3;
     /* 264 */ uint64 t4;
     /* 272 */ uint64 t5;
     /* 280 */ uint64 t6;
   };
   
   ```

2. <img src="http://cdn.zhengyanchen.cn/img202209271047447.png" alt="截屏2022-09-27 10.47.27" style="zoom:50%;" />

   之前可以看到跳入内核后的第二个命令，这里做的是将`sd`寄存器的内容加载到内存中`a0`+40的位置,此后做的指令基本相似，就是把31个其他的用户寄存器依`trapframe`的结果保存。

<img src="http://cdn.zhengyanchen.cn/img202209271056754.png" alt="截屏2022-09-27 10.56.51" style="zoom:50%;" />

​		在这一部分的结尾，还做了一个小操作

<img src="http://cdn.zhengyanchen.cn/img202209271438774.png" alt="截屏2022-09-27 14.38.45" style="zoom:50%;" />

​			将原`a0`的值也保存在`trapframe`上（`sccratch`依然保留了原`a0`的值）。

* ==Q：其实我有点好奇为什么不直接利用`sccratch`加上偏移值得到用户寄存器加载到内存的地址？==

  3. ![截屏2022-09-27 14.55.42](http://cdn.zhengyanchen.cn/img202209271455464.png)

     这一部分注释已经讲解得很清楚了，我们跳到`sfence.vma`,看看此时的栈指针和页表：

     <img src="http://cdn.zhengyanchen.cn/img202209271459173.png" alt="截屏2022-09-27 14.59.46" style="zoom:50%;" />
     
     ![截屏2022-09-27 15.25.31](http://cdn.zhengyanchen.cn/img202209271525226.png)
     
     这就是完全不一样很长的页表。栈顶指针可以推断出是`sh`是第二个进程。
     
  3. 最后，将执行
  
     ```assembly
     jr t0
     ```
     
     `uservec`将跳入`usertrap`函数。
  
  * 可以看到`trampoline page`的神奇之处，这段代码在所有用户和内核的页表有完全一样的映射，所以切换页表时程序不会崩溃。而每个用户进程有它们独享的`trapframe`,虽然它们自己并不能访问。



## usertrap

[这里又有坑...需要重新编译工具链,怎么坑那么多，重新编译真的很费时间好吧。。。](https://stackoverflow.com/questions/67162162/the-gdb-in-my-machine-does-not-support-tui-display)

现在跳进`usertrap`,下面来分步讲解这里的代码

* ```c
  w_stvec((uint64)kernelvec);
  ```

  首先做的是更改`stvec`寄存器的内容，之前已经知道该寄存器存放的是处理`trap`的最初代码的地址，然而那是处理来自用户空间的`trap`,`trap`也可能由内核空间发起，这里切换后`stvec`存放的`kernelvec`正是处理内核空间`trap`代码的地址。

* ```c
  struct proc *p = myproc();
  ```

  确定用户空间运行的进程。之前我们已经多次调用了`myproc`函数，现在来看看它是如何工作的，

* 

* 

​	

