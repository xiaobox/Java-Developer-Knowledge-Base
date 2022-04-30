# CompletionService

## 简介

**将生产新的异步任务与使用已完成任务的结果分离开来的服务。**

生产者 submit 执行的任务，使用者take 已完成的任务，并按照完成这些任务的完成顺序处理它们的结果。

例如，CompletionService 可以用来管理异步 IO ，执行读操作的任务作为程序或系统的一部分提交，然后，当完成读操作时，会在程序的不同部分执行其他操作，执行操作的顺序可能与所请求的顺序不同。

*   CompletionService是一个接口，它的实现类为 `ExecutorCompletionService` ，ExecutorCompletionService主要是为了增强executor线程池

*   CompletionService的实现目标是**任务先完成可优先获取到，即结果按照完成先后顺序排序。**

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h0vep91hb3j20pc0ba74y.jpg)

由于内部使用了 [BlockingQueue](../并发容器/BlockingQueue/BlockingQueue.md "BlockingQueue") 总体来说符合先到先得原则（**结果按照完成先后顺序排序**），比如以下伪代码：

```java
void solve(Executor e,Collection<Callable<Result>> solvers) throws InterruptedException, ExecutionException {
   
   CompletionService<Result> cs= new ExecutorCompletionService<>(e);
   solvers.forEach(cs::submit);
   
   for (int i = solvers.size(); i > 0; i--) {
     Result r = cs.take().get();
     if (r != null)
       use(r);
   }
 } 
```

当然如果你想只处理第一个非空的任务结果，当第一个 ready的时候取消掉其他的任务，可以这样写：

```java
 void solve(Executor e,Collection<Callable<Result>> solvers) throws InterruptedException {
   
   CompletionService<Result> cs = new ExecutorCompletionService<>(e);
   int n = solvers.size();
   List<Future<Result>> futures = new ArrayList<>(n);
   
   Result result = null;
   try {
     solvers.forEach(solver -> futures.add(cs.submit(solver)));
     for (int i = n; i > 0; i--) {
       try {
         Result r = cs.take().get();
         if (r != null) {
           result = r;
           break;
         }
       } catch (ExecutionException ignore) {}
     }
   } finally {
     futures.forEach(future -> future.cancel(true));
   }

   if (result != null)
     use(result);
 } 
```

## 方法概述

*   take()方法：取得最先完成任务的Future对象，谁执行时间最短谁最先返回。

*   poll() 方法：获取并移除表示下一个已完成任务的Future，如果不存在这样的任务，则返回null，方法poll是无阻塞的。

*   poll和take的区别在于：take如果获取不到值则会等待，而poll则会返回null。

*   submit()方法： 提交一个Callable或者Runnable类型的任务，并返回Future

## 总结

CompletionService 好比我同时种了几块地的麦子，然后就等待收割。收割时，则是那块先成熟了，则先去收割哪块麦子。
