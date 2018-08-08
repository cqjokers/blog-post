---
title: java线程池分析
tags:
  - pool
categories:
  - java
abbrlink: bdea3b94
date: 2018-07-05 16:22:41
---

## 为什么需要线程池
***
* 线程是稀缺资源，频繁的创建对系统资源消耗较大
* 线程池可以减少创建和销毁线程的次数并且可以复用
## 线程池的创建
***
线程池的创建会借助于它的工厂类`Executors`来创建，主要有如下几种(jdk1.7)：
* `newSingleThreadExecutor`
```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
}
```
初始化一个只有一个线程的线程池，内部使用的是`LinkedBlockingQueue`作为阻塞队列，如果该线程异常结束则会创建一个新的线程来继续执行任务，这就保证了所提交的任务顺序执行。
*  `newFixedThreadPool`
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
```
初始化一个指定固定大小的线程池，其中核心线程数与最大线程数是一样的，其内部使用`LinkedBlockingQueue`作为阻塞队列，当线程池没有可执行任务时，也不会释放线程。
*  `newCachedThreadPool`
```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
 }
```
初始化一个可缓存线程的线程池，可创建的最大线程数为Integer.MAX_VALUE，内部使用的是`SynchronousQueue`作为阻塞队列，默认缓存时间为60s，在没有任务可执行时，当线程的空闲时间超过`keepAliveTime`时会自动释放线程资源，当提交新任务时，如果此时没有空闲的线程，则会创建新线程执行任务，因此会导致一定的系统开销，所以在使用此线程时需要注意并发的任务数，以防创建大量的线程导致性能降低。
*  `newScheduledThreadPool`
```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
}
```
初始化的线程池可以在指定的时间内周期性的执行所提交的任务，在实际的业务场景中可以使用该线程池定期的同步数据.
1、`ScheduledExecutorService`其中的`scheduleAtFixedRate `方法，当程序执行时间<频率时间（period）时，则下次执行开始时间=上一次执行开始时间+频率时间（period），当程序执行时间>频率时间（period）时，则会在上一次执行结束后立即进入下一次执行
2、`ScheduledExecutorService`其中的`scheduleWithFixedDelay`方法,程序会上一次执行结束后等待(period)时间再进行下一次执行
>以上几个除了`newScheduledThreadPool`内部实现使用的是`ScheduledThreadPoolExecutor`外其它都是基于`ThreadPoolExecutor`类实现
<!--more-->
## ThreadPoolExecutor对象
***
```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
}
```
>- corePoolSize：线程池核心线程数（提交新任务时，如果当前线程数小于corePoolSize则会创建新的线程执行任务直到当前线程数等于corePoolSize，如果当前线程数等于corePoolSize则会将新提交的任务放入阻塞队列中，如果执行了线程池的`prestartAllCoreThreads()`方法，线程池会提前创建并启动所有核心线程）
>- maximumPoolSize：线程池最大线程数(当前队列已满时，如果线程数小于maximumPoolSize则会创建新的线程执行新提交的任务)
>- keepAliveTime：线程空闲时存活时间，只有当线程数大于corePoolSize时此参数才会生效
>- unit：时间单位（keepAliveTime）
>- workQueue：保存实现了`Runable`接口的任务阻塞队列,关于workQueue值jdk提供了以下几种供选择：
1、ArrayBlockingQueue：基于数组结构的有界阻塞队列，按FIFO排序任务
2、LinkedBlockingQuene：基于链表结构的阻塞队列，按FIFO排序任务，吞吐量通常要高于ArrayBlockingQuene
3、SynchronousQuene：一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQuene
4、priorityBlockingQuene：优先级无界阻塞队列
>- threadFactory:创建线程的工厂，可以通过自定义的线程工厂给每个新建的线程设置一个具有识别度的线程名
>-  handler：线程池的饱和策略，当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了以下几种：
1、`ThreadPoolExecutor.AbortPolicy`：直接抛出`RejectedExecutionException`异常，默认策略
2、`ThreadPoolExecutor.DiscardPolicy`：直接丢弃任务与`AbortPolicy`不同的是不会抛出异常
3、`ThreadPoolExecutor.DiscardOldestPolicy`：丢弃阻塞队列中最靠前的任务，并执行当前任务
4、`ThreadPoolExecutor.CallerRunsPolicy`：用调用者所在的线程来执行任务
可以根据应用场景实现RejectedExecutionHandler接口，自定义饱和策略


执行流程
***
* 如果线程池中的线程数量少于corePoolSize，就创建新的线程来执行新添加的任务
*  如果线程池中的线程数量大于等于corePoolSize，但队列workQueue未满，则将新添加的任务放到workQueue中
* 如果线程池中的线程数量大于等于corePoolSize，且队列workQueue已满，但线程池中的线程数量小于maximumPoolSize，则会创建新的线程来处理被添加的任务
* 如果线程池中的线程数量等于了maximumPoolSize，就用RejectedExecutionHandler来执行拒绝策略
![](https://i.loli.net/2018/07/06/5b3edbc6e090c.png)
## 线程池状态
***
-  RUNNING：-1<<COUNT_BITS，即高3位为111，低29位为0，该状态的线程池会接收新任务，也会处理在阻塞队列中等待处理的任务
-  SHUTDOWN：0<<COUNT_BITS，即高3位为000，低29位为0，该状态的线程池不会再接收新任务，但还会处理已经提交到阻塞队列中等待处理的任务
-  STOP：1<<COUNT_BITS，即高3位为001，低29位为0，该状态的线程池不会再接收新任务，不会处理在阻塞队列中等待的任务，而且还会中断正在运行的任务
- TIDYING：2<<COUNT_BITS，即高3位为010，低29位为0，所有任务都被终止了，workerCount为0，为此状态时还将调用terminated()方法
- TERMINATED：3<<COUNT_BITS，即高3位为100，低29位为0，terminated()方法调用完成后变成此状态
## execute
***
```JAVA
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        //根据workerCountOf方法计算出当前线程数，判断当前线程数是否小于核心线程数
        if (workerCountOf(c) < corePoolSize) {
            // 执行addWorker方法创建新的线程执行任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //执行addWorker失败，判断线程池是否正在运行并且将任务加入队列中
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //再次校验，线程池不是RUNNING状态且成功从阻塞队列中删除任务
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //如果当前worker数量为0，通过addWorker(null, false)创建一个线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);//未指定firstTask,第二个参数 ? corePoolSize：maxPoolSize
        }
        //如果线程池不是运行状态或者进入队列失败，执行addWorker方法
        else if (!addWorker(command, false))
            reject(command);//addWorker方法执行失败，则拒绝当前command
    }
```
执行流程如下
>* 通过`workerCountOf`方法得到线程池当前线程数，如果当前线程数小于`corePoolSize`则通过`addWorker(command, true)`方法创建新worker线程，如创建成功返回，否则继续下面的步骤
>* 如果线程池处于RUNNING状态，且把任务成功放入阻塞队列中,如果加入失败则继续执行后面的步骤
1、如果线程池已经不是running状态了且从workQueue中删除任务成功，则拒绝添加新任务
2、如果线程池是运行状态，或者从workQueue中删除任务失败且当前worker数量为0则执行`addWorker(null, false)`创建一个线程
>*  如果线程池不是运行状态或者无法入队列则拒绝添加新任务
## addWorker
***
字面意思就是添加worker线程,主要负责创建新的线程并执行任务,代码如下：
```JAVA
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
            //如果线程池状态大于或等于SHUTDOWN且后面三个条件中有一个为false则不会继续进行
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);//worker数量
                 //如果worker数量>线程池最大上限CAPACITY
                //或者( worker数量>corePoolSize 或  worker数量>maximumPoolSize )则返回不再进行曳光弹
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //cas操作，使得worker数量加1
                if (compareAndIncrementWorkerCount(c))
                    break retry;//跳出retry循环
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)//如果状态不等于之前获取的state，跳出内层循环，继续去外层循环判断
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);//创建一个worker线程
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();//加锁
                try {
                    int rs = runStateOf(ctl.get());
                    //线程池状态小于SHUTDOWN或者（为SHUTDOWN状态且firstTask==null）
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);//将worker线程加入HashSet中(workers是一个HashSet)
                        //设置最大的池大小largestPoolSize，workerAdded设置为true
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();//释放锁
                }
                if (workerAdded) {//加入HashSet成功
                    t.start();//启动线程
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
addWorker执行流程如下：
>-  判断线程池状态是否为可以添加worker线程的状态，如果是则继续下一步，否则返回false
>-  判断线程池当前线程数量是否超过上限（corePoolSize 或 maximumPoolSize），如果超过则return false，否则对workerCount+1，继续往下执行
>- 在线程池的ReentrantLock保证下，向Workers中添加新创建的worker实例，添加完成后解锁，并启动worker线程，如果这几步操作都成功则返回true否则调用addWorkerFailed()逻辑
## worker类
***
线程池的工作线程通过Woker类实现，在ReentrantLock锁的保证下，把Woker实例添加到HashSet后，并启动Woker中的线程，其中Worker类设计如下：
```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
```
-  继承了AQS(AbstractQueuedSynchronizer)类，可以方便的实现工作线程的中止操作；
-  实现了Runnable接口，可以将自身作为一个任务在工作线程中执行；
-  当前提交的任务firstTask作为参数传入Worker的构造方法
创建worker时初始化AQS的状态为-1，将当前提交的任务firstTask作为参数传入构造方法并创建Thread
```JAVA
Worker(Runnable firstTask) {
            setState(-1); // 这所以初始化为-1是为了防止被中断
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
```
初始AQS状态为-1，此时不允许中断interrupt()，只有在worker线程启动了，执行了runWoker()，将state置为0，才能中断，为了防止运行中的worker被中断，所以runWorker()每次运行任务时都会lock()上锁，在调用shutdown时会先去获得worker锁后才能再中断这样就防止了中断运行中的worker
```JAVA
protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {//这里不是加1操作而是将state从0设置为1所以规避了重入锁
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
```
## runWorker
***
```JAVA
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // 释放锁,将state置为0，interruptIfStarted只有state>=0时才能调用中断
        boolean completedAbruptly = true;
        try {
            //如果task不等于null或者从队列中取出的task不等于null
            while (task != null || (task = getTask()) != null) {
                w.lock();//上锁，防止调用shutdown时中断正在运行的任务
                //两种情况：
                //1、当前线程池状态>=stop且没有设置中断状态则wt.interrupt
                //2、如果第一个条件<stop，但线程已经被中断，又清除了中断标示，再次判断线程池状态是否>=stop
                //如果是则wt.interrupt()
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();//调用中断操作
                try {
                    beforeExecute(wt, task);//任务执行前的操作
                    Throwable thrown = null;
                    try {
                        task.run();//执行任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);//任务执行后的操作
                    }
                } finally {
                    task = null;
                    w.completedTasks++;//完成任务数+1
                    w.unlock();//释放锁
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);//执行worker退出
        }
    }
```
执行流程如下：
>- 线程启动后通过释放锁将AQS的state置为0，允许中断当前worker线程
>- 执行firstTask调用task.run()，在执行之前会进行加锁操作，执行完后会释放锁，目的是防止在任务运行时被线程池一些中断操作中断
>- 如果自己的业务需要在任务执行前后进行操作，可以自定义`beforeExecute`和`afterExecute`方法
>- 当前任务执行完后，会通过getTask()从阻塞队列中获取新任务，如果队列中没有任务或者获取超时则当前worker则会执行退出流程
## getTask
***
```JAVA
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 如果当前线程池状态>=SHUTDOWN 且(线程池状态>=STOP 或者队列为空)时，会减少worker数量，并返回null
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();//cas操作减少worker数量，循环操作
                return null;
            }
            int wc = workerCountOf(c);
            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                //wc>maximumPoolSize且队列为空，或者timed与timedOut都为ture且(wc > 1 || 队列为空)
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) ://wc > corePoolSize
                    workQueue.take();//wc< corePoolSize
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```
执行流程如下：
>- 判断线程池状态是否满足，线程池为shutdown且( workQueue为空或者stop状态)都不满足，则会cas操作减少worker数量并返回null
>- 线程数量是否超过maximumPoolSize或者获取任务超时且(线程数大于1或者队列为空)，则会cas操作减少worker数量并返回null
>- 如果满足获取任务条件，根据是否需要定时获取调用不同方法：
    1、workQueue.poll()：如果在keepAliveTime时间内，阻塞队列还是没有任务，返回null
    2、workQueue.take()：如果阻塞队列为空，当前线程会被挂起等待；当队列中有任务加入时，线程被唤醒，take方法返回任务
>- 在阻塞从workQueue中获取任务时，可以被interrupt()中断，代码中捕获了InterruptedException，重置timedOut为初始值false，再次执行第1步中的判断，满足就继续获取任务，不满足return null，会进入worker退出的流程

参考资料：
[Java线程池ThreadPoolExecutor使用和分析](https://www.cnblogs.com/trust-freedom/p/6681948.html)
