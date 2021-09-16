Rxjava中可以通过subscribeOn和observeOn来切换被观察者与观察者运行的线程

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
  @Override public void subscribe(ObservableEmitter<Integer> emitter)
      throws Exception {
    emitter.onNext(1);
  }
}).subscribeOn(Schedulers.io()) // 指定被观察者运行在IO线程，适用于IO密集型
    .observeOn(AndroidSchedulers.mainThread()) // 指定观察者运行在子线程，需要依赖RxAndroid
    .subscribe(new Observer<Integer>() {
    	 ...
    });
```



## 一、RxJava中的线程池

### 1. IO密集型线程池

subscribeOn方法中接收一个通过Schedulers.io()方法得到的Scheduler实例，其源码如下：

```java
public final class Schedulers {
  static final Scheduler IO;
	public static Scheduler io() {
  	  return RxJavaPlugins.onIoScheduler(IO);
	}
  
  static {
    // 实例化一个线程的xian'cheng'c
    SINGLE = RxJavaPlugins.initSingleScheduler(new SingleTask());
    // CPU密集型
    COMPUTATION = RxJavaPlugins.initComputationScheduler(new ComputationTask());
		// IO 密集型线程池
    IO = RxJavaPlugins.initIoScheduler(new IOTask());
    
    TRAMPOLINE = TrampolineScheduler.instance();
		// 实例化新线程，
    NEW_THREAD = RxJavaPlugins.initNewThreadScheduler(new NewThreadTask());
}
  
  // ...
}
```

这个方法中通过RxJavaPlugins.onIoScheduler(IO)返回了一个Scheduler实例，这个实例其实就是Schedulers的成员变量IO。代码中可以看到IO是一个静态的Scheduler成员，并且在Schedulers静态代码块中通过 ` RxJavaPlugins.initIoScheduler(new IOTask())` 来初始化Scheduler。这个方法接受一个IOTask参数，IOTask源码如下：

```java
// Schedulers.java

static final class IOTask implements Callable<Scheduler> {
    @Override
    public Scheduler call() throws Exception {
        return IoHolder.DEFAULT;
    }
}

static final class IoHolder {
    static final Scheduler DEFAULT = new IoScheduler();
}
```

可以看到这里实例化了一个IoScheduler，IoScheduler继承自Scheduler，这个类中封装了一个适用于IO密集型的java线程池。先看下它的构造方法

```java
static final RxThreadFactory WORKER_THREAD_FACTORY;

    static {
        // 线程工厂
        WORKER_THREAD_FACTORY = new RxThreadFactory(WORKER_THREAD_NAME_PREFIX, priority);
				// ... 
    }

public IoScheduler() {
    // 传入默认的线程工厂
    this(WORKER_THREAD_FACTORY);
}

public IoScheduler(ThreadFactory threadFactory) {
    this.threadFactory = threadFactory;
    this.pool = new AtomicReference<CachedWorkerPool>(NONE);
    // 调用start
    start();
}
```

start方法中初始化了线程池，不过是通过CachedWorkerPool进行初始化的，线程池被封装到了这个CachedWorkerPool中。

```java
public void start() {
    // KEEP_ALIVE_TIME默认60s，即空闲线程的存活时间是60s
    CachedWorkerPool update = new CachedWorkerPool(KEEP_ALIVE_TIME, KEEP_ALIVE_UNIT, threadFactory);
    if (!pool.compareAndSet(NONE, update)) {
        update.shutdown();
    }
}
```

接着看CachedWorkerPool的构造方法

```java
CachedWorkerPool(long keepAliveTime, TimeUnit unit, ThreadFactory threadFactory) {
    this.keepAliveTime = unit != null ? unit.toNanos(keepAliveTime) : 0L;
    this.expiringWorkerQueue = new ConcurrentLinkedQueue<ThreadWorker>();
    this.allWorkers = new CompositeDisposable();
    this.threadFactory = threadFactory;

    ScheduledExecutorService evictor = null;
    Future<?> task = null;
    if (unit != null) {
        // 通过Executors工具进行初始化线程池，核心线程数是1，
        evictor = Executors.newScheduledThreadPool(1, EVICTOR_THREAD_FACTORY);
        // 设置空闲线程存活时间
        task = evictor.scheduleWithFixedDelay(this, this.keepAliveTime, this.keepAliveTime, TimeUnit.NANOSECONDS);
    }
    evictorService = evictor;
    evictorTask = task;
}
```

了解过线程池的话应该知道Executors这个线程池工具类，它可以方便快捷的创建线程池。看下它newScheduledThreadPool方法的源码：

```java
// Executors.java
public static ScheduledExecutorService newScheduledThreadPool(
        int corePoolSize, ThreadFactory threadFactory) {
    // 调用线程池构造方法，传入核心线程数为1
    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
```



```java
// ScheduledThreadPoolExecutor.java
public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory) {
    // 调用自身构造方法，传入最大线程数为最大Int值，阻塞队列是DelayedWorkQueue
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue(), threadFactory);
}
```

最终可以看到，线程池配置的参数为：

>  核心线程数是1，最大线程数是Integer.MAX_VALUE，阻塞队列为DelayedWorkQueue。

DelayedWorkQueue是一个高度定制化的延迟阻塞队列，它的核心数据结构是二叉最小堆的优先队列。队列满时会自动扩容，所以offer方法用于不会被阻塞，这也意味着maximumPoolSize其实并没有用，线程池中始终会保持最多corePoolSize个线程运行。

从这个线程池配置的参数可以看得出来，使用Schedulers.IO创建的线程池，会始终保持只有1个核心线程在运行，所有来不及执行的任务都会被保存到DelayedWorkQueue中。

### 2. CPU 密集型线程池

Scheduler中除了提供 io 方法创建IO密集型线程池外还提供了computation方法来创建一个CPU密集型的线程池，如下：

```java
public static Scheduler computation() {
    return RxJavaPlugins.onComputationScheduler(COMPUTATION);
}
```

它的创建过程与上一节IO的创建是一致的，过程就不再分析，主要看一下这里创建线程池的参数

ComputationScheduler中start方法如下：

```java
public void start() {
    
    FixedSchedulerPool update = new FixedSchedulerPool(MAX_THREADS, threadFactory);
    if (!pool.compareAndSet(NONE, update)) {
        update.shutdown();
    }
}
```

而MAX_THREADS是通过` cap(Runtime.getRuntime().availableProcessors(), Integer.getInteger(KEY_MAX_THREADS, 0));` 计算得到的，availableProcessors方法即计算虚拟机可用的核心数。

FixedSchedulerPool的构造方法如下：

```java
FixedSchedulerPool(int maxThreads, ThreadFactory threadFactory) {
    // initialize event loops
    this.cores = maxThreads;
    this.eventLoops = new PoolWorker[maxThreads];
    for (int i = 0; i < maxThreads; i++) {
        this.eventLoops[i] = new PoolWorker(threadFactory);
    }
}
```

先实例化了一个长度为maxThreads的PoolWorker数组，然后又实例化了maxThreads个PoolWorker实例，PoolWorker是什么东西？其源码如下：

```java
static final class PoolWorker extends NewThreadWorker {
    PoolWorker(ThreadFactory threadFactory) {
        // 调用父类的构造方法
        super(threadFactory);
    }
}
```

NewThreadWorker的构造方法如下:

```java
// NewThreadWorker.java
public NewThreadWorker(ThreadFactory threadFactory) {
    executor = SchedulerPoolFactory.create(threadFactory);
}
```

通过SchedulerPoolFactory创建线程池：

```java
// SchedulerPoolFactory.java
public static ScheduledExecutorService create(ThreadFactory factory) {
    final ScheduledExecutorService exec = Executors.newScheduledThreadPool(1, factory);
    tryPutIntoPool(PURGE_ENABLED, exec);
    return exec;
}
```

通过线程池工具类的newScheduledThreadPool方法实例化线程池，同时，核心线程数还是1，最终调用线程池的构造方法如下：

```java
// ScheduledThreadPoolExecutor.java
public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue(), threadFactory);
}
```

可以看到依然是核心线程数为1，最大线程数为Integer.MAX_VALUE，阻塞队列为DelayedWorkQueue的线程池。

跟IO的区别是，这里创建了多个线程池。。。。

## 二、切换线程操作符



```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```

