# CompletableFuture

## 先谈谈 Future

Callable 与 Runnable 的功能大致相似，但是 call() 函数有返回值。Callable 一般是和 ExecutorService 配合来使用的
   
        Future 就是对于具体的 Runnable 或者 Callable 任务的执行结果进行取消、查询是否完成
   
        在 Future 接口中声明了 5 个方法
   

*   cancel 方法用来取消任务，如果取消任务成功则返回 true，如果取消任务失败则返回 false。

*   isCancelled 方法表示任务是否被取消成功，如果在任务正常完成前被取消成功，则返回 true。&#x20;

*   isDone 方法表示任务是否已经完成，若任务完成，则返回 true；&#x20;

*   get() 方法用来获取执行结果，这个方法会**产生阻塞**，会一直等到任务执行完毕才返回；(所以一般需要使用future.isDone先判断任务是否全部执行完成，完成后再使用future.get得到结果)&#x20;

*   get(long timeout, TimeUnit unit) 用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回 null。

也就是说 Future 提供了三种功能：
    1）判断任务是否完成；
    2）能够中断任务；
    3）能够获取任务执行结果。

因为 Future 只是一个接口，所以是无法直接用来创建对象使用的，因此就有了 FutureTask。

来两个 demo:

```java
public static void futureDemo1() throws ExecutionException, InterruptedException {
    ThreadPoolExecutor pool = CommonThreadPool.getPool();    
    Future<Integer> f = pool.submit(() -> {      
        // 长时间的异步计算      
        Thread.sleep(2000);      
        // 然后返回结果      
        return 100;    
    });    
    while (!f.isDone()) {      
        System.out.println(System.currentTimeMillis() + " 还没结束");   
    }    
    //结束后，获取结果    
    System.out.println(f.get());
  }
```

Future 只实现了异步，而没有实现回调，主线程 get 时会阻塞，可以轮询以便获取异步调用是否完成。在实际的使用中建议使用 Guava ListenableFuture 来实现异步非阻塞，目的就是多任务异步执行，通过回调的方式来获取执行结果而不需轮询任务状态。

```java
public static void futureDemo2() {
    ListeningExecutorService executorService = MoreExecutors.listeningDecorator(CommonThreadPool.getPool());
    IntStream.rangeClosed(1, 10).forEach(i -> {      
        ListenableFuture<Integer> listenableFuture = executorService.submit(() -> {            
            // 长时间的异步计算            
            // Thread.sleep(3000);            
            // 然后返回结果            
            return 100;          
        });
      Futures.addCallback(listenableFuture, new FutureCallback<Integer>() {        
          @Override        
          public void onSuccess(Integer result) {          
              System.out.println("get listenable future's result with callback " + result);        
          }
          @Override        
          public void onFailure(Throwable t) {          
              t.printStackTrace();        
          }      
      }, executorService);    
    });  
}
```

## CompletableFuture

Futrue 对于结果的获取却是很不方便，只能通过阻塞或者轮询的方式得到任务的结果。
在 Java 8 中，新增加了一个包含 50 个方法左右的类：CompletableFuture，提供了非常强大的 Future 的扩展功能。

*   CompletableFuture 能够将回调放到与任务不同的线程中执行，也能将回调作为继续执行的同步函数，在与任务相同的线程中执行。它避免了传统回调最大的问题，那就是能够将控制流分离到不同的事件处理器中。

*   CompletableFuture 弥补了 Future 模式的缺点。在异步的任务完成后，需要用其结果继续操作时，无需等待。可以直接通过 thenAccept、thenApply、thenCompose 等方式将前面异步处理的结果交给另外一个异步事件处理线程来处理。

下面将会一个个的例子来说明 CompletableFuture
       

### 异步执行

```java
/**
   *
   * public static CompletableFuture<Void>   runAsync(Runnable runnable)
   * public static CompletableFuture<Void>   runAsync(Runnable runnable, Executor executor)
   * public static  CompletableFuture  supplyAsync(Supplier supplier)
   * public static  CompletableFuture  supplyAsync(Supplier supplier, Executor executor)
   *
   * 以 Async 结尾并且没有指定 Executor 的方法会使用 ForkJoinPool.commonPool() 作为它的线程池执行异步代码。
   *
   * runAsync 方法也好理解，它以 Runnable 函数式接口类型为参数，所以 CompletableFuture 的计算结果为空。
   *
   * supplyAsync 方法以 Supplier函数式接口类型为参数，CompletableFuture 的计算结果类型为 U。
   */
public static void runAsyncExample() throws ExecutionException, InterruptedException {

    CompletableFuture<Void> cf = CompletableFuture.runAsync(() -> {
      System.out.println("异常执行代码");
    });

    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
      //长时间的计算任务
      return "·00";
    });

    System.out.println(future.join());

  }
```

### 计算结果完成时的处理

```java
/**
   *
   * 当 CompletableFuture 的计算结果完成，或者抛出异常的时候，我们可以执行特定的 Action。主要是下面的方法：
   *
   * whenComplete(BiConsumer<? super T,? super Throwable> action) public CompletableFuture<T>
   * whenCompleteAsync(BiConsumer<? super T,? super Throwable> action) public CompletableFuture<T>
   * whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor) public
   * CompletableFuture<T>     exceptionally(Function<Throwable,? extends T> fn)
   *
   * 不以 Async 结尾的方法由原来的线程计算，以 Async 结尾的方法由默认的线程池 ForkJoinPool.commonPool() 或者指定的线程池 executor 运行。
   * Java 的 CompletableFuture 类总是遵循这样的原则
   *
   * 如果你希望不管 CompletableFuture 运行正常与否 都执行一段代码，如释放资源，更新状态，记录日志等，但是同时不影响原来的执行结果。
   * 那么你可以使用 whenComplete 方法。exceptionally 非常类似于 catch()，而 whenComplete 则非常类似于 finally:
   */
  public static void whenComplete() throws ExecutionException, InterruptedException {

    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(new Supplier<Integer>() {
      @Override
      public Integer get() {
        return 2323;
      }
    });
    Future<Integer> f = future.whenComplete((v, e) -> {
      System.out.println(v);
      System.out.println(e);
    });
    System.out.println(f.get());

  }
```

### handle 是执行任务完成时对结果的处理

```java
private static class HttpResponse {

    private final int status;
    private final String body;

    public HttpResponse(final int status, final String body) {
      this.status = status;
      this.body = body;
    }

    @Override
    public String toString() {
      return status + " - " + body;
    }
  }

  /**
   * handle 是执行任务完成时对结果的处理。handle 方法和 thenApply 方法处理方式基本一样。不同的是 handle 是在任务完成后再执行，还可以处理异常的任务。
   * 这组方法兼有 whenComplete 和转换的两个功能
   *
   * public  CompletionStage handle(BiFunction<? super T, Throwable, ? extends U> fn);
   * public  CompletionStage handleAsync(BiFunction<? super T, Throwable, ? extends U> fn);
   * public  CompletionStage handleAsync(BiFunction<? super T, Throwable, ? extends U> fn,Executor executor);
   *
   * thenApply 只可以执行正常的任务，任务出现异常则不执行 thenApply 方法。
   * public  CompletableFuture thenApply(Function<? super T,? extends U> fn)
   * public  CompletableFuture thenApplyAsync(Function<? super T,? extends U> fn)
   * public  CompletableFuture thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
   */
  public static void handle() throws ExecutionException, InterruptedException {

    for (final boolean failure : new boolean[]{false, true}) {

      CompletableFuture<Integer> x = CompletableFuture.supplyAsync(() -> {
        if (failure) {
          throw new RuntimeException("Oops, something went wrong");
        }
        return 42;
      });

      /** * Returns a new CompletableFuture that, when this CompletableFuture completes either normally or exceptionally, * is executed with this stage's result and exception as arguments to the supplied function. */
      CompletableFuture<HttpResponse> tryX = x
          // Note that tryX and x are of different type.
          .handle((value, ex) -> {
            if (value != null) {
              // We get a chance to transform the result...
              return new HttpResponse(200, value.toString());
            } else {
              // ... or return details on the error using the ExecutionException's message:
              return new HttpResponse(500, ex.getMessage());
            }
          });

      // Blocks (avoid this in production code!), and either returns the promise's value:
      System.out.println(tryX.get());
      System.out.println("isCompletedExceptionally = " + tryX.isCompletedExceptionally());

    }
```

### 转换

```java
/**
   * 转换
   * @throws ExecutionException
   * @throws InterruptedException
   */
  public static void thenApply() throws ExecutionException, InterruptedException {
    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
      return 100;
    });
    CompletableFuture<String> f =  future.thenApplyAsync(i -> i * 10).thenApply(i -> i.toString());
    //"1000"
    System.out.println(f.get());
  }
```

### Action

```java
/**
   * 上面的方法是当计算完成的时候，会生成新的计算结果 (thenApply, handle)，或者返回同样的计算结果 whenComplete
   * CompletableFuture 还提供了一种处理结果的方法，只对结果执行 Action, 而不返回新的计算值，因此计算值为 Void:
   *
   * public CompletableFuture<Void>   thenAccept(Consumer<? super T> action)
   * public CompletableFuture<Void>   thenAcceptAsync(Consumer<? super T> action)
   * public CompletableFuture<Void>   thenAcceptAsync(Consumer<? super T> action, Executor executor)
   */
  public static void action() throws ExecutionException, InterruptedException {

    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
      return 100;
    });
    CompletableFuture<Void> f =  future.thenAccept(System.out::println);
    System.out.println(f.get());

  }
```

### thenAccept

```java
/**
   * thenAcceptBoth 以及相关方法提供了类似的功能，当两个 CompletionStage 都正常完成计算的时候，就会执行提供的 action，它用来组合另外一个异步的结果。
   * runAfterBoth 是当两个 CompletionStage 都正常完成计算的时候，执行一个 Runnable，这个 Runnable 并不使用计算的结果。
   *
   * public  CompletableFuture<Void>   thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action)
   * public  CompletableFuture<Void>   thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action)
   * public  CompletableFuture<Void>   thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action, Executor executor)
   * public     CompletableFuture<Void>   runAfterBoth(CompletionStage<?> other,  Runnable action)
   */
  public static void thenAcceptBoth() throws ExecutionException, InterruptedException {
    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
      return 100;
    });
    CompletableFuture<Void> f =  future.thenAcceptBoth(CompletableFuture.completedFuture(10), (x, y) -> System.out.println(x * y));
    System.out.println(f.get());

  }
```

### thenRun

```java
/**
   * 当计算完成的时候会执行一个 Runnable, 与 thenAccept 不同，Runnable 并不使用 CompletableFuture 计算的结果。
   *
   * public CompletableFuture<Void>   thenRun(Runnable action)
   * public CompletableFuture<Void>   thenRunAsync(Runnable action)
   * public CompletableFuture<Void>   thenRunAsync(Runnable action, Executor executor)
   */
  public static void  thenRun() throws ExecutionException, InterruptedException {

    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
      return 100;
    });
    CompletableFuture<Void> f =  future.thenRun(() -> System.out.println("finished"));
    System.out.println(f.get());

  }

```

### 复合

```java
/**
   * thenCombine 用来复合另外一个 CompletionStage 的结果。它的功能类似
   *
   * A +
   *   |
   *   +------> C
   *   +------^
   * B +
   *
   * 两个 CompletionStage 是并行执行的，它们之间并没有先后依赖顺序，other 并不会等待先前的 CompletableFuture 执行完毕后再执行。
   *
   * public <U,V> CompletableFuture<V>   thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
   * public <U,V> CompletableFuture<V>   thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
   * public <U,V> CompletableFuture<V>   thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor)
   *
   * 其实从功能上来讲，它们的功能更类似 thenAcceptBoth，只不过 thenAcceptBoth 是纯消费，它的函数参数没有返回值，而 thenCombine 的函数参数 fn 有返回值。
   */
  public static void thenCombine() throws ExecutionException, InterruptedException {

    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
      return 100;
    });
    CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
      return "abc";
    });
    CompletableFuture<String> f =  future.thenCombine(future2, (x,y) -> y + "-" + x);
    System.out.println(f.get()); //abc-100

  }
```

### 组合

```java
/**
   * 组合
   * 这一组方法接受一个 Function 作为参数，这个 Function 的输入是当前的 CompletableFuture 的计算值，返回结果将是一个新的 CompletableFuture，
   * 这个新的 CompletableFuture 会组合原来的 CompletableFuture 和函数返回的 CompletableFuture。因此它的功能类似：A +--> B +---> C
   *
   * thenCompose 返回的对象并不是函数 fn 返回的对象，如果原来的 CompletableFuture 还没有计算出来，它就会生成一个新的组合后的 CompletableFuture。
   */
  public static void thenCompose() throws ExecutionException, InterruptedException {

    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
      return 100;
    });
    CompletableFuture<String> f =  future.thenCompose( i -> {
      return CompletableFuture.supplyAsync(() -> {
        return (i * 10) + "";
      });
    });
    System.out.println(f.get()); //1000

  }
```

### Either

```java
/**
   * Either 语义：表示的是两个 CompletableFuture，当其中任意一个 CompletableFuture 计算完成的时候就会执行。
   *
   * public CompletableFuture<Void>   acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action)
   * public CompletableFuture<Void>   acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action)
   * public CompletableFuture<Void>   acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action, Executor executor)
   * public  CompletableFuture   applyToEither(CompletionStage<? extends T> other, Function<? super T,U> fn)
   * public  CompletableFuture   applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn)
   * public  CompletableFuture   applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn, Executor executor)
   *
   * acceptEither 方法是当任意一个 CompletionStage 完成的时候，action 这个消费者就会被执行。这个方法返回 CompletableFuture<Void>
   *
   * applyToEither 方法是当任意一个 CompletionStage 完成的时候，fn 会被执行，它的返回值会当作新的 CompletableFuture的计算结果。
   */
  public static void either() {

    Random random = new Random();

    CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {

      try {
        Thread.sleep(random.nextInt(1000));
      } catch (InterruptedException e) {
        e.printStackTrace();
      }

      return "from future1";
    });

    CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {

      try {
        Thread.sleep(random.nextInt(1000));
      } catch (InterruptedException e) {
        e.printStackTrace();
      }

      return "from future2";
    });

    CompletableFuture<Void> haha = future1
        .acceptEitherAsync(future2, str -> System.out.println("The future is " + str));

    try {
      System.out.println(haha.get());
    } catch (InterruptedException e) {
      e.printStackTrace();
    } catch (ExecutionException e) {
      e.printStackTrace();
    }

  }
```

### All

```java
/**
   * allOf 方法是当所有的 CompletableFuture 都执行完后执行计算。
   * anyOf 接受任意多的 CompletableFuture
   *
   * anyOf 方法是当任意一个 CompletableFuture 执行完后就会执行计算，计算的结果相同。
   */
  public static void allOfAndAnyOf() throws ExecutionException, InterruptedException {

    Random rand = new Random();
    CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
      try {
        Thread.sleep(10000 + rand.nextInt(1000));
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      return 100;
    });
    CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
      try {
        Thread.sleep(10000 + rand.nextInt(1000));
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      return "abc";
    });
    //CompletableFuture<Void> f =  CompletableFuture.allOf(future1,future2);
    CompletableFuture<Object> f =  CompletableFuture.anyOf(future1,future2);
    System.out.println(f.get());

  }
```

### allOf 如果其中一个失败了如何快速结束所有？

```java
/**
   * allOf 如果其中一个失败了如何快速结束所有？
   *
   * 默认情况下，allOf 会等待所有的任务都完成，即使其中有一个失败了，也不会影响其他任务继续执行。但是大部分情况下，一个任务的失败，往往意味着整个任务的失败，继续执行完剩余的任务意义并不大。
   * 在 谷歌的 Guava 的 allAsList 如果其中某个任务失败整个任务就会取消执行：
   *
   * 一种做法就是对 allOf 数组中的每个 CompletableFuture 的 exceptionally 方法进行捕获处理：如果有异常，那么整个 allOf 就直接抛出那个异常：
   */

  public static void allOfOneFail(){
    CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
      System.out.println("-- future1 -->");
      try {
        Thread.sleep(1000);
      } catch (InterruptedException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
      }
      System.out.println("<-- future1 --");
      return "Hello";
    });

    CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
      System.out.println("-- future2 -->");
      try {
        Thread.sleep(2000);
      } catch (InterruptedException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
      }
      System.out.println("<-- future2 --");
      throw new RuntimeException("Oops!");
    });

    CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
      System.out.println("-- future3 -->");
      try {
        Thread.sleep(4000);
      } catch (InterruptedException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
      }
      System.out.println("<-- future3 --");
      return "world";
    });

    // CompletableFuture<Void> combinedFuture = CompletableFuture.allOf(future1, future2, future3);
    // combinedFuture.join();

    CompletableFuture<Void> allWithFailFast = CompletableFuture.allOf(future1, future2, future3);
    Stream.of(future1, future2, future3).forEach(f -> f.exceptionally(e -> {
      allWithFailFast.completeExceptionally(e);
      return null;
    }));

    allWithFailFast.join();
  }
```

### 我自己的一个 demo

```java
/**
   * 假设你有一个集合，需要请求 N 个接口，接口数据全部返回后进行后续操作。
   */
  public static void myDemo(){

    ArrayList<String> strings = Lists.newArrayList("1", "2", "3", "4");

    CompletableFuture[] cfs = strings.stream()
        .map(s -> CompletableFuture.supplyAsync(() -> {
          return s + " $";
        }).thenAccept(s1 -> {
          System.out.println(s1+ " #");
        }).exceptionally(t -> {
          return null;
        })).toArray(CompletableFuture[]::new);

    // 等待 future 全部执行完
    CompletableFuture.allOf(cfs).join();

  }
```
