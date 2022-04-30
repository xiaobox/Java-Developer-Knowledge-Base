# Lock（并发编程的锁机制）

## 定义

### 可重入锁

可重入锁（ 如果锁具备可重入性，则称作为可重入锁。像 synchronized 和 ReentrantLock 都是可重入锁，可重入性在我看来实际上表明了锁的分配机制：基于线程的分配，而不是基于方法调用的分配。举个简单的例子，当一个线程执行到某个 synchronized 方法时，比如说 method1，而在 method1 中会调用另外一个 synchronized 方法 method2，此时线程不必重新去申请锁，而是可以直接执行方法 method2。

### 读写锁&#x20;

读写锁将对一个资源的访问分成了 2 个锁，如文件，一个读锁和一个写锁。正因为有了读写锁，才使得多个线程之间的读操作不会发生冲突。ReadWriteLock 就是读写锁，它是一个接口，ReentrantReadWriteLock 实现了这个接口。可以通过 readLock() 获取读锁，通过 writeLock() 获取写锁。

### 可中断锁

可中断锁，即可以中断的锁。在 Java 中，**synchronized 就不是可中断锁，而 Lock 是可中断锁**。 如果某一线程 A 正在执行锁中的代码，另一线程 B 正在等待获取该锁，可能由于等待时间过长，线程 B 不想等待了，想先处理其他事情，我们可以让它中断自己或者在别的线程中中断它，这种就是可中断锁。

Lock 接口中的 lockInterruptibly() 方法就体现了 Lock 的可中断性。

### 公平锁

公平锁即尽量以请求锁的顺序来获取锁。同时有多个线程在等待一个锁，当这个锁被释放时，等待时间最久的线程（最先请求的线程）会获得该锁，这种就是公平锁。
非公平锁即无法保证锁的获取是按照请求锁的顺序进行的，这样就可能导致某个或者一些线程永远获取不到锁。
synchronized 是非公平锁，它无法保证等待的线程获取锁的顺序。对于 ReentrantLock 和 ReentrantReadWriteLock，默认情况下是非公平锁，但是可以设置为公平锁。

## 由来

synchronized 是 java 中的一个关键字，也就是说是 Java 语言内置的特性。那么为什么会出现 Lock 呢？

### synchronized 的缺陷

*   不能响应中断

*   没有读写锁，资源全部串行锁定，性能不高

*   其他

### synchronized 和 lock 的区别

*   Lock 是一个**接口**，而 synchronized 是 Java 中的关键字，synchronized 是内置的语言实现；

*   synchronized 在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而 Lock 在发生异常时，如果没有主动通过 unLock() 去释放锁，则很可能造成死锁现象，因此使用 Lock 时需要在 finally 块中释放锁；

*   Lock 可以让等待锁的线程**响应中断**，而 synchronized 却不行，使用 synchronized 时，等待的线程会一直等待下去，不能够响应中断；

*   通过 Lock 可以知道有没有成功获取锁，而 synchronized 却无法办到。

*   Lock 可以**提高多个线程进行读操作的效率**。（可以通过 readwritelock 实现读写分离）

*   性能上来说，在资源竞争不激烈的情形下，Lock 性能稍微比 synchronized 差点（编译程序通常会尽可能的进行优化 synchronized）。但是当同步非常激烈的时候，synchronized 的性能一下子能下降好几十倍。而 ReentrantLock 确还能维持常态。

### 性能比较

*   JDK1.5 中，synchronized 是性能低效的。因为这是一个重量级操作，它对性能最大的影响是阻塞的是实现，挂起线程和恢复线程的操作都需要转入内核态中完成，这些操作给系统的并发性带来了很大的压力。相比之下使用 Java 提供的 Lock 对象，性能更高一些。多线程环境下，synchronized 的吞吐量下降的非常严重，而 ReentrankLock 则能基本保持在同一个比较稳定的水平上。

*   到了 JDK1.6，发生了变化，对 synchronize 加入了很多优化措施，有自适应自旋，锁消除，锁粗化，轻量级锁，偏向锁等等。导致在 JDK1.6 上 synchronize 的性能并不比 Lock 差。官方也表示，他们也更支持 synchronize，在未来的版本中还有优化余地，所以还是提倡在 synchronized 能实现需求的情况下，优先考虑使用 synchronized 来进行同步。

### 中断响应

当通过 lockInterruptibly() 方法获取某个锁时，如果不能获取到，**只有进行等待的情况下，是可以响应中断的**。

而用 synchronized 修饰的话，当一个线程处于等待某个锁的状态，是无法被中断的，只有一直等待下去。
