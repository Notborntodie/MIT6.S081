# Lab7

## Uthread: switching between threads







## Using thread

* [pthread_create](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_create.html)

  ```c
  #include <pthread.h>
  
  int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
      void *(*start_routine)(void*), void *arg);
  ```

* [pthread_join]()

  ```c
  #include <pthread.h>
  
  int pthread_join(pthread_t thread, void **value_ptr);
  

* [pthread_mutex](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_mutex_lock.html)

  ```c
  int pthread_mutex_lock(pthread_mutex_t *mutex);
  int pthread_mutex_trylock(pthread_mutex_t *mutex);
  int pthread_mutex_unlock(pthread_mutex_t *mutex);
  ```



* 只加一把大锁

  <img src="http://cdn.zhengyanchen.cn/img202211130050478.png" alt="截屏2022-11-13 00.50.26" style="zoom:43%;" />

<img src="http://cdn.zhengyanchen.cn/img202211130049837.png" alt="截屏2022-11-13 00.49.22" style="zoom:43%;" />

* 每散列桶的数据是不相同的，其数据结构如下

## Barrier

[Barrier](https://en.wikipedia.org/wiki/Barrier_(computer_science))

* 定义：在[并行计算](https://en-m-wikipedia-org.translate.goog/wiki/Parallel_computing?_x_tr_sl=auto&_x_tr_tl=zh-CN&_x_tr_hl=zh-CN&_x_tr_pto=wapp)中，**屏障**是一种[同步](https://en-m-wikipedia-org.translate.goog/wiki/Synchronization_(computer_science)?_x_tr_sl=auto&_x_tr_tl=zh-CN&_x_tr_hl=zh-CN&_x_tr_pto=wapp)方法。源代码中一组线程或进程的屏障意味着任何线程/进程都必须在此时停止，并且在所有其他线程/进程到达此屏障之前无法继续。

* assert是一个宏

  使用

  ```c
  assert(expr)
  ```

  等同于

  ```c
  if(expr)
  {
      break；
  }
  else
  {
       terminate the program;
  }
  ```

  
