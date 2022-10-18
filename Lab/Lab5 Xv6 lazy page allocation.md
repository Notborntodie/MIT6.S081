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



到目前为止，已经可以通过`lazytests`

* 



<img src="http://cdn.zhengyanchen.cn/img202210171010882.png" alt="截屏2022-10-17 10.10.12" style="zoom:30%;" />