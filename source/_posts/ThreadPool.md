---
title: Java线程池---Executor框架源码深度解析
date: 2019-06-11 17:30:37
categories: 
- Java
tags:
- Java多线程
- 线程池
- 源码解析
---
## 为什么会需要线程池技术？

* (1)Thread是一个重量级的资源，它的创建，启动以及销毁都是比较耗费性能的；重复利用线程，减少线程创建，销毁的开销，是一种好的程序设计习惯。
* (2)通过new Thread的方法创建线程难以管理，并且难以控制数量，线程的数量通常和系统的性能呈抛物线关系，合理控制线程数量才能发挥出系统最强的性能。
* (3)使用new Thread的方式创建的线程不利于扩展，比如定时，定期执行任务实现起来相对麻烦，但线程池提供了相应的稳定并且高效的解决方案。

<!-- more -->

## 线程池的原理

所谓线程池，我们可以简单地把它理解成一个池子，只不过里面装的都是已经创建好的线程，当我们向这个池子提交任务时，池子中的某个线程就会主动地执行这个任务。当我们提交的任务数量大于池子中的线程时，线程池会自动创建线程加入池子中，但是会有数量的控制，就像游泳池里的水一样，当水达到一定的量时，就会溢出了。当任务比较少的时候，线程池会在系统空闲的时候自动地回收资源。为了能够异步地提交任务和缓存未处理的任务，需要有一个任务缓存队列。

```
小插曲：
线程池是如何提升性能的？
解：比如你们的服务器执行一项任务的时候为T，
创建一个线程的时间为T1,
执行任务花了T2的时间，
线程销毁用了T3的时间。

那么显然  T ＝ T1 + T2 + T3

因为我们是从池子里面拿出线程出来执行任务的，
用完之后，又把线程放回池子里，
所以T1和T3就基本可以忽略不计了。

因此  T ＝ T2
```

## 线程池的组成要素

* **任务列队：**用来缓存提交的任务
* **工作线程队列：**一个线程池要想很好地管理和控制线程数量，可以通过后面的两个参数实现，线程池需要维护的核心数量corePoolSize，线程池自动扩充线程时最大的数量：maximumPoolSize。
* **任务拒绝策略：**如果线程数量已达上限并且任务队列也满了，就需要相应的拒绝策略来告诉提交任务的人。
* **线程工厂：**用于创建线程的，按需定制线程，执行我们的提交的任务。

## JDK提供的线程池实现方案---Executor框架

前面三个小节主要是给大家简单回忆一下线程池的一些基础知识，接下来就正式进入我们这次的主题，看看Doug Lea大神是如何实现整个线程池框架的。

Executor是一个强大且非常灵活的异步执行框架，它以Runnable为任务对象，并且提供了一套将任务提交和任务执行分离开来的机制，让提交任务的过程和执行任务的过程得到充分的解耦。Executor还提供了对线程池生命周期的管理，以及统计信息收集，和应用程序管理机制和性能监控，以及任务调度功能。随后，我们就开始看看这个强大的Executor框架是怎么样一步一步实现的。

### Executor UML

我们先通过Executor的UML整体了解一下Executor框架：

![Executor UML](https://upload-images.jianshu.io/upload_images/1640787-db4d27d74fc437e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/865)

* **Executor(解耦利器)：** Executor采用了**命令设计模式**，让任务的提交和任务的执行之间得到了充分的解耦。
*  **ExecutorSevice:** 它扩展了Executor接口，我们常用的线程池基本都是实现了ExecutorService，它提供了丰富的操作。
* **ScheduledExecutorService:** 可定时调度的接口，它可以对提交了的任务进行延迟执行或者周期性执行。
* **AbstractExecutorService：** ExecutorService接口的默认实现，他只实现了Executor中的部分方法，并提供了一个newTaskFor()方法，返回了一个RunnableFuture对象。
* **ForkJoinPool：** ForkJoinPool可以充分利用CPU，多核CPU的优势，它支持将一个任务拆分成多个小任务的，然后把这些小任务放到CPU的多个核心并行处理，最后再把结果合并起成，得到最终的结果的特殊线程池。
* **ThreadPoolExecutor：** 线程池，所有的核心实现都在这里了！
*  **ScheduledThreadPoolExecutor：** 可调度的线程池。 

### 细说Executor源码

从整体看完Executor框架后，现在我们要开始，我们就采取自顶向下的方式详细深入地聊聊Executor了。

#### Executor接口源码分析
```
public interface Executor {  
    /**
     * 在未来的那个时间点执行command任务，
     * 具体是在新的线程中执行呢，还是在线程池，
     * 又或者是调用线程中执行，则由Executor的实现者决定 
     */ 
    void execute(Runnable command);
}
```
像我们前面说的一样，Executor接口的主要实现的功能是，让任务的提交和任务的执行得到解耦，通常，Executor是用来代替显式创建线程的。比如：相比通过
```
//eg1:
new Thread(new RunnableTask()).start().
```
创建一组线程，更好的方式是：
```
//eg2:
Executor executor = anExecutor;
executor.execute(new RunnableTask1());
executor.execute(new RunnableTask2());
...
```
为什么会更好？这个问题就充分体现了我们这篇文章一开始所讲的，为什么需要线程池技术这一个论点了。

```
但需要注意的一个点是：
我们通过eg1方式一定是异步执行的。
而eg2这种方式并不严格要求异步执行某个，比如下面的例子中，他会在调用线程中立即执行任务：
//eg3:
class DirectExecutor implements Executor {
     public void execute(Runnable r) {
         r.run();
     }
}

但更具代表性的还是在其它线程中执行任务：
//eg4:
class DirectExecutor implements Executor {
     public void execute(Runnable r) {
         new Thread(r).start();
     }
}
```
#### ExecutorService接口源码分析

```
public interface ExecutorService extends Executor {

    void shutdown();
    List<Runnable> shutdownNow();

    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

ExecutorService提供了终止线程池的管理方法，并且还提供了一些返回一个Future对象的方法，通过Future对象，我们就可以跟踪到异步任务的进程了。

一个ExecutorService是可以被关闭的，如果ExecutorService被关闭了，它将会拒绝接收新的任务。有两个不同的方法可以关闭ExecutorService
* shutdown()  允许先前提交的任务在终止之前执行。
* shutdownNow()  会阻止开启新的任务并且尝试停止当前正在执行的任务。

当一个线程池被终止时，没有正在执行的任务，也没有等待执行的任务，也没有新的任务可以提交，一个没有被使用的池程池应该关闭以允许回收它的资源。

submit()方法扩展了Executor#execute(Runnable)方法，创建被返回一个Future对象，这个对象可以用于取消任务的执行或者等待任务完成并取出返回值，至于如何取消任务，或者取值，大家可以参考一些对Future接口的使用案例，这里就不扩展了。

invokeAny()和invokeAll()方法是以批量的形式执行一组任务，然后等待至少一个或者全部的任务完成。
ExecutorCompletionService类可以用来定义于这些方法，包括submit(),invokeAll()和invokeAny().

#### AbstractExecutorService抽象类源码分析

AbstractExecutorService类提供了ExecutorService接口部分方法的默认实现，这个类使用了newTaskFor()方法返回的RunnableFuture对象实现了submit()，invokeAny()和invokeAll()方法，如果你熟悉Future体系，那在看源码就是水到渠成的事情了。如果你还不熟悉，我之前也写了一篇关于Future体系，深入分析的文章。
[Future体系源码深度解析](https://www.jianshu.com/p/70de9fb543d8)  
```
public abstract class AbstractExecutorService implements ExecutorService {
    //通过runnable和一个value参数，创建一个FutureTask
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
    //通过一个callable创建一个FutureTask
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }

}
```
第一个方法中的value其实就是当你的任务执行完成时，你想要通过Future#get()拿到的预期的结果，而第二个方法就是借助了Callable接口，拿到了任务的执行结果。上面我提到的那篇文章有深入详细的分析。如果你感兴趣这整个过程，可以看看，都是干货。

接下来，我们就开始分析**submit()方法**：
```
//通过Future#get()获取出来的值，永远都为null
public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
}

//通过Future#get()获取出来的值为result。
public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
}
//通过Future#get()获取出来的值为任务执行的结果。
public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
}
```
sumbit()方法有三个重载，主要是提交的参数不同。然后由这些参数来创建一个FutureTask.然后交给Executor#execute()方法执行，最后再通过他的get()方法就可以获取到一个执行结行的结果了，但是需要注意的一点就是，这三个方法的返回值是不一样的，前面两个方法的返回情况还是比较简单的，但是第三个方法，再获取返回值的时候，不一定能获取到执行结果的，需要结合任务执行的情况一起合析，还有一个点就是，再调用get()方法使，会让当前线程阻塞，直到任务执行完成。


#### invokeAll()方法
```
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        //如果任务集后为空，则抛出一个NullPointerException
        if (tasks == null)
            throw new NullPointerException();
        //创建一个和任务数量等大的ArrayList.
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
         //标志任务是否完成
        boolean done = false;
        try {
            //将tasks里面的每个Callable转化成Future,添加到futures里面，并交给Executor#execute()方法执行
            for (Callable<T> t : tasks) {
                RunnableFuture<T> f = newTaskFor(t);
                futures.add(f);
                execute(f);
            }
            //判断futures里面的Future是否执行结束，如果还没有完成，通过get()阻塞直到任务完成
            //也是就说执行完这一段代码，futures里面的每一个任务都是执行完成的情况   
            for (int i = 0, size = futures.size(); i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) {
                    try {
                        f.get();
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    }
                }
            }
            done = true;
            //返回ArrayList
            return futures;
        } finally {
            if (!done)//如果任务没有完成的，就全部都取消，并释放内存
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);//采用中断线程的方式取消任务
        }
}
```
invokeAll()方法整体分析下面，也是很简单的，可以批量提交并执行任务，当所有任务都执行完成时，返回一个保存任务状态和执行结果的Future列表。它的重载方法也是类似的，就是加入了一个超时时间，不管是所有的任务都执行完，还是已经到达超时的时间，只有两个有中满足其中一个，就会返回一个保存任务状态和执行结果的Future列表。下面简单分析一下：
```
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
        //如果任务集后为空，则抛出一个NullPointerException
        if (tasks == null)
            throw new NullPointerException();
        //将超时时间转化成纳秒
        long nanos = unit.toNanos(timeout);
        //创建一个和任务数量等大的ArrayList.
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        //标志任务是否完成
        boolean done = false;
        try {
            //将tasks里面的每个Callable转化成Future,添加到futures里面
            for (Callable<T> t : tasks)
                futures.add(newTaskFor(t));
            //超时时间点
            final long deadline = System.nanoTime() + nanos;
            //记录一下futures的大小，
            final int size = futures.size();
            //执行任务，如果超时就直接返回futures
            for (int i = 0; i < size; i++) {
                execute((Runnable)futures.get(i));
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L)
                    return futures;
            }
            //判断futures里面的Future是否执行结束，如果还没有完成，通过get()阻塞直到任务完成
            //也是就说执行完这一段代码，要么futures里面的每一个任务都是执行完成了，要么就是超时了  
            for (int i = 0; i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) {
                    if (nanos <= 0L)
                        return futures;
                    try {
                        //这里捕捉了get()方法可能出现了所有的异常
                        f.get(nanos, TimeUnit.NANOSECONDS);
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    } catch (TimeoutException toe) {
                        return futures;
                    }
                    nanos = deadline - System.nanoTime();
                }
            }
            //标记任务完成
            done = true;
            return futures;
        } finally {
            if (!done)//如果任务没有完成的，就全部都取消，并释放内存
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);//采用中断线程的方式取消任务
        }
}
```

invokeAny()方法也是执行给定的一批量任务，通是通过内部的doInvokeAny()方法完成了，与invokeAll()方法不同的是这一批任务中的某个任务完成了，就返回他的结果，所以他返回的是最快执行完成的那个任务的结果，他的重载方法，加入了超时机制，如果超过了限制的时间也没有一个任务完成，那么就会抛出超时异常了。
 
## 前方高能

经过前面这么多知识的铺垫，是时候要开始核心**ThreadPoolExecutor线程池**的学习了，我们先来认识它的成员变量，认清了他们有助于更好地进行源码阅读。
```
public class ThreadPoolExecutor extends AbstractExecutorService {
    //默认为false,用于控制核心线程在空闲状态是否会被回收。
    private volatile boolean allowCoreThreadTimeOut;
    
    //存活的时间，当非核心线程闲置的时间超过它时，将会被回收
    private volatile long keepAliveTime;

    //核心线程的数量，默认情况下即使核心线程处于空闲状态也不会被回收， 
    //除非把allowCoreThreadTimeOut设置为true.
    private volatile int corePoolSize;

    //存放任务的队列，如果 当前线程数>核心线程数  新提交的任务将会被放入到该任务对列。
    //它是一个阻塞队列，通常有SynchronousQueue(直接提交)，LinkedBlockingQueue(无界队列)，   
    //ArrayBlockingQueue (有界队列)，三种
    private final BlockingQueue<Runnable> workQueue;
    
    //线程池能容纳最大的线程数
    private volatile int maximumPoolSize; 
    
    //创建新线程使使用的线程工厂
    private volatile ThreadFactory threadFactory;
    
    //当任务队列满并且当前线程等于线程池能容纳的最大线程数时所采用的拒绝策略
    private volatile RejectedExecutionHandler handler;
    
     //默认的拒绝策略
     private static final RejectedExecutionHandler defaultHandler = new AbortPolicy();
    
     //统计线程池完成的任务数量 
     private long completedTaskCount;
     
    //工作线程池，存放工作线程的地方,Worker包含一个线程和对应的任务
    private final HashSet<Worker> workers = new HashSet<Worker>();      
      
    //用于记录最大的工作线程池的工作线程的大小.
     private int largestPoolSize;
}
```
以上是线程池的核心参数，整个线程池主要就是围绕着这些参数开展的，所以牢牢记住这些参数，对阅读线程池的源码的帮助是非常大的。接下来我们了解一下线程池的状态，他和FutureTask的状态一样，也是由一组int类型变量也标识的
```
    //ctl是控制线程的状态的，里面包含两个状态，线程的数量和线程池运行的状态
    //它是一AtomicInteger类型，由后面的操作可以得知，
    //高3位是用来保存线程池运行的状态的，低29位用于保存线程的数量，这里的限制是2^29-1，大约有5亿3千6百万左右。
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    //COUNT_BITS ＝29，用于位于操作
    private static final int COUNT_BITS = Integer.SIZE - 3;
    //2^29-1 ＝536,870,911 线程的容量
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    //高3位：111：接受新任务并且继续处理阻塞队列中的任务
    private static final int RUNNING    = -1 << COUNT_BITS;
    //高3位：000：不接受新任务但是会继续处理阻塞队列中的任务
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    //高3位：001：不接受新任务，不在执行阻塞队列中的任务，中断正在执行的任务
    private static final int STOP       =  1 << COUNT_BITS;
    //高3位：010：所有任务都已经完成，线程数都被回收，线程会转到TIDYING状态会继续执行钩子方法
    private static final int TIDYING    =  2 << COUNT_BITS;
    //高3位：110：钩子方法执行完毕
    private static final int TERMINATED =  3 << COUNT_BITS;

    // 用于打包ctl或者拆包ctl的，如果你源码看得比较多，这种操作应该是经常看到的
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }

```
线程池之间的状态转化情况：
* RUNNING -> SHUTDOWN: 调用了shutdown()方法，也可能隐含在finalize()方法中
* (RUNNING or SHUTDOWN) -> STOP: 调用了shutdownNow()方法
* SHUTDOWN -> TIDYING: 队列和线程池都是空的时候
* STOP -> TIDYING: 线程池为空的时候
* TIDYING -> TERMINATED: terminated()钩子方法执行完成

在了解了线程池的这些基本参数之后，就可以开始正式看实现过程了。但是阅读了这么久，先休息一下再继续往下面读吧，不着急，慢慢来。

如果休息够了，我们就继续吧，先从构造函数入手，ThreadPoolExecutor有四个重载的构造函数，但是我们看参数最多的一个就知道这个构造函数怎么用了，

```
 public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        //这里主要是对线程池的基本参数的判断，
        //其中keepAliveTimen必须大于0，
        //并且maximumPoolSize>corePoolSize>0
        //否则就抛出 IllegalArgumentException异常
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();

        //构造函数中传入的workQueue，threadFactory，handler均不能为null
        //否则就会抛出NullPointerException
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        
        //然后就开始赋值操作。
        this.acc = System.getSecurityManager() == null ?null :AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        //unit是控制keepAliveTime的时间单位的
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
在有了前面的对线程池重要参数讲解的铺垫,ThreadPoolExecutor的构造函数就变得特别的清淅和简单了，其中keepAliveTime的时间单位由unit参数控制，必须>0,然后maximumPoolSize>corePoolSize>0，任务队列，线程工厂，拒绝策略均不能为null。如果在使用了其它的构造函数，可以会使用默认的的线程工厂和默认的拒绝策略。

接下来我们先来看一个案例

```
public static void main(String[] args) throws Exception {
   ExecutorService executor = new ThreadPoolExecutor(2, 4, 10, TimeUnit.SECONDS, new ArrayBlockingQueue<>(3));
   for (int i = 1; i <= 10; ++i) {
            final int index = i;
            Runnable runnable = () -> {
                try {
                    System.out.println(Thread.currentThread().getName() + "我是任务" + index);
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            executor.submit(runnable);
            //这里确认id小的任务先提交到线程池
            TimeUnit.MILLISECONDS.sleep(30);
        }
   executor.shutdown();
}
//输入
pool-1-thread-1 我是任务1
Exception in thread "main" java.util.concurrent.RejectedExecutionException: Task java.util.concurrent.FutureTask@6d03e736 rejected from java.util.concurrent.ThreadPoolExecutor@568db2f2
[Running, pool size = 4, active threads = 4, queued tasks = 3, completed tasks = 0]
pool-1-thread-2 我是任务2
pool-1-thread-3 我是任务6
pool-1-thread-4 我是任务7
pool-1-thread-1 我是任务3
pool-1-thread-2 我是任务4
pool-1-thread-3 我是任务5
```
我们创建了一个核心线程为：2，最大线程数为：4，空闲线程存活的时间为：10秒，有界队列的容量为：3的线程池。然后我们模拟提交10个任务，为了让id小的任务先有机会运行，我们提交一个任务后先休眠30ms，然后模拟每个任务需要执行1秒，确认10个任务都是先提交了，才有任务执行完，我们先分析一下运行的结程：
```
任务1：被核心线程1执行
任务2：被核心线程2执行
任务3：此时已经没有核心线程了，所以被放到有界面队列里面，任务个数：1
任务4：此时已经没有核心线程了，所以被放到有界面队列里面，任务个数：2
任务5：此时已经没有核心线程了，所以被放到有界面队列里面，任务个数：3 (任务队列满)
任务6：核心线程都被占用了，队列也已经满了，所有创建新线程3来执行
任务7：核心线程都被占用了，队列也已经满了，所有创建新线程4来执行(达到最大线程数)
任务8：队列满了，也达到最大线程数了，被拒绝掉(执行任务拒绝策略)
任务9：队列满了，也达到最大线程数了，被拒绝掉(执行任务拒绝策略)
任务10：队列满了，也达到最大线程数了，被拒绝掉(执行任务拒绝策略)
核心线程1把任务1执行完毕，从任务队列中拿出任务3执行
核心线程2把任务2执行完毕，从任务队列中拿出任务4执行
线程3把任务6执行完毕，从任务队列中拿出任务5执行，
...
然后回收空闲线程
```
这个是它的大概过程，但是至于那个线程执行那个任务，这个就由线程抢到CPU资源来决定了。多次运行后，结果不一致。

大家看到我在这个案例中并没有使用**Executors**来创建线程池，为什么？这里有一个特别主要的原因就是，我们通过ThreadPoolExecutor的构造函数来创建一个线程池的时候，我对整个过程是非常清淅，但是如果采有Executors来创建的时候，可能这种感觉就会慢慢下降！**然后还有另外一个重要的依据是来自阿里巴巴的Java开发规范，我全力呼吁大家都遵守里面的条例，让我们的编码风格规范起来!**

![线程池使用规范](https://upload-images.jianshu.io/upload_images/1640787-fcd1b50d641e1828.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来我们就开始分析execute()方法了：
```
  public void execute(Runnable command) {
        //提交的任务command不能为null，不然会抛出NullPointerException
        if (command == null)
            throw new NullPointerException();
        /**
         * 紧接着会进行如下三个步骤：
         *
         * 1：如果当前运行的线程数小于 corePoolSize，则马上尝试使用command对象创建一个新线程。
         * 调用addWorker()方法进行原子性检查runState和workerCount,然后通过返回false来防止在不应该
         * 添加线程时添加了线程产生的错误警告。
         *
         * 2：如果一个任务能成功添加到任务队列，在我们添加一个新的线程时仍然需要进行双重检查
         * (因为自 上一次检查后，可能线程池中的其它线程全部都被回收了) 或者在进入此方法后，
         * 线程池已经 shutdown了。所以我们必须重新检查状态，如果有必要，就在线程池shutdown时采取
         * 回滚入队操作移除任务，如果线程池的工作线程数为0，就启动新的线程。
         * 
         * 3：如果任务不能入队，那么需要尝试添加一个新的线程，但如果这个操作失败了，那么我们知道线程 
         * 池可能已经shutdown了或者已经饱和了，从而拒绝任务。
         */ 
        
        //获取线程池控制状态
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {//如果工作线程数<核心线程数
            if (addWorker(command, true))//添加一个工作线程来运行任务，如果成功了，则直接返回
                //if 还有要有大括号比较好，至少让阅读的人看起来更清楚，这里我要小小地批评一下小Lea
                return;
            //如果添加线程失败了，就再次获取线程池控制状态
            c = ctl.get();
        }
        //如果线程池处理RUNNING状态，则尝试把任务添加到任务队列
        if (isRunning(c) && workQueue.offer(command)) {
            // // 再次检查，获取线程池控制状态
            int recheck = ctl.get();
            //如果线程池已经不是RUNNING状态了，把任务从队列中移除，并执行拒绝任务策略
            //「可能线程池已经被关闭了」
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //如果工作线程数为0，就添加一个新的工作线程
            //「因为旧线程可能已经被回收了，所以工作线程数可能为0」 
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //再次尝试创建一个新线程来执行任务，如果还是不行，并执行拒绝任务策略
        else if (!addWorker(command, false))
            reject(command);
}
```
我们可以从execute()方法看出ThreadPoolExecutor线程池的三个原则：

* 如果运行的线程少于 corePoolSize，则 Executor 始终首选添加新的线程，而不进行排队。
* 如果运行的线程等于或多于 corePoolSize，则 Executor 始终首选将请求加入队列，而不添加新的线程。
* 如果无法将请求加入队列，则创建新的线程，除非创建此线程超出 maximumPoolSize，在这种情况下，任务将被拒绝。

很明显要开始分析addWorker()方法了，在这个方法里面有一个语法，估计有小部分同学还没见过呢。
```
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {//一上来就是一个无限循环，不过外面有一个标签噢
            int c = ctl.get();//先获取线程池控制状态
            int rs = runStateOf(c);//获取线程池的状态

            //如果线程池的状态为RUNNING状态，直接跳过这个检查
            //如果线程池状态不为RUNNING，那么当满足
            // 状态为SHUTDOWN并且任务为null，任务队列不为空的时候也跳过这个检查
            //「也就是说：如果线程池的状态SHUTDOWN时,它不接收新任务,但是会继续运行任务队列中的任务」               
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))//worker队列不为空
                return false;//返回不能新增线程

            for (;;) {//如果线程池继续工作！
                int wc = workerCountOf(c);//获取工作线程的数量
                if (wc >= CAPACITY ||//如果工作线程的数量>=最大容量
                //据据core来控制工作线程数量是>=corePoolSize 还是 >=maximumPoolSize
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;//返回不能新增线程
                if (compareAndIncrementWorkerCount(c))//如果原子性增加工作线程数(+1)成功，跳出大循环
                    break retry;
                c = ctl.get();  // Re-read ctl//否则，再次获取线程池控制状态
                if (runStateOf(c) != rs)//如果线程池的状态发生改变了，再次重复大循环
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
        //工作线程开始的标记
        boolean workerStarted = false;
        //工作线程被添加的标记
        boolean workerAdded = false;
       //Worker包装了线程和任务
        Worker w = null;
        try {
            w = new Worker(firstTask);//初始化worker
            final Thread t = w.thread;//获取worker对应的线程
            if (t != null) {//如果线程不为null
                final ReentrantLock mainLock = this.mainLock;// 线程池锁
                mainLock.lock();//获取锁
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());//拿着锁重新检查池程池的状
                    //线程池的状态为RUNNING或者(线程池的状态SHUTDOWN并且提交的任务为null时)
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //如果线程已经运行了或者还没有死掉，抛出一个IllegalThreadStateException异常
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //把worker加入到工作线程Set里面
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)//如果工作线程池的大小大于largestPoolSize
                            largestPoolSize = s;//让largestPoolSize记录工作线程池的最大的大小
                        workerAdded = true;//工作线程被添加的标记置为true
                    }
                } finally {
                    mainLock.unlock();//释放锁
                }
                if (workerAdded) {//如果工作线程已经被添加到工作线程池了
                    t.start();//开始执行任务
                    workerStarted = true;//把工作线程开始的标记置为true
                }
            }
        } finally {
            if (! workerStarted)//如果没有添加，那么移除任务，并减少工作线程的数量(-1)
                addWorkerFailed(w);
        }
        return workerStarted;
}
```
我们先在总结一下，addWorker()方法都做了什么:
* **池程的状态为RUNNING**
* 1.采用原子性增加工作线程的数量(+1)
* 2.将提交的任务封装成一个worker,并将此worker添加到workers中
* 3.启动worker对应线程，然后开始执行提交的任务
* 4.如果没有把工作线程添加到工作线程池中，那么会移除任务，并原子性减少工作线程的数量(-1)
* **如果池程池的状态是SHUTDOWN**
* 它不接收新任务,但是会继续运行任务队列中的任务

真正执行任务的runWorker()方法：
```
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();//获取当前线程（和worker绑定的线程）
        Runnable task = w.firstTask;//用task保存在worker中的任务
        w.firstTask = null;//把worker中的任务置为null  
        w.unlock(); //释放锁
        boolean completedAbruptly = true;
        try {
            //这个while循环，保证了如果任务队列中还有任务就继续拿出来执行，注意这里的短路情况
            while (task != null || (task = getTask()) != null) {//如果任务不为空，或者任务队列中还有任务
                w.lock();//获取锁
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||//如果线程池的状态>STOP，直接中断
                     (Thread.interrupted() &&//调用者线程被中断
                      runStateAtLeast(ctl.get(), STOP))) &&//再次检查线程池的状态如果>STOP
                    !wt.isInterrupted())//当前线程还没有被中断
                    wt.interrupt();//中断当前线程
                try {
                    beforeExecute(wt, task);//在任务执行前的钩子方法
                    Throwable thrown = null;//用于记录运行任务时可能出现的异常
                    try {
                        //开始正式运行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        //任务执行完成后的钩子方法
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;//把任务置为null
                    w.completedTasks++;//把任务完成的数量+1
                    w.unlock();//释放锁
                }
            }
            completedAbruptly = false;
        } finally {
            //当所有任务完成之后的一个钩子方法
            processWorkerExit(w, completedAbruptly);
        }
}
```
runWorker()方法除了会运行我们提交的任务外，还会自动取出从任务队列中的任务，与此同时还提供了任务执行之前和任务执行之后的钩子方法，并且提供了所有任务都执行完的钩子方法。通过这些钩子方法，我们就可以做我们的一些处理操作了。

```
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {//又是一个无限循环
            int c = ctl.get();//先获取线程池的控制状态
            int rs = runStateOf(c);//获取线程池的当前状态

            // Check if queue empty only if necessary.
            //如果池程的状态>= STOP或者 任务队列为空
            //这里rs>=SHUTDOWN这个判断其实体现的是一种优化的策略
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();//交工作线程数原子性减少1
                return null;//返回null
            }

            int wc = workerCountOf(c);//获取工作线程的数量

            // Are workers subject to culling?
            // 是否允许核心线程超时或者当前工作线程数是否大于核心线程数
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
      
            //如果当前工作线程数>maximumPoolSize>（1或者任务队列为空）,那么减少工作线程数,并返回null
            //如果当前工作线程数>corePoolSize>（1或者任务队列为空）,且超过keepAliveTime还没有拿到任务，那么减少工作线程数,并返回null
           //如果允许核心线程超时，且超过keepAliveTime还没有拿到任务，且当前线程数（>1或者任务队列为空），那么减少工作线程数,并返回null
  
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) ://这里就是控制一个工作线程能闲置的时间
                    workQueue.take();//一直等下面，直到拿到任务
                if (r != null)//如果拿的任务不为null，直到返回任务
                    return r;
                timedOut = true;//等待指定的时间后，还没有获取到任务，则超时 
            } catch (InterruptedException retry) {
                timedOut = false;//如果被中断了，将超时置为false
            }
        }
}
```
小小地总结一下getTask()方法，首先他会不断地检查线程池的状态，如果线程池的状态为SHUTDOWN或者STOP时，它会直接返回null，同时他在特定情况下会减少工作线程数，也是返回null，比较重要的一点是，因为getTask()方法是从阻塞队列中获取任务的，所以他支持有限时间的等待poll()，和无限时间的等待take().

接下来就是我们的shutdown()方法了
```
public void shutdown() {
        //重入锁
        final ReentrantLock mainLock = this.mainLock;
        //获取锁
        mainLock.lock();
        try {
            //检查shutdown权限
            checkShutdownAccess();
            //设置线程池控制状态为SHUTDOWN
            advanceRunState(SHUTDOWN);
            //中断所有空闲的工作线程
            interruptIdleWorkers();
            //shutdown的钩子方法
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            //释放锁
            mainLock.unlock();
        }
        //尝试终止
        tryTerminate();
}
```
整个shutdown()方法的流程是很简单明了的,他会执行所有已经提交到线程池的任务,但是不会再接收新任务了.还有一个shutdownNow()方法,他和shutdown()方法是非常相似的,不过shutdownNow()会尝试停止所有的任务,暂停正在任务队列中等待的任务等等,如果有兴趣的同学就要自行去阅读他的源码了,这里就不带着大家读了.

接下来我们就分析我们的本篇文章的最后一个方法吧. tryTerminate()

```
final void tryTerminate() {
        for (;;) {//线程池里面用了大量的这种无限循环
            int c = ctl.get();//获取线程池的控制状态
            if (isRunning(c) ||// 线程池的运行状态为RUNNING
                runStateAtLeast(c, TIDYING) || // 线程池的运行状态最小要大于TIDYING
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))// 线程池的运行状态为SHUTDOWN并且workQueue队列不为null
                return;// 不能终止，直接返回
             // 线程池正在运行的worker数量不为0
            if (workerCountOf(c) != 0) { // Eligible to terminate
                  // 仅仅中断一个空闲的worker
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
            // 获取线程池的锁
            final ReentrantLock mainLock = this.mainLock;
             // 获取锁
            mainLock.lock();
            try {
                 // 比较并设置线程池控制状态为TIDYING
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        //终止线程池的钩子方法
                        terminated();
                    } finally {
                        // 设置线程池控制状态为TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                         // 释放在termination条件上等待的所有线程
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                 // 释放锁
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
}
```
最后和大家简单介绍一下ThreadPoolExecutor给我们提供的四种拒绝策略:

```
ThreadPoolExecutor.AbortPolicy:默认的拒绝策略，处理程序遭到拒绝将抛出运行时RejectedExecutionException
ThreadPoolExecutor.CallerRunsPolicy:它直接在 execute 方法的调用线程中运行被拒绝的任务；如果执行程序已关闭，则会丢弃该任务
ThreadPoolExecutor.DiscardPolicy:不能执行的任务将被删除。
ThreadPoolExecutor.DiscardOldestPolicy:如果执行程序尚未关闭，则位于工作队列头部的任务将被删除，然后重试执行程序（如果再次失败，则重复此过程）
```

到这里,线程池的核心代码,我们都已经阅读了一篇了,如果你现在头脑里面,有一幅ThreadPoolExecutor的流程图,那这一次你的阅读是非常成功的.线程池在多线程开发的环境下的作用是非常大的，只有深刻理解了线程池的基本原理，了解线程池细致的实现过程，你才能更好地使用它，再出现问题之后，也更容易排查。与此同时，将大大增加你的战斗力。

文章写得比较长了,有一万多字,能够坚持读完的同学,你们也是非常非常不得了了,至少你是一位非常有毅力的人,相信在你肯定能走得更远!当然,线程池里面的技术点,技术细节非常多，一篇文章难以全部讲完，有一些知识点只能分开来一个一个地讲，比如:ReentrantLock ，阻塞列队等等。最后建议大家要多读几遍，线程池的代码有很多细节写得都是有点绕的，很明显是经过了精心的优化的，如果有什么感想想法，欢迎你们留言，我们交流一下。有你赞扬，我们可以一起走得更远！欢迎大家关注我的公众号，那样有什么好文你就可以第一时间收到啦，谢谢你的阅读。

![扫一扫加入公众号,第一次时间获取新知识](https://upload-images.jianshu.io/upload_images/1640787-d25d0d3ec6ea54a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
