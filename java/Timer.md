---
title: Timer底层分析
date: 2018-07-14 22:24:08
tags:
- timer
categories:
- java
---
Timer是jdk1.3中自带的一种线程调度任务的工具。可以执行一个只调度一次的任务也可以重复调度一个一定间隔时间的任务。
## 成员变量
```JAVA
//任务队列
private final TaskQueue queue = new TaskQueue();
//定时调用任务的线程类
private final TimerThread thread = new TimerThread(queue);
private final Object threadReaper = new Object();
//用于生成调度线程的名字
private final static AtomicInteger nextSerialNumber = new AtomicInteger(0);
```
## 构造器
```java
public Timer() {
    this("Timer-" + serialNumber());
}
public Timer(boolean isDaemon) {
    this("Timer-" + serialNumber(), isDaemon);
}
public Timer(String name) {
    thread.setName(name);
    thread.start();
}
public Timer(String name, boolean isDaemon) {
    thread.setName(name);
    thread.setDaemon(isDaemon);
    thread.start();
}
```
当我们实例化一个Timer类的时候会为线程设置一个名字并启动此线程然后一直处于等待状态直到到队列中加入任务
<!--more-->
## 关键方法
```JAVA
//在指定延迟后执行指定的任务
public void schedule(TimerTask task, long delay)
//在指定时间执行任务
public void schedule(TimerTask task, Date time)
//任务从指定的延迟后开始进行重复的固定延迟执行
public void schedule(TimerTask task, long delay, long period)
//任务在指定的时间开始进行重复的固定延迟执行
public void schedule(TimerTask task, Date firstTime, long period)
//任务在指定的延迟后开始进行重复的固定速率执行
public void scheduleAtFixedRate(TimerTask task, long delay, long period)
//任务在指定的时间开始进行重复的固定速率执行
public void scheduleAtFixedRate(TimerTask task, Date firstTime,long period)
```
schedule的重复执行与scheduleAtFixedRate的区别在于任务执行起始的时间基准点不一样，schedule下一次执行时间相对于上一次任务实际执行完成的时间点 ，而scheduleAtFixedRate下一次执行时间相对于上一次开始的 时间点 ，因此执行时间不会延后
## 核心方法
```JAVA
private void sched(TimerTask task, long time, long period) {
        if (time < 0)
            throw new IllegalArgumentException("Illegal execution time.");
        if (Math.abs(period) > (Long.MAX_VALUE >> 1))
            period >>= 1;
        synchronized(queue) {
            if (!thread.newTasksMayBeScheduled)
                throw new IllegalStateException("Timer already cancelled.");

            synchronized(task.lock) {
                if (task.state != TimerTask.VIRGIN)
                    throw new IllegalStateException(
                        "Task already scheduled or cancelled");
                task.nextExecutionTime = time;
                task.period = period;
                task.state = TimerTask.SCHEDULED;
            }
            queue.add(task);
            if (queue.getMin() == task)
                queue.notify();
        }
    }
```
关键方法里最终都会调用`sched`方法，该方法主要是设置任务的开始执行时间，周期与状态，将任务加入到队列中，如果加入的任务与队列中的第一个相等则唤醒线程去执行。
## TimerThread类
```JAVA
private void mainLoop() {
        while (true) {
            try {
                TimerTask task;
                boolean taskFired;
                synchronized(queue) {
                    while (queue.isEmpty() && newTasksMayBeScheduled)//队列为空一直等待
                        queue.wait();
                    if (queue.isEmpty())
                        break; 
                    long currentTime, executionTime;
                    task = queue.getMin();//获取任务
                    synchronized(task.lock) {
                        if (task.state == TimerTask.CANCELLED) {
                            queue.removeMin();
                            continue;  
                        }
                        currentTime = System.currentTimeMillis();
                        executionTime = task.nextExecutionTime;
                        if (taskFired = (executionTime<=currentTime)) {
                            if (task.period == 0) { //代表任务仅执行一次
                                queue.removeMin();//从队列中移除
                                task.state = TimerTask.EXECUTED;//改变任务状态
                            } else { //周期的执行
                                queue.rescheduleMin(
                                  task.period<0 ? currentTime   - task.period
                                                : executionTime + task.period);//重新排列队列里的任务
                            }
                        }
                    }
                    if (!taskFired) //未到达时间则等待
                        queue.wait(executionTime - currentTime);
                }
                if (taskFired) //达到条件则执行任务
                    task.run();
            } catch(InterruptedException e) {
            }
        }
    }
```
`TimerThread`类中主要的就是`mainLoop`方法，通过无限循环来从队列中取任务来执行，如果队列为空且newTasksMayBeScheduled为true时则会一直等待，在executionTime 时间小于当前时间的情况下会去判断下周期值period，如果period小于0则会更新更新任务上的nextExecutionTime时间为当前时间减去period时间，此时更新后的nextExecutionTime时间就大于了当前时间，并且当前任务会在队里重新排序，当下一次循环到此任务时则会等待(executionTime - currentTime)时间后才执行任务。如果在执行任务中出现了异常这里并没有做任务处理，在mainLoop最外层中的finally中会清除队列中的任务，并将newTasksMayBeScheduled设置为false。一旦一个任务中出现异常那么这个定时器将会失效，后面的任务就无法执行
## TaskQueue类
对于TaskQueue类我理解为是一个最小堆，里面的任务都是时间离当前时间最近的放在前面，添加任务时会将任务放在末尾然后从上下往上更新堆，删除时会将末尾的替换掉第一个，从上往下更新。TaskQueue实例化时，会默认初始一个长度为128的TimerTask数组，来存储TimerTask，如下：
```JAVA
TimerTask[] queue = new TimerTask[128]
```
当想TaskQueue中add任务时，若内部数组已满，则将数组长度扩展为当前的2倍。下面看下几个主要方法
### add方法
```JAVA
void add(TimerTask task) {
        if (size + 1 == queue.length)//判断是否需要扩容
            queue = Arrays.copyOf(queue, 2*queue.length);
        queue[++size] = task;//这里首个任务的下标是1并不是1
        fixUp(size);//更新最小堆，从下往上
    }
```
添加任务时会判断是否需要扩容，需要则扩为原来的2倍，不需要则将任务加在末尾然后更新最小堆
### removeMin方法
```JAVA
void removeMin() {
        queue[1] = queue[size];//用末尾替换掉第一个
        queue[size--] = null;  
        fixDown(1);//删除时的更新最小堆,从上往下
    }
```
删除时会将末尾的替换成第一个，然后将末尾设置为null且大小减1，然后通过`fixDown`方法从上往下更新。
### fixUp方法
```JAVA
private void fixUp(int k) {
        while (k > 1) {
            int j = k >> 1;
            if (queue[j].nextExecutionTime <= queue[k].nextExecutionTime)
                break;
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }
```
添加任务时更新堆，首先找到子节点的父节点然后进行比较，如果子小于父则两个进交换，进入下一次循环，若子大于父则跳出循环，这就保证了永远都是一个最小堆
### fixDown方法
```JAVA
private void fixDown(int k) {
        int j;
        while ((j = k << 1) <= size && j > 0) {
            if (j < size &&
                queue[j].nextExecutionTime > queue[j+1].nextExecutionTime)
                j++; // j indexes smallest kid
            if (queue[k].nextExecutionTime <= queue[j].nextExecutionTime)
                break;
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }
```
删除任务时更新堆，从根部从上往下，先找子节点，然后再进行比较，将最小的那个子节点与它交换位置，然后进入下次循环
## 总结
* Timer的任务是单线程来执行的，即只有一个线程来执行所有的任务
* 由于Timer只有一个线程来按顺序执行任务，当某一个任务执行失败而抛异常，这会导致后面的任务将无法被执行，所以建议当有定时任务的时候最好使用`ScheduledThreadPoolExecutor`来完成
