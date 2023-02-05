---
title: Java中的异步任务
date: "2021/8/20 20:26:25"
tags: [Java]
categories: Java
---

## 前言

### Asynchrony

【图片来自Wikipedia】
![20230205202737](https:image.codingoer.top/blog/20230205202737.jpg)

refers to the occurrence of events independent of the main program flow and ways to deal with such events 。**指独立于主程序流程的事件的发生和处理此类事件的方法、**

### Asynchronous vs Synchronous

同步(Sync)和异步(Async)编程可以在一个或多个线程中完成。两者之间的主要区别在于，当使用同步编程时，我们可以一次执行一个任务，而当使用异步编程时，我们可以同时执行多个任务。

1. Asynchronous
在同步操作中，每次只执行一个任务，只有当一个任务完成时，才解除阻塞。换句话说，需要等待一个任务完成后再转移到下一个任务。
2. Synchronous
在异步操作中，可以在前一个任务完成之前转移到另一个任务。这样，通过异步编程，就能够同时处理多个请求，从而在更短的时间内完成更多任务。

![20230205202832](https:image.codingoer.top/blog/20230205202832.jpg)

Example：

- Sync
Single Thread：我开始煮鸡蛋，煮好后我就可以开始烤面包了。为了开始另一项工作，我不得不等待一项工作完成。
Multi-Thread：我开始煮鸡蛋，煮熟后，我妈妈会烤面包。这些任务由不同的人(线程)一个接一个地执行。
- Async
Single Thread：我开始煮鸡蛋，设好计时器，然后烤面包，开始另一个计时器。在异步模式中，我不需要等待一个任务完成才能开始另一个任务。
Multi-Thread：我雇了两个厨师，他们会为我煮鸡蛋和烤面包。他们可以同时做，他们不需要等待一个完成另一个开始。

### Async in Java

1. Async with Thread
2. Async with Future
3. Async with CompletableFuture

<!-- more --> 

## FutureTask

### 使用的几种场景

#### 执行多任务计算

Future直译是未来的意思，主要是将一些耗时的操作交给一个线程去执行，从而达到异步的目的。提交线程在提交任务和获得计算结果的过程可以进行其他任务执行，而不至于傻傻的等待结果的返回。

```java
public class ExExecutorFuture {
 
    private static ExecutorService executorService = Executors.newFixedThreadPool(2);
 
    public static Future<Integer> calculate(ComputeTask workThread) {
        return executorService.submit(workThread);
    }
 
    public class ComputeTask implements Callable<Integer> {
        private Integer result;
        private String taskName;
 
        public ComputeTask(Integer result, String taskName) {
            this.result = result;
            this.taskName = taskName;
            System.out.println("生成子线程计算任务 " + taskName);
        }
 
        @Override
        public Integer call() throws Exception {
            for (int i = 0; i < 100; i++) {
                result = +i;
            }
            TimeUnit.SECONDS.sleep(10);
            System.out.println("子线程计算任务: " + taskName + " 执行完成!");
            return result;
        }
    }
 
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExExecutorFuture exExecutorFuture = new ExExecutorFuture();
        List<Future<Integer>> taskList = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            ComputeTask computeTask = exExecutorFuture.new ComputeTask(i, "Task" + i);
            Future<Integer> futureTask = calculate(computeTask);
            taskList.add(futureTask);
        }
        for (int i = 0; i < 5; i++) {
            System.out.println("所有计算任务提交完毕, 主线程接着干其他事情！");
        }
        Integer totalResult = 0;
        for (Future<Integer> future : taskList) {
            totalResult += future.get();
        }
        System.out.println("多任务计算后的总结果是:" + totalResult);
        executorService.shutdown();
    }
}
```

#### 高并发环境确保执行一次

FutureTask的代码实现满足这种场景的使用。

```java
public class ExFutureTaskCallable implements Callable <String> {
    private final int count = 20;
    @Override
    public String call() throws Exception {
        System.out.println("开始执行了，在 " + Thread.currentThread().getName());
        for(int i = count; i > 0; i--)
        {
            System.out.println(Thread.currentThread().getName() + "当前票数：" + i);
        }
        return "sale out";
    }
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        Callable < String > callable = new ExFutureTaskCallable();
        FutureTask < String > futureTask = new FutureTask < > (callable);
        Thread mThread1 = new Thread(futureTask);
        Thread mThread2 = new Thread(futureTask);
        Thread mThread3 = new Thread(futureTask);
        mThread1.start();
        mThread2.start();
        mThread3.start();
    }
```

### 源码分析

#### UML图

![20230205203637](https:image.codingoer.top/blog/20230205203637.jpg)

FutureTask提供了Future的基础实现，并提供了开始和取消计算的方法，查询计算是否完成，检索计算结果。**只有在计算完成完成后才能检索结果，get方法在计算没完成前会一直阻塞。**

一旦计算结果完成，就不能重新开始或者取消。

因为实现了Runnable接口，FutureTask可以提交到Executor执行。

#### 实现思想

FutureTask定义了volatile变量

```java
private volatile int state;
private volatile Thread runner;
private volatile WaitNode waiters;
```

在JDK8中，设计的同步控制依赖于，使用CAS更新的state状态来跟踪完成情况。使用的是unsafe内联函数

```java
// Unsafe mechanics
private static final sun.misc.Unsafe UNSAFE;
private static final long stateOffset;
private static final long runnerOffset;
private static final long waitersOffset;
static {
    try { // 对象的字段在主存中的偏移位置
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> k = FutureTask.class;
        stateOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("state"));
        runnerOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("runner"));
        waitersOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("waiters"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```

如上，静态代码块获取到对象中属性在主内存的偏移位置，后面的方法都是都围绕这些个状态，比如，run方法或cancel方法中

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    ........ 省略
}
 
 
public void run() {
    if (state != NEW ||
    !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                 null, Thread.currentThread()))
    ........ 省略
    return;
}
```

任务状态的几种流转情况：

1. NEW -> COMPLETING -> NORMAL
2. NEW -> COMPLETING -> EXCEPTIONAL
3. NEW -> CANCELLED
4. NEW -> INTERRUPTING -> INTERRUPTED

在完成过程中，状态可能具有完成（当结果被设置）或者中断（正在执行的满足取消）的瞬时值。

#### outcome为什么没有使用volatile

state变量使用了volatile关键字修饰，保证可见性。更新state使用了CAS(Compare And Swap)，是用于实现多线程同步的原子指令。将内存位置的内容与给定的值进行比较。只有在相同的情况下，将该内存位置的内容修改为新的给定值。

源码中的outcome为什么没有使用volatile ????

Java 内存模型中的 happen-before规则，该规则定义了 Java 多线程操作的有序性和可见性，防止了编译器重排序对程序结果的影响。

当一个变量被多个线程读取并且至少被一个线程写入时，如果读操作和写操作没有happen-before原则，则将会产生数据竞争问题。要想保证操作B的线程看到操作A的结果，那么A和B之间必须满足happen-before原则。

Happen-before规则：（部分）

1. volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作。
2. 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于C。
3. 程序次序规则：一个线程内，按照代码执行顺序，书写在前面的操作先行发生于书写再后面的操作。

根据happen-before原理，state是volatile修饰的，所以对state的操作一定对其他线程可见；同一线程内，先执行的语句一定对后执行语句可见，所以只要其他线程看到任务state是normal，那一定可以最新的outcome。

get()拿到值的前提是state > COMPLETING， 否则就等待。

state 变为 NORMAL 的前提是 outcome = v。

#### 重点代码分析

1. 构造方法【FutureTask提供了两个构造方法】

一个是创建一个任务，改任务运行后执行指定的Callable方法

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
```

一个是运行时执行给定的Runnable方法，get方法将返回给定的结果

```java
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result); // 这里创建一个RunnableAdapter,Callable的变体
    this.state = NEW;       // ensure visibility of callable
}
```

2. run方法

```java
public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset, // 通过CAS设置当前执行的线程
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) { //callable不为空，状态为new才执行
            V result;
            boolean ran; // 局部变量，记录执行是否成功
            try {
                result = c.call(); // 回调callable的call方法
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex); // 失败时将异常放到outcome中
            }
            if (ran)
                set(result); // 执行成功将结果放到outcome中
        }
    } finally {
        runner = null;
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

3. get方法【重点看是如何阻塞的】

FutureTask提供了两个get方法：get()和get(long time, TimeUnit unit).

区别在于：get()方法在任务没有执行完成前，会一直阻塞。带有超时时间的get，在到时间后会抛出TimeOutException。

阻塞的代码在awaitDone方法中

```java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) { // 判断当前线程是否被中断
            removeWaiter(q); // 如果当前线程被中断，移除线程
            throw new InterruptedException();
        }
 
        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            q = new WaitNode();
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q); // 超时，移除等待的线程
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        else
            LockSupport.park(this); // 暂停当前线程
    }
}
```

4. cancel方法【状态可能会变成CANCELLED，或者INTERRUPTED。取消的任务不能get了】

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null) // 给执行的线程发出中断信号，告诉它应该中断了。
                    t.interrupt(); // 如果线程处于被阻塞状态，会立即退出阻塞状态并抛出InterruptedException，仅此而已。
            } finally { // 如果线程处于正常活动状态，会设置该线程的中断标识为true，仅此而已。
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED); // final state
            }
        }
    } finally {
        finishCompletion();
    }
    return true;
}
```

5. finishCompletion方法【移除并标志所有等待的线程，调用done方法扩展】

```java
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t); // 恢复暂停的线程
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }
 
    done(); // 扩展方法
 
    callable = null;        // to reduce footprint
}
```

## ExecutorService&Future

从上面的示例中可以看到，在执行多任务计算时，首先我们创建了一个线程池，然后将任务都放进去，下面说下执行流程。
ExecutorService接口中提供了submit方法，用于返回异步执行的结果。其本质还是使用的FutureTask。

### 执行流程

如下代码

```java
public static Future<Integer> calculate(ComputeTask workThread) {
    return executorService.submit(workThread);
}
```

1. `executorService.submit()`;

```java
<T> Future<T> submit(Callable<T> task);
```

2. `AbstractExecutorService.submit()`

```java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task); // 创建一个Runnable + Callable
    execute(ftask); // 由执行器决定怎么执行
    return ftask;
}
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```

3. `ThreadPoolExecutor.execute()`

```java
public void execute(Runnable command) {
     .....省略
     addWorker(null, false);
     .....省略
}
```

4. `ThreadPoolExecutor.addWork()`

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    .....省略
   Worker w = null;
   try {
      w = new Worker(firstTask);
      final Thread t = w.thread;
      if (t != null) {
         .....省略
         if (workerAdded) {
            t.start();
            workerStarted = true;
        }
      }
   }
}
```

5. `Worker.runWork`

```java
task.run();
```

6. `FutureTask.run()`
7. `Callable.call()`

## Future模式

### Future模式介绍

Future设计模式提供了一种凭据式的解决方案。

比如，今天你女朋友过生日，早上去蛋糕店预定了个蛋糕，蛋糕都是现做的，可能要到下午才能来取。一般情况下蛋糕店会给你开一个凭据，此凭据就是Future。下午的时候可以拿着凭据去取蛋糕。

使用场景：

当某个任务运行需要很长时间时，调用线程在提交任务之后的徒劳等待，对CPU资源来说是一种浪费，在这段等待时间里，完全可以进行其他任务的执行，这种场景完全符合Future设计模式的应用。

### Future模式代码实现

#### Future设计模式的UML类图。

![20230205205957](https:image.codingoer.top/blog/20230205205957.jpg)

#### 类说明

| class | 说明 |
| ------ | ----- |
| FutureService | 接口，主要用于提交任务 |
| FutureServiceImpl | FutureService实现类，主要作用在于当提交任务时创建一个新的线程来受理任务，进而达到异步执行的效果 |
| Future | 接口，提供了获取计算结果和判断任务是否完成等 |
| FutureTask | Future实现类，获取结果时进入阻塞 |
| Task | 接口，提供给调用者实现计算逻辑用的 |
| Callback | 接口，回调接口，避免阻塞 |

#### 代码实现示例

1. FutureService【定义了提交任务的接口】
```java
public interface FutureService<IN, OUT> {
 
    Future<?> submit(Runnable runnable);
 
    Future<OUT> submit(Task<IN, OUT> task, IN input);
 
    FutureTask<OUT> submit(Task<IN, OUT> task, IN input, Callback<OUT> callback);
}
```

2. FutureServiceImpl【提交任务后，具体任务怎么实现】
```java
public class FutureServiceImpl<IN, OUT> implements FutureService<IN, OUT> {
 
    // 为执行的线程指定名字前缀，为线程起一个特殊的名字是一个好的编码习惯
    private final static String FUTURE_THREAD_PREFIX = "future-";
 
    private final AtomicInteger nextCounter = new AtomicInteger(0);
 
    private String getNextName() {
        return FUTURE_THREAD_PREFIX + nextCounter.getAndIncrement();
    }
 
    @Override
    public Future<?> submit(Runnable runnable) {
        final FutureTask<Void> futureTask = new FutureTask<>();
        new Thread(() -> {
            runnable.run();
            futureTask.finish(null);
        }, getNextName()).start();
 
        return futureTask;
    }
 
    @Override
    public Future<OUT> submit(Task<IN, OUT> task, IN input) {
        final FutureTask<OUT> futureTask = new FutureTask<>();
        new Thread(() -> {
            OUT result = task.get(input);
            futureTask.finish(result);
        }, getNextName()).start();
 
        return futureTask;
    }
 
    @Override
    public FutureTask<OUT> submit(Task<IN, OUT> task, IN input, Callback<OUT> callback) {
        final FutureTask<OUT> futureTask = new FutureTask<>();
        new Thread(() -> {
            OUT result = task.get(input);
            futureTask.finish(result);
            if (Objects.isNull(callback)) {
                callback.call(result);
            }
        }, getNextName()).start();
 
        return futureTask;
    }
}
```

3. Future【异步计算结果】
```java
public interface Future<T> {
 
    T get() throws InterruptedException;
 
    boolean done();
}
```

4. FutureTask【Future的基础实现】
```java
public class FutureTask<T> implements Future<T> {
 
    private T result;
    private boolean isDone = false;
    private final Object LOCK = new Object();
 
    @Override
    public T get() throws InterruptedException {
        synchronized (LOCK) {
            // 当任务还没完成时，调用get方法会被挂起而进入阻塞
            while (!isDone) {
                LOCK.wait();
            }
 
            return result;
        }
    }
 
    protected void finish(T result) {
        synchronized (LOCK) {
            if (isDone) {
                return;
            }
 
            this.result = result;
            this.isDone = true;
            // 利用线程间的通信,知道任务完成唤醒阻塞线程
            LOCK.notifyAll();
        }
    }
 
    @Override
    public boolean done() {
        return isDone;
    }
}
```

5. Task接口【提供调用者实现计算的逻辑】与Callback接口【回调】
```java
OUT get(IN input);
 
 
void call(T t);
```

#### 测试类

1. 有返回值任务
```java
public void submitWithReturn() throws InterruptedException {
    FutureService<String, Integer> service = FutureService.newService();
 
    Future<Integer> future = service.submit(input -> {
        try {
            TimeUnit.SECONDS.sleep(20);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("finish done.");
        return input.length();
    }, "Hello");
 
    System.out.println("do something else.");
 
    // 当前main线程进入阻塞
    Integer length = future.get();
    System.out.println("计算完成，长度：" + length);
}
```

2. 有返回值任务，使用callback回调函数
```java
public void submitWithCallback() {
    FutureService<String, Integer> service = FutureService.newService();
 
    Future<Integer> future = service.submit(input -> {
        try {
            TimeUnit.SECONDS.sleep(20);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("finish done.");
        return input.length();
    }, "Hello", System.out::println);
 
 
    System.out.println("do something else.");
}
```

### JDK内置Future模式

#### UML类图

![20230205211537](https:image.codingoer.top/blog/20230205211537.jpg)

#### 类说明

| class | 说明 |
| ------ | ----- |
| Executor | 接口，顶层接口，提供了一种将任务提交与每个任务将如何运行分离的方法 |
| ExecutorService | 扩展接口，提供了一些方法来管理终止，以及一些方法可以生成一个Future来跟踪一个或多个异步任务的进度 |
| AbstractExecutorService | 抽象类，提供ExecutorService执行方法的默认实现，使用newTaskFor返回的RunnableFuture实现submit等方法。|
| ThreadPoolExecutor | 一个ExecutorService实现，使用可能的几个池线程之一执行每个提交的任务。常说的线程池 |
| RunnableFuture | Future接口与Runnable接口的结合. |
| Future | 接口，代表的是异步计算的结果，提供的方法用于检查计算是否完成、等待计算完成和检索计算结果。 |
| FutureTask | Future的基础实现，并提供了开始和取消计算的方法，查询计算是否完成，检索计算的结果。 |

#### 代码示例

基于ExecutorService和Future实现的异步处理工具类。

1. CallableFuture【自定义的FutureTask，用来处理被线程池拒绝的任务】
```java
public class CallableFuture<V> implements Future<V>, Callable<V> {
 
    private V result;
 
    private Callable<V> callable;
 
    public CallableFuture(Callable<V> callable) {
        this.callable = callable;
    }
 
    @Override
    public V call() {
        try {
            return result = callable.call();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
 
    @Override
    public boolean cancel(boolean mayInterruptIfRunning) {
        return false;
    }
 
    @Override
    public boolean isCancelled() {
        return false;
    }
 
    @Override
    public boolean isDone() {
        return false;
    }
 
    @Override
    public V get() throws InterruptedException, ExecutionException {
        return result;
    }
 
    @Override
    public V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
        return result;
    }
}
```

2. ExecutorUtils【异步执行工具类】
```java
public static <Q> List<Q> execute(ExecutorService executor, List<? extends Callable<Q>> taskList) {
    int size = taskList.size();
    List<Q> results = new ArrayList<>(size);
    if (size < 1) {
        return results;
    }
 
    List<CallableFuture<Q>> rejectedList = new ArrayList<>();
    List<Future<Q>> futureList = new ArrayList<>(size);
    for (final Callable<Q> callable : taskList) {
        try {
            Future<Q> future = executor.submit(callable);
            futureList.add(future);
        } catch (RejectedExecutionException e) {
            CallableFuture<Q> callFuture = new CallableFuture<>(callable);
            rejectedList.add(callFuture);
            futureList.add(callFuture);
        }
    }
    try {
        for (CallableFuture<Q> future : rejectedList) {
            future.call();
        }
        for (Future<Q> future : futureList) {
            results.add(FutureUtils.get(future));
        }
        return results;
    } catch (RuntimeException e) {
        for (Future<Q> future : futureList) {
            future.cancel(true);
        }
        throw e;
    }
}
```

#### 测试类

```java
private static final ExecutorService executor = new ThreadPoolExecutor(2, 5, 0L,TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>(50));
 
public static <Q> List<Q> execute(List<Callable<Q>> taskList) {
    return ExecutorUtils.execute(executor, taskList);
}
 
public static void main(String[] args) {
    List<Callable<Integer>> callableList = new ArrayList<>(100);
 
    for (int i = 0; i < 100; i++) {
        callableList.add(() -> {
            TimeUnit.SECONDS.sleep(1);
            return 1;
        });
    }
 
    List<Integer> executeResult = execute(callableList);
    Integer totalCount = 0;
    for (Integer value : executeResult) {
        totalCount += value;
    }
 
    System.out.println(totalCount);
    executor.shutdown();
}
```

线程池设置：拒绝策略使用默认的AbortPolicy，抛出RejectedExecutionHandler。当线程池拒绝任务提交后，创建CallableFuture，同步执行，保证任务不丢失。
