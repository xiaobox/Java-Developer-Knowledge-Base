# CountDownLatch和CyclicBarrier

## CountDownLatch

### 介绍

*   CountDownLatch： 利用它可以实现类似计数器的功能。比如有一个任务A，它要等待其他4个任务执行完毕之后才能执行，此时就可以利用CountDownLatch来实现这种功能了。

*   从名字可以看出，CountDownLatch是一个倒数计数的锁，当倒数到0时触发事件，也就是开锁，其他人就可以进入了。在一些应用场合中，需要等待某个条件达到要求后才能做后面的事情；同时当线程都完成后也会触发事件，以便进行后面的操作。

*   CountDownLatch最重要的方法是countDown()和await()，前者主要是倒数一次，后者是等待倒数到0，如果没有到达0，就只有阻塞等待了。

*   一个CountDouwnLatch实例是不能重复使用的，也就是说它是一次性的，锁一经被打开就不能再关闭使用了，如果想重复使用，请考虑使用CyclicBarrier。

*   类比 CountDownLatch 的作用和 Thread.join() 方法类似，可用于一组线程和另外一组线程的协作，但本质上还是有区别的，Thread.join()等待的是线程的结束，而 CountDownLatch 等待的是计数器的归零，与线程生命周期没有丝毫关系

### 原理

CountDownLatch其实可以把它看作一个计数器，这个计数器的操作是**原子操作**，同时只能有一个线程去操作这个计数器，也就是同时只能有一个线程去减这个计数器里面的值。

内部使用AQS同步状态来表示计数。计数为0时，所有的Acquire操作（CountDownLatch的await方法）才可以通过。具体可以参考 [彻底理解 AQS（AbstractQueuedSynchronizer）](../彻底理解%20AQS（AbstractQueuedSynchro/彻底理解%20AQS（AbstractQueuedSynchronizer）.md "彻底理解 AQS（AbstractQueuedSynchronizer）")

### 例子

下面的例子简单的说明了CountDownLatch的使用方法，模拟了100米赛跑，10名选手已经准备就绪，只等裁判一声令下。当所有人都到达终点时，比赛结束。

```java
public class TestCountDownLatch {

    public static void main(String[] args) throws InterruptedException {
        // 开始的倒数锁
        final CountDownLatch begin = new CountDownLatch(1);
        // 结束的倒数锁
        final CountDownLatch end = new CountDownLatch(10);
        // 十名选手
        final ExecutorService exec = Executors.newFixedThreadPool(10);
        for (int index = 0; index < 10; index++) {
            final int NO = index + 1;
            Runnable run = new Runnable() {
                public void run() {
                    try {
                        begin.await();
                        Thread.sleep((long) (Math.random() * 10000));
                        System.out.println("No." + NO + " arrived");
                    } catch (InterruptedException e) {
                    } finally {
                        end.countDown();
                    }
                }
            };
            exec.submit(run);
        }
        System.out.println("Game Start");
        begin.countDown();
        end.await();
        System.out.println("Game Over");
        exec.shutdown();
    }

}
```

### 使用场景

CountDownLatch 适用于一组线程和另一个主线程之间的工作协作，一个主线程等待一组工作线程的任务完毕才继续它的执行是使用 CountDownLatch 的主要场景，异步回调大部分也基于 CountDownLatch 实现

## CyclicBarrier

> 字面意思回环栅栏，通过它可以实现让一组线程等待至某个状态之后再全部同时执行。 &#x20;
> 叫做回环是因为当所有等待线程都被释放以后，CyclicBarrier可以被重用。 &#x20;
> 叫做栅栏，大概是描述所有线程被栅栏挡住了，当都达到时，一起跳过栅栏执行，也算形象。我们可以把这个状态就叫做barrier。

功能类似CountDownLatch,CyclicBarrier可以被重用。

CountDownLatch 和 CycliBarrier 二者的区别在于：

*   CyclicBarrier的某个线程运行到某个点上之后（执行`await` 方法），该线程即停止运行，直到所有的线程都到达了这个点，所有线程才重新运行；CountDownLatch则不是，某线程运行到某个点上之后，执行 `countDown` 方法后， 只是给某个数值-1而已，该线程继续运行

*   **CyclicBarrier可重用**，CountDownLatch不可重用，计数值为0该CountDownLatch就不可再用了

*   CountDownLatch一般用于一个或多个线程，等待其他线程执行完任务后，再才执行;CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行

### 循环使用

循环使用指的是在大门被打开后，可以再次关闭；即再让之前指定数目的线程在屏障前阻塞等待，然后再次打开大门。

方法**reset()** 的作用就是重置屏障，以保证循环使用。

> Resets the barrier to its initial state. If any parties are currently waiting at the barrier, they will return with a BrokenBarrierException. Note that resets after a breakage has occurred for other reasons can be complicated to carry out; threads need to re-synchronize in some other way, and choose one to perform the reset. It may be preferable to instead create a new barrier for subsequent use.
>
> 将屏障重置为其初始状态。此时，如果有任何的一个参与者正在屏障前等待，它将会返回一个 BrokenBarrierException异常。
> 注意：如果因为其他原因使屏障发生损坏，此时屏障的重置将会变得很复杂；为了将来的使用，相比需要考虑使用其他方式重新同步线程，并选择其中一个线程来执行重置，更好的解决办法是创建一个新的屏障。

```java
// 将屏障重置为其初始化状态即重置为构造函数传入的parties值。
public void reset() 
```

### 例子

构建函数 有两个：

```java
public CyclicBarrier(int parties)
public CyclicBarrier(int parties, Runnable barrierAction)

```

解释下第二个：CyclicBarrier 支持一个可选的 Runnable 动作，该动作在队伍中的最后一个线程到达之后执行。这个方法对于在所有任务开始之前更新一下共享状态很有用。

下面是一个例子：

```java
public class Client {

    public static void main(String[] args) throws Exception{

        CyclicBarrier cyclicBarrier = new CyclicBarrier(3,new TourGuideTask());
        Executor executor = Executors.newFixedThreadPool(3);
        //登哥最大牌，到的最晚
        executor.execute(new TravelTask(cyclicBarrier,"哈登",5));
        executor.execute(new TravelTask(cyclicBarrier,"保罗",3));
        executor.execute(new TravelTask(cyclicBarrier,"戈登",1));
    }
}

public class TravelTask implements Runnable{

    private CyclicBarrier cyclicBarrier;
    private String name;
    private int arriveTime;//赶到的时间

    public TravelTask(CyclicBarrier cyclicBarrier,String name,int arriveTime){
        this.cyclicBarrier = cyclicBarrier;
        this.name = name;
        this.arriveTime = arriveTime;
    }

    @Override
    public void run() {
        try {
            //模拟达到需要花的时间
            Thread.sleep(arriveTime * 1000);
            System.out.println(name +"到达集合点");
            cyclicBarrier.await();
            System.out.println(name +"开始旅行啦～～");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}

/**
 * 导游线程，都到达目的地时，发放护照和签证
 */
public class TourGuideTask implements Runnable{

    @Override
    public void run() {
        System.out.println("****导游分发护照签证****");
        try {
            //模拟发护照签证需要2秒
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

 
```

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h0vjqz61b0j20r606ut98.jpg)

### 原理

原理上是利用 可重入锁 ReentrantLock 和 Condition 来实现的。可以参考 [ReentrantLock](../锁/ReentrantLock/ReentrantLock.md "ReentrantLock") [condition](../condition/condition.md "condition")

## 参考

*   [https://www.iteye.com/blog/zapldy-746458](https://www.iteye.com/blog/zapldy-746458 "https://www.iteye.com/blog/zapldy-746458")

*   [https://www.jianshu.com/p/4ef4bbf01811](https://www.jianshu.com/p/4ef4bbf01811 "https://www.jianshu.com/p/4ef4bbf01811")

*   [https://www.cnblogs.com/boothsun/p/7248735.html](https://www.cnblogs.com/boothsun/p/7248735.html "https://www.cnblogs.com/boothsun/p/7248735.html")
