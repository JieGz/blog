---
title: 你是如何拿到一个线程的执行结果的？Future体系源码深度解析
date: 2019-06-11 17:18:37
categories: 
- Java
tags:
- Java多线程
- Future
- 源码解析
---

在Java多线程开发中，我们经常把Thread#run()方法称为线程的执行单元，执行单元通常就是编写我们的业务逻辑。我们可以通过继承Thread然后重写run方法实现自己的业务逻辑，也可以实现Runnable接口实现自己的业务逻辑。而Runnable接口的职责主要是想把线程控制本身和业务逻辑分离开来，但在其它的很多文章中，经常可以看到这样一句话，创建线程的方式有两种，第一种是构造一个Thread,第二种是实现Runnable接口，其实这种说法，我认为是错误，至少他是特别不严谨的。为什么我这么说呢？我们看一下Thread类中的run()方法。

<!-- more -->
```
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}

public class Thread implements Runnable {
    private Runnable target;
    
     @Override
    public void run() {
        //如果构造Thread的时候传递了Runnable，那么将调用Runnable的run()方法，
        if (target != null) {
            target.run();
        }
       //否则就需要我们重写Thread类run()方法了
    }
}
```
通过上面这两行注释可以清晰地看到，执行的单元的实现方式是有两种的。而不是说创建线程的方式有两种，准确地讲，创建线程的方式只有一种那就是构造Thread类，而实现线程的执行单元的方式有两种，第一种是重写Thread的run方法，第二种是实现Runnable接口的run方法，并且将Runnable实例用作构造Thread的参数。

现在我们反过来重新观察一下，不管我们是采用重写Thread的run，还是实现Runnable接口的run方法，都无法拿到线程的执行结果，为了解决这个问题，我们常用的一种方式就是使用一个共享变量，间接地返回线程执行的结果。但在JDK1.5之后，在JUC包中提供了Future接口，而在JDK1.8中更是提供了CompletableFuture。

但本文我们主要讲的是Future体系。**因为Future体系作为线程池的重要载体，所以我们要想理解好线程池，就非常有必要先了解Future体系。**

## 深入理解Future体系

**Future体系UML**
![Future体系](https://upload-images.jianshu.io/upload_images/1640787-68dbe48eb1412ee8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

FutureTask类实现了RunnableFuture接口，而RunnableFuture继承了Runnable和Future，也就是说FutureTask既是Runnable，也是Future。因此FuntureTask可以直接作为Thread的构造参数直接使用了。

### Callable接口
```
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```
Callable接口类似于Runnable，都是为了成为其它线程的执行单元而被设计出来的，但与Runnable不同的是，Callable接口不仅拥有返回值，还会抛出异常。


### Future接口

```
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);

    boolean isCancelled();

    boolean isDone();

    V get() throws InterruptedException, ExecutionException;

    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
Future表示的是异步计算的结果，在Future接口中，提供了一些用于检查任务执行是否完成，等待任务执行完成和取出任务执行结果的方法。
* 当运算完成后只能通过get()方法进行检索，并且调用了get()方法后出阻塞当前线程直到任务执行完成。
* 通过cancel()方法，可以取消任务。
* 通过isCancelled()方法，可以判断任务是否被取消了
* 通过isDone()方法，可以判断任务是否已经完成了

### RunnableFuture接口
```
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```
RunnableFuture 继承自Runnable和Future，即提供了可以使用Runnable来执行任务，又可以使用Future执行获取结果的功能，同时还拥有了取消任务，判断任务状态的功能。

### FutureTask
```
public class FutureTask<V> implements RunnableFuture<V> {
    /**
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
}
```
一个异步可取消计算，FutureTask提供了Future接口的基本实现，其中包含开始执行任务和结束任务的方法，查询任务执行是否完成的方法，或者获取任务结果的方法，等等。仅当任务执行完成了，才能获取到结果。
并且调用get()方法会阻塞当前线程直到任务执行完成，如果任务已经完成了，不能重新开始或者取消，除非这个任务调用了runAndReset()方法。

FutureTask可以包装一个Callable或者是Runnable,因为FutureTask实现了Runnable对象（Callable接口类似于Runnable,Callable相对于Runnable来说，仅仅多了一个返回值和Exception抛出而已），我们可以把一个FutureTask提交给线程池的Executor来执行。FutureTask，除了作为一个单独的类之外，它的protected 方法在我们自定义Task的时候是非常有用的。

看到了这里，大家有没有思考过这样的一个问题呢？一个正在执行的任务，他是怎么判断已经取消了的，又是怎么判断执行的任务是否已经完成了呢，等等Future接口提供的功能。如果是你，你会怎么做呢？不访花上几分钟先思考一下。

从上面的FutureTask的一些成员变量或者你已经看出了端倪.但再详细分析之前，我们先看看FutureTask类怎么使用吧.

![FutureTask UML](https://upload-images.jianshu.io/upload_images/1640787-5004891c90dc60a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过UML，我们可以看到，有两个构造函数，所以说FutureTask可以包装一个Callable或者是Runnable。

```
 public static void main(String[] args) throws Exception {
        Callable<String> callable = new Callable<String>() {
            @Override
            public String call() throws Exception {
                System.out.println("正在下载中...");
                TimeUnit.SECONDS.sleep(3);
                return "Hello World!";
            }
        };
        FutureTask<String> futureTask = new FutureTask<>(callable);
        new Thread(futureTask).start();
        
        System.out.println("我们做的其他什么吧...");
        System.out.println("从网络下载的结果为:" + futureTask.get());
        System.out.println("Finish!");


     Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println("正在下载中...");
                try {
                    TimeUnit.SECONDS.sleep(3);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        FutureTask<String> runnableTask = new FutureTask<>(runnable, "我是返回的结果");
        new Thread(runnableTask).start();
        System.out.println("我们做的其他什么吧...");
        System.out.println("从网络下载的结果为:" + runnableTask.get());
        System.out.println("Finish!");
 }
//输出
我们做的其他什么吧...
正在下载中...(然后等待3秒，继续输出)
从网络下载的结果为:Hello World!
Finish!
我们做的其他什么吧...
正在下载中...(然后等待3秒，继续输出)
从网络下载的结果为:我是返回的结果
Finish!
```
使用FutureTask包装Runnable和Callable,通过Future体系，我们就可以拿到异步任务的执行结果了,看完例子，你应该已经知道FutureTask怎么使用了，但只有这个级别怎么能满足，做人要有点追求，不然和八戒有什么区别。

### FutureTask源码分析

```
public class FutureTask<V> implements RunnableFuture<V> {
    /** 任务可能出现的状态转换
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;    //任务当前状态
    private static final int NEW          = 0;    //新建一个任务时的状态
    private static final int COMPLETING   = 1;    //任务即将执行完成的状态
    private static final int NORMAL       = 2;    //任务正常执行结束的状态
    private static final int EXCEPTIONAL  = 3;    //任务异常时的状态
    private static final int CANCELLED    = 4;    //任务被取消的状态
    private static final int INTERRUPTING = 5;    //任务即将被中断的状态
    private static final int INTERRUPTED  = 6;    //任务已经被中断的状态
}
```
其中可以把一个FutureTask的这7种状态分为三种：
* 第一种：初始化状态：NEW
* 第二种：中间状态：COMPLETING，INTERRUPTING
* 第三种：终端状态：NORMAL，EXCEPTIONAL，CANCELLED，INTERRUPTED

初始化一个FutureTask的时候，state为初始化值NEW，当仅且当：调用了set()，setException()和cancel()方法，才会将状态转换成终端状态。并且中间状态继续的时间是比较短暂的。

```
//这是一个内部的callable对象，我们通过构造函数传入的callable对象将会保存在这里
//当任务执行完成后，callable会被置为null
private Callable<V> callable;
//保存任务执行的结果或者是get()方法抛出的异常，通过state来实现同步的
private Object outcome; 
//执行callable任务的线程，它是CAS操作。
private volatile Thread runner;
//等待线程的Treiber栈，Treiber是一种算法，Treiber栈是一种无阻塞栈。
private volatile WaitNode waiters;

public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
}

public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
}
```
创建FutureTask的时候，会把runnable或者callable保存到 callable成员变量里边，同时会把state置为NEW

```
 // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long stateOffset;      //任务状态的偏移量
    private static final long runnerOffset;   //runner线程的偏移量   
    private static final long waitersOffset;  //Treiber栈的偏移量   
  //有了这些偏移量后，UNSAFE就能得到他们对应的值了
    static {
        try {
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

//任务起动就调这个方法。
public void run() {
        //compareAndSwapObject方法有四个参建：第一个是某个对象，第二个是相对偏移量，有了这个偏移量 
        //就可以知道，内存块对应变量X的值了，第三个是：预期值，第四个是:更新值,如果X ＝＝ 预期值，那 
        //么就将X的值更新为更新值，并返回true，或者返回fasle.「是一个CAS操作」
        
        //如果state == NEW,则runner = Thread.currentThread() ,返回true，取反之后，进入后面的计算。

        //如果state不是NEW的情况，说明任务已经被执行了，直接返回  
       //避免任务重新执行
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                //这一块的代码，建议大家反复读多几次
                //这一个ran变量用得真的是精彩，他主要的为了：他主要是不捕捉set()方法的异常
                //如果这里我们直接在  result = c.call();后面直接调set();那么最终的done()方法很可能出现异常
                //就会导致 setException()调用了，从而生命周期变成了
                //NEW -> COMPLETING -> NORMAL-> EXCEPTIONAL
                boolean ran;
                try {
                    result = c.call();//如果任务执行的时间比较少，那么在这里就体现出来了
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
         
            runner = null;
            //在任务执行的过程中，可能会调用cancel()
            //这里主要是不想让中断操作逃逸到run()方法之外
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
}

//更改当前任务的状态并把任务执行的结果写入到outcome当中，最后由get()取出来用
protected void set(V v) {
        //将当前任务状态置为：COMPLETING
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            //将当前任务状态置为：NORMAL
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
}

//完成任务后的收尾操作
private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
             //  将Treiber栈的栈顶置为null，
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                //遍历Treiber栈并唤醒所有节点的线程
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;//把当前节点置为无效节点
                        LockSupport.unpark(t);//这里唤醒的是awaitDone()阻塞的线程
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
        //一个钩子方法，本类中，它是一个空实现，在子类中可以重写它。
        done();
        //最后把callable置为null.
        callable = null;        // to reduce footprint
    }
//修改当前任务的状态
protected void setException(Throwable t) {
        //将当前任务状态置为：COMPLETING
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            //将当前任务状态置为：EXCEPTIONAL
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
}

//取消任务。如果是false  NEW->CANCELLED     true: NEW ->INTERRUPTING->INTERRUPTED
public boolean cancel(boolean mayInterruptIfRunning) {
         //如果当前任务是新建任务，则将其置为INTERRUPTING或者CANCELLED
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    //使用了线程中断的方法来达到取消任务的目的
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    //如果当前任务不是新建任务，则将其状态置为INTERRUPTING
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
}
```
通过前面的这些分析，相应FutureTask的整个生命周期应该特别清淅了

![FutureTask生命周期](https://upload-images.jianshu.io/upload_images/1640787-bc9de7f15268ed93.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/825)

```
    //任务是滞被取消了。包括cancel(false);和cancel(true);
    public boolean isCancelled() {
        return state >= CANCELLED;
    }
    
    //任务是否结束，不一定是成功任务，取消了，出异常了，被中断了，也是结束
    public boolean isDone() {
        return state != NEW;
    }
```
这两个方法比较简单，我们就多说什么，接下来看FutureTask的核心， get()的过程
```
//获取任务执行的结果
 public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)//如果任务还没执行完成，就等待任务先执行完成
            s = awaitDone(false, 0L);
        return report(s);
}
//获取任务执行的结果，并且有超时机制
public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        //带有超时时间的get()，如果超过指定时间，就会抛出一个TimeoutException
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
}
 //等待任务完成
 private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        //超时时间，如果使用了超时的get()才起作用，否则这个值不起作用
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {//如果线程被中断了
                removeWaiter(q);//移除无效节点
                throw new InterruptedException();//就抛出一个中断异常
            }

            int s = state;//把任务的当前状态保存到s变量里
            if (s > COMPLETING) {//如果当前状态大于COMPLETING，说明任务已经结果了
                if (q != null)//如果q不为空，就说明已经被初始化了
                    q.thread = null;//回收q.thread
                return s;//返回任务的状态
            }
           // 如果当前的任务状态为COMPLETING，因为他的停留的时间非常短，通过yield()尝试把时间片
          //交给其他线程处理，然后重试
            else if (s == COMPLETING)
                Thread.yield();
          //初始化q节点，然后重试
            else if (q == null)
                q = new WaitNode();
            //将q节点压入栈，它是一个cas操作
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            //如果有超时限制的话，判断是否超时，如果没有超时就重试，如果超时了，就把q节点从栈中遇除
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {//超时了
                    removeWaiter(q);//移除无效节点
                    return state;
                }
                LockSupport.parkNanos(this, nanos);//LockSupport是一个并发工具，这里表示等待nanos秒后唤醒
            }
            else
                LockSupport.park(this);//开始阻塞线程，直到任务完成了才会再次唤醒了在finishCompletion()中唤醒
        }
}
/**
 *  将某个节点置为无效节点，并清除栈中所有的无效节点
 * （通过前面的分析，应该可以推断出，无效的节点，其实就是指节点内部的thread ＝＝ null） 
 *  那么产生无效节点的情况就有三种了
 *  (1):线程被中断了
 *  (2):s > COMPLETING，当前的任务状态> COMPLETING
 *  (3):超时
 */
private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;//为了GC回收node节点中的thread成员变量
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;//将q节点的下一个节点，保存到s
                    if (q.thread != null)//如果当前节点q为有效节点，则前pred节点置为当前节点，继续遍历
                        pred = q;
                    //如果当前节点q为无效节点，并且有前驱节点
                    else if (pred != null) {
                        //删除当前节点
                        pred.next = s;
                        //如果前驱节点也是一个无效节点，则重新遍历，否则就代表清理完成了
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    //如果当前节点q是无效节点并且没有前驱节点（也就是栈顶节点），则将栈顶置为当前节点q的    后继节点，再遍历
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
}

//取出执行结果
 private V report(int s) throws ExecutionException {
        Object x = outcome;//我们在set()方法的时候把结果写进outcome的
        if (s == NORMAL)//如果是正常直接返回任务执行的结果
            return (V)x;
        if (s >= CANCELLED)//如果是被取消了，或者是被中断了就返回一个CancellationException
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);//或者返回一个ExecutionException
}
```
至此，整个FutureTask的源码已经分析完了，最后总结一下关于Future体系的一些重要点，第一，关于保存任务执行的结果，Future体系主要是利用了Callable接口的call函数，如果你想在任务执行结束后返回一些你预期的值，就可以使用**public FutureTask(Runnable runnable, V result){}**，这里的result就是你的预期值。第二个就是FutureTask的生命周期了，通过前面的分析，相信生命周期的流程认真阅读的你已经理解记忆了。第三点就是，获取任务执行的结果了，这一块其实是贯穿全文，可能需要你多读几遍源码，但其实思路也是比较简单的，①如果在获取值是，任务已经完成了，则直接就返回结果②如果在获取任务执行的结果时，任务还没有完成，则开始阻塞，直到任务任务完成。其中还涉汲到一个栈对线程的管理，如果在阻塞其间，任务被中断了，或者超时了，又或者任务已经完成了，都需要进行资源的回收。

文章到这里，相信如何拿到一个线程的执行结果？这个题目，你自己已经有答案了，但其实这个只是开始，有这个知道，相信在阅读线程池的源码时，你将更加的如鱼得水。当然，紧接着，我也会推送JDK线程池的源码解析。好了，谢谢您的阅读，下期再见。
