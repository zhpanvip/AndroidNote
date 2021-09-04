## Window的添加过程概述

1. 通过getSystemService获取WindowManagerImpl实例，通过这个实例的LayoutParams设置窗口的类型、宽高等参数。
2. 调用WindowManagerImpl的addView方法添加View，WindowManagerImpl的addView 方法调用了WindowManagerGlobal去addView。
3. WindowManagerGlobal是一个单利，它的addView方法中会实例化ViewRootImpl，并且会调用ViewRootImpl的setView将要添加的View传递给ViewRootImpl。
4. ViewRootImpl 是 Window 和 View 之间的通信纽带，在ViewRootImpl的构造方法中做了两件重要的事情。
   - 通过WindowManagerGlobal 的getWindowSession方法发起一次IPC，请求获取 WMS 端的WindowSession的Binder服务。
     - getWindowSession中会通过getWindowManagerService拿到WMS的代理类IWindowManager。
     - IWindowManager 通过 openSession 方法向WMS发起一次IPC，请求获取 WMS 端的WindowSession的Binder服务。至此以上所有的操作都是在APP端完成的。
     - WMS接收到openSession的请求后调用自身的openSession方法实例化一个Session对象
     - 这个Session类继承了 IWindowSession.Stub，它是WMS端一个Binder通信的Stub端。
     - APP端的openSession请求最终会拿到这个WMS端的Session代理，并将其保存到ViewRootImpl的成员变量mWindowSession中。mWindowSession将作为APP与WMS通信的桥梁。
   - 实例化Window，这个Window是一个Binder服务对象可以看做是APP端的窗口对象，它会被传递给WMS，作为WMS向APP端发送消息的通道。
5. 完成ViewRootImpl的实例化后会调用ViewRootImpl的setView方法，setView方法中做了两件重要的事：
   - 首先调用requestLayout 方法post出来一个Message，这个Message一定会被插入到WMS发送来的输入事件之前，即先执行一次View的绘制流程。
   - mWindowSession的addToDisplay方法发起一次IPC，向Session请求添加窗口的操作。这个操作会在执行View的第一次绘制流程前执行，即将Window添加到WMS后才会执行第一次View的绘制流程，然后才会接着执行WMS发送来的输入事件等消息。
6. WMS端的Session会接受来自APP的addToDisplay请求，并且通过执行自己的onTranscation方法来调用自己的addToDisplay。
7. Session 的 addToDisplay 方法中又调用了WMS的addWindow方法，即最终Window的添加还是交给了WMS。
8. WMS的addWindow方法中会对要添加的Window进行校验、并获取Window的token以及创建WindowState
   - token标志着Window的分组
   - WindowState 与单个窗口一一对应，可以看做WMS中窗口的抽象实体，内部记录了Window的参数、z-index、状态信息等。

## WMS概述概述

WindowManagerService(简称WMS)是负责Android的窗口管理，但是它其实只负责管理，比如窗口的添加、移除、调整顺序等，至于图像的绘制与合成之类的都不是WMS管理的范畴，WMS更像在更高的层面对于Android窗口的一个抽象，真正完成图像绘制的是APP端，而完成图层合成的是SurfaceFlinger服务。

这里通过一个简单的悬浮窗口来探索一下大概流程：


```java
    TextView textView = new TextView(context);
    ...
    // 设置颜色 样式
    WindowManager mWindowManager  = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
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

通过 `getSystemService` 获取 `WindowManager` 的实例，WindowManager 是一个接口，它的实现类是WindowManagerImpl，接着实例化WindowManager.LayoutParams，将窗口类型指定为TOAST，设置颜色模式以及宽高。最后通过WindowManager的addView 方法将View添加到Window。由于WindowManager的实现类是WindowManagerImpl，因此 addView 方法的实际实现是在 WindowManagerImpl 中。

```java
// WindowManagerImpl

private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mDisplay, mParentWindow);
}
```

WindowManagerImpl中委托WindowManagerGlobal来进行 View 的添加。WindowManagerGlobal是一个单利，添加窗口精简后的代码如下：

```java
// WindowManagerGlobal
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        // 创建ViewRootImpl，内部会
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

WindowManagerGlobal添加View时首先创建了一个ViewRootImpl。ViewRootImpl 是 Window 和 View 之间的通信纽带，负责将 View 添加到 WMS ，处理 WMS 传入的触摸事件，通知 WMS 更新窗口大小，以及负责通知View的绘制与更新。

通过构造方法实例化 ViewRootImpl 后，将 LayoutParams 参数设置到了 ViewRootImpl 中，这个参数将在 View 的绘制流程中起到重要的作用。关于上边这部分代码需要重点分两部分来看，第一部分是ViewRootImpl实例化的过程，第二部分是调用ViewRootImpl 的 setView 的过程。

### 1. ViewRootImpl 的构造方法

在WindowManagerGlobal的addView方法中首先通过ViewRootImpl的构造方法实例化了一个ViewRootImplement对象。ViewRootImpl的构造方法如下：

```java
// ViewRootImpl
final IWindowSession mWindowSession;
public ViewRootImpl(Context context, Display display) {
    mContext = context;
     ...
       
    // 获取 服务端 Session 的代理
    mWindowSession = WindowManagerGlobal.getWindowSession();
  
    ...
      
    // 实例化 Window ，并将自身作为参数传入
    mWindow = new W(this);
}
```

ViewRootImpl 的构造方法中比较重要的是上边两行代码，虽然代码很少，但却涉及到向 WMS 的IPC，因此并不容易理解。这里仍然分为两部分来分析。

#### （1）获取 WMS 端 Session 的代理

> 温馨提示：想要读懂本节内容，必须有Binder和AIDL的相关知识储备，否则，你可能会看得一头雾水。

通过 `WindowManagerGlobal.getWindowSession` 获取 `IWindowSession`的实例，IWindowSession是一个由IWindowSession.aidl生成的接口，IWindowSession的实例实际上是服务端(WMS)  Session(Session自身是一个Binder，后边分析)的代理对象。

看下 WindowManagerGlobal 的 getWindowSession 方法的源码，如下：

```java
// WindowManagerGlobal  
private static IWindowSession sWindowSession;
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                InputMethodManager imm = InputMethodManager.getInstance();
                // 获取 WMS 的代理
                IWindowManager windowManager = getWindowManagerService();
                // 发起 IPC 调用 WMS 中的方法,并得到服务端Session的代理 sWindowSession
                sWindowSession = windowManager.openSession(
                        new IWindowSessionCallback.Stub() {
                            @Override
                            public void onAnimatorScaleChanged(float scale) {
                                 alueAnimator.setDurationScale(scale);
                            }
                ...
        return sWindowSession;
    }
}
```

`getWindowManagerService` 会去获取 WMS 的代理，到这里如果你了解 Binder 与 AIDL ，那么应该清楚，  getWindowManagerService方法返回的IWindowManager 是个服务端Binder的代理对象，IWindowManager的实现类(即代理类)是由AIDL自动生成。IWindowManager的实现类中会持有服务端（即 WMS）的Binder实例。

看下 ` getWindowManagerService`  的源码，如下：

```java
public static IWindowManager getWindowManagerService() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowManagerService == null) {
            // 获取 IWindowManager 的实现类的实例，即服务端Binder的代理对象。
            sWindowManagerService = IWindowManager.Stub.asInterface(
                    ServiceManager.getService("window"));
            try {
                sWindowManagerService = getWindowManagerService();
                       ValueAnimator.setDurationScale(sWindowManagerService.getCurrentAnimatorScale());
            } catch (RemoteException e) {
                Log.e(TAG, "Failed to get WindowManagerService, cannot set animator scale", e);
            }
        }
        return sWindowManagerService;
    }
}
```

通过getWindowManagerService方法获取到代理对象 windowManager 后，便通过windowManager的 openSession 方法会向WMS发起一次 IPC。在 WMS 的 onTransact 方法中接收到这次IPC请求，然后调用 WindowManagerService 中的openSession方法。由于这里是使用 AIDL 生成的代码，所以在Android 源码中并不能找到代理类和 IWindowManager.Stub 等相关类，这里理解过程即可。可以看下 WMS 中 openSession 的实现：

```java
// WindowManagerService
@Override
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
        IInputContext inputContext) {
    if (client == null) throw new IllegalArgumentException("null client");
    if (inputContext == null) throw new IllegalArgumentException("null inputContext");
    Session session = new Session(this, callback, client, inputContext);
    return session;
}
```

openSession 中会实例化一个 Session 对象。Session 继承了  **IWindowSession.Stub** 因此，它是一个Binder通信的Stub端，封装了每一个Session会话的操作。客户端这次IPC调用会在APP端拿到这个Session的代理，即WindowManagerGlobal中的 sWindowSession，也是ViewRootImpl中的mWindowSession。sWindowSession 接下来会作为 APP 与 WMS 间通信的桥梁。

#### （2）实例化Window

ViewRootImpl 的 第二部分是去实例化了一个 W 类型的 Window对象，并赋值给了成员变量mWindow。W类是ViewRootImpl中的内部类，它继承了 **IWindow.Stub** 也是一个Binder 服务对象。可以看做是APP端的窗口对象，它会传递给WMS，并作为WMS向APP端发送消息的通道。它的使用是在ViewRootImpl的setView方法中，接下来对setView的分析将更加清晰的认识这个mWindow。

### 2. ViewRootImpl 的 setView 过程

接下来看 ViewRootImp 的 setView 方法，源码简化后如下：

```java
// ViewRootImpl 
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
                  ...
                // 在添加窗口前先进行一次布局绘制，以确保之后WMS传来的各种事件能够正常处理。
                requestLayout();
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    mInputChannel = new InputChannel();
                }
                try {
                    // 真正添加添加窗口的地方
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                } catch (RemoteException e) {
                  ...
                }
  }                 
```



这里重要的有两部分，首先在添加Window前，调用了一次requestLayout，这个方法中会post出一个Message，排在所有WMS发送来的消息之前。即，先布局绘制一次View，View测量、布局、绘制好后，然后才会处理WMS传来的各种事件，如触摸事件等。

接着通过 mWindowSession 调用 addToDisplay 方法开始真正的添加Window。上一小节中已经分析了 mWindowSession，它也是服务端 Binder 的代理对象。因此 addToDisplay 是通过这个代理向Session发起的一次IPC。这段代码的执行过程其实是在requestLayout后View第一次绘制流程之前执行的。因为requestLayout是通过post 消息的方式执行的绘制流程。

接下来重点分析addToDisplay的过程，因为mWindowSession.addToDisplay这个方法是一次IPC，那么真正的执行应该是在WMS端执行的，这个流程的实现是在Session类中，当客户端发起addToDisplay请求后，Session的onTransact方法会收到请求，并执行真正添加窗口的逻辑。onTransact的逻辑是在 Session 父类 IWindowSession.Stub 中执行的，显然这里又是一个AIDL生成的类，依然无法看到代码。但是，最终会调用 Session 类中的 addToDisplay 方法，代码如下：

```java
// Session
@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
        Rect outOutsets, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
            outContentInsets, outStableInsets, outOutsets, outInputChannel);
}
```

可以看到 Session 的 addToDisplay 方法中并没有实质性的添加窗口的逻辑，而是又调用了 mService 的 addWindow 方法，这里的mServices 显然是 WindowManagerService ，也就是添加窗口的操作实质上还是在WMS中执行的。



### WMS 中添加Window的过程

饶了好大一圈，最终还是回到了WMS中来，看下WMS中addWindow方法简化后的代码：

```java 
public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            InputChannel outInputChannel) {
        ...
        synchronized(mWindowMap) {
        ...
            // 1.检查窗口是否已添加，不能重复添加
            if (mWindowMap.containsKey(client.asBinder())) {
                return WindowManagerGlobal.ADD_DUPLICATE_ADD;
            }
            // 2.对于子窗口类型的处理:
              // 1）必须有父窗口 
              // 2）父窗口不能是子窗口类型
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
            // 3.WindowManager.LayoutParams中有一个token字段，该字段标志着窗口的分组属性
            WindowToken token = mTokenMap.get(attrs.token);
            // 4.对于Toast类系统窗口，其attrs.token可以看做是null， 如果目前没有其他的类似系统窗口展示，token仍然获取不到，仍然要走新建流程
            if (token == null) {
            ...    
                token = new WindowToken(this, attrs.token, -1, false);
                addToken = true;
            } 
            ...
             // 5.新建WindowState，WindowState与窗口是一对一的关系，可以看做是WMS中与窗口的抽象实体，详细记录了窗口的参数、z-index、状态等各种信息
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



这里先了解几个概念

- **IWindow：** 是APP端暴露给WMS的抽象实例，对应的ViewRootImpl构造方法中实例化的mWindow对象，它与ViewRootImpl 一一对应，同时也是WMS向APP端发送消息的Binder。
- **WindowState：**WMS端窗口令牌，与IWindow，或者说窗口一一对应，是WMS管理窗口的重要依据。
- **WindowToken：** 窗口令牌，也可以看做是窗口分组的依据。在WMS中，与分组对应的数据结构是WindowToken，而与组内每个窗口对应的是WindowState。每块令牌（AppWindowToken、WindowToken）都对应一组窗口(WindowState)。Activity 与 Dialog 对应的是 AppWindowToken、PopupWindow对应的是普通的WindowToken。
- **AppToken：** 是ActivityRecord 里面的IApplicationToken.Stub appToken 代理，也是ActivityClientRecord里面的token。可以看做是Activity在其他服务（非WMS服务）的抽象。

至此，向WMS注册窗口的流程已经走完了，但是WMS还需要向SurfaceFlinger申请Surface才算真正分配了窗口。关于SurfaceFlinger这部分内容，请参考 [SurfaceFlinger](https://github.com/zhpanvip/AndroidNote/wiki/SurfaceFlinger)



参考：https://www.jianshu.com/p/40776c123adb
