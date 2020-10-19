---
layout: article
title: JUC 并发工具笔记
tags: java juc concurrent
---

# 并发工具

> 使用并发工具优于使用 Object::wait 和 Object::notify  -- Effective Java

<!--more-->

## 基础的 Monitor-Object 协同

以下代码提供了对资源 resourceNum 的数量控制，实现了平衡的生产和消费过程。

对于多个消费者对 resourceNum 的并发修改，此处使用了 `synchronized` 关键字，表示 `remove` 和 `add` 都是同步方法。 

**synchronized**

JVM 基于 monitorenter 和 monitorexit 指令实现同步锁的获取功能

synchronized 声明代码块会与其他线程竞争 synchronized(object) 中的 object对象的引用，即获取对 object 对象的锁。一旦获取到锁，synchronized 代码块的内容就会被执行，执行完成（完整执行或是异常）后都会对获取的同步锁解锁。

synchronized 方法和 synchronized 声明代码块的执行过程类似, 但其利用了 ACC_SYNCHRONIZED 标识实现了方法调用时自动加上对同步锁的获取过程。

同步锁是互斥的(mutex)，即在任意时间只会有一个线程占用object资源；并且通过 wait-notify 的信号机制控制锁的释放和获取的时机

Object::wait, Object::notify, Object::notifyAll 需要在 synchronized 声明的同步块中才能使用


```java
    public synchronized void remove() {
        if (resourceNum > ABUNDANT_THRESHOLD) {
            this.resourceNum--;
        } else {
            try {
                wait();

                System.out.println("current waiting thread: " + Thread.currentThread());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public synchronized void add() {
        if (resourceNum <= ABUNDANT_THRESHOLD) {
            this.resourceNum++;
            notifyAll();
        }
    }
```

注意：标准模式的条件等待需要在 while 中判断，例如 `while(!condition) { wait(); } `, 这是为了确保条件等待的安全性，因为无法保证通知线程是否满足了条件，也有概率是'虚假唤醒 (spurious wakeup)'

自 jdk5 开始, java 提供了更多优于 wait, notify 的并发工具，使得编写正确的并发控制流程更加的简单，其原理大多也是用效率更高或者更容易控制或扩展的数据结构实现了 获取锁-等待队列-加锁-解锁 的流程

`java.util.concurrent` 中更高级的工具分成三类：
* Executor Framework 
    - ExecutorService
    - ThreadPoolExecutor (使用 BlockingQueue 作为 worker Queue)
* 并发集合（Concurrent Collection）
    - ConcurrentHashMap
* 同步器（Synchronizer）使线程能够等待另一个线程，实现线程间的协调作业
    - CountDownLatch
    - Semaphore
    - CyclicBarrier
    - Exchanger
    - **Phaser**

## BlockingQueue 阻塞队列

```java
    private BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(QUEUE_SIZE);

    @Override
    public void remove() {
        try {
            queue.take();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void add() {
        try {
            queue.put(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

```

## Conditions await signal

```java
    public void remove() {

        if (lock.tryLock()) {

            try {
                if (resourceNum > 80) {
                    this.resourceNum--;
                    producerCondition.signalAll();
                } else {
                    try {
                        consumerCondition.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("current waiting thread: " + Thread.currentThread());
                }
            } finally {
                lock.unlock();
            }
        }
    }

    public void add() {

        if (lock.tryLock()) {

            try {
                if (resourceNum <= 90) {
                    this.resourceNum++;
                    consumerCondition.signalAll();
                } else {
                    producerCondition.await();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
    }

```

## 信号量 Semaphore

Semaphore (信号量)的设计是限制一定数量的线程同时运行，超过限制的线程等待。


```java
    private Semaphore mutex = new Semaphore(1);
    private Semaphore fulled = new Semaphore(MAX_RESOURCE_NUM);
    private Semaphore lacked = new Semaphore(0);

    @Override
    public void remove() {
        try {
            lacked.acquire();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        if (mutex.tryAcquire()) {
            try {
                resourceNum--;
            } finally {
                mutex.release();
                fulled.release();
            }

            System.out.println("lacked : " + lacked.availablePermits() + " fulled: " + fulled.availablePermits());
        }
    }

    @Override
    public void add() {
        try {
            fulled.acquire();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        if (mutex.tryAcquire()) {
            try {
                resourceNum++;
            } finally {
                mutex.release();
                lacked.release();
            }
            System.out.println("lacked : " + lacked.availablePermits() + " fulled: " + fulled.availablePermits());
        }
    }

```


## CyclicBarrier

栅栏，当达到设置数量上限的线程执行到了 `await()` 方法时，才会继续。

和 Semaphore 的区别就是：

* Semaphore 等待获准进入执行
* CyclicBarrier 等待一起执行完成则进行下一步

## CountDownLatch

令牌计数，使用 `await()` 阻塞, 当 countDown 为0时，才能唤醒

与 CyclicBarrier 的区别是 CountDownLatch 不可重入