- [一个线程有几个Handler？一个线程有几个Looper？如何保证？](#%E4%B8%80%E4%B8%AA%E7%BA%BF%E7%A8%8B%E6%9C%89%E5%87%A0%E4%B8%AAhandler%E4%B8%80%E4%B8%AA%E7%BA%BF%E7%A8%8B%E6%9C%89%E5%87%A0%E4%B8%AAlooper%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81)
- [Handler线程是如何切换的？](#handler%E7%BA%BF%E7%A8%8B%E6%98%AF%E5%A6%82%E4%BD%95%E5%88%87%E6%8D%A2%E7%9A%84)
- [Handler内存泄漏的原因是什么？如何解决?](#handler%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E7%9A%84%E5%8E%9F%E5%9B%A0%E6%98%AF%E4%BB%80%E4%B9%88%E5%A6%82%E4%BD%95%E8%A7%A3%E5%86%B3)
- [子线程中使用Looper应该注意什么？有什么用？](#%E5%AD%90%E7%BA%BF%E7%A8%8B%E4%B8%AD%E4%BD%BF%E7%94%A8looper%E5%BA%94%E8%AF%A5%E6%B3%A8%E6%84%8F%E4%BB%80%E4%B9%88%E6%9C%89%E4%BB%80%E4%B9%88%E7%94%A8)
- [MessageQueue是如何保证线程安全的？](#messagequeue%E6%98%AF%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E7%9A%84)
- [我们使用Message的时候如何创建它？](#%E6%88%91%E4%BB%AC%E4%BD%BF%E7%94%A8message%E7%9A%84%E6%97%B6%E5%80%99%E5%A6%82%E4%BD%95%E5%88%9B%E5%BB%BA%E5%AE%83)
- [Looper死循环为什么不会导致应用卡死？](#looper%E6%AD%BB%E5%BE%AA%E7%8E%AF%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E4%BC%9A%E5%AF%BC%E8%87%B4%E5%BA%94%E7%94%A8%E5%8D%A1%E6%AD%BB)
- [能不能让一个Message被加急处理？](#%E8%83%BD%E4%B8%8D%E8%83%BD%E8%AE%A9%E4%B8%80%E4%B8%AAmessage%E8%A2%AB%E5%8A%A0%E6%80%A5%E5%A4%84%E7%90%86)
- [Handler的同步屏障是什么？](#handler%E7%9A%84%E5%90%8C%E6%AD%A5%E5%B1%8F%E9%9A%9C%E6%98%AF%E4%BB%80%E4%B9%88)
- [Handler的阻塞唤醒机制是什么？](#handler%E7%9A%84%E9%98%BB%E5%A1%9E%E5%94%A4%E9%86%92%E6%9C%BA%E5%88%B6%E6%98%AF%E4%BB%80%E4%B9%88)
- [ThreadLocal的实现原理](post/ThreadLoacal%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.md)



## 一个线程有几个Handler？一个线程有几个Looper？如何保证？
Handler的个数与所在线程无关，可以在线程中实例化任意多个Handler。一个线程中只有一个Looper。Looper的构造方法被声明为了private，我们无法通过new关键字来实例化Looper，唯一开放的可以实例化Looper的方法是prepare。prepare方法的源码如下：

```java
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
我们知道ThreadLocal是一个线程内部的数据存储类，当某个线程调用prepare方法的时候，会首先通过ThreadLocal检查这个线程是否已经创建了Looper,如果还没创建，则实例化Looper并将实例化后的Looper保存到ThreadLocal中，而如果ThreadLocal中已经保存了Looper，则会抛出一个RuntimeException的异常。那么意味着在一个线程中最多只能调用一次prepare方法，这样就保证了Looper的唯一性。

## Handler线程是如何切换的？

（1）假设现在有一个线程A，在A线程中通过Looper.prepare和Looper.loop来开启Looper,并且在A线程中实例化出来一个Handler。Looper.prepare()方法被调用时会为会初始化Looper并为ThreadLocal 设置Looper，此时ThreadLocal中就存储了A线程的Looper。另外MessageQueue也会在Looper中被初始化。

（2）接着当调用Loop.loop方法时，loop方法会通过myLooper得到A线程中的Looper，进而拿到Looper中的MessageQueue，接着开启死循环等待执行MessageQueue中的方法。
（3）此时，再开启一个线程B，并在B线程中通过Handler发送出一个Message，这个Message最终会通过sendMessageAtTime方法调用到MessageQueue的equeueMessage方法将消息插入到队列。

（4）由于Looper的loop是一个死循环，当MessageQueue中被插入消息的时候，loop方法就会取出MessageQueue中的消息，并执行callback。而此时，Looper是A线程的Looper,进而调用的Message或者Handler的Callback都是执行在A线成中的。以此达到了线程的切换。

## Handler内存泄漏的原因是什么？如何解决?
通常在使用Handler的时候回通过匿名内部类的方式来实例化Handler，而非静态的匿名内部类默认持有外部类的引用，即匿名内部类Handler持有了外部类。而导致内存泄漏的根本原因是是因为Handler的生命周期与宿主的生命周期不一致。比如说在Activity中实例化了一个非静态的匿名内部类Handler，然后通过Handler发送了一个延迟消息，但是在消息还未执行时结束了Activity，此时由于Handler持有Activity，就会导致Activity无法被GC回收，也就是出现了内存泄漏的问题。解决方式是可以把Handler声明为静态的匿名内部类，但这样一来，在Handler内部就没办法调用到Activity中的非静态方法或变量。那么最终的解决方案可以使用静态内部类 + 弱引用来解决。代码如下：

```java
public class MainActivity extends AppCompatActivity {

    private MyHandler mMyHandler = new MyHandler(this);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    private void handleMessage(Message msg) {

    }

    static class MyHandler extends Handler {
        private WeakReference<Activity> mReference;

        MyHandler(Activity reference) {
            mReference = new WeakReference<>(reference);
        }

        @Override
        public void handleMessage(Message msg) {
            MainActivity activity = (MainActivity) mReference.get();
            if (activity != null) {
                activity.handleMessage(msg);
            }
        }
    }

    @Override
    protected void onDestroy() {
        mMyHandler.removeCallbacksAndMessages(null);
        super.onDestroy();
    }
}
```


## 子线程中使用Looper应该注意什么？有什么用？
子线程中开启的Looper，在没有消息时一定要调用Looper的quit或者quitSafely方法。不然子线程会一直处于阻塞状态，无法被回收，进而可能导致内存泄漏的问题。
## MessageQueue是如何保证线程安全的？
enqueueMessage方法和next方法中的代码都加了synchronize关键字来保证线程安全。

## 我们使用Message的时候如何创建它？
这里推荐使用 Message.obtain() 方法来实例化一个 Message，好处在于它会从消息池中取，而避免了重复创建的开销。虽然直接实例化一个 Message 其实并没有多大开销，但是我们知道 Android 是消息驱动的，这也就说明 Message 的使用量是很大的，所以当基数很大时，消息池就显得非常有必要了。

```java
public final class Message implements Parcelable {
    
   //消息的标示
    public int what;
    //系统自带的两个参数
    public int arg1;
    public int arg2;
    //处理消息的相对时间
    long when;
	
    Bundle data;
    Handler target;
    Runnable callback;
    
    Message next;	//消息池是以链表结构存储 Message
    private static Message sPool;	//消息池中的头节点
    
    //公有的构造方法，所以我们可以通过 new Message() 实例化一个消息了
    public Message() {
    }
    
    //推荐以这种方式实例化一个 Message，
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                //取出头节点返回
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
    
    //回收消息
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }

    void recycleUnchecked() {
        //清空消息的所有标志信息
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                //链表头插法
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
}

```
Message中维护了一个sPools的消息池，obtain方法会从消息池中取出一个消息，避免了实例化。而在Message使用结束后会通过recycle或者recycleUnchedked来回收Message。
## Looper死循环为什么不会导致应用卡死？
如果按照 Message.next 方法的注释来解释的话，如果返回的 Message 为空，就说明消息队列已经退出了，这种情况下只能说明应用已经退出了。这也正符合我们开头所说的，Android 本身是消息驱动，所以没有消息几乎是不可能的事；如果按照源码分析，Message.next() 方法可能会阻塞是因为如果消息需要延迟处理（sendMessageDelayed等），那就需要阻塞等待时间到了才会把消息取出然后分发出去。然后这个 ANR 完全是两个概念，ANR 本质上是因为消息未得到及时处理而导致的。同时，从另外一方面来说，对于 CPU 来说，线程无非就是一段可执行的代码，执行完之后就结束了。而对于 Android 主线程来说，不可能运行一段时间之后就自己退出了，那就需要使用死循环，保证应用不会退出。这样一想，其实这样的安排还是很有道理的。

这里，推荐一个说法：https://www.zhihu.com/question/34652589/answer/90344494
## 能不能让一个Message被加急处理？
系统内部可以通过开启同步屏障，并发送异步消息来优先处理异步消息。但是由于开启同步屏障的方法postSyncBarrier被隐藏，外部无法调用，并且发送异步消息的方法也是被hide的，所以在APP开发中实际是没有办法去发送一个加急消息的。
## Handler的同步屏障是什么？
在 Handler 中还存在了一种特殊的消息，它的 target 为 null，并不会被消费，仅仅是作为一个标识处于 MessageQueue 中。它就是 SyncBarrier (同步屏障)这种特殊的消息。当读取到这个同步屏障后就会开启一个循环取查找msg.isAsynchronous()为true的消息。如果找到则终止循环，优先执行msg.isAsynchronous()为true的这个Message。这个逻辑是在MessageQueue的next方法中实现的，代码如下

```java
// MessageQueue
Message next() {
			// ...省略无关代码
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) { // 读取到msg.target为null,即这里被添加了一个同步屏障
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do { // 开启循环查找Message
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous()); // 找到isAsynchronous为true的消息，则终止循环，执行后边的代码来处理这个Message
                }
              // ... 省略无关代码
        }
    }
```
同步屏障的原理其实就这么简单，那么这个同步屏障有什么用呢？其实同步屏障仅仅是在系统内部使用，我们无法通过公开的API来开启一个同步屏障。具体的使用案例来看View的绘制流程，在ViewRootImp类中有如下代码：

```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            // 调用MessageQueue的postSyncBarrier开启同步屏障
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            // 此处会发送一个synchronus为true的Message
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```
接着来看MessageQueue中的postSyncBarrier方法：

```java
	/*
     * @hide
     */
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```
可以看到postSyncBarrier使用了@hide注解隐藏了API，意味着这个方法外部是无法使用的。而在postSyncBarrier(long when)方法中为MessageQueue插入了一个Message，而这个Message并没有给target赋值。至此可以联系到next方法中对target为null情况的处理。

接下来看下mChoreographer.postCallback(
Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null)这个方法的源码


```java
    public void postCallbackDelayed(int callbackType,
            Runnable action, Object token, long delayMillis) {
        if (action == null) {
            throw new IllegalArgumentException("action must not be null");
        }
        if (callbackType < 0 || callbackType > CALLBACK_LAST) {
            throw new IllegalArgumentException("callbackType is invalid");
        }

        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }

    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        if (DEBUG_FRAMES) {
            Log.d(TAG, "PostCallback: type=" + callbackType
                    + ", action=" + action + ", token=" + token
                    + ", delayMillis=" + delayMillis);
        }

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                // 设置异步消息
                msg.setAsynchronous(true);
                // 发送异步消息到MessageQueue
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
```
可以看到在postCallbackDelayed方法中最终发送了一个异步消息，因为有了前边的同步屏障，那么这个异步消息就会优先执行。可见这里通过同步屏障和异步消息来保证了View的绘制会优先执行，避免了消息过多的情况下出现掉帧的情况。


## Handler的阻塞唤醒机制是什么？
Handler 中其实还存在着一种阻塞唤醒机制，我们都知道不断地进行循环是非常消耗资源的，有时我们 MessageQueue 中的消息都不是当下就需要执行的，而是要过一段时间，此时如果 Looper 仍然不断进行循环肯定是一种对于资源的浪费。

因此 Handler 设计了这样一种阻塞唤醒机制使得在当下没有需要执行的消息时，就将 Looper 的 loop 过程阻塞，直到下一个任务的执行时间到达或者一些特殊情况下再将其唤醒，从而避免了上述的资源浪费。

这个阻塞唤醒机制是基于 Linux 的 I/O 多路复用机制 epoll 实现的，它可以同时监控多个文件描述符，当某个文件描述符就绪时，会通知对应程序进行读/写操作。

Handler的阻塞机制的实现是在MessageQueue的next方法中，通过调用nativePollOnce的一个native方法实现的。

遇到以下情况，Java 层会调用 natvieWake 方法进行唤醒。

MessageQueue 类中调用 nativeWake 方法主要有下列几个时机：

调用 MessageQueue 的 quit 方法进行退出时，会进行唤醒

消息入队时，若插入的消息在链表最前端(最早将执行)或者有同步屏障时插入的是最前端的异步消息(最早被执行的异步消息)

移除同步屏障时，若消息列表为空或者同步屏障后面不是异步消息时

可以发现，主要是在可能不再需要阻塞的情况下进行唤醒。(比如加入了一个更早的任务，那继续阻塞显然会影响这个任务的执行)