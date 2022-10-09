# Lab4 traps

## Risc-v assembly

* ![截屏2022-10-07 11.35.54](http://cdn.zhengyanchen.cn/img202210071135665.png)

  

  观察`main`的汇编，似乎压根就没有调用f(x),而是已经算出了结果12放进寄存器

* 对于`printf("%d %d\n",)`三个参数

  ```assembly
  28:	00000517          	auipc	a0,0x0
  2c:	79050513          	addi	a0,a0,1936 # 7b8 <malloc+0xea>
  ```

  1. `"%d %d\n"` 内存地址放在`a0`中，数值为0x7b8(1936)

  *这个位置由`malloc`调用产生，所以在heap上*

  ```assembly
    24:	4635                	li	a2,13
  ```

  2. `13`存放在寄存器`a2`中

  ```assembly
    26:	45b1                	li	a1,12
  ```

  3. `f(8)+1`的结果12存放在寄存器`a1`中

     

* 对于`exit`,参数0放进了寄存器`a0`

* `printf()`地址位于0x616(可以看到这位于用户空间的第一个page)

* `ra`=0x616

* 输出

  <img src="http://cdn.zhengyanchen.cn/img202210071155361.png" alt="截屏2022-10-07 11.55.10" style="zoom: 25%;" />

  如果大端，i修改如下,57616 无需修改

  ```c
  unsigned int i=0x726c6400;
  ```

* <img src="http://cdn.zhengyanchen.cn/img202210071204946.png" alt="截屏2022-10-07 12.04.28" style="zoom:25%;" />

  y输出的是`a1` 寄存器的原来内容。



## Backtrace



![截屏2022-10-07 21.22.37](http://cdn.zhengyanchen.cn/img202210072122637.png)



## Alarm

再次复习一遍，`extern` 关键字用来辨识该函数的定义不在当前文件，提醒编译器去其他模块寻找该函数的定义。

**要做的主要添加系统调用和修改`trap`**



1. **系统调用与`sigalarm`**

* 在`user/user.h`中添加

  ```c
  int sigalarm(int ticks, void (*handler)());
  int sigreturn(void);
  ```

* 在`user/usys.pl`添加

  ```c
  entry("sigalarm");
  entry("sigreturn");
  ```

* 在`kernel/syscall.h`添加

  ```c
  #define SYS_sigalarm 22
  #define SYS_sigreturn 23
  ```

* 在`kernel/syscall.c`添加

  ```c
  extern uint64 sys_sigalarm(void);
  extern uint64 sys_sigreturn(void);
  ```

* 同样文件的函数`(*syscalls[])(void)`继续添加

   ```c
   [SYS_sigalarm] sys_sigalarm,
   [SYS_sigreturn] sys_sigreturn,
   ```

* 在`kerne/sysproc.c`中添加

  ```c
  uint64
  sys_sigalarm(void){
    struct proc *p=myproc();
    if (argint(0,&p->alarm)){
      return -1;
    }
    p->countTick=0;
    if (argaddr(1,&p->handler)){
      return -1;
    }
    return 0;      
  }
  ```

  这里已经在`kernel/proc.h`的 `struct proc`添加

  ```c
  int alarm;//the interval of the alarm
  uint64 handler;//the alarm handler处理程序的地址
  int countTick;//这个参数是用来在计数发生计数器中断的次数，也就是从系统调用开始后tick的发生次数。这里需要初始化为0
  ```

* 在`kernel/proc.c`的`allocproc`添加对此的初始化

  ```c
  p->alarm=-1;
  p->countTick=0;
  ```

系统调用以后，回到用户空间。每过一个tick将发生一次计时器中断，进入`trap`:



2. `trap`和`sigreturn`

* 需要修改`kernel/trap.c`里的`usertrap`函数

  ```c
   if(which_dev == 2){
      if (p->alarm>0){
        p->countTick++;
        if (p->countTick==p->alarm && p->handle_working==0 ){
          p->handle_working=1;
          p->ra=p->trapframe->ra;
          p->sp=p->trapframe->sp;
          p->gp=p->trapframe->gp;
          p->tp=p->trapframe->tp;
          p->t0=p->trapframe->t0;
          p->t1=p->trapframe->t1;
          p->t2=p->trapframe->t2;
          p->s0=p->trapframe->s0;
          p->s1=p->trapframe->s1;
          p->a0=p->trapframe->a0;
          p->a1=p->trapframe->a1;
          p->a2=p->trapframe->a2;
          p->a3=p->trapframe->a3;
          p->a4=p->trapframe->a4;
          p->a5=p->trapframe->a5;
          p->a6=p->trapframe->a6;
          p->a7=p->trapframe->a7;
          p->s2=p->trapframe->s2;
          p->s3=p->trapframe->s3;
          p->s4=p->trapframe->s4;
          p->s5=p->trapframe->s5;
          p->s6=p->trapframe->s6;
          p->s7=p->trapframe->s7;
          p->s8=p->trapframe->s8;
          p->s9=p->trapframe->s9;
          p->s10=p->trapframe->s10;
          p->s11=p->trapframe->s11;
          p->t3=p->trapframe->t3;
          p->t4=p->trapframe->t4;
          p->t5=p->trapframe->t5;
          p->t6=p->trapframe->t6;
          p->tmp_pc=p->trapframe->epc;
          p->trapframe->epc=p->handler;
          p->countTick=0;
        }
      }
   }
  ```

  这里又需要在`kernel/proc.h`的 `struct proc`添加

  ```c
  uint64 tmp_pc;
  uint64 ra;
  uint64 sp;
  uint64 gp;
  uint64 tp;
  uint64 tmp_pc;
  uint64 ra;
  uint64 sp;
  uint64 gp;
  uint64 tp;
  uint64 t0;
  uint64 t1;
  uint64 t2;
  uint64 s0;
  uint64 s1;
  uint64 a0;
  uint64 a1;
  uint64 a2;
  uint64 a3;
  uint64 a4;
  uint64 a5;
  uint64 a6;
  uint64 a7;
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
  uint64 t3;
  uint64 t4;
  uint64 t5;
  uint64 t6;
  uint64 handler_working;
  ```

  之所以需要在`proc`字段添加保存寄存器的变量，是因为一旦进入`handler`程序，那么就会使用用户寄存器，所以为了`handler`程序结束以后，可以回到原程序`alarmtest`

  **同时注意，当再发生计时器中断时，如果handler程序还没结束，==时间已经又过了alarm个tick，为了防止再次进入handler程序==，这里还需要一个变量，我叫它`handler_working`,这个变量表示已经进入handler程序了。**

  

  * 在`kerne/sysproc.c`添加第二个系统调用`sigreturn`

    ```c
    uint64 
    sys_sigreturn(void){
      struct proc *p=myproc();
      p->trapframe->ra=p->ra;
      p->trapframe->sp=p->sp;
      p->trapframe->gp=p->gp;
      p->trapframe->tp=p->tp;
      p->trapframe->t0=p->t0;
      p->trapframe->t1=p->t1;
      p->trapframe->t2=p->t2;
      p->trapframe->s0=p->s0;
      p->trapframe->s1=p->s1;
      p->trapframe->a0=p->a0;
      p->trapframe->a1=p->a1;
      p->trapframe->a2=p->a2;
      p->trapframe->a3=p->a3;
      p->trapframe->a4=p->a4;
      p->trapframe->a5=p->a5;
      p->trapframe->a6=p->a6;
      p->trapframe->a7=p->a7;
      p->trapframe->s2=p->s2;
      p->trapframe->s3=p->s3;
      p->trapframe->s4=p->s4;
      p->trapframe->s5=p->s5;
      p->trapframe->s6=p->s6;
      p->trapframe->s7=p->s7;
      p->trapframe->s8=p->s8;
      p->trapframe->s9=p->s9;
      p->trapframe->s10=p->s10;
      p->trapframe->s11=p->s11;
      p->trapframe->t3=p->t3;
      p->trapframe->t4=p->t4;
      p->trapframe->t5=p->t5;
      p->trapframe->t6=p->t6;
      p->trapframe->epc=p->tmp_pc;
      p->handler_working=0;
    
      return 0;
    }
    ```

    在`handler`程序结束的地方将使用`sigreturn`，将所有的寄存器复原，又回到`alarmtest`的程序。

  思路整理如下：

  ```mermaid
  graph TD;
  A[alarmtest,use sigalarm]-->B[trap--set alarm and handler,from now,count tick ]-->C[alarmtest]-->E[if time interrupt happens,countTick+1]-->F[after alarm tick,enter handler--save all the registers]-->G[at the end of handler,recover all the registers,return alarmtest]-->H[do last two steps by loop]-->I[use sigalarm again, shup dowm the handler]
  ```

  









