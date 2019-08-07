---
layout: article
title: ThreadPoolExecutor 线程池Executor实现解析
tags: ThreadPoolExecutor BlockingQueue Runnable
---



<!--more-->

## 构造方法及使用参数

构造方法：

参数:

* `corePoolSize`
- __核心线程数__: 保留在线程池中的线程数，不管线程是否IDLE，都保留，除非设置了`allowCoreThreadTimeOut`的值
* `maximumPoolSize`
- __线程池最大容量__: 线程池中允许持有的线程数上限
* `keepAliveTime`
- __保活时间__: 当线程数超出核心线程数，IDLE线程有一个保活时间来等待新的任务提交来领走它们，如果最终没有则IDLE线程就会被终止

## 线程池状态的维护原理及生命周期

用一个原子整数值 `ctl` 来标识当前池的状态，将几个字段用二进制的方式聚合在了一起；

```java
ctl = rs | wc; // ( 29 位的 ) runningState || ( 2 位的 ) workerCount
```

* `runningState` rs表示当前线程池是运行中还是被关闭或者其它状态，是**线程池的生命周期标志**
* *RUNNING*: 可接受新任务且执行队列中的任务
* *SHUTDOWN*: 停止接受新任务但执行队列中的任务
* *STOP*: 完全停止了，不接受新任务，不执行队列中的任务
* *TIDYING*: 所有任务都已经终止，workerCount此时是0，正转变状态为TIDYING的线程将会调用 `terminated()` 的钩子函数
* *TERMINATED*: `terminated()` 已完成
* `workerCount` wc表示线程池中可用线程的数量


## 线程任务的执行

执行任务的核心方法

```java
public void execute(Runnable command) {
         if (command == null)
            throw new NullPointerException();
            
        int c = ctl.get();

        // 当前 worker 的数量小于核心线程数 ---> 继续添加核心工作线程
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }

        // 如果worker 数量超过核心线程数， 加入等待的worker队列
        // 线程池正在运行, 将任务加入阻塞队列中等待执行
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();

            // 线程池被SHUTDOWN了, 不再增加等待任务，将重试的任务从等待队列中去掉，
            // 并采取拒绝策略 (`RejectExecutionHandler`) 来对线程池饱和或者线程池SHUTDOWN状态下的等待线程进行处理流程
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果线程池还在运行, 且线程池空了, 新增非核心线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 队列满了, 尝试加入非核心线程中
        else if (!addWorker(command, false))
            reject(command);
}
```



## 线程等待机制

添加任务到等待队列的内部实现： 

```java
private boolean addWorker(Runnable firstTask, boolean core)
```

addWorker 基本流程：

1. 维护ctl状态: 检查当前 rs 是否是RUNNING状态，然后进行 CAS 增加workerCount 的操作
2. 将 Runnable Task 放进 Worker 容器里，其中 Worker 会从ThreadFactory里创建新线程给 Task。Worker 之后被原子地添加到线程池的 HashSet中，如果成功加入则启动线程。
3. 
