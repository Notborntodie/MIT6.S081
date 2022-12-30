# Lab8 Locks



## pre

预编译指令

* `#define`

* `#ifdef`

  ```c
  #ifdef x  //先测试x是否被宏定义过  
  程序段1   //如果x被宏定义过，那么就编译程序段1  
  #else 
  程序段2 //如果x没有被定义过则编译程序段2的语句，“忽视”程序段1。 
  #endif 
  ```





为了统计每个`spinlock`被获取的次数和`spinlock`的在`acquire`中循环的次数，修改`spinlock`数据结构如下

```c
// Mutual exclusion lock.
struct spinlock {
  uint locked;       // Is the lock held?

  // For debugging:
  char *name;        // Name of lock.
  struct cpu *cpu;   // The cpu holding the lock.
#ifdef LAB_LOCK
  int nts;
  int n;
#endif
};

```



* `n`用来记录锁企图`acquire`的次数
* `nts`用来记录锁在`acquire`中循环的次数











## 安全性

* 使用`cpuid` 函数需要关中断，目前的
