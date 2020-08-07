---
layout: article
title: ThreadPoolExecutor 线程池Executor实现解析
tags: ThreadPoolExecutor BlockingQueue Runnable
---

ThreadPoolExecutor 实现了具有线程资源池化特性的执行工具, 内部通过维护workerCount, runningState这两个线程池的实时状态, 以及抽象线程为可复用的独占资源Worker, 并且对 worker进行管理, 达到了对线程资源的调度, 复用。

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
* `workerCount` wc表示线程池中可用线程的数量

* *RUNNING*: 可接受新任务且执行队列中的任务
* *SHUTDOWN*: 停止接受新任务但执行队列中的任务
* *STOP*: 完全停止了，不接受新任务，不执行队列中的任务
* *TIDYING*: 所有任务都已经终止，workerCount此时是0，正转变状态为TIDYING的线程将会调用 `terminated()` 的钩子函数
* *TERMINATED*: `terminated()` 已完成

## 线程任务的执行


### 主要执行入口 execute

执行任务的核心方法 `execute`

方法定义： 

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
        // 队列满了,2 尝试加入非核心线程中
        else if (!addWorker(command, false))
            reject(command);
}
```

### addWorker 流程

按当前线程池状态, 尝试给Runnable Task 安排资源

1. 维护ctl状态: 检查当前线程池的running state (rs) 是否是RUNNING状态，然后进行 CAS 增加线程池 workerCount 的操作
```java
private boolean addWorker(Runnable firstTask, boolean core) {
        // #1
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && ! (rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        // #2  ...
}
```

2. 将 Runnable Task 放进 Worker 容器里，其中 Worker 会从ThreadFactory里创建新线程给 Task。Worker 之后被原子地（获取mainLock）添加到线程池的 HashSet中，如果成功加入则启动线程。


### 解析

从上面代码我们可以看出：

1. 文首提出的 `corePoolSize` 概念
    - 线程池需要维持一定数量的核心线程在池中，所以如果每次执行发现 workCount 少于初始化的核心线程，
    就会新增核心线程来维持这个水平。
2. `Worker`
    - 每个线程资源在线程池中的包装, 基于AQS实现 (当前获取到资源的Runnable Task 独占该线程资源)
    - 其数量对应于线程池状态中的 wc (workerCount)
3. addWorker
    - 每次execute 都会增加 worker, addWorker里面做了包装Runnable Task 的步骤, 第二个参数core 代表是否是核心线程
    - 每次增加worker, 正常情况加入到线程池的HashSet 中管理, 并且启动 worker thread,
    如果 workerCount 超出了核心线程数或者最大线程数 (取决于core 表示现在的worker 是核心线程还是非核心线程),
    那么不会初始化Worker 也不会增加wc, 相当于该Runnable Task 没获取到线程资源, 如果在 `execute` 的逻辑中, 这样的Task 会尝试加入到等待队列 (workerQueue) 中
4. 最坏情况线程池队列都满了, Runnable Task 无法加入线程中, 再次尝试加入非核心线程, 失败了就按线程池的 Reject Handler 进行拒绝流程的处理

*TL;DR*

1. 如果 workerCount < 核心线程数或线程池最大容量, workers hashset 保存占用到线程资源的Runnable Task, 并且启动
2. 获取不到线程, Runnable task 进入 workerQueue 等待

## 线程池执行等待队列的任务

在 `execute` 中无法立即执行的任务被 `addWorker` 加入到内置等待队列中

前文提到 Worker 是一个独占的AQS资源, 同时它也是一个 Runnable Task, 在其独立的线程中执行 runWorker方法
```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {

// ...
        public void run() {
            runWorker(this);
        }
// ...
}
```

**runWorker 方法: **

```java
    final void runWorker(Worker w)
```

以上方法看来, Worker 的执行实际将占用 worker的Runnable Task 委派给了线程池来执行;

线程池会先查看task 是否为空, 其实这就解释了之前代码中 `addWorker(null, false)` 的空worker的去处,
这些空worker是线程池中未销毁可被复用的线程资源, 减免程序每次都要创建线程的开销。 如果task 为空, 那么通过 getTask 方法从 workerQueue里拿任务来执行

```java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

getTask 方法还是会查看当前线程池的状态, 如果线程池状态是STOP了, 那么不再执行队列中的任务, 亦或是workQueue没有任务在等待了, 那么就将现在这个worker 丢弃

timed 和 timeOut 标志, 控制了轮询执行时间. 从代码中可以看出, 在 workerCount 超出核心线程数之前, 获取等待任务是持续进行不会轮询超时的，除非给线程池设置了 `allowCoreThreadTimeOut` 这个参数

如果轮询超时, 或是 wc已经超出了 maximumPoolSize, 但核心线程worker 没有了, workQueue也没有了, 那么这个空worker就没有轮询下去的意义, 直接尝试dicard。

正常情况, 从 workQueue 中拿到 task 就直接返回。

### 线程等待机制

### 线程池超限

### 等待任务移除

## 线程池销毁

* tryTerminal
* shutdown
* shutdownNow
* terminated

