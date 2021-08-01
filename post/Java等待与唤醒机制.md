## 一、从synchronized锁看线程等待与唤醒

初学Java的时候想必大家都用synchronized实现过“生产者-消费者”模型的代码，其中用到了几个Object中的方法如wait()、notify()、notifyAll()，不知道当时的你是否有些困惑，线程等待与唤醒相关的方法为什么会定义在Object类中呢？

什么？你连“生产者-消费者”模型都忘了是什么了？好吧，我们还是先来看下回顾一下“生产者-消费者”模型吧。

### 1.“生产者-消费者”模型

“生产者-消费者”模型是一个典型的线程协作通信的例子。在这一模型中有两类角色，即若干个生产者线程和若干个消费者线程。生产者线程负责提交用户请求，消费者线程负责处理生产者提交的请求。很多情况下，生产者与消费者不能够达到一定的平衡，即有时候生产者生产的速度过快，消费之来不及消费；而有时候可能是消费者过于旺盛，生产者来不及生产。在此情况下就需要一个生产者与消费者共享的内存缓存区来平衡二者的协作。生产者与消费者之间通过共享内存缓存区进行通信，从而平衡生产者与消费者线程，并将生产者和消费者解耦。如下图所示：

![1C2478F7-48B7-4ACA-A575-ABF8B71F40B9.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b129a2e7034c474791c35ceb89e3af18~tplv-k3u1fbpfcp-watermark.image)

当队列容器中没有商品的时候，就需要让消费者处于等待状态，而当容器满了之后就需要生产者处于等待状态。而消费者每消费一个商品，又会通知正在等待的生产者可以进行生产了；当生产则生产一个商品，也会通知正在等待的消费者可以消费了。



### 2.使用synchronized实现“生产者-消费者”模型

了解了“生产者-消费者”模型后，我们尝试使用synchronized关键字结合wait()与notifyAll()方法来实现一个”生产者-消费者“模型的例子。

我们选一个比较经典的生产面包的例子来看，首先需要一个面包容器类，容器类中有放入面包和取出面包两个操作，代码如下：


```java
public class BreadContainer {

    LinkedList<Bread> list = new LinkedList<>();
    // 容器容量
    private final static int CAPACITY = 10;
    /**
     * 放入面包
     */
    public synchronized void put(Bread bread) {
        while (list.size() == CAPACITY) {
            try {
                // 如果容器已满，则阻塞生产者线程
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        list.add(bread);
        // 面包生产成功后通知消费者线程
        notifyAll();
        System.out.println(Thread.currentThread().getName() + " product a bread" + bread.toString() + " size = " + list.size());
    }

    /**
     * 取出面包
     */
    public synchronized void take() {
        while (list.isEmpty()) {
            try {
                // 如果容器为空，则阻塞消费者线程
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        Bread bread = list.removeFirst();
        // 消费后通知生产者生产面包
        notifyAll();
        System.out.println("Consumer " + Thread.currentThread().getName() + " consume a bread" + bread.toString() + " size = " + list.size());
    }
}
```
在上述代码的put方法会将生产好的面包放入到容器中。如果容器已经满了，那么需要阻塞生产者线程来停止生产，当生产者成功将面包放入容器后则需要尝试唤醒等待中的消费者线程进行消费。

而take方法则是取出面包的操作，当容器为空，则阻塞消费者线程，让其进行等待，如果成功消费面包后则通知生产者开始生产。

另外需要注意一下，这两个方法都使用了synchronized关键字，如果你看过[这一次，彻底搞懂Java中的synchronized关键字](https://zhpanvip.gitee.io/2021/06/14/39-synchronized/)这篇文章的话应该知道此时synchronized加锁的对象就是这两个方法所在的实例对象，即BreadContainer对象，而在这两个方法中调用的wait()和notifyAll()两个方法同样属于BreadContainer对象。记住这段话，这里留个Flag,我们后边分析。

接下来生产者与消费者的实现就比较简单了，代码如下：


```java
// 生产者
public class Producer implements Runnable {
    private final BreadContainer container;

    public Producer(BreadContainer container) {
        this.container = container;
    }

    @Override
    public void run() {
        // 生产者生产面包
        container.put(new Bread());
    }
}

// 消费者
public class Consumer implements Runnable {

    private final BreadContainer container;

    public Consumer(BreadContainer container) {
        this.container = container;
    }

    @Override
    public void run() {
        // 消费者消费面包
        container.take();
    }
}
```
接下来在测试代码中，同时开启多个生产者线程与多个消费者线程

```java

    public static void main(String[] args) {
        BreadContainer container = new BreadContainer();

        new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                new Thread(new Producer(container)).start();
            }
        }).start();

        new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                new Thread(new Consumer(container)).start();
            }
        }).start();

    }

```
此时运行main方法，生成者与消费者线程就可以很好的协同工作了。

注意，在main方法中我们实例化了一个BreadContainer对象，上边Flag处说的synchronized锁的对象即为这个container，调用的wait和notifyAll方法也是container实例的方法。到这里不知道你是否会有疑问，究竟container的wait和notify方法对象成做了什么能让线程阻塞和唤醒呢？被阻塞的线程放到哪里去了？为什么要调用container对象中的wait和notifyAll方法？如果换成调用其他对象的wait和notifyAll是否可行呢？


## 二、wait()与notify底层实现原理

在[这一次，彻底搞懂Java中的synchronized关键字](https://zhpanvip.gitee.io/2021/06/14/39-synchronized/)中我们已经知道，使用synchronized关键字后，synchronized锁住的对象会关联一个monitor对象，当有一个线程获得synchronized锁后，monitor对象中的count就会被加1，并且会将这个线程的id存入到monitor的_ower中。此时，如果其他线程来尝试拿锁则会被放入到_WaitSet队列中等待。

还记得上一节中我们立的一个Flag了吗？synchronized锁的是container对象，而wait和notify也是container的方法，这么一看我们上一节中留下的问题就有些眉目了。是不是调用wait方法的时候线程也会被放入到等待队列，而等到notify或者notifyAll的时候再从等待队列中将线程唤醒呢？想要解答这个问题就需要我们来看下wait和notify/notifyAll到底做了什么。

我们看下Object中wait、notify、notifyAll三个方法的实现


```java
public class Object {

    public final native void notify();

    public final native void notifyAll();

    public final void wait() throws InterruptedException {
        wait(0L);
    } 
    public final native void wait(long timeoutMillis) throws InterruptedException;    

}
```
很可惜，这几个方法都是native方法，也就是说这几个方法都是在虚拟机中使用C/C++实现的。既然如此，我们不妨扒一扒虚拟机的代码来一探究竟，毕竟口说无凭。

### 1.虚拟机对wait的实现

承接上一节中”生产者-消费者“模型的代码来分析，当生产者线程往容器里边放面包的时候发现容器已经满了，则调用wait方法，那么此时这个线程就会释放锁并进入到阻塞状态。

Object中wait方法的真实实现是在[objectMonitor.cpp](https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/782f3b88b5ba/src/share/vm/runtime/objectMonitor.cpp)中的ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS)函数中,ObjectMonitor::wait中的核心相关代码如下：


```C++
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) {
    // ...省略其他代码
    
    // 当前线程
    Thread * const Self = THREAD ;
    // 将线程封装成ObjectWaiter
    ObjectWaiter node(Self);
    // 标记为Wait状态
    node.TState = ObjectWaiter::TS_WAIT ;
    Self->_ParkEvent->reset() ;

    Thread::SpinAcquire (&_WaitSetLock, "WaitSet - add") ;
    // 将线程加入到等待队列中
    AddWaiter (&node) ;
    Thread::SpinRelease (&_WaitSetLock) ;
    
    // ...
    
    // 线程退出monitor,释放锁
    exit (true, Self) ; 
}
```
可以看到，调用wait函数后，线程被封装成了一个ObjectWaiter对象，然后调用了addWait函数将线程加入到等待队列中。接着看addWait函数的代码：


```C++
inline void ObjectMonitor::AddWaiter(ObjectWaiter* node) {
  
    if (_WaitSet == NULL) {
        // 初始化_WaitSet,即只有一个node元素
        _WaitSet = node;
        // 这里可以看出_WaitSet是一个双向环形链表
        node->_prev = node;
        node->_next = node;
    } else {
        // 头结点
        ObjectWaiter* head = _WaitSet ;
        // 环形链表头结点的prev就是尾结点
        ObjectWaiter* tail = head->_prev;
        assert(tail->_next == head, "invariant check");
        // 将node插入到_WaitSet的尾结点中
        tail->_next = node;
        head->_prev = node;
        node->_next = head;
        node->_prev = tail;
    }
}
```
AddWaiter函数的实现其实比较简单，会初始化一个_WaitSet链表，并将node插入到_WaitSet的队尾，而这个链表是一个环形的双向链表。

完成线程的插入队列操作后，继续调用exit函数来释放monitor锁,并挂起自己。


### 2.虚拟机对notify的实现

在生产者生产完面包后则会调用notifyAll来唤醒消费者线程。此处我们以notify为例，来看[objectMonitor.cpp](https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/782f3b88b5ba/src/share/vm/runtime/objectMonitor.cpp)中notify函数的实现



```C++
void ObjectMonitor::notify(TRAPS) {
 
    int Policy = Knob_MoveNotifyee ;
    // 从_WaitSet中取出ObjectWaiter队列
    ObjectWaiter * iterator = DequeueWaiter() ;
    if (iterator != NULL) {

        ObjectWaiter * List = _EntryList ;

        // 根据策略执行不同的逻辑，Policy默认值为2
        if (Policy == 0) {       // prepend to EntryList
            // ...
        } else if (Policy == 1) {      // append to EntryList
            // ...
        } else if (Policy == 2) {      // prepend to cxq
            // prepend to cxq
            if (List == NULL) {
                iterator->_next = iterator->_prev = NULL ;
                _EntryList = iterator ;
            } else {
                iterator->TState = ObjectWaiter::TS_CXQ ;
                for (;;) {
                    ObjectWaiter * Front = _cxq ;
                    iterator->_next = Front ;
                    if (Atomic::cmpxchg_ptr (iterator, &_cxq, Front) == Front) {
                        break ;
                    }
                }
            }
        } else if (Policy == 3) {      // append to cxq
            // ...
        }
    }
    // ...
}
```
在notify函数中Policy被赋值了Knob_MoveNotifyee，而Knob_MoveNotifyee是一个常量，值为2,所以这里只关注Policy为2的情况。而DequeueWaiter()是一个函数，代码如下：

```C++
inline ObjectWaiter* ObjectMonitor::DequeueWaiter() {
    // dequeue the very first waiter
    ObjectWaiter* waiter = _WaitSet;
    if (waiter) {
        DequeueSpecificWaiter(waiter);
    }
    return waiter;
}
```
这个函数就是得到_WaitSet集合，并赋值给了iterator，iterator不为NULL说明存在被阻塞的线程。

接着将_EntryList赋值给List，如果List等于NULL，说明此时没有在等待获取锁的线程。那么就将阻塞中的iterator构成一个双向环链表，并将_EntryList指向iterator。标志着阻塞中的线程能够获取锁了，但此时线程还未被唤醒。

如果_EntryList不为NULL，就通过CAS将interator加入_cxq队列，并将_cxq指针指向interator。

可见notify函数中只是对线程进行了队列转移，并没有被实际唤醒。而实际唤醒线程的操作就是在上边提到的exist函数中的，而这个exist函数也正是在虚拟机读取到monitorexist指令后执行，exist函数简化后代码如下：


```C++
void ObjectMonitor::exit(bool not_suspended, TRAPS) {
  // ...
  for (;;) {

        ObjectWaiter * w = NULL ;
        // QMode默认值为0
        int QMode = Knob_QMode ;
        
        if (QMode == 2 && _cxq != NULL) {
            // ... 这里从_cxq队列取头结点并唤醒，无关省略。
            return ;
        }

        // ...
        
        w = _EntryList  ;
        // 先查看_EntryList是否为空
        if (w != NULL) {
            assert (w->TState == ObjectWaiter::TS_ENTER, "invariant") ;
            // _EntryList不为空，直接唤醒_EntryList队列的头结点
            ExitEpilog (Self, w) ;
            return ;
        }
        // 到这里说明_EntryList为空，将w指向_cxq
        w = _cxq ;
        if (w == NULL) continue ;

        // ...

        if (QMode == 1) {
            // ...
        } else {
            // 如果走到此处说明_cxq队列不为空
            // QMode == 0 or QMode == 2
            // 此时_cxq队列是空，将_EntryList指向_cxq队列
            _EntryList = w ;
            ObjectWaiter * q = NULL ;
            ObjectWaiter * p ;
            // 将单向链表构造成双向环链表
            for (p = w ; p != NULL ; p = p->_next) {
                guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
                p->TState = ObjectWaiter::TS_ENTER ;
                p->_prev = q ;
                q = p ;
            }
        }

        if (_succ != NULL) continue;

        w = _EntryList  ;
        if (w != NULL) {
            guarantee (w->TState == ObjectWaiter::TS_ENTER, "invariant") ;
            // 唤醒_EntryList的头结点
            ExitEpilog (Self, w) ;
            return ;
        }
    }
    
// 释放锁并唤醒线程    
void ObjectMonitor::ExitEpilog (Thread * Self, ObjectWaiter * Wakee) {
    assert (_owner == Self, "invariant") ;

    // Exit protocol:
    // 1. ST _succ = wakee
    // 2. membar #loadstore|#storestore;
    // 2. ST _owner = NULL
    // 3. unpark(wakee)

    _succ = Knob_SuccEnabled ? Wakee->_thread : NULL ;
    ParkEvent * Trigger = Wakee->_event ;

    Wakee  = NULL ;

    // 释放锁
    OrderAccess::release_store_ptr (&_owner, NULL) ;
    OrderAccess::fence() ;                               // ST _owner vs LD in unpark()
    // 唤醒线程
    Trigger->unpark() ;

    //...
}    
```

exist函数的代码比较繁杂，这里做了简化，由于QMode默认值是0，因此只讨论这种情况。

1）首先，如果_EntryList不为NULL，那么直接调用ExitEpilog函数从_EntryList中取出头结点并唤醒线程；

2）如果_EntryList为NULL,但是_cxq队列不为NULL，那么取出_cxq队列中的所有元素并构建成一个双向环链表赋
值为_EntryList,然后将_cxq置为NULL；

3）最后，调用ExitEpilog函数释放锁并唤醒_EntryList的头结点。


[深入理解Java线程的等待与唤醒机制（一）](https://juejin.cn/post/6980002998361522190)

[深入理解Java线程的等待与唤醒机制（二）](https://juejin.cn/post/6980655421497278495)