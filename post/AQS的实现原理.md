## 学习AQS的必要性 

队列同步器AbstractQueuedSynchronizer（以下简称同步器或AQS），是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。并发包的大师（Doug Lea）期望它能够成为实现大部分同步需求的基础。

## AQS使用方式和其中的设计模式
 
AQS的主要使用方式是继承，子类通过继承AQS并实现它的抽象方法来管理同步状态，在AQS里有一个int型的state来代表这个状态，在抽象方法的实现过程中免不了要对同步状态进行更改，这时就需要使用同步器提供的3个方法（getState()、setState(int newState)和compareAndSetState(int expect,int update)）来进行操作，因为它们能够保证状态的改变是安全的。

```
/**
 * The synchronization state.
 */
private volatile int state;
```

在实现上，子类推荐被定义为自定义同步组件的静态内部类，AQS自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获取同步状态，这样就可以方便实现不同类型的同步组件（ReentrantLock、ReentrantReadWriteLock和CountDownLatch等）。

同步器是实现锁（也可以是任意同步组件）的关键，在锁的实现中聚合同步器。可以这样理解二者之间的关系：

锁是面向使用者的，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；

同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。锁和同步器很好地隔离了使用者和实现者所需关注的领域。

实现者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。

## 模板方法模式

同步器的设计基于模板方法模式。模板方法模式的意图是，定义一个操作中的算法的骨架，而将一些步骤的实现延迟到子类中。模板方法使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。我们最常见的就是Spring框架里的各种Template。



## AQS中的方法
### 1.模板方法
实现自定义同步组件时，将会调用同步器提供的模板方法，

| 方法名称 | 描述 |
|--|--|
| void acquire(int arg) | 独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回。否则，进入进入同步队列等待。该方法将会调用重写tryAcquire(int arg)方法 |
| void acquireInterruptibly(int arg) | 与acquire(int arg)相同，但是该方法相应中断，当前线程未获取到同步状态而进入队列中，如果当前线程被中断，则方法会抛出InterrputedException并返回。 |
| boolean tryAcquireNanos(int arg, long nanosTimeout) |在acqureInterruptibly(int arg)基础上增加了超时限制，如果当前线程在超时时间内没有获取到同步状态，那么将返回false，如果获取到就返回true  |
| void acquireShared(int arg) |共享式的获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式获取的主要区别在于同一时刻可以有多个线程获取到同步状态。  |
| void acquireSharedInterruptibly(int arg) | 与acquireShared(int arg)相同，该方法相应中断 |
| boolean tryAcquireSharedNanos(int arg,long nanos) | 在acquireSharedInterruptibly(int arg)基础上增加了超时限制  |
| boolean release(int arg) | 独占式的释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒 |
| boolean releaseShared(int arg) | 共享式的释放同步状态 |
| Collection<Thread> getQueuedThreads() | 获取等待在同步队列上的线程集合 |

这些模板方法同步器提供的模板方法基本上分为3类：独占式获取与释放同步状态、共享式获取与释放、同步状态和查询同步队列中的等待线程情况。

### 2.可重写的方法

| 方法名称 | 描述 |
|--|--|
| protected boolean trayAcquire(int arg) | 独占式获取同步状态，实现该方法需要查询当前状态并判断同步状态是否符合预期，然后再进行CAS设置同步状态 |
| protected boolean tryRelease(int arg) | 独占式释放同步状态，等待获取同步状态的线程将有机会获取到同步状态 |
| protected int tryAcquireShared(int arg) | 共享式获取同步状态，返回大于等于0的值表示获取成功，反之获取失败 |
| protected boolean tryReleaseShared(int arg) | 共享式释放同步状态 |
 | protected boolean isHeldExclusively() | 当前同步器是否在独占模式下被线程占用，一般该方法表示是否被当前线程所独占 |

### 3.访问或修改同步状态的方法

重写同步器指定的方法时，需要使用同步器提供的如下3个方法来访问或修改同步状态。

- getState()：获取当前同步状态。
- setState(int newState)：设置当前同步状态。
- compareAndSetState(int expect,int update)：使用CAS设置当前状态，该方法能够保证状态设置的原子性。 

## CLH队列锁 

CLH队列锁即Craig, Landin, and Hagersten (CLH) locks。

CLH队列锁也是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程仅仅在本地变量上自旋，它不断轮询前驱的状态，假设发现前驱释放了锁就结束自旋。

当一个线程需要获取锁时：

- 1.创建一个的QNode，将其中的locked设置为true表示需要获取锁，myPred表示对其前驱结点的引用
![](https://gitee.com/zhpanvip/images/raw/master/project/article/thread/clh1.png)
- 2.线程A对tail域调用getAndSet方法，使自己成为队列的尾部，同时获取一个指向其前驱结点的引用myPred

![](https://gitee.com/zhpanvip/images/raw/master/project/article/thread/clh2.png)

线程B需要获得锁，同样的流程再来一遍

![](https://gitee.com/zhpanvip/images/raw/master/project/article/thread/clh3.png)

- 3.线程就在前驱结点的locked字段上旋转，直到前驱结点释放锁(前驱节点的锁值 locked == false)

- 4.当一个线程需要释放锁时，将当前结点的locked域设置为false，同时回收前驱结点

![](https://gitee.com/zhpanvip/images/raw/master/project/article/thread/clh4.png)

如上图所示，前驱结点释放锁，线程A的myPred所指向的前驱结点的locked字段变为false，线程A就可以获取到锁。

CLH队列锁的优点是空间复杂度低（如果有n个线程，L个锁，每个线程每次只获取一个锁，那么需要的存储空间是O（L+n），n个线程有n个myNode，L个锁有L个tail）。CLH队列锁常用在SMP体系结构下。

Java中的AQS是CLH队列锁的一种变体实现。