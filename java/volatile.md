---
title: volatile关键字
date: 2018-07-24
tags:
 -volatile
categories:
 -java
---
  一个变量被volatile修饰之后，则具备了两层含义：
* 可见性
* 禁止指令重排序（保证有序性）
```java
    private boolean flag = false;
    private void test(){
        new Thread(new Runnable() {
            @Override
            public void run() {
                while(!flag) {
                    doSomeThing();
                }
            }
        });
    }
    private void stop(){
        flag = true;
    }
```
&emsp;&emsp;上面例子是经常用一个标识来控制线程的结束，理想状态下当另一个线程调用stop方法后则会立即停止，但这毕竟只是我们的假设，上面的代码很有可能会出现一个死循环，当另一个线程调用stop方法后将flag的值设置为true后但并没有去更新主内存中的数据此时就会出现死循环，加上volatile后则会解决上面的问题。因为使用volatile关键字后会强制将修改的值立即写入主存，并让其它线程中的该变量的值失效，当其它线程需要此变量值时由于自己工作内中的已经失效了所以会从主内存中去获得所以得到的就是最新值。
```JAVA
int a = 1;    （1）
int b = 2;     （2）
volatile boolean flag = false;   （3）
c = 3;   （4）
d = 4;    （5）
```
<!--more-->
&emsp;&emsp;上面例子中由于flag使用了volatile修饰，所以它不会先于1，2执行也不会后于4，5执行，并且1，2执行的结果是对3，4，5可见的，但是并没有保证1，2和4，5的执行顺序。

volatile关键字禁止指令重排序有两层意思：
　　1）当程序执行到volatile变量的读操作或者写操作时，在其前面的操作肯定全部已经完成，且后面的操作肯定还没有开始，前面的执行结果是对后面的操作可见的
　　2）在进行指令优化时，不能将在对volatile变量访问的语句放在其后面执行，也不能把volatile变量后面的语句放到其前面执行。

**volatile不能保证操作的原子性**
```JAVA
    static volatile int num = 0;
    static CountDownLatch countDownLatch = new CountDownLatch(5);
    private static void test() throws InterruptedException {
        for(int i=0;i<5;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        num++;
                    countDownLatch.countDown();
                };
            }.start();
        }
        //等待计算线程执行完
        countDownLatch.await();
        System.out.println(num);
	}
```
执行结果：
```JAVA
4351  3786  4518  3123  3275
```
对上面代码执行了5次但结果都不是预期的5000，由此可以看出虽然num使用了volatile来修饰但在对其进行自增时并不是原子操作。由于自增操作是分为3步完成，在多线程情况下，假如有一个线程将num读取到了自己的本地内存中且是一个很小的值，此时其他线程很有可能已经将num增大了很多，这个线程依然对num进行自加，然后重新写到主存中，这样就导致了num的结果与预期的不符合。要解决上面的问题只需将num类型替换成`AtomicInteger`就能解决问题
**volatile 的应用**
* 双重检查锁的单例模式
```JAVA
public class Singleton {
  private static volatile Singleton singleton;
  private Singleton() {
  }
  public static Singleton getInstance() {
    if (singleton == null) {
        synchronized (Singleton.class) {
            if (singleton == null) {
                singleton = new Singleton();
            }
        }
    }
    return singleton;
  }
}
```
* 控制停止线程的标记
```java
    private volatile boolean flag = false;
    private void test(){
        new Thread(new Runnable() {
            @Override
            public void run() {
                while(!flag) {
                    doSomeThing();
                }
            }
        });
    }
    private void stop(){
        flag = true;
    }
```