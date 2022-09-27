# LAB2

首先需要跳到lab2的分支，会出现以下的错误。

![截屏2022-06-23 11.01.12](http://cdn.zhengyanchen.cn/img202206231101099.png)

按它说的，进行以下操作，进行切换就可以了。

![截屏2022-06-23 11.07.10](http://cdn.zhengyanchen.cn/img202206231107860.png)



## trace

跟踪系统调用的思路是在结构体`proc`中增加一个变量，我们称它为`mask`。当调用系统调用`trace`时，会将`mask`的值更改为`trace`传递的参数，这个值不一定是syscall number,只是掩码。需要注意由于不仅仅需要追踪当前进程的系统调用，还要追踪子进程的，所以还需要修改系统调用`fork`的代码。



1. `pro.h`中`struct proc`添加

   ```c
     uint32 mask;
   ```

   *其实这里我有一点小小的疑惑，C语言struct内的变量不能初始化，而事实上mask被初始化为0*

2. `syscall.h`添加

   ```c
   #define SYS_trace  22
   ```

3. `syscall.c`添加

   ```c
   extern uint64 sys_trace(void);
   ```

   *引用外部变量或者外部函数时使用 `extern`*

   在`static uint64 (*syscalls[])(void) `中添加

   ```c
   [SYS_trace]   sys_trace,
   ```

   添加函数

   ```c
   static char* sysname[]={
     " ", 
     "fork",
     "exit",
     "wait",
     "pipe",
     "read",
     "kill",
     "exec",
     "fstat",
     "chdir",
     "dup",
     "getpid",
     "sbrk",
     "sleep",
     "uptime",
     "open",
     "write",
     "mknod",
     "unlink",
     "link",
     "mkdir",
     "close",
     "trace",
   };
   ```

   `syscall`函数修改为

   ```c
   void
   syscall(void)
   {
     int num;
     struct proc *p = myproc();
     num = p->trapframe->a7;
     if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
       p->trapframe->a0 = syscalls[num]();
       if ((p->mask)&(1<<num)){
         printf("%d: syscall %s -> %d\n",p->pid,sysname[num],p->trapframe->a0);
       }
     } else {
       printf("%d %s: unknown sys call %d\n",
               p->pid, p->name, num);
       p->trapframe->a0 = -1;
     }
   }
   ```

   

4. `pro.c`中`fork`函数添加

   ```c
   np->mask=p->mask;
   ```

5. `sysproc.c`添加

    ```c
    uint64 
    sys_trace(void){
      int mask;
      if (argint(0,&mask)<0){
        return -1;
      }
      myproc()->mask=mask;
      return 0;
    }
    ```

​		这里需要`syscall.c `里的`argint`来传递参数,有一件事情值得注意，在 `syscall.c`中，像`argint`这样需要被其他文件调用的代码没有加上`static`,而其他自己独享的函数都加上了`static`。

这样就基本完成了`kernel`部分的代码，现在来看如何实现从用户程序跳进内核。

1. 首先是对`user.h`的修改，添加

   ```c
   int trace(int);
   ```

2. 然后再在`usys.pl`添加

   ```c
   entry("trace");
   ```

   这会在`usys.S`中生成如下的汇编代码

   ```assembly
   .global trace
   trace:
    li a7, SYS_trace
    ecall
    ret
   ```

​		 首先将参数传到a7寄存器，然后通过`ecall`跳进内核。

---



## sysinfo

关于如何实现系统调用的代码将不再赘述。

1. 在`sysproc.c`中添加函数

   ```c
   uint64
   sys_sysinfo(void){
     struct sysinfo info;
     uint64 addr;  
     struct proc *p = myproc();
     info.freemem=getfreememory();  
     info.nproc=getusingproc();
     if (argaddr(0,&addr)<0){
       return -1;
     }
     if (copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0){
       return -1;
     }
     return 0;  
   }
   ```

在写这个函数时，需要学会函数`copyout`的使用。

<img src="http://cdn.zhengyanchen.cn/img202206241904353.png" alt="截屏2022-06-24 19.03.55" style="zoom:33%;" />

在阅读`filestat`函数如何使用`copyout`函数时，发现第三个参数发生了指针类型强制转化，其实这里只是为了和函数的参数类型不发生冲突。



下面分别在`proc.c`和`kalloc.c`中添加函数 

2. `getusingproc`

```c
uint64 getusingproc(){
  uint64 count=0;
  for (int i = 0; i < NPROC; i++)
  {
    if (proc[i].state!=UNUSED){
      count++;
    }
  }
  return count;
}

```



3. `getfreememory`

```c
uint64  getfreememory(){
  uint64 count=0;
  struct  run  *p=kmem.freelist;
  while (p)
  {
    count+=PGSIZE;
    p=p->next;
  }
  return count;
    
}
```



<img src="http://cdn.zhengyanchen.cn/img202206241912878.png" alt="截屏2022-06-24 19.12.46" style="zoom:33%;" />

需要注意一个`page`有4096字节。内存使用链表进行管理，只要遍历一遍链表即可。

