# 1.线程池基本概念


线程池（Thread Pool）`是一种管理和复用线程的技术，用于优化线程的创建、调度和销毁过程`。线程池解决了每次需要执行任务时都需要创建新线程的问题，而是在池中预先创建足够数量的线程，这些线程等待执行任务的指令，当有新的任务需要执行时，从线程池中取出一个空闲的线程执行任务。当任务执行完毕之后，线程不会被销毁，而是重新放回池中，等待下一次任务的到来。

线程池中通常会包含一个任务队列，用于存储等待执行的任务。当任务队列已满，不能继续添加任务时，可以采用拒绝策略，具体实现根据需要而定，可以抛出异常、直接丢弃任务或者将任务放入任务队列中等待执行。

线程池的主要优势在于：

1. **降低了资源消耗**：复用线程，避免了线程创建和销毁的成本，降低了资源消耗。
2. **提高线程可管理性**：统一任务调度，避免了线程之间的竞争，提高了系统整体的运行效率和稳定性。
3. **提高响应速度**：当任务到达时，任务可以不需要等待线程创建就能立即执行。



# 2.线程池实现原理

当向线程池提交一个任务之后，线程池时如何管理这个任务？

- 线程池判断核心线程池中的线程是否都在执行任务，如果不是，则创建一个新的工作线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下一个流程。
- 线程池判断工作队列是否已满，如果工作队列没有满，则将新提交的任务存储在工作队列中。反之，进入下一个流程
- 线程池判断线程池中的线程是否都处于工作状态。如果没有，则创建一个新的任务线程来执行任务。反之，交给饱和策略来处理这个任务。

![image-20230502223313164](/pic/image-20230502223313164.png)



以ThreadPoolExecutor为例，流程图如下：

![image-20230502230026273](/pic/image-20230502230026273.png)

# 3.线程池顶层接口Executor

> Executor接口是线程池框架中最基础的部分，位于**package java.util.concurrent 包下**定义了一个用于执行Runnable的execute方 法。 下图为它的继承与实现  

```java
//线程池的顶层父类，抽象了执行线程方法
public interface Executor {

    //execute（Runnable command）：履行Ruannable类型的任务  
    void execute(Runnable command);
}
```

![image.png](/pic/1634568370973-75f1518e-f1ed-4773-bd89-e0512503a9ef.png)

# 4.ExecutorService

> 在线程池顶层父类下面抽象了线程的各种状态与执行方法。所有的线程池都实现了这层抽象

```java
//Executor子类，抽象了任务的状态以及执行方法
public interface ExecutorService extends Executor {

    //在完成已提交的任务后封闭办事，不再接管新任务
    void shutdown();

    //停止所有正在履行的任务并封闭办事
    List<Runnable> shutdownNow();

    //测试是否该ExecutorService已被关闭。
    boolean isShutdown();
 
    //测试是否所有任务都履行完毕了
    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit);

    //可用来提交Callable,并返回代表此任务的Future对象
    <T> Future<T> submit(Callable<T> task);

    //可用来提交Runnable任务，并返回代表此任务的Future对象
    <T> Future<T> submit(Runnable task, T result);

    //可用来提交Runnable任务，并返回代表此任务的Future对象
    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,long timeout, TimeUnit unit) ;


    <T> T invokeAny(Collection<? extends Callable<T>> tasks);

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,long timeout, TimeUnit unit);
}
```



# 5.ThreadPoolExecutor

> 线程池的具体实现，默认线程池。可以通过它创建线池

## 构造

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler){}
 
```

## 构造参数注释

**corePoolSize**

线程池中的核心线程数，当提交一个任务时，线程池创建一个新线程执行任务，直到当前线程数等于corePoolSize；如果当前线程数为corePoolSize，继续提交的任务被保存到阻塞队列中，等待被执行；如果执行了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有核心线程。

**maximumPoolSize**

线程池中允许的最大线程数。如果当前阻塞队列满了，且继续提交任务，则创建新的线程执行任务，前提是当前线程数小于maximumPoolSize；

**keepAliveTime**

线程池维护线程所允许的空闲时间。当线程池中的线程数量大于corePoolSize的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了keepAliveTime；

**unit**

keepAliveTime的单位；

```java
TimeUnit.DAYS          //天
TimeUnit.HOURS         //小时
TimeUnit.MINUTES       //分钟
TimeUnit.SECONDS       //秒
TimeUnit.MILLISECONDS  //毫秒
```

**workQueue**

用来保存等待被执行的任务的阻塞队列，且任务必须实现Runable接口，在JDK中提供了如下阻塞队列：

- 1、ArrayBlockingQueue：基于数组结构的有界阻塞队列，按FIFO排序任务；
- 2、LinkedBlockingQuene：基于链表结构的阻塞队列，按FIFO排序任务，吞吐量通常要高于ArrayBlockingQuene；
- 3、SynchronousQuene：一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQuene；
- 4、priorityBlockingQuene：具有优先级的无界阻塞队列；

**threadFactory**

它是ThreadFactory类型的变量，用来创建新线程。默认使用Executors.defaultThreadFactory() 来创建线程。使用默认的ThreadFactory来创建线程时，会使新创建的线程具有相同的NORM_PRIORITY优先级并且是非守护线程，同时也设置了线程的名称。

**handler**

线程池的饱和策略，当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了4种策略：

- 1、AbortPolicy：直接抛出异常，默认策略；
- 2、CallerRunsPolicy：用调用者所在的线程来执行任务；
- 3、DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
- 4、DiscardPolicy：直接丢弃任务；

上面的4种策略都是ThreadPoolExecutor的内部类。

当然也可以根据应用场景实现RejectedExecutionHandler接口，自定义饱和策略，如记录日志或持久化存储不能处理的任务。

## 方法

```java
public void execute() //提交任务无返回值
public Future<?> submit() //任务执行完成后有返回值
public void shutdown() //关闭线程池,将线程池的状态设置为shutdown状态，然后中断所有没有正在执行的任务的线程，通常使用此方式
public void shutdownNow()//关闭线程池,将线程池的状态设置为stop状态，然后尝试停止所有正在执行或暂停的任务线程，如果任务不需要执行完就结束，可以使用shutdownNow
```

![image-20230503101147938](/pic/image-20230503101147938.png)

## 使用案例

```java
package com.itmck.threadpool;

import java.util.concurrent.*;

public class ThreadPoolTest {

    private static final int maximumPoolSize = 5;
    private static final int corePoolSize = 2;

    public static void main(String[] args) {

        int cpuNum = Runtime.getRuntime().availableProcessors();
        System.out.println("当前系统cpu个数："+cpuNum);

        /**
         * 创建一个线程池
         * corePoolSize ：核心线程池大小
         * maximumPoolSize： 线程池最大数量
         * keepAliveTime：存活时间
         * unit ： 单位
         * workQueue： 阻塞队列
         * handler：处理方式 AbortPolicy：直接抛出异常  CallerRunsPolicy：只用调用者所在线程来运行任务  DiscardPolicy：不处理丢弃  DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务
         *
         */
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, 20, TimeUnit.SECONDS, new LinkedBlockingDeque<>(), new RejectedExecutionHandler() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {

            }
        });

        //execute执行无返回值的任务
        threadPoolExecutor.execute(() -> System.out.println("hello java"));


        try {
            //submit执行带有返回值的任务
            String message = threadPoolExecutor.submit(new Callable<String>() {
                @Override
                public String call() throws Exception {
                    return "hello";
                }
            }).get();
            System.out.println("------>" + message);
        } catch (InterruptedException e) {
            //处理中断异常
            throw new RuntimeException(e);
        } catch (ExecutionException e) {
            //处理无法执行的任务异常
            throw new RuntimeException(e);
        }


        //关闭线程池,将线程池的状态设置为shutdown状态，然后中断所有没有正在执行的任务的线程，通常使用此方式
        threadPoolExecutor.shutdown();

        //关闭线程池,将线程池的状态设置为stop状态，然后尝试停止所有正在执行或暂停的任务线程，如果任务不需要执行完就结束，可以使用shutdownNow
        //threadPoolExecutor.shutdownNow();
    }
}

```



#  6.ScheduledThreadPoolExecutor

> public class ScheduledThreadPoolExecutor extends ThreadPoolExecutor implements ScheduledExecutorService 
>
> 从继承关系来看ScheduledThreadPoolExecutor继承了ThreadPoolExecutor拥有其所有的方法

ScheduledThreadPoolExecutor是一个线程池，继承自ThreadPoolExecutor，可以在指定时间后，周期性地执行任务。它的主要特点有：

1. 定时执行任务：可以通过schedule()、scheduleAtFixedRate()和scheduleWithFixedDelay()方法指定任务的执行时间。

2. 指定初始线程数：可以通过ThreadPoolExecutor的构造函数指定初始的线程数，以及最大线程数和线程空闲时间等参数。

3. 线程复用：该线程池中的线程可以在执行完任务后被复用，而不是立即销毁与新建。

4. 任务队列：可以通过指定任务队列的容量、类型等参数来控制任务队列的行为。

使用ScheduledThreadPoolExecutor的步骤一般包括：创建线程池对象、创建任务对象、调用schedule()、scheduleAtFixedRate()或scheduleWithFixedDelay()方法进行任务的周期性执行。在执行过程中，可以通过调用shutdown()或shutdownNow()方法来停止线程池中的任务。

## 使用案例

```java
package com.itmck.threadpool;

import java.util.concurrent.*;

public class ScheduledThreadPoolExecutorTest {


    private static final int corePoolSize = 2;

    public static void main(String[] args) {

        Executors.newFixedThreadPool(corePoolSize);

				//方式1，自定义线程和丢弃策略
        ScheduledThreadPoolExecutor scheduledThreadPoolExecutor = new ScheduledThreadPoolExecutor(corePoolSize, new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                //这里可以自定义线程
                return new Thread(r);
            }
        }, new RejectedExecutionHandler() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                //这里自定义拒绝策略
                executor.execute(r);
            }
        });


				//使用默认
        ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(corePoolSize);
        long beginTime = System.currentTimeMillis();
        System.out.println("执行定时线程开始，当前时间：" + beginTime);
        executor.schedule(() -> {
            System.out.println("延迟时间：" + (System.currentTimeMillis() - beginTime) / 1000+"s");
            System.out.println("hello ScheduledThreadPoolExecutor");
        }, 2, TimeUnit.SECONDS);
        executor.shutdown();
    }
}
    
```

# 7.Executors

> Executors类主要用于提供线程池相关的操作.提供了一系列工厂方法用于创建线程池。

Executors是Java中Executor框架中的一个工具类，提供了一些静态工厂方法，可以方便地创建常用的线程池实例，包括：

1. newFixedThreadPool(int nThreads)：创建一个核心线程数和最大线程数均为nThreads的线程池，使用LinkedBlockingQueue作为任务队列。

2. newCachedThreadPool()：创建一个具有自动回收空闲线程的线程池，核心线程数为0，最大线程数为Integer.MAX_VALUE，使用SynchronousQueue作为任务队列。

3. newSingleThreadExecutor()：创建一个只有一个核心线程的线程池，使用LinkedBlockingQueue作为任务队列，可以完成对一组任务的顺序执行。

4. newScheduledThreadPool(int corePoolSize)：创建一个可以执行定时任务或周期性任务的线程池，最高可达Integer.MAX_VALUE，初始化核心线程数为corePoolSize，使用DelayedWorkQueue作为任务队列。

Executors工具类可以大大简化线程池的创建和使用，提高效率和减少代码量。但是需要注意使用线程池时需要根据实际情况合理选择不同类型的线程池，避免由于线程池的太多或太少导致程序性能的下降。

**常用静态方法:**

**如下所示,**Executors提供的静态方法,其中重点关注如下.

![image.png](/pic/1647395098052-5c657c1e-2d16-47b7-9652-63b947b073ff.png)

## newCachedThreadPool()

> newCachedThreadPool是Java中Executor框架中的一个线程池工具类，在没有足够工作线程时会创建新的线程，并在60秒钟没有被使用时将其回收。

该线程池的主要特点有：

1. 动态线程数：线程池中的线程数量是根据需要动态调整的，如果当前有可用的空闲线程，则直接复用，否则会创建新的线程，可灵活适应任务数量的变化。

2. 任务队列：使用SynchronousQueue作为任务队列，该队列不会缓存任务，任务提交后会等待线程池中的线程立即处理。

3. 线程复用：线程池维护一定数量的空闲线程，可以<u>重复利用这些空闲线程</u>处理任务，从而避免了频繁地创建和销毁线程的开销，提高了程序的效率。

使用newCachedThreadPool需要注意以下几点：

1. 适合处理大量短时间的任务，但需要注意设置最大线程数，以避免线程数膨胀。

2. 不适合处理长时间的任务，因为使用过程中可能会造成线程数无限制的增长，导致系统崩溃。

3. 任务队列的容量是0，任务提交后需要等待线程池中的线程直接处理，如果没有可用的线程，则创建新线程来处理任务。如果任务数量过多，会导致线程数膨胀，从而影响系统性能。

## newFixedThreadPool(int)

> newFixedThreadPool是Java中Executor框架中的一个线程池工具类，在初始化时会创建指定数量的线程，并将它们放入一个线程池中，对于重复利用的线程进行管理和监控，从而比直接创建线程来说更加高效。

该线程池的主要特点有：

1. 固定线程数：在创建线程池时需要指定线程数量，线程数量固定不变。

2. 线程复用：重复利用空闲线程，不必每次提交任务后重新创建线程，从而提高效率。

3. 任务队列：使用无界队列（LinkedBlockingQueue实现）作为任务队列，等待线程池中的线程来处理。

使用newFixedThreadPool需要注意以下几点：

1. 线程数应该根据实际情况来设定，过多的线程会浪费资源，过少的线程会影响任务执行效率。
2. 在任务数超过线程池的数量时，任务会进入任务队列等待执行，这时需要考虑任务队列的容量问题。
3. 在使用完线程池后，应该使用shutdown()方法来关闭线程池。

## newScheduledThreadPool(int)

> newScheduledThreadPool是Java中Executor框架中的一个线程池工具类，它具有定时或周期性执行任务的功能，可以通过schedule、scheduleAtFixedRate、scheduleWithFixedDelay等方法实现。

该线程池的主要特点有：

1. 定时执行任务：通过指定时间单位，可以实现定时执行任务的功能。

2. 周期性执行任务：可以通过支持.scheduleAtFixedRate和.scheduleWithFixedDelay()方法来实现周期性执行任务的功能。

3. 线程池中线程数量的动态调整：在任务量较大的情况下，可以通过指定核心线程数、最大线程数和线程池中线程关闭策略等参数灵活调整线程数量。

4. 允许重用空闲线程：利用已经存在的线程，避免了频繁地创建和销毁线程的开销，提高了系统的性能。

使用newScheduledThreadPool需要注意以下几点：

1. 在任务数量较大的情况下，为了避免线程池中的线程数不断增加到极限，需要适时地调整线程池中的线程数。

2. 被执行的任务需要考虑异常处理等异常情况，避免因为未处理异常导致线程池中的线程被卡住。

## SingleThreadExecutor()

> SingleThreadExecutor是Java中Executor框架中的一个线程池工具类，它只创建一个核心线程，该线程可重复使用。适用于顺序执行一系列任务的场景

该线程池的主要特点有：

1. 单线程执行：该线程池只创建了一个核心线程，按顺序执行任务，实现了线程安全。

2. 队列：任务使用一个基于链表的阻塞队列来保存，可以根据需要来处理队列任务。

3. 线程池中线程关闭策略：线程池中只有一个核心线程，当该线程结束时，会将相应的工作线程关闭，并回收资源。

4. 可以保证任务的顺序性：由于只有一个线程，可以避免多线程造成的任务顺序性问题，保证任务的有序执行。

使用SingleThreadExecutor需要注意以下几点：

1. 单线程执行可能会影响程序的效率和响应时间，不适合大量的高性能的需要并发处理的场景。

2. 由于只有一个线程，因此任务的执行顺序对程序的执行效率可能会产生重大影响。



# 8.线程池在xxljob中的使用

```java
package com.xxl.job.admin.core.thread;

import com.xxl.job.admin.core.conf.XxlJobAdminConfig;
import com.xxl.job.admin.core.trigger.TriggerTypeEnum;
import com.xxl.job.admin.core.trigger.XxlJobTrigger;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * job trigger thread pool helper
 *
 * @author xuxueli 2018-07-03 21:08:07
 */
public class JobTriggerPoolHelper {
    private static final Logger logger = LoggerFactory.getLogger(JobTriggerPoolHelper.class);


    // ---------------------- trigger pool ----------------------

    // fast/slow thread pool
    private ThreadPoolExecutor fastTriggerPool = null;
    private ThreadPoolExecutor slowTriggerPool = null;

    public void start(){
        fastTriggerPool = new ThreadPoolExecutor(
                10,
                XxlJobAdminConfig.getAdminConfig().getTriggerPoolFastMax(),
                60L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<Runnable>(1000),
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        return new Thread(r, "xxl-job, admin JobTriggerPoolHelper-fastTriggerPool-" + r.hashCode());
                    }
                });

        slowTriggerPool = new ThreadPoolExecutor(
                10,
                XxlJobAdminConfig.getAdminConfig().getTriggerPoolSlowMax(),
                60L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<Runnable>(2000),
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        return new Thread(r, "xxl-job, admin JobTriggerPoolHelper-slowTriggerPool-" + r.hashCode());
                    }
                });
    }

}

```

# 9.如何合理使用线程池

合理使用线程池才能更大限度的使用提高资源利用率。可以从如下角度进行分析：

- 任务的性质：CPU密集型，IO密集型和混合型任务
- 任务的优先级：高，中，低
- 任务的执行时长：长，中，短
- 任务的依赖性：是否依赖其他系统资源，如数据库连接

性质不同的任务可以用不同规模的线程池分开处理。

- CPU密集型任务应配置尽可能小的线程，如配置N（cpu）+1个线程池。
- IO密集型任务并不是一直在执行任务，则应配置尽可能多的线程，如2*N（cpu）。
- 混合型：可以将其拆分成一个cpu密集型，一个IO密集型，只要这两个任务执行时间不是差距太大，分解后执行的吞吐量高于串行吞吐量。如果执行时长相差很大，则没必要拆分。

int cpuNum = Runtime.getRuntime().availableProcessors();  获取当前系统的cpu个数

优先级不同的的任务，可以使用优先级队列PriorityBlockingQueue来处理，他可以让优先级高的任务先执行。（如果一直有优先级高的任务提交到队列，那么优先级低的将永远不会被执行）

扩展：

**CPU密集型（CPU-bound）**

CPU密集型也叫计算密集型，指的是系统的硬盘、内存性能相对CPU要好很多，此时，系统运作大部分的状况是CPU Loading 100%，CPU要读/写I/O(硬盘/内存)，I/O在很短的时间就可以完成，而CPU还有许多运算要处理，CPU Loading很高。

在多重程序系统中，大部份时间用来做计算、逻辑判断等CPU动作的程序称之CPU bound。例如一个计算圆周率至小数点一千位以下的程序，在执行的过程当中绝大部份时间用在三角函数和开根号的计算，便是属于CPU bound的程序。

CPU bound的程序一般而言CPU占用率相当高。这可能是因为任务本身不太需要访问I/O设备，也可能是因为程序是多线程实现因此屏蔽掉了等待I/O的时间。

线程数一般设置为：

线程数 = CPU核数+1 (现代CPU支持超线程)

**IO密集型（I/O bound）**

IO密集型指的是系统的CPU性能相对硬盘、内存要好很多，此时，系统运作，大部分的状况是CPU在等I/O (硬盘/内存) 的读/写操作，此时CPU Loading并不高。

I/O bound的程序一般在达到性能极限时，CPU占用率仍然较低。这可能是因为任务本身需要大量I/O操作，而pipeline做得不是很好，没有充分利用处理器能力。

线程数一般设置为：

线程数 = （（线程等待时间+线程CPU时间）/线程CPU时间 ）* CPU数目 

**CPU密集型 vs IO密集型**

我们可以把任务分为计算密集型和IO密集型。

计算密集型任务的特点是要进行大量的计算，消耗CPU资源，比如计算圆周率、对视频进行高清解码等等，全靠CPU的运算能力。这种计算密集型任务虽然也可以用多任务完成，但是任务越多，花在任务切换的时间就越多，CPU执行任务的效率就越低，所以，要最高效地利用CPU，计算密集型任务同时进行的数量应当等于CPU的核心数。

计算密集型任务由于主要消耗CPU资源，因此，代码运行效率至关重要。Python这样的脚本语言运行效率很低，完全不适合计算密集型任务。对于计算密集型任务，最好用C语言编写。

第二种任务的类型是IO密集型，涉及到网络、磁盘IO的任务都是IO密集型任务，这类任务的特点是CPU消耗很少，任务的大部分时间都在等待IO操作完成（因为IO的速度远远低于CPU和内存的速度）。对于IO密集型任务，任务越多，CPU效率越高，但也有一个限度。常见的大部分任务都是IO密集型任务，比如Web应用。

IO密集型任务执行期间，99%的时间都花在IO上，花在CPU上的时间很少，因此，用运行速度极快的C语言替换用Python这样运行速度极低的脚本语言，完全无法提升运行效率。对于IO密集型任务，最合适的语言就是开发效率最高（代码量最少）的语言，脚本语言是首选，C语言最差



# **Fork/Join**框架

> Fork/Join 框架是 Java7 提供了的一个用于并行执行任务的框架， 是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架

对于线程池来说，我们经常使用的是 ThreadPoolExecutor，可以用来提升任务处理效率。一般情况下，我们使用 ThreadPoolExecutor 的时候，各个任务之间都是没有联系的。但在某些特殊情况下，我们处理的任务之间是有联系的，例如经典的 Fibonacci 算法就是其中一种情况。

**ForkJoinPool 就是设计用来解决父子任务有依赖的并行计算问题的。** 类似于快速排序、二分查找、集合运算等有父子依赖的并行计算问题，都可以用 ForkJoinPool 来解决。

Fork 就是把一个大任务切分为若干子任务并行的执行，Join 就是合并这些子任务的执行结果，最后得到这个大任务的结果。比如计算1+2+.....＋10000，可以分割成 10 个子任务，每个子任务分别对 1000 个数进行求和，最终汇总这 10 个子任务的结果。如下图所示：

![image.png](/pic/image-20230506144840611.png)

Fork/Jion特性：

1. ForkJoinPool 不是为了替代 ExecutorService，而是它的补充，在某些应用场景下性能比 ExecutorService 更好。（见 Java Tip: When to use ForkJoinPool vs ExecutorService ）
2. ForkJoinPool 主要用于实现“分而治之”的算法，特别是分治之后递归调用的函数，例如 quick sort 等。
3. ForkJoinPool 最适合的是计算密集型的任务，如果存在 I/O，线程间同步，sleep() 等会造成线程长时间阻塞的情况时，最好配合使用 ManagedBlocker。

## **工作窃取算法**

工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行。

我们需要做一个比较大的任务，我们可以把这个任务分割为若干互不依赖的子任务，为了减少线程间的竞争，于是把这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应，比如A线程负责处理A队列里的任务。但是有的线程会先把自己队列里的任务干完，而其他线程对应的队列里还有任务等待处理。干完活的线程与其等着，不如去帮其他线程干活，于是它就去其他线程的队列里窃取一个任务来执行。而在这时它们会访问同一个队列，所以为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。

工作窃取算法的优点是充分利用线程进行并行计算，并减少了线程间的竞争，其缺点是在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且消耗了更多的系统资源，比如创建多个线程和多个双端队列。

![image.png](/pic/image-20230506144847475.png)

1. ForkJoinPool 的每个工作线程都维护着一个工作队列（WorkQueue），这是一个双端队列（Deque），里面存放的对象是任务（ForkJoinTask）。
2. 每个工作线程在运行中产生新的任务（通常是因为调用了 fork()）时，会放入工作队列的队尾，并且工作线程在处理自己的工作队列时，使用的是 LIFO 方式，也就是说每次从队尾取出任务来执行。
3. 每个工作线程在处理自己的工作队列同时，会尝试窃取一个任务（或是来自于刚刚提交到 pool 的任务，或是来自于其他工作线程的工作队列），窃取的任务位于其他线程的工作队列的队首，也就是说工作线程在窃取其他工作线程的任务时，使用的是 FIFO 方式。
4. 在遇到 join() 时，如果需要 join 的任务尚未完成，则会先处理其他任务，并等待其完成。
5. 在既没有自己的任务，也没有可以窃取的任务时，进入休眠。



## **fork/join的使用**

ForkJoinTask：我们要使用 ForkJoin 框架，必须首先创建一个 ForkJoin 任务。它提供在任务中执行 fork() 和 join() 操作的机制，通常情况下我们不需要直接继承 ForkJoinTask 类，而只需要继承它的子类，Fork/Join 框架提供了以下两个子类：

RecursiveAction：用于没有返回结果的任务。(比如写数据到磁盘，然后就退出了。 一个RecursiveAction可以把自己的工作分割成更小的几块， 这样它们可以由独立的线程或者CPU执行。 我们可以通过继承来实现一个RecursiveAction)

RecursiveTask ：用于有返回结果的任务。(可以将自己的工作分割为若干更小任务，并将这些子任务的执行合并到一个集体结果。 可以有几个水平的分割和合并)

CountedCompleter： 在任务完成执行后会触发执行一个自定义的钩子函数

![image.png](/pic/image-20230506144855049.png)



ForkJoinPool ：ForkJoinTask 需要通过 ForkJoinPool 来执行，任务分割出的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。当一个工作线程的队列里暂时没有任务时，它会随机从其他工作线程的队列的尾部获取一个任务。

## 使用场景示例：

定义fork/join任务，如下示例，随机生成2000w条数据在数组当中，然后求和

```java
package com.itmck.threadpool.Fibonacci;

import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.ForkJoinTask;
import java.util.concurrent.RecursiveTask;

public class LongSumMain {
    //获取逻辑处理器数量
    static final int NCPU = Runtime.getRuntime().availableProcessors();
    /**
     * for time conversion
     */
    static final long NPS = (1000L * 1000 * 1000);

    static long calcSum;

    static final boolean reportSteals = true;

    public static void main(String[] args) throws Exception {
        int[] array = LongSumMain.buildRandomIntArray(20000000);
        System.out.println("cpu-num:" + NCPU);
        //单线程下计算数组数据总和
        calcSum = seqSum(array);
        System.out.println("seq sum=" + calcSum);

        //采用fork/join方式将数组求和任务进行拆分执行，最后合并结果
        LongSum ls = new LongSum(array, 0, array.length);
        ForkJoinPool fjp = new ForkJoinPool(4); //使用的线程数
        ForkJoinTask<Long> result = fjp.submit(ls);
        System.out.println("forkjoin sum=" + result.get());

        fjp.shutdown();

    }

    private static int[] buildRandomIntArray(int i) {

        int[] arr = new int[i + 1];
        for (int j = 0; j <= i; j++) {
            arr[j] = i;
        }
        return arr;
    }

    static long seqSum(int[] array) {
        long sum = 0;
        for (int j : array) {
            sum += j;
        }
        return sum;
    }
}

/**
 * RecursiveTask 并行计算，同步有返回值
 * ForkJoin框架处理的任务基本都能使用递归处理，比如求斐波那契数列等，但递归算法的缺陷是：
 * 一只会只用单线程处理，
 * 二是递归次数过多时会导致堆栈溢出；
 * ForkJoin解决了这两个问题，使用多线程并发处理，充分利用计算资源来提高效率，同时避免堆栈溢出发生。
 * 当然像求斐波那契数列这种小问题直接使用线性算法搞定可能更简单，实际应用中完全没必要使用ForkJoin框架，
 * 所以ForkJoin是核弹，是用来对付大家伙的，比如超大数组排序。
 * 最佳应用场景：多核、多内存、可以分割计算再合并的计算密集型任务
 */
class LongSum extends RecursiveTask<Long> {
    //任务拆分的最小阀值
    static final int SEQUENTIAL_THRESHOLD = 1000;
    static final long NPS = (1000L * 1000 * 1000);
    static final boolean extraWork = true; // change to add more than just a sum
    int low;
    int high;
    int[] array;

    LongSum(int[] arr, int lo, int hi) {
        array = arr;
        low = lo;
        high = hi;
    }


    /**
     * fork()方法：将任务放入队列并安排异步执行，一个任务应该只调用一次fork()函数，除非已经执行完毕并重新初始化。
     * tryUnfork()方法：尝试把任务从队列中拿出单独处理，但不一定成功。
     * join()方法：等待计算完成并返回计算结果。
     * isCompletedAbnormally()方法：用于判断任务计算是否发生异常。
     */
    protected Long compute() {
        //任务被拆分到足够小时，则开始求和
        if (high - low <= SEQUENTIAL_THRESHOLD) {
            long sum = 0;
            for (int i = low; i < high; ++i) {
                sum += array[i];
            }
            return sum;
        } else {//如果任务任然过大，则继续拆分任务，本质就是递归拆分
            int mid = low + (high - low) / 2;
            LongSum left = new LongSum(array, low, mid);
            LongSum right = new LongSum(array, mid, high);
            left.fork();
            right.fork();
            long rightAns = right.join();
            long leftAns = left.join();
            return leftAns + rightAns;
        }
    }
}
```



https://juejin.cn/post/6906424424967667725

https://juejin.cn/post/7221559896214028346

