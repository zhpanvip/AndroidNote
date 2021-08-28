## IMS

IMS是Android为了处理各种用户操作而抽象的一个服务，自身是一个Binder服务实体，在SystemServer进程启动时被初始化并注册到ServiceManager中。这个服务主要是用来提供一些输入设备信息的作用，作为Binder服务的作用比较小

```java
private void startOtherServices() {
        ...
        // 实例化IMS
        inputManager = new InputManagerService(context);
        // 实例化WMS，并将IMS与WMS关联
        wm = WindowManagerService.main(context, inputManager,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                !mFirstBoot, mOnlyCore);
        // WMS注册到ServiceManager
        ServiceManager.addService(Context.WINDOW_SERVICE, wm);
        // IMS注册到ServiceManager
        ServiceManager.addService(Context.INPUT_SERVICE, inputManager);
       ...
       }
```

 InputManagerService与WIndowManagerService几乎同时被添加，触摸事件的处理也涉及到了这两个服务。WMS持有IMS的引用，IMS负责触摸事件的采集，WMS负责找到目录窗口。

### 捕获触摸事件



IMS中会单独开一个线程专门读取触摸事件

```cpp
NativeInputManager::NativeInputManager(jobject contextObj,
        jobject serviceObj, const sp<Looper>& looper) :
        mLooper(looper), mInteractive(true) {
     ...
    sp<EventHub> eventHub = new EventHub();
    mInputManager = new InputManager(eventHub, this, this);
}
```

EventHub利用Linux的inotify和epoll机制监听设备事件。包括设备插拔以及各种触摸事件、点击事件。可看做一个不同的设备的集线器，主要面向/dev/Input目录下的设备节点，如/dev/input/event0上的事件就是输入事件。通过EventHub的getEvents就可以监听并获取该事件。

![](https://upload-images.jianshu.io/upload_images/1460468-b6d934a08d75bdfc.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/893/format/webp)



InputManager实例化的时候回创建一个InputReader对象，以及InputReaderThread Loop线程。loop线程作用是通过EventHub的getEvents获取Input事件。

![](https://upload-images.jianshu.io/upload_images/1460468-57833de14e98f7f0.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)



```cpp
InputManager::InputManager(
        const sp<EventHubInterface>& eventHub,
        const sp<InputReaderPolicyInterface>& readerPolicy,
        const sp<InputDispatcherPolicyInterface>& dispatcherPolicy) {
    <!--事件分发执行类-->
    mDispatcher = new InputDispatcher(dispatcherPolicy);
    <!--事件读取执行类-->
    mReader = new InputReader(eventHub, readerPolicy, mDispatcher);
    initialize();
}

void InputManager::initialize() {
    mReaderThread = new InputReaderThread(mReader);
    mDispatcherThread = new InputDispatcherThread(mDispatcher);
}

bool InputReaderThread::threadLoop() {
    mReader->loopOnce();
    return true;
}

void InputReader::loopOnce() {
        int32_t oldGeneration;
        int32_t timeoutMillis;
        bool inputDevicesChanged = false;
        Vector<InputDeviceInfo> inputDevices;
        {
      ...<!--监听事件-->
        size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);
       ....<!--处理事件-->
           processEventsLocked(mEventBuffer, count);
       ...
       <!--通知派发-->
        mQueuedListener->flush();
    }
```

通过上面的流程，输入事件就可以被读取，经过processEventsLocked初步封装成RawEvent，最后发通知，请求派发消息。

### 事件派发

在InputManager初始化的时候还会创建一个事件派发线程。在事件读取完毕后向派发线程发送一个通知，请派发线程去处理。



![](https://upload-images.jianshu.io/upload_images/1460468-5692afaef6fdc134.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

InputReader中的mQueueListener是一个InputDispatcher对象，所以mQueuedListener->flush就是通知InputDispatcher事件读取完毕。可以派发事件了。InputDispatcherThead是一个Looper，基于native的Looper实现了Handler消息处理模型。如果有Input事件到来，就唤醒处理事件，处理完毕继续睡眠等待。

```cpp
bool InputDispatcherThread::threadLoop() {
    mDispatcher->dispatchOnce();
    return true;
}

void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    {
      <!--被唤醒 ，处理Input消息-->
        if (!haveCommandsLocked()) {
            dispatchOnceInnerLocked(&nextWakeupTime);
        }
       ...
    }
    nsecs_t currentTime = now();
    int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
    <!--睡眠等待input事件-->
    mLooper->pollOnce(timeoutMillis);
}
```



dispatchOnceInnerLocked是具体的派发处理逻辑，看其中的一个分支：

```cpp
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
        ...
    case EventEntry::TYPE_MOTION: {
        MotionEntry* typedEntry = static_cast<MotionEntry*>(mPendingEvent);
        ...
        done = dispatchMotionLocked(currentTime, typedEntry,
                &dropReason, nextWakeupTime);
        break;
    }

bool InputDispatcher::dispatchMotionLocked(
        nsecs_t currentTime, MotionEntry* entry, DropReason* dropReason, nsecs_t* nextWakeupTime) {
    ...
    Vector<InputTarget> inputTargets;
    bool conflictingPointerActions = false;
    int32_t injectionResult;
    if (isPointerEvent) {
    <!--关键点1 找到目标Window-->
        injectionResult = findTouchedWindowTargetsLocked(currentTime,
                entry, inputTargets, nextWakeupTime, &conflictingPointerActions);
    } else {
        injectionResult = findFocusedWindowTargetsLocked(currentTime,
                entry, inputTargets, nextWakeupTime);
    }
    ...
    <!--关键点2  派发-->
    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}
```

触摸事件会通过findTouchedWindowTargetsLocked找到目标Window，进而通过dispatchEventLocked将消息发送到目标窗口。

### 查找触摸事件目标窗口

Android系统能够同时支持多块屏幕。每块屏幕都会被抽象成一个DisplayContent，内部维护了一个WindowList列表。用于记录当前屏幕的所有窗口，包括状态栏、导航栏、应用窗口、子窗口等。

如何确定触摸事件对应的窗口是状态栏、导航栏还是应用窗口呢？DisplayContent持有所有窗口信息，因此可以根据触摸事件的位置以及窗口的属性来确定将事件发送到哪个窗口。查找目标窗口的这个过程还跟窗口的状态、透明、分屏等信息有关系：

```cpp
int32_t InputDispatcher::findTouchedWindowTargetsLocked(nsecs_t currentTime,
        const MotionEntry* entry, Vector<InputTarget>& inputTargets, nsecs_t* nextWakeupTime,
        bool* outConflictingPointerActions) {
        ...
        sp<InputWindowHandle> newTouchedWindowHandle;
        bool isTouchModal = false;
        <!--遍历所有窗口-->
        size_t numWindows = mWindowHandles.size();
        for (size_t i = 0; i < numWindows; i++) {
            sp<InputWindowHandle> windowHandle = mWindowHandles.itemAt(i);
            const InputWindowInfo* windowInfo = windowHandle->getInfo();
            if (windowInfo->displayId != displayId) {
                continue; // wrong display
            }
            int32_t flags = windowInfo->layoutParamsFlags;
            if (windowInfo->visible) {
                if (! (flags & InputWindowInfo::FLAG_NOT_TOUCHABLE)) {
                    isTouchModal = (flags & (InputWindowInfo::FLAG_NOT_FOCUSABLE
                            | InputWindowInfo::FLAG_NOT_TOUCH_MODAL)) == 0;
         <!--找到目标窗口-->
                    if (isTouchModal || windowInfo->touchableRegionContainsPoint(x, y)) {
                        newTouchedWindowHandle = windowHandle;
                        break; // found touched window, exit window loop
                    }
                }
              ...
```



这个方法会根据点击位置、窗口z-order等特性从mWindowHandles中找到目标窗口。mWindowHandles是怎么来的？mWindowHandles会在InputDispatcher::setInputWindows中设置的。



```cpp
void InputDispatcher::setInputWindows(const Vector<sp<InputWindowHandle> >& inputWindowHandles) {
        ...
        mWindowHandles = inputWindowHandles;
       ...
}
```

WindowManagerService中的InputMonitor会调用setInputWindows。这个时机主要跟窗口增、改、删除等逻辑相关。如下图：

![](https://upload-images.jianshu.io/upload_images/1460468-aaf6043e34fc61ec.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

从上面流程可以理解为什么说WindowManagerService跟InputManagerService是相辅相成的了。

### 如何将事件发送到窗口上

获取事件和查找目标窗口的逻辑都是在SystemServer进程中的，要通知的目标窗口则是位于APP端的用户进程。所以需要进程间通信，这里使用的是Socket方式实现的。

```cpp
void InputDispatcher::dispatchEventLocked(nsecs_t currentTime,
        EventEntry* eventEntry, const Vector<InputTarget>& inputTargets) {
    pokeUserActivityLocked(eventEntry);
    for (size_t i = 0; i < inputTargets.size(); i++) {
        const InputTarget& inputTarget = inputTargets.itemAt(i);
        ssize_t connectionIndex = getConnectionIndexLocked(inputTarget.inputChannel);
        if (connectionIndex >= 0) {
            sp<Connection> connection = mConnectionsByFd.valueAt(connectionIndex);
            prepareDispatchCycleLocked(currentTime, connection, eventEntry, &inputTarget);
        } else {
        }
    }
}
```

逐层跟踪代码，最后会调用InputChannel的sendMessage函数，最后通过Socket发送到APP端。

![](https://upload-images.jianshu.io/upload_images/1460468-6977678cba0df4b7.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

两者通信的Socket是怎么来的？它牵扯到了WindowManagerService，在APP向WMS请求添加窗口的时候，会伴随着Input的创建。窗口的添加一定会调用ViewRootImpl的setView方法。

```java
// ViewRootImpl
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
                ...
            requestLayout();
            if ((mWindowAttributes.inputFeatures
                    & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                 <!--创建InputChannel容器-->
                mInputChannel = new InputChannel();
            }
            try {
                mOrigWindowType = mWindowAttributes.type;
                mAttachInfo.mRecomputeGlobalAttributes = true;
                collectViewAttributes();
                <!--添加窗口，并请求开辟Socket Input通信通道-->
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(),
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mOutsets, mInputChannel);
            }...
            <!--监听，开启Input信道-->
            if (mInputChannel != null) {
                if (mInputQueueCallback != null) {
                    mInputQueue = new InputQueue();
                    mInputQueueCallback.onInputQueueCreated(mInputQueue);
                }
                mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                        Looper.myLooper());
            }
```

在IwindowSession.aidl定义中InputChannel是out类型，也就是说需要服务端进行填充。服务端WMS填充如下：

```java
public int addWindow(Session session, IWindow client, int seq,
        WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
        Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
        InputChannel outInputChannel) {
          ...
        if (outInputChannel != null && (attrs.inputFeatures
                & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
            String name = win.makeInputChannelName();
            <!--关键点1创建通信信道 -->
            InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
            <!--本地用-->
            win.setInputChannel(inputChannels[0]);
            <!--APP端用-->
            inputChannels[1].transferTo(outInputChannel);
            <!--注册信道与窗口-->
            mInputManager.registerInputChannel(win.mInputChannel, win.mInputWindowHandle);
        }
```

WMS首先创建socketPair作为全双工通道，并分别填充client与Server的InputChannel中。之后让InputManager将Input通信信道与当前窗口ID绑定，这样就能知道哪个窗口用哪个信道通信了，最后通过BInder将outPutChannel回传到APP端。下面是SocketPair的创建代码：

```java
status_t InputChannel::openInputChannelPair(const String8& name,
        sp<InputChannel>& outServerChannel, sp<InputChannel>& outClientChannel) {
    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets)) {
        status_t result = -errno;
        ...
        return result;
    }

    int bufferSize = SOCKET_BUFFER_SIZE;
    setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
    <!--填充到server inputchannel-->
    String8 serverChannelName = name;
    serverChannelName.append(" (server)");
    outServerChannel = new InputChannel(serverChannelName, sockets[0]);
     <!--填充到client inputchannel-->
    String8 clientChannelName = name;
    clientChannelName.append(" (client)");
    outClientChannel = new InputChannel(clientChannelName, sockets[1]);
    return OK;
}
```

这里SocketPair的创建与访问其实是借助文件描述符，WMS需要借助Binder通信向APP端回传文件描述符fd，主要是内核层面实现两个进程fd的转换，窗口添加成功后，Socket被创建，被传到APP端。但是通信信道并未完全建立。因为还需要一个主动监听，因为消息到来时需要通知的。

![](https://upload-images.jianshu.io/upload_images/1460468-1de01a884b9c26d9.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

App端监听的手段是将Socket添加到Looper线程的epoll数组中。一有消息到来，Looper线程就会被唤醒。并获取事件内容。从代码来看，通信信道的打开是伴随WindowInputEventReceiver的创建来完成的。

![](https://upload-images.jianshu.io/upload_images/1460468-67a08564ae0785ae.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

消息到来，Looper根据fd找到对应的监听器NativeInputReceiver，并调用handleEvent处理对应事件。

```cpp
int NativeInputEventReceiver::handleEvent(int receiveFd, int events, void* data) {
   ...
    if (events & ALOOPER_EVENT_INPUT) {
        JNIEnv* env = AndroidRuntime::getJNIEnv();
        status_t status = consumeEvents(env, false /*consumeBatches*/, -1, NULL);
        mMessageQueue->raiseAndClearException(env, "handleReceiveCallback");
        return status == OK || status == NO_MEMORY ? 1 : 0;
    }
  ...
```

之后进一步读取事件并封装成java层对象传递给java层

```cpp
status_t NativeInputEventReceiver::consumeEvents(JNIEnv* env,
        bool consumeBatches, nsecs_t frameTime, bool* outConsumedBatch) {
        ...
    for (;;) {
        uint32_t seq;
        InputEvent* inputEvent;
        <!--获取事件-->
        status_t status = mInputConsumer.consume(&mInputEventFactory,
                consumeBatches, frameTime, &seq, &inputEvent);
        ...
        <!--处理touch事件-->
      case AINPUT_EVENT_TYPE_MOTION: {
        MotionEvent* motionEvent = static_cast<MotionEvent*>(inputEvent);
        if ((motionEvent->getAction() & AMOTION_EVENT_ACTION_MOVE) && outConsumedBatch) {
            *outConsumedBatch = true;
        }
        inputEventObj = android_view_MotionEvent_obtainAsCopy(env, motionEvent);
        break;
        }
        <!--回调处理函数-->
       if (inputEventObj) {
                    env->CallVoidMethod(receiverObj.get(),
                            gInputEventReceiverClassInfo.dispatchInputEvent, seq, inputEventObj);
                    env->DeleteLocalRef(inputEventObj);
                }
```

触摸事件最终被封装成InputEvent，并通过InputEventReceiver的dispatchInputEvent（WindowInputEventReceiver）进行处理。

后续事件的传递过程参看[ViewRootImpl与事件分发](post/ViewRootImpl.md#%E4%B8%89viewrootimpl%E4%B8%8E%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91)



## 内容来源

[十分钟了解Android触摸事件原理（InputManagerService）](https://www.jianshu.com/p/f05d6b05ba17)


