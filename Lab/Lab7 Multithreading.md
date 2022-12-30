# Lab7

[TOC]

## Uthread: switching between threads



* 首先是`thread`的数据结构

  ```c
  struct thread {
    struct     ucontext  context;
    int        state;             /* FREE, RUNNING, RUNNABLE */
    char       stack[STACK_SIZE];
  };
  ```



* **特别注意的是数组地址自小到大的，但是栈的增长是自高向低的，所以线程的栈并不是`thread->stack`**

  ```c
  void 
  thread_create(void (*func)())
  {
    struct thread *t;
  
    for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
      if (t->state == FREE) break;
    }
    t->state = RUNNABLE;
    // YOUR CODE HERE
    t->context.ra=(uint64)func;
    t->context.sp=(uint64)(t->stack+STACK_SIZE-1);
  }
  
  ```

* ```c
  void 
  thread_schedule(void)
  {
    struct thread *t, *next_thread;
  
    /* Find another runnable thread. */
    next_thread = 0;
    t = current_thread + 1;
    for(int i = 0; i < MAX_THREAD; i++){
      if(t >= all_thread + MAX_THREAD)
        t = all_thread;
      //printf("%d %d ",(int)(t-&all_thread[0]),t->state);    
      if(t->state == RUNNABLE) {
        next_thread = t;
        break;
      }
      t = t + 1;
    }
  
    if (next_thread == 0) {
      printf("thread_schedule: no runnable threads\n");
      exit(-1);
    }
  
    //printf("%d",(int)(next_thread-&all_thread[0]));
  
    if (current_thread != next_thread) {         /* switch threads?  */
      next_thread->state = RUNNING;
      t = current_thread;
      current_thread = next_thread;
      
      //YOUR CODE HERE
    
      thread_switch((uint64)(&t->context),(uint64)(&next_thread->context));
  
    } else
      next_thread = 0;
  }
  
  ```

* `uthread_switch`

  ```assembly
   	.text
  
  	/*
           * save the old thread's registers,
           * restore the new thread's registers.
           */
  
  .globl thread_switch
  thread_switch:
  	/* YOUR CODE HERE */
          sd sp, 0(a0)
          sd ra, 8(a0)
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
  
          ld sp, 0(a1)
          ld ra, 8(a1)
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
          
  
  		ret    /* return to ra */
  ```

## Using thread

* answer-thread txt

```tex
nsert(key,value, &table[i], table[i]);//thread1
insert(key,value, &table[i], table[i]);//thread1

static void 
insert(int key, int value, struct entry **p, struct entry *n)
{
  struct entry *e = malloc(sizeof(struct entry));
  e->key = key;
  e->value = value;
  e->next = n;
  *p = e;
}

e1->next=table[i];
e2->next=table[i];
*p=e1;
*p=e2;

e1 is lost
```





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

  ```c
  struct entry {
    int key;
    int value;
    struct entry *next;
  };
  
  struct entry *table[NBUCKET];
  ```

   可以拆分大锁，给每个散列桶用一个锁

全局

```c
pthread_mutex_t tablemutex[NBUCKET];
```

在 `puts`前初始化

```c
for (int i = 0; i <NBUCKET; i++)
  {
    pthread_mutex_init(&tablemutex[i],NULL);
  }
  
```

在`puts`后销毁

```c
for (int i = 0; i <NBUCKET; i++)
  {
    pthread_mutex_destroy(&tablemutex[i]);
  }
```

修改`put`，==需要注意的是只在`put`修改桶的数据地方加锁==

 ```c
 static 
 void put(int key, int value)
 {
   int i = key % NBUCKET;
 
   // is the key already present?
   struct entry *e = 0;
  
   for (e = table[i]; e != 0; e = e->next) {
     if (e->key == key)
       break;
   }
   if(e){
     // update the existing key.
     pthread_mutex_lock(&tablemutex[i]);
     e->value = value;
     pthread_mutex_unlock(&tablemutex[i]);
 
   } else {
     // the new is new.
     pthread_mutex_lock(&tablemutex[i]);
     insert(key, value, &table[i], table[i]);
     pthread_mutex_unlock(&tablemutex[i]);  
   }
 
 }
 ```

* 虽然这已经能通过测试，但我认为似乎有点问题，在查看是否有散列是否有键，这里是不加锁的（不然不能通过速度测试）。但是这可能会导致键冲突。

  ```c
  for (e = table[i]; e != 0; e = e->next) {
      if (e->key == key)
        break;
    }
  ```

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

* [pthread_cond_wait](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_cond_wait.html)

   ```c
   int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
   ```

  和`sleep`类似,第一个参数为condition variable

* [pthread_cond_broadcast](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_cond_broadcast.html)

   ```c
   int pthread_cond_broadcast(pthread_cond_t *cond);
   ```

  和`wakeup`类似，参数为condition variable

`barrier`函数为

```c
static void 
barrier()
{
  pthread_mutex_lock(&bstate.barrier_mutex);
  int rd=bstate.round;
  bstate.nthread++;
  if (bstate.nthread==nthread){
    bstate.round++;
    bstate.nthread=0;
    pthread_cond_broadcast(&bstate.barrier_cond);  
  }

  while (rd==bstate.round)
  {
    pthread_cond_wait(&bstate.barrier_cond,&bstate.barrier_mutex);
  }

  pthread_mutex_unlock(&bstate.barrier_mutex);

}
```





最后的测试

![截屏2022-12-15 13.59.14](http://cdn.zhengyanchen.cn/img202212151359820.png)
