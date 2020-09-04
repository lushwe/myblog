# 源码系列 - 一行一行分析 `ThreadPoolExecutor` 源码

## 成员属性
```java
// ctl = RUNNING
// RUNNING | 0 = 1010 0000 0000 0000 0000 0000 0000 0000 | 0000 0000 0000 0000 0000 0000 0000 0000 = 1010 0000 0000 0000 0000 0000 0000 0000 = RUNNING
// 简单说下ctl，作者把ctl分为两部分
// 第一部分为高3位，表示线程池状态；
// 第二部分为剩余29位，表示线程池容纳的最大线程数
// 初始情况，线程池状态为RUNNING，线程数量为0
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 计算线程数量的二进制位数，COUNT_BITS = 32 - 3 = 29
private static final int COUNT_BITS = Integer.SIZE - 3;
// 线程池最大容量: 2^29-1，二进制为：0001 1111 1111 1111 1111 1111 1111 1111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
// -1 << COUNT_BITS 二进制为: 1010 0000 0000 0000 0000 0000 0000 0000
//  0 << COUNT_BITS 二进制为: 0000 0000 0000 0000 0000 0000 0000 0000
//  1 << COUNT_BITS 二进制为: 0010 0000 0000 0000 0000 0000 0000 0000
//  2 << COUNT_BITS 二进制为: 0100 0000 0000 0000 0000 0000 0000 0000
//  3 << COUNT_BITS 二进制为: 0110 0000 0000 0000 0000 0000 0000 0000
```

## `execute` 方法
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    // 拿到线程池变量ctl的值
    int c = ctl.get();
    // workerCountOf 方法得到当前线程池线程数量
    if (workerCountOf(c) < corePoolSize) {
        // 进入这里，表示当前线程数量小于核心线程数，新建一个线程执行任务
        if (addWorker(command, true))
            // 新建成功，则执行返回
            return;
        // 进入这里，表示上面 addWorker 方法执行失败，两种情况：
        // 1、多个线程同时执行 addWorker, 另外一个线程执行成功，此时线程数大于等于核心线程数
        // 2、线程池刚刚 SHUTDOWN, （所以下面首先判断一下线程的状态）
        // 这里重新获取一下ctl的值，用于下面判断线程池状态
        c = ctl.get();
    }
    // 判断当前线程池是否运行状态，若线程池是运行状态，则执行 workQueue.offer 方法是把任务加到阻塞队列
    if (isRunning(c) && workQueue.offer(command)) {
        // 进入这里，表示当前线程池是运行状态，且加入队列成功
        // 这里重新获取一下ctl的值，双重校验
        // 第一，看当前线程池是否运行状态
        // 第二，看当前线程池线程数量是否为0
        int recheck = ctl.get();
        // 判断是否运行状态，若非运行状态，则从队列中去掉刚才的添加的任务
        if (!isRunning(recheck) && remove(command))
            // 进入这里，表示线程池非运行状态，且任务已从队列中删除
            // 执行拒绝策略
            reject(command);
        else if (workerCountOf(recheck) == 0)
            // 进入这里，表示当前线程池线程数量为0，则新建一个线程，不然没有线程执行上面入队的任务
            // 这里 firstTask 设置为 null, 则该线程会直接从阻塞队列中取任务执行
            addWorker(null, false);
    }
    // 进入这里，表示线程池非运行状态，或加入阻塞队列失败
    // 若线程池非运行状态，addWorker 方法会返回 false
    // 若是加入阻塞队列失败，则说明阻塞队列满了，需要新建线程执行任务
    else if (!addWorker(command, false))
        // 进入这里，表示线程池非运行状态，或线程池线程数超过最大线程数
        // 执行拒绝策略
        reject(command);
}
```

## `addWorker` 方法
```java
private boolean addWorker(Runnable firstTask, boolean core) {
    // 定义一个变量
    retry:
    // 下面是循环
    for (;;) {
        // 拿到ctl的值
        int c = ctl.get();
        // 获取线程池的状态
        int rs = runStateOf(c);

        // 这里条件比较多，作者这里给了注释，其实就是
        // 不是运行状态执行返回false，这里多了一个条件
        // !(rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
        // 因为状态为SHUTDOWN 且 firstTask==null 且 队列不为空，则表示需要添加一个worker来处理队列中的任务，所以要通过，其他条件都不通过
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && !(rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
            // 进入这里，表示线程池状态非运行状态，且不满足创建一个线程执行队列里的任务
            return false;

        // 下面又是一个循环
        for (;;) {
            // 获取线程池中线程数量
            int wc = workerCountOf(c);
            // 判断线程数量是否超过阀值（核心线程数、最大线程数）
            if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
                // 进入这里，表示当前线程数超过了阀值，返回false
                return false;
            // CAS方式将线程数加1
            if (compareAndIncrementWorkerCount(c))
                // 加1成功，则跳出retry
                break retry;
            // 进入这里，表示加1失败，重新获取ctl的值
            c = ctl.get();  // Re-read ctl
            // 判断当前线程池状态是否发生了改变
            if (runStateOf(c) != rs)
                // 进入这里，表示当前线程池状态发生了改变，跳到retry继续执行
                continue retry;
            // 走到这里，下面注释也说了，是由于线程数量发生了改变导致cas失败，继续当前这个内部循环
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    // 到这里可以总结下上面两个循环，第一个循环针对线程池状态判断，第二个循环针对线程池中线程数量判断

    // 经历千辛万苦，终于到了这里
    
    // 记录线程启动标识
    boolean workerStarted = false;
    // 记录线程添加标识
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 创建一个Worker对象
        w = new Worker(firstTask);
        // 拿到Workder对象的属性
        final Thread t = w.thread;
        // 判断一下线程是否为空，通过 `new Worker(firstTask)` 可以知道线程 t 一般不会为空
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // 加锁，开始添加
            mainLock.lock();
            try {
                // 获得锁后，双重检查线程池状态
                // 退出 ThreadFactory 失败（这点暂时不太理解），或者获得锁之前线程池已经关闭
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                // 判断线程池状态
                if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
                    // 这里判断线程是否已经启动，
                    if (t.isAlive()) // precheck that t is startable
                        // 新建的线程已经启动，这里抛出异常
                        throw new IllegalThreadStateException();
                    // 任务线程加到线程池
                    workers.add(w);
                    // 记录线程池最大数量
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    // 线程添加标识改为true
                    workerAdded = true;
                }
            } finally {
                // 添加结束，解锁
                mainLock.unlock();
            }
            // 判断线程是否添加成功
            if (workerAdded) {
                // 进入这里，表示线程添加到线程池成功，启动线程
                t.start();
                // 线程启动标识改为true
                workerStarted = true;
            }
        }
    } finally {
        // 判断线程是否启动
        if (! workerStarted)
            // 进入这里，表示线程没有启动，执行addWorkerFailed方法
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
```java
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            // 从队列中删除添加的任务
            workers.remove(w);
        // 任务数量减1（因为在上面方法 compareAndIncrementWorkerCount 任务数量加了1）
        decrementWorkerCount();
        // 尝试Terminate
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

## `runWorker` 方法
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 循环获取任务，直到返回null
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) || (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

## `getTask` 方法
```java
private Runnable getTask() {
    // 获取任务是否超时，看下面哪里赋值为 true 可知
    boolean timedOut = false; // Did the last poll() time out?

    // 这里是一个大循环
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 这里主要是判断线程池状态，改成下面两种情况更好理解一些
        // 情况一：rs >= STOP
        // 情况二：rs == SHUTDOWN && workQueue.isEmpty()
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 任务线程是否要被淘汰？有两种情况
        // 情况一：我们设置了允许核心线程超时淘汰
        // 情况二：任务线程数超过了核心线程数
        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // (任务线程数大于最大线程数，或者当前任务线程要被淘汰) && (任务线程数大于1 或者 阻塞队列为空)
        // 这里(wc > 1 || workQueue.isEmpty()))可以这么理解：
        // 要么阻塞队列不为空，当前线程池里面的任务线程数大于1，当前任务线程被淘汰了还有其他线程可以去处理阻塞队列里的任务;
        // 要么阻塞队列为空，不需要线程处理了
        // 也就是说，如果阻塞队列不为空，且 wc == 1 , 此时该任务线程不能被淘汰，因为只能自己去处理阻塞队列中的任务了
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            // 这里任务线程数减1，并返回null
            if (compareAndDecrementWorkerCount(c))
                return null;
            // 进入这里，说明上面减1没成功，继续循环处理
            continue;
        }

        try {
            // 这个三元运算符很好理解：
            // 如果当前线程要需要超时淘汰，则使用带超时时间的API获得任务，否则使用不带超时时间的API
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            // 如果 r 不为空，则返回，由该线程处理该任务
            if (r != null)
                return r;
            // 能进入这里，说明上面 timed==true 并且 获取任务超时
            // 所以这里 timedOut置为true，继续回到上面的循环，再次走到这里：if ((wc > maximumPoolSize || (timed && timedOut))
            // 条件 (timed && timedOut) 就会成立，最终返回null
            timedOut = true;
        } catch (InterruptedException retry) {
            // 因为上面 poll、take方法都是阻塞的，遇到中断会抛出InterruptedException异常
            // 这里的处理逻辑是：遇到中断异常，循环再来一次
            timedOut = false;
        }
    }
}
```

## `processWorkerExit` 方法
```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate();

    int c = ctl.get();

    // 状态小于STOP
    if (runStateLessThan(c, STOP)) {
        // 没有被中断
        if (!completedAbruptly) {
            // 计算线程池需要保留的最小线程数
            // 若允许核心线程超时，则最小为0，否则最小为核心线程数
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            // 若最小线程数为0，并且阻塞队列不为空，则至少要有一个线程来处理，所以此处最小数量重置为1
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            // 判断当前线程数量是否大于等于要求的最小数量
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        // 进入这里有两种情况
        // 情况一：线程被中断，需要新建一个替换
        // 情况二：当前线程池的数量小于最小数量了，需要补足
        addWorker(null, false);
    }
}
```

---
本文完。
