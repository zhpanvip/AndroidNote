## Window的添加过程概述

1. 通过getSystemService获取WindowManagerService的代理对象WindowManagerImpl，调用其addView将View添加到WindowManager。
2. WindowManagerImpl的addView方法委托WindowManagerGlobal进行添加View。
3. WindowManagerGlobal是一个单利，它的addView方法中会实例化ViewRootImpl，并且会调用ViewRootImpl的setView将View添加到ViewRootImpl中。
4. ViewRootImpl是Window和View之间的通信纽带。它会将View添加到WMS，并且处理WMS传来的触摸事件及View的绘制与更新。
5. ViewRootImpl的构造方法中会通过WindowManagerGlobal的getWindowSession获取Binder的服务代理WindowSession，还会实例化一个mWindow，
6. mWindow是一个W类继承了IWindow.Stub的Binder服务对象，可视为APP端的窗口对象，主要作用是传递给WMS，并作作为WMS向APP端发送消息的通道。
7. WindowManagerGlobal的getWindowSession方法中通过WindowManagerService的代理的openSession方法打开一个Session返回给APP端
8. Session继承了IWindowSession.Stub，是一个Binder通信的Stub端，它封装了每一个Session会话的操作。
9. 接下来调用Session的addToDisplayWithoutInputChannel添加一个窗口
10. addToDisplayWithoutInputChannel方法中调用WindowManagerService的addWindow来添加Window，即最终通过WMS来添加Window


## WMS概述概述

WindowManagerService(简称WMS)是负责Android的窗口管理，但是它其实只负责管理，比如窗口的添加、移除、调整顺序等，至于图像的绘制与合成之类的都不是WMS管理的范畴，WMS更像在更高的层面对于Android窗口的一个抽象，真正完成图像绘制的是APP端，而完成图层合成的是SurfaceFlinger服务。

这里通过一个简单的悬浮窗口来探索一下大概流程：


```java
    TextView textView = new TextView(context);
    ...
    // 设置颜色 样式
    WindowManager mWindowManager##  = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
    WindowManager.LayoutParams wmParams = new WindowManager.LayoutParams();
    wmParams.type = WindowManager.LayoutParams.TYPE_TOAST;
    wmParams.format = PixelFormat.RGBA_8888;
    wmParams.width = 800;
    wmParams.height = 800;
    mWindowManager.addView(textView, wmParams);
```

以上代码可以在主屏幕上添加一个TextView并展示，并且这个TextView独占一个窗口。在利用WindowManager.addView添加窗口之前，TextView的onDraw不会被调用，也就说View必须被添加到窗口中，才会被绘制，或者可以这样理解，只有申请了依附窗口，View才会有可以绘制的目标内存。当APP通过WindowManagerService的代理向其添加窗口的时候，WindowManagerService除了自己进行登记整理，还需要向SurfaceFlinger服务申请一块Surface画布，其实主要是画布背后所对应的一块内存，只有这一块内存申请成功之后，APP端才有绘图的目标，并且这块内存是APP端同SurfaceFlinger服务端共享的，这就省去了绘图资源的拷贝，示意图如下：

![](https://upload-images.jianshu.io/upload_images/1460468-3cddb5d035046beb.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

以上是抽象的图层对应关系，可以看到，APP端是可以通过unLockCanvasAndPost直接同SurfaceFlinger通信进行重绘的，就是说图形的绘制同WMS没有关系，WMS只是负责窗口的管理，并不负责窗口的绘制


## Window的添加过程

添加一个只有TextView的悬浮窗Window，代码如下：

```java
private void addTextViewWindow(Context context){

    TextView mview=new TextView(context);
    ...<!--设置颜色 样式-->
    <!--关键点1-->
    WindowManager mWindowManager = (WindowManager) context.getApplicationContext().getSystemService(Context.WINDOW_SERVICE);
    WindowManager.LayoutParams wmParams = new WindowManager.LayoutParams();
    <!--关键点2-->
    wmParams.type = WindowManager.LayoutParams.TYPE_TOAST;
    wmParams.format = PixelFormat.RGBA_8888;
    wmParams.width = 800;
    wmParams.height = 800;
    <!--关键点3-->
    mWindowManager.addView(mview, wmParams);
}

```

关键点1获取WindowManagerService的代理对象--WindowManagerImpl的实例。接着通过WindowManager.LayoutParams.type指定窗口类型为TOAST。最后调用addView将View添加到WMS。



```java
// WindowManagerImpl
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mDisplay, mParentWindow);
}
```

WindowManagerImpl中委托WindowManagerGlobal来进行添加。WindowManagerGlobal是一个单利，添加窗口精简后的代码如下：

```java
// WindowManagerGlobal
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        // 创建ViewRootImpl
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
    }
   // 将View添加到ViewRootImpl
    try {
        root.setView(view, wparams, panelParentView);
    }           
     ...  
}
```



WindowManagerGlobal添加View时首先创建了一个ViewRootImpl。ViewRootImpl是Window和View之间的通信纽带，负责将View添加到WMS，处理WMS传入的触摸事件，通知WMS更新窗口大小，以及负责通知View的绘制与更新。

ViewRootImpl的构造方法如下：

```java
public ViewRootImpl(Context context, Display display) {
    mContext = context;
    mWindowSession = WindowManagerGlobal.getWindowSession();
    mWindow = new W(this);
}
```

通过WindowManagerGlobal.getWindowSession获取Binder服务代理WindowSession，它是APP端向WMS发送消息的通道。mWindow是一个**W extends IWindow.Stub** Binder服务对象，可视为APP端的窗口对象，主要作用是传递给WMS，并作为WMS向APP端发送消息的通道。



首先通过getWindowManagerService 获取WMS的代理，之后通过WMS的代理在服务端open一个Session，并在APP端获取该Session的代理：

```jsx
 public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                InputMethodManager imm = InputMethodManager.getInstance();
                <!--关键点1-->
                IWindowManager windowManager = getWindowManagerService();
                <!--关键点2-->
                sWindowSession = windowManager.openSession(***）
                ...
        return sWindowSession;
    }
}
```

getWindowManagerService其实就是获得WindowManagerService的代理。openSession会打开一个Session返回给APP端。而**Session extends IWindowSession.Stub**，是一个Binder通信的Stub端，封装了每一个Session会话的操作。

```java
@Override
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
        IInputContext inputContext) {
    if (client == null) throw new IllegalArgumentException("null client");
    if (inputContext == null) throw new IllegalArgumentException("null inputContext");
    Session session = new Session(this, callback, client, inputContext);
    return session;
}
```



接下调用Session的addToDisplayWithoutInputChannel方法添加一个窗口：

```java
@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
        Rect outOutsets, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
            outContentInsets, outStableInsets, outOutsets, outInputChannel);
}
```

最终又通过WMS来addWindow。

```java
public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            InputChannel outInputChannel) {
        ...
        synchronized(mWindowMap) {
        ...
        		// 1.不能重复添加窗口
            if (mWindowMap.containsKey(client.asBinder())) {
                return WindowManagerGlobal.ADD_DUPLICATE_ADD;
            }
            // 2.对于子窗口类型的处理 1、必须有父窗口 2，父窗口不能是子窗口类型
            if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
                parentWindow = windowForClientLocked(null, attrs.token, false);
                if (parentWindow == null) {
                    return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
                }
                if (parentWindow.mAttrs.type >= FIRST_SUB_WINDOW
                        && parentWindow.mAttrs.type <= LAST_SUB_WINDOW) {
                    return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
                }}
           ...
           boolean addToken = false;
            //3.根据IWindow 获取WindowToken WindowToken是窗口分组的基础，每个窗口必定有一个分组
            WindowToken token = mTokenMap.get(attrs.token);
          //4.对于Toast类系统窗口，其attrs.token可以看做是null， 如果目前没有其他的类似系统窗口展示，token仍然获取不到，仍然要走新建流程
            if (token == null) {
            ...    
                token = new WindowToken(this, attrs.token, -1, false);
                addToken = true;
            } 
            ...
             // 5. 新建WindowState，WindowState与窗口是一对一的关系，可以看做是WMS中与窗口的抽象实体
            WindowState win = new WindowState(this, session, client, token,
                    attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);
            ...
            if (addToken) {
                mTokenMap.put(attrs.token, token);
            }
            win.attach();
            mWindowMap.put(client.asBinder(), win);
            ... 
          
          addWindowToListInOrderLocked(win, true);
        return res;
    }
```

至此，APP向WMS添加窗口的流程就结束了。但是WMS还要向SurfaceFlinger申请Surface才算真正完成窗口的添加。



## SurfaceFlinger

WMS在添加窗口后需要向SurfaceFlinger申请Surface。首先WMS需要先获得SurfaceFlinger的代理。在WindowState对象创建后利用win.attach()为APP申请建立SurfaceFlinger的链接。

```java
void attach() {
    if (WindowManagerService.localLOGV) Slog.v(
    mSession.windowAddedLocked();
}

void windowAddedLocked() {
    if (mSurfaceSession == null) {
       // SurfaceSession新建
        mSurfaceSession = new SurfaceSession();
        mService.mSessions.add(this);
       ...
    }
    mNumWindow++;
}
```

SurfaceSession持有SurfaceFlinger的代理SurfaceComposerClient，如下：



```java
    public SurfaceSession() {
        mNativeClient = nativeCreate();
    }

    static jlong nativeCreate(JNIEnv* env, jclass clazz) {
        SurfaceComposerClient* client = new SurfaceComposerClient();
        client->incStrong((void*)nativeCreate);
        return reinterpret_cast<jlong>(client);
    }
```

Session与APP进程一一对应，它会进一步为当前进程建立SurfaceSession会话。



Session是APP同WMS通信的通道，SurfaceSession是WMS为APP向SurfaceFlinger申请的通信通道，同样 SurfaceSession与APP也是一一对应的，既然是同SurfaceFlinger通信的信使，那么SurfaceSession就应该握着SurfaceFlinger的代理，其实就是SurfaceComposerClient里的ISurfaceComposerClient mClient对象，它是SurfaceFlinger为每个APP封装一个代理，也就是 **进程 <-> Session <-> SurfaceSession <-> SurfaceComposerClient <-> ISurfaceComposerClient(BpSurfaceComposerClient) **五者是一条线。



窗口的添加流程简化如下，这里暂且忽略窗口的分组管理。

- APP首先去WMS登记窗口
- WMS端登记窗口
- APP新建Surface壳子，请求WMS填充Surface
- WMS请求SurfaceFlinger分配窗口图层
- SurfaceFlinger分配Layer，将结果回传给WMS
- WMS将窗口信息填充到Surface传输到APP
- APP端获得填充信息，获取与SurfaceFlinger通信的能力



[Android窗口管理分析（1）：View如何绘制到屏幕上的主观理解](https://www.jianshu.com/p/e4b19fc36a0e)