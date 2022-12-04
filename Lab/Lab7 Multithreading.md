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



