ViewRootImpl是一个Android视图层次结构的顶部，是View和WindowManager的桥梁。ViewRootImpl与Choreographer协同完成View的绘制，也负责接收底层的触摸事件和对触摸事件的中转分发。

## 一、ViewRootImpl与窗口的添加

在WMS中提到WindowManager在添加窗口的时候会调用WindowManagerGlobal代理的addView方法来添加View。

```java
// WindowManagerGlobal
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ...
        ViewRootImpl root;
        View panelParentView = null;
        
        ...
        // 实例化ViewRootImpl    
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
        //将View添加到ViewRootImpl,开始绘制view
        root.setView(view, wparams, panelParentView);
        ...
    }
```

WindowManagerGlobal的addView、removeView、UpdateView都是通过ViewRootImpl来执行的。

调用的顺序为 WindowManagerImpl -> WindowManagerGlobal -> ViewRootImpl

而ViewRootImpl的setView如下：



```java
    // ViewRootImpl 
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
                ...
                // 添加到Window前先调用requestLayout进行布局，确定收到任何系统事件后重新布局。  
                // requestLayout方法会完成View的绘制流程 
                requestLayout();
                ...
                try {
                    ...
                    // 通过WindowSession完成Window的添加
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                } 
    }
```

mWindowSession类型是IWindowSession，它是一个Binder对象，真正的实现类是Session，也就是说这其实是一次IPC过程，远程调用了Session中的addToDisPlay方法。

```java
    @Override
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
            Rect outOutsets, InputChannel outInputChannel) {
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
                outContentInsets, outStableInsets, outOutsets, outInputChannel);
    }
```

这里的mService就是WindowManagerService，也就是说Window的添加请求，最终是通过WindowManagerService来添加的。



### Activity中的Window与ViewRootImpl

Activity的setContentView被调用时窗口还未被添加到WindowManager中，也就是此时的View不会开启测量绘制流程。当Activity的handleResumeActivity方法被调用时才会将DecorView添加到WindowManager。而在windowManager.addView方法中调用到windowManagerGlobal.addView，开始创建初始化ViewRootImpl。也就是上面添加Window的过程。



## 二、ViewRootImpl与View的绘制

ViewRootImpl的requestLayout开启View的绘制流程：



```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        // 检查线程是不是主线程，不是主线程则抛出异常
        checkThread();
        mLayoutRequested = true;
        // 开启绘制流程
        scheduleTraversals();
    }
}
```





```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
          	// 通过Handler发送同步屏障消息
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            // 通过Choreographer发送一个Runnable，Choreographer接收到Vsync信号后通过异步消息执行Runnable
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            ...
    }
```



```java
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            // 调用doTraversal开启View的绘制
            doTraversal();
        }
    }
```



```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        // 移除同步屏障，让非异步消息能够执行
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
				// 在这里开启View的测量、布局、绘制三大流程
        performTraversals();

       ...
    }
}
```





```java
private void performTraversals() {  
        // 测量  
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        // 布局
        performLayout(lp, desiredWindowWidth, desiredWindowHeight);
        // 绘制
        performDraw();
        }
        ...
    }  
```

## 三、ViewRootImpl与事件分发

关于触摸事件时如何从驱动到APP的过程可以参考[InputManagerService](https://github.com/zhpanvip/AndroidNote/wiki/InputManagerService)



ViewRootImpl接收到输入事件，

```java
public void dispatchInputEvent(InputEvent event, InputEventReceiver receiver) {
    // InputEvent是接收到的输入事件，可能是KeyEvent，也可能是MotionEvent
    // InputEventReceiver用来接收输入事件
    SomeArgs args = SomeArgs.obtain();
    args.arg1 = event;
    args.arg2 = receiver;
    // MSG_DISPATCH_INPUT_EVENT
    Message msg = mHandler.obtainMessage(MSG_DISPATCH_INPUT_EVENT, args);
    msg.setAsynchronous(true);
    // 通过Handler将消息发送到UI线程
    mHandler.sendMessage(msg);
}
```

mHandler是ViewRootImpl的内部类ViewRootHandler,在接收到上述消息后对消息进行处理

```java
   final class ViewRootHandler extends Handler {
        ...
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                 ...
                case MSG_DISPATCH_INPUT_EVENT: {
                    SomeArgs args = (SomeArgs)msg.obj;
                    InputEvent event = (InputEvent)args.arg1;
                    InputEventReceiver receiver = (InputEventReceiver)args.arg2;
                    // 在UI线程中调用enqueueInputEvent
                    enqueueInputEvent(event, receiver, 0, true);
                    args.recycle();
                } break;
                ...
            }
        }
   }
```



```java
  void enqueueInputEvent(InputEvent event,
            InputEventReceiver receiver, int flags, boolean processImmediately) {
        adjustInputEventForCompatibility(event);
        //将当前输入事件加入队列中排列等候执行
        QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);
        //输入事件添加进队列后，加入输入事件的默认尾部
        QueuedInputEvent last = mPendingInputEventTail;
        if (last == null) {
            mPendingInputEventHead = q;
            mPendingInputEventTail = q;
        } else {
            last.mNext = q;
            mPendingInputEventTail = q;
        }
        //队列计数
        mPendingInputEventCount += 1;
        ...
        //processImmediately则是判断是同步还是异步，前面我们在handler中调用的，因为是在UI线程，肯定是同步的，所以传递了参数是true，如果是异步，则调用到scheduleProcessInputEvents()
        if (processImmediately) {
            // 循环获取队列中的输入事件
            doProcessInputEvents();
        } else {
            // 最终通过Handler post出来，还是调用doProcessInputEvents方法
            scheduleProcessInputEvents();
        }
    }
```



上述代码是将消息封装成QueuedInputEvent，并加入链表的操作。QueuedInputEvent是一个链表结构，代码如下：

```java
  private static final class QueuedInputEvent {
        ...
        public QueuedInputEvent mNext;
        public InputEvent mEvent;
        public InputEventReceiver mReceiver;
        ...
}
```

doProcessInputEvents代码如下：

```java
void doProcessInputEvents() {
        //循环取出队列中的输入事件
        while (mPendingInputEventHead != null) {
            QueuedInputEvent q = mPendingInputEventHead;
            mPendingInputEventHead = q.mNext;
            ...
            mPendingInputEventCount -= 1;
            ...
            //分发处理
            deliverInputEvent(q);
        }

        //处理完所有输入事件，清楚标志
        if (mProcessInputEventsScheduled) {
            mProcessInputEventsScheduled = false;
            mHandler.removeMessages(MSG_PROCESS_INPUT_EVENTS);
        }
    }
```

通过deliverInputEvent进行分发：

```java
 private void deliverInputEvent(QueuedInputEvent q) {
        //校验输入事件
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
        }

        InputStage stage;
        if (q.shouldSendToSynthesizer()) {
            stage = mSyntheticInputStage;
        } else {
            stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
        }

        if (stage != null) {
            stage.deliver(q);
        } else {
            finishInputEvent(q);
        }
    }
```

这里InputStage则是一个实现处理输入事件责任的阶段，提供一系列处理输入事件的方法。每种InputStage可以处理一定的事件类型，比如AsyncInputStage、ViewPreImeInputStage、ViewPostImeInputStage等。当一个InputEvent到来时，ViewRootImpl会寻找合适它的InputStage来处理。



InputStage会先调用deliver开始处理，最终调用onProcess方法。对于View的点击事件可以用ViewPostImeInputStage来处理，它的onProcess方法如下：



```java
        @Override
        protected int onProcess(QueuedInputEvent q) {
            if (q.mEvent instanceof KeyEvent) {
                return processKeyEvent(q);
            } else {
                // If delivering a new non-key event, make sure the window is
                // now allowed to start updating.
                handleDispatchWindowAnimationStopped();
                final int source = q.mEvent.getSource();
                if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                    // 判断是触摸事件调用该方法
                    return processPointerEvent(q);
                } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
                    return processTrackballEvent(q);
                } else {
                    return processGenericMotionEvent(q);
                }
            }
        }
```



```java
        private int processPointerEvent(QueuedInputEvent q) {
            // 触摸事件
            final MotionEvent event = (MotionEvent)q.mEvent;
						
            mAttachInfo.mUnbufferedDispatchRequested = false;
            // 调用DocerView的dispatchPointerEvent,将分发事件交给View
            boolean handled = mView.dispatchPointerEvent(event);
            ...
            return handled ? FINISH_HANDLED : FORWARD;
        }
```



View的dispatchPointerEvent代码如下：

```java
 public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            // 如果是TouchEvent，则开启事件分发
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }
```



https://www.jianshu.com/p/9da7bfe18374

https://www.jianshu.com/p/9e6c54739217