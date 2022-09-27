# 	LAB3

 首先切换`pgtbl`分支

```shell
git fetch
git checkout pgtbl
make clean
```

这里使用`git fetch`老是出错，不过也不影响写lab。

![截屏2022-09-15 16.52.31](http://cdn.zhengyanchen.cn/img202209151652331.png)

## Print a page table

* 首先在`exec.c`中的`exec`函数添加以下代码。在第一个用户进程调用`exec`时使用`vmprint` 来打印该进程的用户页表。

  ```c
  if (p->pid==1) vmprint(p->pagetable);
  return argc; // this ends up in a0, the first argument to main(argc, argv)
  ```

  * 在`vm.c`里添加`vmprint`函数,使用递归的方法，打印各级的PTE。在写`pgprint`函数的时候，由于我不想其他文件访问该函数，所以添加了`static`.


  ```c
  static void pgprint(pagetable_t pagetable,int level){
    if (level<0) return;
    for (int i=0; i < 512; i++)
      {
        pte_t pte=pagetable[i];
        if ((pte & PTE_V)!=0){
          uint64 pa=PTE2PA(pte);
          for (int k = 2; k>level; k--)
          {
            printf(".. ");
          }
          printf("..%d: pte %p pa %p\n",i,pte,pa);
          pgprint((pagetable_t)pa,level-1);
        }
    }        
  }
  
  void vmprint(pagetable_t pagetable){
    printf("page table %p\n",pagetable);  
    pgprint(pagetable,2);
  }
  ```

* 需要在`def.h`里添加`vmprint`函数的定义，这样才可以在`exec`里调用

  ```c
  void            vmprint(pagetable_t);
  ```

在`userinit`中给第一个用户程序`initcode`创建了第一个用户进程，并为其分配了第一个进程号 `pid=1`,正是这个程序系统调用了`exec`。

## A kernel page table per process

* 首先在`proc.h`里的`proc`结构体中添加

```c
pagetable_t kpagetable;      // Kernel page table per process 
```

* 下面在`allocproc`中在每一个用户进程的`kpagetable`中填入和`kernel_pagetable`一样的映射。这里需要注意两点：

1. 用户的内核页表其他部分需要和全局的`kernel_pagetable`保持一致，但用户的内核栈却不必。事实上在全局内核页表的映射中，每个进程的内核栈自高到低地分布，因此建立映射时虚拟地址是这样的：

   ```c
    uint64 va = KSTACK((uint64)(p-proc));
   ```

   但用户的内核页表很自由，我干脆把每个用户进程的内核栈都放在`TRAMPOLINE`下面。

```c
	// A kernel page table of user process 
	p->kpagetable=newkvminit();
	if (p->kpagetable==0){
  	freeproc(p);
   	release(&p->lock);
   	return 0;
	}

  // Allocate a page for the process's kernel stack.
	char *pa = kalloc();
  if(pa == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  uint64 va = KSTACK(0）;
  newkvmmap(p->kpagetable,va,(uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack=va;
```

​		还有一个很重要的问题，就是需要==修改`kvmpa` 函数==（这个函数用于内核栈地址的翻译）,尤其是在我修改了内核栈的虚拟地址以后，于是将

```c
  pte = walk(kernel_pagetable, va, 0);
```

修改为

```c
  pte = walk(myproc()->kpagetable, va, 0);
```

但会报错说结构体不完全，加上包含结构体信息的头文件就好了。

```c
#include "spinlock.h"
#include "proc.h"
```

2. 这里调用了我新写的函数`newkvminit`和`newkvmmap`,用来建立映射，这两个函数写在`vm.c`中。

```c

pagetable_t newkvminit(){
  pagetable_t kpagetable=(pagetable_t)kalloc();
  memset(kpagetable,0,PGSIZE);// every byte is 0
  // uart registers
  newkvmmap(kpagetable,UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  newkvmmap(kpagetable,VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  newkvmmap(kpagetable,CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  newkvmmap(kpagetable,PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  newkvmmap(kpagetable,KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  newkvmmap(kpagetable,(uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  newkvmmap(kpagetable,TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);  

  return  kpagetable;
}


void 
newkvmmap(pagetable_t kpagetable,uint64 va,uint64 pa,uint64 sz,int perm)
{
  if(mappages(kpagetable,va,sz,pa,perm)!=0)
    panic("newkvmmap");
}
```

* 还需要在`freeproc`添加一段释放用户的内核页表内存的代码

```c
if(p->kpagetable){
    uvmunmap(p->kpagetable,p->kstack,1,1);   
    p->kstack=0;
    freekpagetable(p->kpagetable);
}
```

直接借用用户页表`unmap`的函数，来看看`uvmunmap`的代码

```c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

简单来说，设`do_free`参数为`1`,所做的就是通过`walk`函数找到最低一级的PTE，释放掉指向的内核栈内存，然后将PTE归0。特别需要注意这里==并没有回收掉各级页表的内存==。这时候就轮到`freekpagetable`登场了,通过递归的方法，将每一级的PTE归0并释放掉当前级的页表。

```c
void freekpagetable(pagetable_t kpagetable){
  for (int i = 0; i < 512; i++)
  {
    if (kpagetable[i] & PTE_V){
      if ((kpagetable[i] & (PTE_R | PTE_W | PTE_X))==0){
        freekpagetable((pagetable_t)PTE2PA(kpagetable[i]));
      }
      kpagetable[i]=0;
    }
  }
  kfree((void *)kpagetable);
}
```



* 修改`scheduler`函数,使切换进程时更换页表。这部分的代码我有疑惑，保留下来。

```c
p->state = RUNNING;
c->proc = p;
w_satp(MAKE_SATP(p->kpagetable));//now satp is p->kpagetable
sfence_vma();
swtch(&c->context, &p->context);
w_satp(MAKE_SATP(kernel_pagetable));//now satp is kernel_kpagetable
sfence_vma();
```





很奇怪的是，到这里已经可以通过所有的`usertests`了，但是会出现一些`usertrap`





![截屏2022-09-17 18.42.04](http://cdn.zhengyanchen.cn/img202209171847256.png)



![截屏2022-09-17 18.44.37](http://cdn.zhengyanchen.cn/img202209171847898.png)

![截屏2022-09-17 18.44.00](http://cdn.zhengyanchen.cn/img202209171848675.png)

​	 

很奇怪的是，到这里已经可以通过所有的`usertests`了，但是









## Simplify `copyin`/`copyinstr`



