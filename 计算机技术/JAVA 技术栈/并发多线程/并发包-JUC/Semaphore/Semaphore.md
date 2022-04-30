# Semaphore

## 简介

> A counting semaphore. Conceptually, a semaphore maintains a set of permits. Each [acquire()](<dfile:///Users/helong/Library/Application Support/Dash/DocSets/Java_SE11/Java.docset/Contents/Resources/Documents/java.base/java/util/concurrent/Semaphore.html#acquire()> "acquire()") blocks if necessary until a permit is available, and then takes it. Each [release()](<dfile:///Users/helong/Library/Application Support/Dash/DocSets/Java_SE11/Java.docset/Contents/Resources/Documents/java.base/java/util/concurrent/Semaphore.html#release()> "release()") adds a permit, potentially releasing a blocking acquirer. However, no actual permit objects are used; the `Semaphore` just keeps a count of the number available and acts accordingly.
>
> Semaphores are often used to restrict the number of threads than can access some (physical or logical) resource. For example, here is a class that uses a semaphore to control access to a pool of items:

Semaphore它是一种基于计数的信号量。它可以设定一个阈值，基于此，多个线程竞争获取许可信号，做自己的申请后归还，超过阈值后，线程申请许可信号将会被阻塞。更形象的说法应该是许可证管理器，**作用**： 控制某个资源可被同时访问的个数，通过 acquire() 获取一个许可，如果没有就等待，用 release() 释放一个许可。

无论是Synchroniezd还是ReentrantLock,一次都只允许一个线程访问一个资源,但是Semaphore可以指定多个线程同时访问某一个资源.

### 例子

```java
import java.util.concurrent.Semaphore;

class MyThread extends Thread {
    private Semaphore semaphore;
    
    public MyThread(String name, Semaphore semaphore) {
        super(name);
        this.semaphore = semaphore;
    }
    
    public void run() {        
        int count = 3;
        System.out.println(Thread.currentThread().getName() + " trying to acquire");
        try {
            semaphore.acquire(count);
            System.out.println(Thread.currentThread().getName() + " acquire successfully");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            semaphore.release(count);
            System.out.println(Thread.currentThread().getName() + " release successfully");
        }
    }
}

public class SemaphoreDemo {
    public final static int SEM_SIZE = 10;
    
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(SEM_SIZE);
        MyThread t1 = new MyThread("t1", semaphore);
        MyThread t2 = new MyThread("t2", semaphore);
        t1.start();
        t2.start();
        int permits = 5;
        System.out.println(Thread.currentThread().getName() + " trying to acquire");
        try {
            semaphore.acquire(permits);
            System.out.println(Thread.currentThread().getName() + " acquire successfully");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            semaphore.release();
            System.out.println(Thread.currentThread().getName() + " release successfully");
        }      
    }
}
   
```

## 原理

与 [ReentrantLock](../锁/ReentrantLock/ReentrantLock.md "ReentrantLock") 类似，内部也是利用了 `AQS` 来实现。默认为**非公平**锁

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}

```

## 场景&#x20;

1.  加锁（类似于ReentrantLock）

2.  异步任务同步返回

3.  控制线程并发数（限流）

## 注意点

*   release会添加令牌，令牌数量并不会以初始化的大小为准

*   Semaphore中release方法的调用并没有限制要在acquire后调用

## 参考

*   [https://pdai.tech/md/java/thread/java-thread-x-juc-tool-semaphore.html](https://pdai.tech/md/java/thread/java-thread-x-juc-tool-semaphore.html "https://pdai.tech/md/java/thread/java-thread-x-juc-tool-semaphore.html")

*   [http://soiiy.com/java/14346.html](http://soiiy.com/java/14346.html "http://soiiy.com/java/14346.html")
