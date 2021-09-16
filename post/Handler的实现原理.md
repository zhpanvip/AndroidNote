
Android 应用是通过消息驱动运行的，在 Android 中一切皆消息，包括触摸事件，视图的绘制、显示和刷新等等都是消息。Handler 是消息机制的上层接口，平时开发中我们只会接触到 Handler 和 Message，内部还有 MessageQueue 和 Looper 两大助手共同实现消息循环系统。
**（1）Handler**
通过Handler的sendXXX或者postXXX来发送一个消息，这里要注意post(Runnable r)方法也会将Runnable包装成一个Message，代码如下：

```java
	public final boolean post(Runnable r){
    	return  sendMessageDelayed(getPostMessage(r), 0);
    }
    public final boolean postDelayed(Runnable r, long delayMillis){
    	return sendMessageDelayed(getPostMessage(r), delayMillis);
    }
private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

```
从代码中可以看到将Runnable赋值给了Message.callback了。最终sendXXX和postXXX都会调用到sendMessageAtTime，代码如下：

```java
 public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```
在这个方法中最终调用了enqueueMessage方法，这里注意将this赋值给了Message.target，而此处this就是Handler。

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        //转到 MessageQueue 的 enqueueMessage 方法
        return queue.enqueueMessage(msg, uptimeMillis);
    }

```
enqueueMessage方法最终调用了MessageQueue的enqueueMessage方法，将消息放入队列。
**（2）MessageQueue**
MessageQueue是一个优先级队列，核心方法是enqueueMessage和next方法，也就是将插入队列，将消息取出队列的操作。
之所以说MessageQueue是一个优先级队列是因为enqueueMessage方法中会根据Message的执行时间来对消息插入，这样越晚执行的消息会被插入到队列的后边。

而next方法是一个死循环，如果队列中有消息，则next方法会将Message移除队列并返回该Message，如果队列中没有消息该方法则会处于阻塞状态。

**（3）Looper**
Looper可以理解为一个消息泵，Looper的核心方法是loop。注意loop方法的第一行会首先通过myLooper来得到当前线程的Looper,接着拿到Looper中的MessageQueue，然后开启一个死循环，它会不断的通过MessageQueue的next方法将消息取出来，并执行。代码如下：

```java
public static void loop() {
        final Looper me = myLooper();// 这里要特别注意，是从ThreadLocal中拿到当前线程的Looper。
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        for (;;) {
            //从 MessageQueue 中取消息
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
			//通过 Handler 分发消息
            msg.target.dispatchMessage(msg);
            //回收消息
            msg.recycleUnchecked();
        }
    }
```
可以看到在取出Message后则会调用Message.target调用dispatchMessage方法，这里target就是Handler，它是在Handler的enqueueMessage时赋值的。紧接着将Message进行了回收。
接下来再回到Handler看dispatchMessage，代码如下：

```java
	public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            //通过 handler.postXxx 形式传入的 Runnable
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                //以 Handler(Handler.Callback) 写法
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            //以 Handler(){} 内存泄露写法
            handleMessage(msg);
        }
    }
```
可以看到，这里最终会调用到我们自己的实现方法。





