# Thread

[TOC]

## 线程Intro

**线程(Thread)**是什么? 

这没有一个明确的定义，但这个概念起源于我们期望在计算机上多核运行多任务时，我们可以简化一个任务为线程 ，换而言之，它可以被认为是单个串行执行代码的单元，并且它只占有一个CPU。在之前的使用中，我们多少有点混用线程和进程的概念，比如在Lecture2中，进程被认为是CPU的抽象，但线程作为CPU单个核某个时段执行程序的抽象才更合适, 毕竟在更复杂的OS比如Linux中，进程中可以有多个线程。

**为什么需要多线程**？

* **并行运算** ：一个程序可以被拆分成多个线程来进行并行运算，这会提高程序的运算速度。
* **多任务** : 我们期望计算可以同时间执行多个任务，比如多用户可以同时登陆一台计算机各自运行它们的程序。
* **简化程序** ：对于程序员而言，多线程还可以简化程序结构

###  线程的状态

如果被线程暂停运行，我们需要保存一些东西来使得该线程下一次再占有CPU的时候可以继续运行，这就是**线程的状态**。线程的状态包含三个部分

* **PC** 程序计数器，当前线程执行指令的位置。
* **Registers** 保存变量的寄存器
* **Stack**,通常来说，每个线程都有自己的栈。栈上记录了函数调用和一些变量。

三者在一起才能准确地反映出线程的执行点。

### 线程系统

线程系统要做的就是管理多线程运行，我们可能启动了成百上千的线程，需要一些**Interleave**策略来保证每个线程都能运行。

* 第一个策略就是在多核处理器上，**多核运行多线程**。每个核有自己的自己硬件寄存器--PC和Registers，那么每个线程的状态也就在这里。但是，如果有4个核可以应付4个线程，却无法同时运行成百上千的线程。

* 所以需要第二个策略，**单核在多线程间`switch`**。我们将在Xv6中看到通过实现线程切换，完成以下过程：

  运行第一个进程，保存其状态，再运行第二个进程，**依次让每个线程**都运行一会，并保存它们的状态，最后回来重新执行第一个进程。

和大多数OS一样，Xv6结合了这两种策略：**多核处理器每个核都运行着线程，每个核再在多线程切换**。

#### 共享内存

不同线程系统主要的区别就是，不同线程是否**share memory**。

* **Xv6 Kernel threads**,Xv6支持内核线程的概念。在Xv6中，每个核都可以执行来自用户的系统调用，进入内核线程。**内核线程间都是共享内存的**，这意味着多核的多线程运行在同一个地址空间里，线程彼此之间是能看到彼此的更新。更具体一点，它们共享所有的数据，所以可能会出现race condition的情况，这也就是上一讲中Lock出现的原因。
* **Xv6 User process**,在Xv6中，每一个用户进程都有自己独立的地址空间，同时每个进程包含了一个线程，线程的状态决定了程序的执行点。所以在 Xv6中，**用户线程之间是不共享内存** 。

* **Linux**，在Linux中，用户进程可以包含多个线程，在同一个进程中，线程间是共享内存的（在一个地址空间）。

### 线程外的其他技术

计算机交织运行多个任务除线程外还有其他的技术，比如`event-driven programming ` 或者`state machine` 。线程并不是支持多任务最有效的技术，但通常是最方便的，对程序员最友好的。

## Xv6线程调度

线程系统存在以下的挑战

* **如何实现线程切换(switching)**，我们将暂停一个线程并启动另一个线程的过程称为*线程调度（Scheduling）*。 Xv6为每个核都创建一个`Scheduler`。
* **what/where/when to save/restore in a thread**

* **处理compute bound thread**, 大多数线程自愿接受调度，但有一些线程可能正在执行花费很长时间的计算，这时候它不愿意接受调度让出CPU给其他的线程运行。所以我们还需要提供某种机制强制从这类compute bound thread夺回CPU的控制权，让它稍后再运行。

### 处理compute bound thread

#### 硬件支持

在每个核上，都有一个硬件设备，它会定时（比如每隔10ms）产生中断。这个中断如前面会强制cpu进入内核的中断处理程序，即便是cpu正在执行一个非常耗时的线程。我们正要利用**定时器中断**来处理compute bound thread。

####  pre-emptive sheduling

内核的中断处理程序，会自愿将cpu **yield** 给`Scheduler`。`Scheduler`再去做线程调度。

```c
  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2 && myproc() != 0 && myproc()->state == RUNNING)
    yield();
```

流程如下

```mermaid
graph LR;
A[Thread1]--timeinterrupt-->B[interrupt handler]--yield-->C[Scheduler]-->D[Thread2]
```

* 这样的流程的前半部分，定时器中断强制把cpu控制权交给内核的过程被称为**pre-emptive sheduling**，

* 后半部分对应着当前用户进程的内核线程会使用调度器调度进入新线程，这是**voluntary scheduling**。

### 线程状态续

除了可以前面提的可以确定线程执行点的状态(pc,regs,stack)。我们还需要可以使用在内核线程调度的线程状态，我们先需要了解以下三种

1. **RUNNING**，表示线程正在某个CPU运行。
2. **RUNABLE**，表示线程还没有在某个CPU运行，但是如何有CPU调度到它，它马上就可以运行。
3. **SLEEPING**,我们会着重在下一讲介绍。这个状态意味着线程在等待一些I/O事件，它需要等待I/O事件结束后唤醒才能运行。

回到pre-emptive sheduling：pre-emptive sheduling进入内核后做的就是将一个RUNNING线程转化为一个RUNABLE线程。

对于RUNNING线程,它的PC和Regs都在cpu硬件里，而RUNABLE线程并没有和CPU关联，所以RUNNING线程转化为RUNABLE线程的时候，内核会将cpu硬件里pc和regs的值都放在内存某处，以待线程的再次运行时恢复。

## Thread switch

### Thread switch 1

系统调用的时候，会把进程CC的状态保存在`trapframe`里，cpu会从ustack（用户栈）切换到kstack（内核栈），内核处理完后将`trapframe` 的保存进程的状态恢复，回到进程CC。

<img src="http://cdn.zhengyanchen.cn/img202211041623112.png" alt="截屏2022-11-04 16.23.03" style="zoom:50%;" />

我们认为进程进入内核后可以认为是进入该进程对应的内核线程。如果用户进程进入内核不是因为系统调用而是因为定时器中断，那么在Xv6中它会经过以下过程切换到另一个用户进程。（以CC和LS为例）

<img src="http://cdn.zhengyanchen.cn/img202211041644205.png" alt="截屏2022-11-04 16.44.38" style="zoom:50%;" />

* CC的用户进程因为定时器中断进入CC的内核线程。

* 将CC的内核线程的寄存器保存在对应的 `context`，CC的进程状态从RUNNING变为RUNABLE。

* Xv6将LS的内核线程对应的`context`内容恢复到寄存器上，LS的进程状态从RUNABLE变为RUNNING。**这就完成了内核线程切换**。

  *==LS的进程原状态是RUNABLE,这意味着它可能之前运行了一半，也因为定时器中断进入内核被调度。这进一步意味，该用户进程的trapframe已经存了用户空间的状态，以及内核线程对应的context也已经存了内核线程的寄存器，这是完成线程调度的前提==*

* LS的内核线程会继续在内核完成工作，再通过恢复LS trapframe中的用户进程回到LS用户进程。

换而言之，在Xv6中，一个用户进程要切换到另一个用户进程都需要(以CC和LS为例)

```mermaid
graph LR;
A[CC的用户进程]-->B[CC对应的内核线程]-->C[LS对应的内核线程]-->D[LS的用户进程]
```

这么一个曲折的过程。

### Thread switch 2

在内核线程里 `context` 是一个很重要的概念，其实它和用户进程保存在`trapframe`里的用户状态差不多 ，==是可以准确反映内核线程执行点的一些寄存器== 。它包含以下内容

```c
// Saved registers for kernel context switches.
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```

​	其中最重要的是两个

* `ra`(return  address)寄存器，它存储了调用函数的下一条指令。
* `sp`(stack pointer)寄存器，它存储了内核线程的栈地址。

现在我们更详细地描述`context`切换的过程。我们将使用线程切换最核心的函数`swtch`，这个函数将接受两个内核线程在内存中`context`的地址，把当前cpu硬件里`context`保存到其中一个，将另一个的`context`加载到cpu硬件上。==其中微妙的是，由于ra寄存器已经被换成了新内核线程的ra，所以当函数`swtch`返回的时候，就跳进了新内核线程 。==

### Thread switch 3

 依然以CC和LS的内核线程切换为例，在Xv6中，它并不是直接通过`swtch`函数直接完成

```mermaid
graph LR;
A[CC内核线程]-->B[schedulder线程]-->C[LS内核线程]
```

而是CC内核线程通过`swtch`进入`schedulder`线程，`schedulder`线程中的`schedulder`函数完成调度，再通过`swtch`函数进入LS内核线程。

所以==`context`会保存在两个位置==

* 每个cpu`schedulder`线程的`context`都保存在和cpu相关的数据结构—cpu结构体里。

* 和用户进程对应的内核线程的`context`保存在和进程相关的数据结构—proc结构体里。

`schedulder`线程会在后面代码的部分详谈，它在`main.c`的最后被启动 ,Xv6给每个cpu的`schedulder`线程都分配了独立的栈。

最后提一嘴，本讲只讲了pre-emptive sheduling，而用户进程主动让出cpu是什么场景呢？比如一个用户进程要进行等待I/O操作的结束，它会使用系统调用`sleep` ,在`sleep`的最后也会调用`swtch`函数进入`schedulder`线程，这里就是主动让出cpu。

## proc结构体

我们现在观察`proc`结构体

```c
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};

```

这里有很多之前提到和线程切换的内容。

* **trapframe**,保存用户进程寄存器的`trapframe`字段，需要注意的是这里放的是**指向`trapframe`的虚拟地址**。而==只有用户进程的`trapframe`虚拟地址映射的物理地址和`proc`结构体`trapframe`的虚拟地址映射的物理地址相同时，`trapframe`才真正地起作用==。因为这两者的映射一个在用户进程的页表里一个在内核页表里，所以特别注意。
* **context**,保存内核线程寄存器的`context` 字段 。
* **kstack**,每个用户进程对应的内核线程的内核栈。
* **state**,`state`字段保存了RUNNING,RUNABLE或者SLEEPING等等的进程状态。
* **lock**，在`proc`结构体里有许多需要维护一致性的数据，所以当然需要`lock`。比如有两个cpu的`schedulder`线程同时想要改变某个进程的状态，锁就会防止这种情况。

## Xv6进程切换实例

单核运行Xv6,用如下的示例函数来看看Xv6的进程切换

```c
#include "kernel/types.h"
#include "user/user.h"

int main(){
    int pid;
    char c;
    pid=fork();
    if(pid!=0){
        c='1';
        printf("parent pid=%d ,child pid=%d\n",getpid(),pid);
    }else{
        c='0';
    }
    for(int i=0;i<100000000;i++)
    {
        if (i%1000000==0){
            write(1,&c,1);
        }
    }
    
    exit(0);
}
```

会出现如下的结果

<img src="http://cdn.zhengyanchen.cn/img202211071128648.png" alt="截屏2022-11-07 11.28.42" style="zoom: 33%;" />

* 该函数做的就是让两个进程间隔一段时间分别打印 0和1，两个进程都是compute bound thread，它们并没有使用`sleep` 来主动cpu。
* 由于这是单核在运行，所以出现了上图的结果说明进程是在不断切换的，这就是定时器中断在起作用。

现在进行调试

1. 设置断点

   ```shell
   b trap.c:207
   ```

   进入

<img src="http://cdn.zhengyanchen.cn/img202211071224649.png" alt="截屏2022-11-07 12.24.09" style="zoom: 25%;" />

2. `devintr()`返回值是2，这说明这是个定时器中断

<img src="http://cdn.zhengyanchen.cn/img202211071224153.png" alt="截屏2022-11-07 12.24.30" style="zoom: 25%;" />

3. 查看此时的进程名称

   <img src="http://cdn.zhengyanchen.cn/img202211071519790.png" alt="截屏2022-11-07 14.53.33" style="zoom:25%;" />

   和进程的pid（父进程）

   <img src="http://cdn.zhengyanchen.cn/img202211071518467.png" alt="截屏2022-11-07 15.17.58" style="zoom:25%;" />

4. 查看用户进程的pc

   <img src="http://cdn.zhengyanchen.cn/img202211071523631.png" alt="截屏2022-11-07 15.23.45" style="zoom:25%;" />

   观察`spin.asm`对应的指令

   <img src="http://cdn.zhengyanchen.cn/img202211071524827.png" alt="截屏2022-11-07 15.24.27" style="zoom:25%;" />

   说明进程切换的时候父进程正在循环中，这符合我们的预期。

5. 当调度完成，再回到`trap`,此时cpu上运行的进程的pid为4。

   <img src="http://cdn.zhengyanchen.cn/img202211071538834.png" alt="截屏2022-11-07 15.38.55" style="zoom:25%;" />

#### 错误

最开始想打印变量如`p->name`的时候会出现错误

```ABAP
dwarf2_find_location_expression: Corrupted DWARF expression.
```

按照[这篇的指导](https://www.reddit.com/r/RISCV/comments/plgwyk/riscv64unknownelfgdb_gives_dwarf_error_when/)需要修改`Makefile`对 Xv6进行重新编译。

```Makefile
CFLAGS = -Wall -Werror -O -fno-omit-frame-pointer -ggdb -gdwarf-2
```



## yield,sched,swtch函数

在`usertrap`中,如果是定时器中断则会进入`yield`函数。

```c
if(which_dev == 2)
  yield();
```

### yield

来看 `yield`函数

```c
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```

* 获取当前cpu用户进程的锁，这里加锁很重要，因为我们将修改进程的状态为RUNNABLE。我们依然用CC和LS进程切换的例子：

  ==如果这里不加锁，其他cpu核也会见到CC进程的状态（RUNNABLE），它可能会运行CC进程对应的内核线程。事实上该CC进程虽然状态字段修改了，但还没有完成调度，也就是说它的内核线程依然在运行，一旦cpu运行CC的内核线程==，那么就意味着CC的内核线程就被两个cpu核使用，而且它们共用一个内核栈，**这会导致程序直接崩溃**。

`yield`函数会调用`sched`函数

### sched

```c
void
sched(void)
{
  int intena;
  struct proc *p = myproc();

  if(!holding(&p->lock))
    panic("sched p->lock");
  if(mycpu()->noff != 1)
    panic("sched locks");
  if(p->state == RUNNING)
    panic("sched running");
  if(intr_get())
    panic("sched interruptible");

  intena = mycpu()->intena;
  swtch(&p->context, &mycpu()->context);
  mycpu()->intena = intena;
}
```

* `sched`中间的部分主要是一些常规的检查，不详谈。
* `intena`表示当前的cpu是否开中断，这个字段在将来恢复进程状态有用。
* 主要是`swtch`函数

### swtch

`sched`函数中会这样使用`swtch`，

```c
swtch(&p->context, &mycpu()->context);
```

* 第一个参数是结构体`proc`中`context`的地址
* 第二个参数是结构体`cpu`中`context`的地址

`swtch`函数

```assembly
.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret
```

* a0寄存器对应着`swtch`函数的第一个参数,a1寄存器对应着`swtch`函数的第二个参数
* `swtch`函数会将当前内核线程的寄存器保存到`p->context`中，同时将`mycpu()->context`加载到对应的寄存器上。`mycpu()->context`存放的正是调度器现场，我们可以简单查看一下：

<img src="http://cdn.zhengyanchen.cn/img202211072347633.png" alt="截屏2022-11-07 23.47.37" style="zoom:43%;" />

<img src="http://cdn.zhengyanchen.cn/img202211080005622.png" alt="截屏2022-11-08 00.04.29" style="zoom:43%;" />

* 可以看到`ra`寄存器存放的正是`scheduler`函数的地址 。
* `sp`在`swtch`前是一个很高的地址，这是内核栈。

<img src="http://cdn.zhengyanchen.cn/img202211080847266.png" alt="截屏2022-11-08 08.47.11" style="zoom:20%;" />

* 在`swtch`后，`sp`离内核数据区很近，这个栈是调度器栈，是在`entry.S`初始化的。

<img src="http://cdn.zhengyanchen.cn/img202211080852974.png" alt="截屏2022-11-08 08.52.04" style="zoom:20%;" />

### why 14 registers?

 现在还有一个问题，那就是RISC-V有32个寄存器，为什么只保存`context`这14个寄存器?

除了`ra`和`sp`寄存器外，==需要保存的`context`都是callee saved register== 。那些caller saved registers呢？事实上，swtch函数是被C代码调用，这意味着caller saved registers在进入callee之前就保存caller frame里。

<img src="http://cdn.zhengyanchen.cn/img202210071540327.png" alt="截屏2022-10-07 15.40.25" style="zoom:25%;" />

* 以从`swtch`函数从内核切换到调度器进程为例，当`swtch`返回后，caller saved registers保存在内核栈，caller saved registers保存在`p->context`里，所有硬件寄存器已经是调度器线程的了 。

**所以现在我们反思，==到底什么是线程？，在我理解，其实就是完整的寄存器。==所以线程的状态，就是线程**

## scheduler函数

<img src="http://cdn.zhengyanchen.cn/img202211080005622.png" alt="截屏2022-11-08 00.04.29" style="zoom:43%;" />

* 如上一小节所说，内核线程会通过`swtch`函数跳到调度器进程这个位置，这是什么代码呢？来看

  <img src="http://cdn.zhengyanchen.cn/img202211081939160.png" alt="截屏2022-11-08 19.39.12" style="zoom:43%;" />

再看`scheduler`完整的函数

```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    
    int nproc = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state != UNUSED) {
        nproc++;
      }
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
      }
      release(&p->lock);
    }
    if(nproc <= 2) {   // only init and sh exist
      intr_on();
      asm volatile("wfi");
    }
  }
}
```

* 所以内核线程进入调度器的`c->proc=0`位置，有趣的是它正在`swtch`后面。

  *`swtch`函数仿佛是一道传送门，此内核线程在之前的调度中被调度器线程通过`swtch`函数进入，现在又回到调度器线程。*

* `c->proc=0`做的是清除当前cpu结构体里指向的进程。

* `release(&p->lock)`,之前在`yield`函数中,我们获得了该进程的锁，现在内核线程的所有信息都已经保存，我们在调度器线程中，

  ```c
  void
  yield(void)
  {
    struct proc *p = myproc();
    acquire(&p->lock);
    p->state = RUNNABLE;
    sched();
    release(&p->lock);
  }
  ```

* `scheduler`函数将通过遍历所有的进程，查看是否有处于RUNABLE的进程，进行调度。

* ```c
  p->state = RUNNING;
  c->proc = p;
  swtch(&c->context, &p->context);
  ```

  这三步的操作应该是原子性的，所以应该在前面加锁。

* 有趣的是，这个锁在`yield`被释放。









































