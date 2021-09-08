IdleHandler是一个位于MessageQueue中的接口，源码如下：

```java
public static interface IdleHandler {
    /**
     * Called when the message queue has run out of messages and will now
     * wait for more.  Return true to keep your idle handler active, false
     * to have it removed.  This may be called if there are still messages
     * pending in the queue, but they are all scheduled to be dispatched
     * after the current time.
     */
    boolean queueIdle();
}
```

IdleHandler中有一个queueIdle方法，根据方法上的注释可以知道这个方法会在MessageQueue中没有消息时或者只有延迟消息时才会被调用。方法的返回值是一个boolean值，返回true会将IdleHandler保留，否则会将其移除。

MessageQueue中关于IdleHandler的方法有两个：

```java
// 存放IdleHandler的集合
private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();

// 添加IdleHandler
public void addIdleHandler(@NonNull IdleHandler handler) {
    if (handler == null) {
        throw new NullPointerException("Can't add a null IdleHandler");
    }
    synchronized (this) {
        mIdleHandlers.add(handler);
    }
}

// 移除IdleHandler
public void removeIdleHandler(@NonNull IdleHandler handler) {
    synchronized (this) {
        mIdleHandlers.remove(handler);
    }
}
```

在MessageQueue中维护了一个IdleHandler集合，并且提供了添加IdleHandler和移除IdleHandler的方法。可以通过 `Looper.myQueue().addIdleHandler(new Idler())` 添加一个IdleHandler。

mIdleHandlers是在MessageQueue中被执行的，相关代码如下;

```java
Message next() {
		// 初次执行pendingIdleHandlerCount 的值是-1
    int pendingIdleHandlerCount = -1; 
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
				// 阻塞方法，调用native层的epoll监听文件描述符的写入事件来实现
        // 如果 nextPollTimeoutMillis = -1，则会一直阻塞，不会超时
        // 如果 nextPollTimeoutMillis = 0，不会阻塞,立即返回
        // 如果 nextPollTimeoutMillis >0,表示最长的阻塞时间为nextPollTimeoutMillis，如果期间被唤醒则立即返回
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
           	// ... 省略消息处理相关逻辑
          
						// 1.pendingIdleHandlerCount默认值为-1，满足第一个条件
            // 2.没有消息，或者当前没有要执行的消息
            // 同时满足以上两个条件才会进入if语句
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                // 获取IdleHandler的个数并赋值给pendingIdleHandlerCount
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            
            if (pendingIdleHandlerCount <= 0) {
                // 没有要执行的IdleHandler
                mBlocked = true;
                continue;
            }
						
            if (mPendingIdleHandlers == null) {
                // 初始化一个最小值为4的IdleHandler数组
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            // 将mIdleHandlers添加到数组中
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

				// 遍历数组执行IdleHandler
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            // 取出IdleHandler
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                // 执行IdleHandler的queueIdle，返回值表示是否要保留IdleHandler
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    // 不保留则从集合中移除IdleHandler
                    mIdleHandlers.remove(idler);
                }
            }
        }

        pendingIdleHandlerCount = 0;

        nextPollTimeoutMillis = 0;
    }
}
```



- IdleHandler是在MessageQueue中没有可用消息时发生作用的，如果想要在消息处理空闲时做一些事情，就可以在当前线程的消息队列中加入IdleHandler，并重写Idle的queueIdle函数。

- IdleHandler的作用次数可为一次或者多次，取决于queueIdle方法的返回值，如果返回false，那么执行完这个方法后便会从集合中移除，如果返回true，意味着每次只要消息队列空闲就会调用一次queueIdle方法。

## 相关面试题

### 1. IdleHandler是什么，有什么用？

- IdleHandler 是 Handler 提供的一种在消息队列空闲时，执行任务的机制；

- 当 MessageQueue 当前没有立即需要处理的消息时（消息队列为空，或者消息未到执行时间），会执行 IdleHandler；

### 2. 如果消息队列一直没有消息，为什么queueIdle方法不会被无限调用（死循环）？

当MessageQueue为空时，会循环遍历一遍mIdleHandler，并执行IdleHandler.queueIdle方法。如果某些IdleHandler的queueIdle返回了true，则会被保留在mIdleHandlers中，下次消息队列空闲时依然会执行。

在调用next方法的时候pendingIdleHandlerCount被赋值为-1，也就是开始循环前值是-1，在循环中如果pendingIdleHandlerCount小于0时才会通过mIdleHandlers.size给pendingIdleHandlerCount赋值，即只有第一次循环才会改变pendingIdleHandlerCount的值。然后在MessageQueue空闲的时候执行一遍IdleHandler。执行完之后pendingIdleHandlerCount被赋值了0，再次进入循环的时候判断pendingIdleHandlerCount <= 0则直接执行continue跳过了后边的执行逻辑。

### 2. MessageQueue 提供了 add/remove IdleHandler 的方法，是否需要成对使用？

不是必须要成对使用。MessageQueue中会根据IdleHandler中queueIdle方法的返回值来决定是否移除MessageQueue中的IdleHandler。

### 3. 是否可以将一些不重要的启动服务移到IdleHandler中处理？

不建议这样做，因为IdleHandler的执行时机不可控，如果MessageQueue一直有待处理的消息，那么IdleHandler会一直得不到执行。

### 4. IdleHandler 的 queueIdle运行在哪个线程？

 queueIdle() 运行的线程，只和当前 MessageQueue 的 Looper 所在的线程有关；子线程一样可以构造 Looper，并添加 IdleHandler；

### 5. 为什么 Activity.finish() 之后 10s 才 onDestroy ？

**问题描述：** 在A Activity启动B Activity，并结束A 页面，B页面在启动时进行大量的动画场景，源源不断的向主线程消息队列发送消息。A Activity的onPause正常执行，但是onStop与onDestory都延迟了10s才执行。为什么会出现这样的情况？

Activity 的 onStop/onDestroy 是依赖 IdleHandler 来回调的，正常情况下当主线程空闲时会调用。但是由于某些特殊场景下的问题，导致主线程迟迟无法空闲，onStop/onDestroy 也会迟迟得不到调用。但这并不意味着 Activity 永远得不到回收，系统提供了一个兜底机制，当 onResume 回调 10s 之后，如果仍然没有得到调用，会主动触发。