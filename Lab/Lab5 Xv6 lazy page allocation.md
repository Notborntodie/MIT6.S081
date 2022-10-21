# Lab5

## Eliminate allocation from sbrk()

修改`sys_sbrk`如下

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;
  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz=myproc()->sz+n;
//  if(growproc(n) < 0)
//    return -1;
  return addr;
}
```

 出现page fault<img src="http://cdn.zhengyanchen.cn/img202210162313076.png" alt="截屏2022-10-16 23.13.46" style="zoom:50%;" />



## Lazy allocation

在`usertrap`里添加

```c
  }else if (r_scause()==13 || r_scause()==15){//load/store page fault
    uint64 vadd=PGROUNDDOWN(r_stval());
    void* padd=kalloc();
    if (padd==0){
      p->killed=1;
      exit(-1);
    }
    if (mappages(p->pagetable,vadd,PGSIZE,(uint64)padd,PTE_V|PTE_U|PTE_W|PTE_R)!=0){
      kfree(padd);
      p->killed=1;
      exit(-1);
    }
```

修改`uuvmunmap`

```c
   if((*pte & PTE_V) == 0)
      //panic("uvmunmap: not mapped");
      continue;
```



## Lazytest and Usertest

* **处理n为负值的情况**,修改`sys_sbrk`

  直接释放内存

  ```c
   if (n<0) {
      if (growproc(n)){
        return -1;
      }
    }else myproc()->sz=myproc()->sz+n;
  ```

  **处理高于当前sz的地址**，修改`usertrap`,添加

  ```c
     if (vadd>=p->sz){
        p->killed=1;
        exit(1);
     }
  ```



* `fork`

  修改从父进程向子进程拷贝的`uvmcopy`,这时候pte可能是不存在的，就不拷贝这页

  ```c
   for(i = 0; i < sz; i += PGSIZE){
      if((pte = walk(old, i, 1)) == 0)
        panic("uvmcopy: pte should exist");
      if((*pte & PTE_V) == 0)
        //panic("uvmcopy: page not present");
        continue;
  ```

  同样的对`uvmunmap`

  ```c
      if((pte = walk(pagetable, a, 1)) == 0)
        panic("uvmunmap: walk");
        //continue;
      if((*pte & PTE_V) == 0)
        //panic("uvmunmap: not mapped");
        continue;
  ```

  

* 出现错误，

<img src="http://cdn.zhengyanchen.cn/img202210170958028.png" alt="截屏2022-10-17 09.58.14" style="zoom:33%;" />

 这是因为`if((*pte & PTE_V) == 0)`之后继续，所以还有pagetable的叶节点是有效的，所以修改`freewalk`. 

```c
   } else if(pte & PTE_V){
      //panic("freewalk: leaf");
      pagetable[i]=0;
    }
```



到目前为止，已经可以通过`lazytests`,

* <img src="http://cdn.zhengyanchen.cn/img202210202107474.png" alt="截屏2022-10-20 21.07.15" style="zoom:50%;" />

但要通过`usertests`

* 还需要修改`read`和`write`相关的系统调用，

  给`sys_write`和`sys_pipe`添加如下代码

  ```c
   if (!walkaddr(p->pagetable,fdarray)){
      if (lazyalloc(p,fdarray)<0)
      return -1;
    }
  ```

  `lazyalloc`这个函数被定义在`vm.c`里

  ```c
  uint64 lazyalloc(struct proc* p ,uint64 va){
  
      if (va>=p->sz){
        return -1;
      }
      void* pa=kalloc();
      if (pa==0){
        return -1;
      }
      if (mappages(p->pagetable,va,PGSIZE,(uint64)pa,PTE_V|PTE_U|PTE_W|PTE_R)!=0){
        kfree(pa);
        return -1;
      }
    return 0;
  
  }
  ```

* 处理用户栈下的无效页面, `trap.c`里添加

  ```c
      if (r_stval()<p->trapframe->sp){
        p->killed=1;
        exit(-1);
      }
  ```

* 这样就可以通过`usertests`

<img src="http://cdn.zhengyanchen.cn/img202210202129700.png" alt="截屏2022-10-20 21.29.30" style="zoom:50%;" />





### Stack

关于测试stack的部分，我觉得`stacktest`是不够的，因为这里测试只是直接访问了Guard page, 假如说因为无限递归，导致栈指针移动到Guard Page,也会触发页错误，这时候有处理嘛？，并没有。![截屏2022-10-21 11.39.00](http://cdn.zhengyanchen.cn/img202210211139303.png)

* 我修改`stacktest`如下

  ```c
    if(pid == 0) {
      p_sp();
      exit(1);
    }
  
  	
  void p_sp(){
    p_sp();
  }
  
  ```

  添加了一个无限递归的函数。这时候现在的`usetrap`并不能处理这种页错误。所以我添加

  ```c
      if (!walkaddr(p->pagetable,p->trapframe->sp)){
        p->killed=1;
        printf("stack overflow\n");
        exit(-1);
      }
  ```

  现在来测试新的`stacktest`

  可以看到，现在如果出现栈溢出就会直接杀死进程.

  <img src="http://cdn.zhengyanchen.cn/img202210211245721.png" alt="截屏2022-10-21 12.45.29" style="zoom: 50%;" />