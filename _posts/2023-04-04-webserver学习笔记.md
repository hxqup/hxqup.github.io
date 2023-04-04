---
layout:     post
title:      "webserver笔记(一)"
subtitle:   " \"MutexLock类与Condition类的实现\""
date:       2023-04-04 21:00:00
author:     "Hxq"
header-img: "img/post-bg-2015.jpg"
tags:
    - 网络编程
---
# webserver学习笔记(一)

> 从自己实现一个锁和条件变量开始，在陈硕的《Linux多线程服务端编程》中提到：“应该避免用信号量(semaphore)。它的功能与条件变量重合，但容易用错。"

## MutexLock类与Condition类的实现

互斥锁和条件变量构成了多线程编程的全部必备同步原语，用它们即可完成任何多线程同步任务，二者不能相互替代。

### MutexLock类

单独使用mutex时，可以采用以下四点：

* 用RAII手法封装mutex的创建、销毁、加锁、解锁四个操作。
* 只用非递归(不可重入)的mutex

> 可重入函数是指一个函数同时被多个线程调用，任务在调用时不用担心出错。同一线程不能对non-recursive mutex加锁

* 不手动调用lock与unlock函数
* 在每次构造Guard对象时，思考栈上已经持有的锁，防止因加锁顺序不同而造成死锁。

>* pthread_mutex_init用于初始化互斥锁
>* pthread_mutex_destroy用于销毁互斥锁
>
>* pthread_mutex_lock函数以原子操作的方式给互斥锁加锁
>* pthread_mutex_unlock函数以原子操作的方式给互斥锁解锁

以上函数，成功则返回0，失败返回errno

### Condition类

互斥量是加锁原语，用来排他性地访问共享数据，在使用mutex时，我们一般会期望加锁不要阻塞，总是能立刻拿到锁，然后尽快访问数据，这样才不会影响并发性和性能。

如果需要等待某个条件成立，则需要使用条件变量(condition variable)。条件变量顾名思义是一个或多个线程等待某个布尔表达式为真，即等待别的线程唤醒它。

条件变量只有一种正确的使用方式：

**对于wait端**

1. 必须与mutex一起使用，该布尔表达式的读写需要受到mutex的保护，即如果一个class要包含MutexLock与Condition，请注意它们的声明顺序与初始化顺序，mutex\_应先于condition\_构造，并作为后者的构造函数
2. 在mutex已经上锁的时候才能调用wait()
3. 把判断布尔条件和wait一起放到while循环中

**对于signal/broadcast端**

1. 不一定要在mutex上锁的情况下调用signal
2. 在signal之前一般要修改布尔表达式
3. 修改布尔表达式通常要用mutex保护
4. 注意区分signal与broadcast:"broadcast通常用于表明状态变化，signal通常用于表示资源可用“

>* pthread_cond_init函数用于初始化条件变量
>* pthread_cond_destroy函数销毁条件变量
>* pthread_cond_broadcast函数以广播的方式唤醒**所有**等待条件变量的线程
>
>* pthread_cond_wait函数用于等待目标条件变量。该函数使用时需要传入mutex参数(加锁的互斥锁),函数执行时，先把调用线程放入条件变量的请求队列，然后把互斥锁mutex解锁，当函数成功返回为0时，互斥锁会再次锁上，也就是说函数内部有一次解锁和解锁的操作。
>* pthread_cond_signal只唤醒一个等待的线程，不保证唤醒等待线程中的哪一个线程
>* pthread_cond_timedwait设置了一个超时时间，如果超出了超时时间，那么该函数会退出，并返回errno，避免因为死锁等原因造成线程长时间等待

---

> **notify_all与notify**

> 使用notify_all可能更安全，但可能带来明显的效率下降，因为容易造成惊群现象。notify则在某些情况下导致并发不充分。

参考资料：

陈硕 《Linux多线程服务端编程》

游双 《Linux高性能网络服务器编程》