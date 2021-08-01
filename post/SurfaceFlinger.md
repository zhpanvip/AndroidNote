可以先了解[WMS与Window的添加过程](https://github.com/zhpanvip/AndroidNote/wiki/WMS%E6%A0%B8%E5%BF%83%E5%88%86%E6%9E%90)，WMS添加Window主要做了两个事情，第一是将Window添加到WMS，另外一个是通过SurfaceFlinger为Window申请Surface。

## SurfaceFlinger与图形系统概述

SurfaceFlinger是负责绘制应用UI的核心，它的功能是合成所有Surface。在Android系统中所有的View最终都是被渲染到Surface上的，绘制过程可以看做是一个生产者-消费者模型。Surface是BufferQueue的生产者，SurfaceFlinger可以看做是消费者。Android平台的所有Window都是由Surface支持的，所有课件的Surface渲染到显示设备上都是通过SurfaceFlinger来完成的。

![surface_rendered](http://gityuan.com/images/surfaceFlinger/surface_rendered.png))



1. **Image Stream Producers(图形流的生产者):** 可产生graphic buffers的生产者. 例如OpenGL ES, Canvas 2D, mediaserver的视频解码器.
2. **Image Stream Consumers(图形流的消费者):** 最场景的消费者便是SurfaceFlinger,它使用OpenGL和Hardware Composer来组合一组surfaces.
   - OpenGL ES应用能消费图形流, 比如camera app消费camera预览图形流;
   - 非OpenGL ES应用也能消费, 比如ImageReader类
3. **Window Manager:** 用于管理window, 这是一组view的容器. WM将手机的window元数据(包括屏幕放心,z-order等)信息发送给SurfaceFlinger,因此SurfaceFlinger 能使用这些信息来合成surfaces,并输出到显示设备.
4. **Hardware Composer(硬件合成器):** 这是显示子系统的硬件抽象层, SurfaceFlinger能将一些合成工作委托给Hardware Composer,从而降低来自OpenGL和GPUd的负载.
5. **Gralloc:** 全称为graphics memory allocator,图像内存分配器, 用于图形生产这来请求分配内存.





SurfaceFlinger进程是由init进程创建的，运行在独立的SurfaceFlinger进程。Android应用程序 必须跟SurfaceFlinger进程交互，才能完成将应用UI绘制到frameBuffer(帧缓冲区)。这个交互便涉及到 进程间的通信，采用的Binder IPC方式，名为”SurfaceFlinger”的Binder服务端运行在[SurfaceFlinger进程](http://gityuan.com/2017/02/11/surface_flinger/)。



![](http://gityuan.com/images/surfaceFlinger/class_surface.jpg)

Client，SurfaceFlinger这两个Binder服务运行在SurfaceFlinger进程. SurfaceComposerClient对象的两个成员变量分别跟着两个Binder服务通信：

- 其成员变量mClient通过Binder调用Client服务，
- 其成员变量mComposer经过Composer(位于SurfaceComposerClient.cpp文件)，ComposerService对象，再通过Binder调用SurfaceFlinger。

也就是说只需要调用`new SurfaceComposerClient()`便建立应用程序跟SurfaceFlinger服务建立连接， 获取了其中两个Binder的代理类。每一个app在SurfaceFlinger中都有一个Client对象相对应。

对于大多数的app来说都有3个layers: 状态栏,导航栏, 应用UI. 每一个layer都是独立更新的. 状态栏和导航栏是由系统进程负责渲染, app层是由app自己渲染,两者直接并没有协作.



## SurfaceFlinger与Surface

在ViewRootImpl的scheduleTraversals方法中使用Handler发送了一个同步屏障，这个同步屏障作用是阻塞普通的同步消息的执行，在同步屏障撤销之前Handler只会执行异步消息。

```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            // 通过Handler发送同步屏障阻塞同步消息
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            // 通过Choreographer发出一个mTraversalRunnable，会在这里执行
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

mTraversalRunnable任务的主要作用是：如果Surface未分配，则请求分配Surface，并测量、布局、绘图，其执行主体其实是performTraversals()方法。该方法包含了APP端View绘制大部分的逻辑：



```java
// ViewRootImpl
private void performTraversals() {
     // 申请Surface或者重新设置参数
     relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
     ...
    //measur过程
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
     ...
    //layout过程
    performLayout(lp, desiredWindowWidth, desiredWindowHeight);
     ...
    //draw过程
    performDraw();
}
```

relayoutWindow主要是通过mWindowSession.relayout向WMS申请或者更新Surface如下。在Binder通信中mSurface是一个out类型的参数，也就是Surface内部的内容需要WMS端负责填充，并回传给APP端：

```java
   private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {
       ...
        int relayoutResult = mWindowSession.relayout(
                mWindow, mSeq, params, ...  mSurface);
        ...
        return relayoutResult;
    }
```



relayout是如何向SurfaceFlinger申请Surface的？我们知道每个窗口都有一个WindowState与其对应，另外每个窗口也有自己的动画，比如入场/出厂动画，而WindowStateAnimator就是与WindowState的动画



```java
 public int relayoutWindow(Session session, IWindow client, int seq,... Surface outSurface) {
   			// 通过windowForClientLocked找到WindowState
        WindowState win = windowForClientLocked(session, client, false);
   
        WindowStateAnimator winAnimator = win.mWinAnimator;
        // 建一个SurfaceControl
        SurfaceControl surfaceControl = winAnimator.createSurfaceLocked();
           if (surfaceControl != null) {
             outSurface.copyFrom(surfaceControl);
            } else {
                outSurface.release();
            }
 }   
```

SurfaceControl会调用native函数nativeCreate(session, name, w, h, format, flags)创建Surface

```c++
static jlong nativeCreate(JNIEnv* env, jclass clazz, jobject sessionObj,
        jstring nameStr, jint w, jint h, jint format, jint flags) {
    ScopedUtfChars name(env, nameStr);
    // 取到SurfaceSession对象中SurfaceComposerClient对象
    sp<SurfaceComposerClient> client(android_view_SurfaceSession_getClient(env, sessionObj));
    // 创建Surface
    sp<SurfaceControl> surface = client->createSurface(
            String8(name.c_str()), w, h, format, flags);
    surface->incStrong((void *)nativeCreate);
    return reinterpret_cast<jlong>(surface.get());
}
```



```c
sp<SurfaceControl> SurfaceComposerClient::createSurface(
        const String8& name,
        uint32_t w,
        uint32_t h,
        PixelFormat format,
        uint32_t flags)
{
    sp<SurfaceControl> sur;
    if (mStatus == NO_ERROR) {
        sp<IBinder> handle;
        sp<IGraphicBufferProducer> gbp;
        // SurfaceComposerClient的mClient是一个BpSurfaceComposerClient对象
        status_t err = mClient->createSurface(name, w, h, format, flags,
                &handle, &gbp);
         // 根据返回的图层关键信息 创建SurfaceControl对象
        if (err == NO_ERROR) {
            sur = new SurfaceControl(this, handle, gbp);
        }
    }
    return sur;
}
```

SurfaceComposerClient的mClient其实是一个BpSurfaceComposerClient对象，它是SurfaceFlinger端Client在WMS端的代理，因此创建Surface的代码还是在SurfaceFlinger服务端的Client对象中，这里有两个关键的变量sp<IBinder> handle与 sp<IGraphicBufferProducer> gbp，前者标志在SurfaceFlinger端的图层，后者用来创建GraphicBuffer，两者类型都是IBinder类型，同时也是需要SurfaceFlinger填充的对象，这两者是一个图层对应的最关键的信息：

```cpp
status_t Client::createSurface(
        const String8& name,
        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
        sp<IBinder>* handle,
        sp<IGraphicBufferProducer>* gbp){
    ...
    // 这里并未直接创建 ，而是通过发送了一个MessageCreateLayer消息
    sp<MessageBase> msg = new MessageCreateLayer(mFlinger.get(),
            name, this, w, h, format, flags, handle, gbp);
    mFlinger->postMessageSync(msg);
    return static_cast<MessageCreateLayer*>( msg.get() )->getResult();
}
```

Client 并不会直接新建图层，而是向SurfaceFlinger发送一个MessageCreateLayer消息，通知SurfaceFlinger服务去执行，其handler代码如下：

```cpp
class MessageCreateLayer : public MessageBase {
        SurfaceFlinger* flinger;
        Client* client;
            virtual bool handler() {
            result = flinger->createLayer(name, client, w, h, format, flags,
                    handle, gbp);
            return true;
        }
    };
```

其实就是调用SurfaceFlinger的createLayer，创建一个图层，到这里才是真正的创建图层：

```cpp
status_t SurfaceFlinger::createLayer(
        const String8& name,
        const sp<Client>& client,
        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
        sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp)
{
    if (int32_t(w|h) < 0) {
        return BAD_VALUE;
    }

    status_t result = NO_ERROR;

    sp<Layer> layer;
  <!--关键点1 新建不同图层-->
    switch (flags & ISurfaceComposerClient::eFXSurfaceMask) {
        case ISurfaceComposerClient::eFXSurfaceNormal:
            result = createNormalLayer(client,
                    name, w, h, flags, format,
                    handle, gbp, &layer);
            break;
        case ISurfaceComposerClient::eFXSurfaceDim:
            result = createDimLayer(client,
                    name, w, h, flags,
                    handle, gbp, &layer);
            break;
        default:
            result = BAD_VALUE;
            break;
    }

    if (result != NO_ERROR) {
        return result;
    }
   ...
}
```

SurfaceFlinger会根据不同的窗口参数，创建不同类型的图层，这里只看一下createNormalLayer普通样式的图层

```cpp
status_t SurfaceFlinger::createNormalLayer(const sp<Client>& client,
        const String8& name, uint32_t w, uint32_t h, uint32_t flags, PixelFormat& format,
        sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp, sp<Layer>* outLayer)
{
    // initialize the surfaces
    switch (format) {
    case PIXEL_FORMAT_TRANSPARENT:
    case PIXEL_FORMAT_TRANSLUCENT:
        format = PIXEL_FORMAT_RGBA_8888;
        break;
    case PIXEL_FORMAT_OPAQUE:
        format = PIXEL_FORMAT_RGBX_8888;
        break;
    }
    <!--关键点 1 -->
    *outLayer = new Layer(this, client, name, w, h, flags);
    status_t err = (*outLayer)->setBuffers(w, h, format, flags);
    <!--关键点 2-->
    if (err == NO_ERROR) {
        *handle = (*outLayer)->getHandle();
        *gbp = (*outLayer)->getProducer();
    }
  return err;
}
```

可以看到 图层最终对应的是Layer，这里会新建一个Layer对象，Layer中包含着与这个图层对应的Handle及Producer对象，Handle可以看做是Surface的唯一性标识，不过好像没太大的作用，最多是一个标识，将来清理的时候有用。相比之下gbp = (*outLayer)->getProducer()比较重要，它实际是一个BufferQueueProducer对象，关系到共享内存的分配问题，后面会专门分析，这里到此打住，我们终于得到了一个图层对象，到这里之后，我们梳理一下，图层如何建立的：

- 首先APP端新建一个Surface图层的容器壳子，

- APP通过Binder通信将这个Surface的壳子传递给WMS，

- WMS为了填充Surface去向SurfaceFlinger申请真正的图层，

- SurfaceFlinger收到WMS请求为APP端的Surface分配真正图层

- 将图层相关的关键信息Handle及Producer传递给WMS

Layer建立之后，SurfaceFlinger会将图层标识信息Handle及Producer传递给WMS，WMS利用这两者创建一个SurfaceControl对象，之后再利用该对象创建Surface，具体代码如下：

```java
void getSurface(Surface outSurface) {
    outSurface.copyFrom(mSurfaceControl);
}

public void copyFrom(SurfaceControl other) {
long surfaceControlPtr = other.mNativeObject;
long newNativeObject = nativeCreateFromSurfaceControl(surfaceControlPtr);
synchronized (mLock) {
    setNativeObjectLocked(newNativeObject);
}
}
```

可以看到Surface的拷贝函数其实就是直接修改Surface native对象指针值，native的Surface对象中包含mGraphicBufferProducer对象，很重要，会被传递给APP端。

```cpp
static jlong nativeCreateFromSurfaceControl(JNIEnv* env, jclass clazz,
        jlong surfaceControlNativeObj) {

    sp<SurfaceControl> ctrl(reinterpret_cast<SurfaceControl *>(surfaceControlNativeObj));
    sp<Surface> surface(ctrl->getSurface());
    if (surface != NULL) {
        surface->incStrong(&sRefBaseOwner);
    }
    return reinterpret_cast<jlong>(surface.get());
}

sp<Surface> SurfaceControl::getSurface() const
{
    Mutex::Autolock _l(mLock);
    if (mSurfaceData == 0) {
        mSurfaceData = new Surface(mGraphicBufferProducer, false);
    }
    return mSurfaceData;
}
```

到这里WMS端Surface创建及填充完毕，并且Surface其实与WMS的SurfaceControl一一对应，当APP端需要在图层级别进行操控的时候，其实还是要依靠SurfaceControl的，WMS的Surface创建完毕后，需要传递给APP端，之后APP端就获得直接同SurfaceFlinger通信的能力，比如绘图与UI更新，怎传递的呢？我们知道Surface实现了Parcel接口，因此可以传递序列化的数据，其实看一下Surface nativeReadFromParcel就知道到底是怎么传递的了，利用readStrongBinder获取IGraphicBufferProducer对象的句柄，之后转化为IGraphicBufferProducer代理其实就是BpGraphicBufferProducer，之后利用BpGraphicBufferProducer构建Surface，这样APP端Surface就被填充完毕，可以同SurfaceFlinger通信了：

```cpp
static jlong nativeReadFromParcel(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject parcelObj) {
    Parcel* parcel = parcelForJavaObject(env, parcelObj);
    if (parcel == NULL) {
        doThrowNPE(env);
        return 0;
    }
     sp<Surface> self(reinterpret_cast<Surface *>(nativeObject));
    sp<IBinder> binder(parcel->readStrongBinder());
    if (self != NULL
            && (IInterface::asBinder(self->getIGraphicBufferProducer()) == binder)) {
        return jlong(self.get());
    }
    sp<Surface> sur;
    sp<IGraphicBufferProducer> gbp(interface_cast<IGraphicBufferProducer>(binder));
    if (gbp != NULL) {
        sur = new Surface(gbp, true);
        sur->incStrong(&sRefBaseOwner);
    }

    if (self != NULL) {
        self->decStrong(&sRefBaseOwner);
    }

    return jlong(sur.get());
}
```



http://gityuan.com/2017/02/05/graphic_arch/

https://www.jianshu.com/p/40776c123adb





