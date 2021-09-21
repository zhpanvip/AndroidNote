## ActivityThread初始化

Android程序的应用入口，main方法中主要初始化了Looper，实例化了ActivityThread，最后开启Looper循环。

```java
public static void main(String[] args) {
    // ...

    Looper.prepareMainLooper();
    // 初始化ActivityThread
    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
    sMainThreadHandler = thread.getHandler();
    }

    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

prepareMainLooper方法会通过myLooper去初始化主线程的Looper，主线程Looper是一个位于Looper内部的静态成员变量sMainLooper

```java
public static void prepareMainLooper() {
    // 初始化Looper,设置为禁止退出
    prepare(false);
synchronized (Looper.class) {
    if (sMainLooper != null) {
    throw new IllegalStateException("The main Looper has already been prepared.");
    }
    sMainLooper = myLooper();
    }
    }

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
    throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
    }

public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
    }
```

关于Looper更多知识参考Handler。

## ActivityThread与Activity启动

main方法中会调用ActivityThread的构造方法进行初始化，它的构造方法如下：

```java
ActivityThread() {
    // 初始化ResourcesManager
    mResourcesManager = ResourcesManager.getInstance();
    }
```

构造方法初始化了ResourcesManager的单利，ResourcesManager是APP的资源管理类，所有的资源文件都是由它来管理的。

ActivityThread实例化成功后会调用自身的的attach方法，

```java
private void attach(boolean system) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
// ...
final IActivityManager mgr = ActivityManagerNative.getDefault();
    try {
    mgr.attachApplication(mAppThread);
    }
    // ...
    }
    // ...
    }
```

在AMS中已经分析了ActivityManagerNative.getDefault()方法，这个方法会返回一个AMS的Binder代理ActivityManagerProxy。接着通过ActivityManagerProxy调用attachApplication方法，

```java
// ActivityManagerProxy
public void attachApplication(IApplicationThread app) throws RemoteException
    {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(app.asBinder());
    mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
    reply.readException();
    data.recycle();
    reply.recycle();
    }
```

这是一次跨进程的通信，注意ATTACH_APPLICATION_TRANSACTION这个参数，AMS收到请求后会在onTranscation方法中判断如果code是ATTACH_APPLICATION_TRANSACTION则会调用attachApplication

```java
// ActivityManagerService
@Override
public final void attachApplication(IApplicationThread thread) {
synchronized (this) {
    int callingPid = Binder.getCallingPid();
final long origId = Binder.clearCallingIdentity();
    attachApplicationLocked(thread, callingPid);
    Binder.restoreCallingIdentity(origId);
    }
    }
```

在AMS中调用了attachApplicationLocked方法

```java
// ActivityManagerService

private final boolean attachApplicationLocked(IApplicationThread thread,
    int pid) {
    // 通过进程id查找进程
    ProcessRecord app;
    if (pid != MY_PID && pid >= 0) {
synchronized (mPidsSelfLocked) {
    app = mPidsSelfLocked.get(pid);
    }
    } else {
    app = null;
    }

    if (app == null) {
    if (pid > 0 && pid != MY_PID) {
    // 杀进程操作
    Process.killProcessQuiet(pid);
    // 这边的TODO在后边版本中实现，避免多进程保活
    // TODO: killProcessGroup(app.info.uid, pid);
    } else {
    try {
    thread.scheduleExit();
    } catch (Exception e) {
    // Ignore exceptions.
    }
    }
    return false;
    }

    // ...

    try {
    // ...

    // 创建Application
    thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
    isRestrictedBackupMode || !normalMode, app.persistent,
    new Configuration(mConfiguration), app.compat,
    getCommonServicesLocked(app.isolated),
    mCoreSettingsObserver.getCoreSettingsLocked());

    }


    // See if the top visible activity is waiting to run in this process...
    if (normalMode) {
    try {
    // 用于创建Activity
    if (mStackSupervisor.attachApplicationLocked(app)) {
    didSomething = true;
    }
    }
    // ...
    }

    // ...


    return true;
    }
```

这个方法中有两点比较重要

1. thread.bindApplication 用于创建Application，这里的thread对象是ApplicationThread在AMS中的代理对象。这里的bindApplication方法最终会调用ApplicationThread.bindApplication方法，该方法会向ActivityThread的消息对应发送BIND_APPLICATION的消息。消息的处理最终会调用Application.onCreate方法，因此Application的onCreate方法的调用时机比任何Activity的onCreate方法都要早。

```java
public final void bindApplication(String processName, ApplicationInfo appInfo,
    // ...
    // 通过Handler发送 BIND_APPLICATION 的消息。                         
    sendMessage(H.BIND_APPLICATION, data);
    }
```

在ActivityThread的内部类H（继承Handler）中的handleMessage方法中判断如果是BIND_APPLICATION类型的消息则会调用了handleBindApplication(data)方法。

```java
private void handleBindApplication(AppBindData data) {
    // ...
  
		// 创建Instrumentation
    if (data.instrumentationName != null) {
        InstrumentationInfo ii = null;
        mInstrumentation.init(this, instrContext, appContext,
               new ComponentName(ii.packageName, ii.name), data.instrumentationWatcher,
               data.instrumentationUiAutomationConnection);
    } else {
        mInstrumentation = new Instrumentation();
    }

    try {
				// 创建Application
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;
        try {
            // 通过Instrumentaion初始化Application
            mInstrumentation.callApplicationOnCreate(app);
        } 
        
        // ...
    } 
    // ...
}
```

handleBindApplication方法做了简化，这里主要是初始化Instrumentation，并调用Instrumentation.callApplicationOnCreate方法执行Application的onCreate。callApplicationOnCreate源码如下：

```java
public void callApplicationOnCreate(Application app) {
    app.onCreate();
}
```

代码很简单，仅仅是调用了Application的onCreate方法，此时，我们项目里边声明的Application的onCreate方法被调用。

2. mStackSupervisor.attachApplicationLocked(app) 用于创建Activity。 mStackSupervisor是AMS的成员变量，为Activity堆栈管理辅助类的实例。该方法最终会通过ApplicationThread类的scheduleLaunchActivity方法，该方法也是通过向ActivityThread的消息队列发送创建Activity的Message，并最终在ActivityThread中完成Activity的创建。

```java
// ActivityThread#ApplicationThread
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
        CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

		// 发送启动Activity的Message
    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```

同样， 在ActivityThread的内部类H（继承Handler）中的handleMessage方法中判断如果是BIND_APPLICATION类型的消息则会调用了handleLaunchActivity(r, null)方法来启动Activity。

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
		// 调用performLaunchActivity创建Activity
    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        Bundle oldState = r.state;
        // Resume Activity
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed);

        if (!r.activity.mFinished && r.startsNotResumed) {

            try {
                r.activity.mCalled = false;
                // callActivityOnPause
                mInstrumentation.callActivityOnPause(r.activity);
								// ...

            } 
            // ...
    } 
    // ...  
}
```

Activity的创建是在performLaunchActivity方法中

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        // 通过mInstrumentation实例化Activity
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
			// ...
    } 

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (activity != null) {
            // 创建Activity Context
            Context appContext = createBaseContextForActivity(r, activity);
            // ...
          
            // attach Activity
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor);

            // Activity onCreate
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            r.activity = activity;
            if (!r.activity.mFinished) {
                // Activity start
                activity.performStart();
                r.stopped = false;
            }
     
				// ...
        r.paused = true;

        mActivities.put(r.token, r);

    }

    return activity;
}
```

可以看到performLaunchActivity中首先通过Instrumentation实例化了一个Activity对象，接着调用Activity的attach方法。

```java
// Activity.java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
    attachBaseContext(context);

    mFragments.attachHost(null /*parent*/);
		// 实例化PhoneWindow
    mWindow = new PhoneWindow(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
    if (info.uiOptions != 0) {
        mWindow.setUiOptions(info.uiOptions);
    }
    mUiThread = Thread.currentThread();

    // ...
		// PhoneWindow 关联 WindowManager
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    // Activity 中维护一个 WindowManagerImpl
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;
}
```

Activity 的 attach 方法做了两件比较重要的事情：

- 实例化了一个 PhoneWindow , PhoneWindow 中持有 DecorView，在 Activity 调用 setContentView 的时候回初始化 DecorView。
- 将 PhoneWindow通过 setWindowManager 关联 WindowManager，注意这里通过 `context.getSystemService` 来获取 WindowManager ，关于这个方法在WindowManagerService中已经有讲。它会获取到一个WindowManagerImpl实例。

```java
// Window.java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName) {
    setWindowManager(wm, appToken, appName, false);
}

/**
 * Set the window manager for use by this Window to, for example,
 * display panels.  This is <em>not</em> used for displaying the
 * Window itself -- that must be done by the client.
 *
 * @param wm The window manager for adding new windows.
 */
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated
            || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
    if (wm == null) {
        // 如果WindowManager为null，则重新获取
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    // 通过WindowManagerImpl的createLocalWindowManager方法实例化新的WindowManagerImpl
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

createLocalWindowManager方法中仅仅是实例化了一个WindowManagerImpl。

```java
// WindowManagerImpl.java
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    return new WindowManagerImpl(mDisplay, parentWindow);
}
```

可以看出Activity的attach方法中主要去实例化了PhoneWindow并关联了WindowManager。在执行完 Activity 的 attach 方法后，接着通过 Instrumentation 的 callActivityOnCreate 方法来执行 Activity 的 onCreate 生命周期。在Activity的onCreate方法中的setContentView方法会将Activity的布局文件设置到PhoneWindow的DecorView中，当然，是需要先实例化DecorView之后再执行添加 。此时DecorView虽然被实例化了，但并没有真正添加到Window中，也就是DecorView实际上还没有执行第一次执行 requestLayout，那么意味着 DecorView 还没有经历测量、布局、绘制的流程。这也是为什么在Activity的onCreate方法中无法获取到View的宽高的原因。

继续回到 handleLaunchActivity 这个方法，在调用 performLaunchActivity 成功创建 Activity 后，紧接着就调用了 handleResumeActivity 。这个方法简化后的代码如下：

```java
// ActivityThread.java
final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    // TODO Push resumeArgs into the activity for consideration
    ActivityClientRecord r = performResumeActivity(token, clearHide);

    if (r != null) {
        final Activity a = r.activity;

        // ...
      
        if (r.window == null && !a.mFinished && willBeVisible) {
            // 获取 PhoneWindow
            r.window = r.activity.getWindow();
            // 从 PhoneWindow 中获取 DecorView
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            // 获取 WindowManagerImpl，WindowManagerImpl实现WindowManager接口，WindowManager继承自ViewManager
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (a.mVisibleFromClient) {
                a.mWindowAdded = true;
                // 将 DecorView 添加至窗口
                wm.addView(decor, l);
            }

            // ...
    }
}
```

可以看到在这个方法中通过Activity首先拿到了DecorView与WindowManagerImpl。然后调用WindowManagerImpl的addView开启将DecorView添加至WindowManager的操作。关于Window的添加过程在WindowManagerService中已经详细解读。在向WMS执行添加Window之前先调用requestLayout方法，等到Vsync信号到来时就会首次执行DecorView的绘制流程，执行完之后便可以获取到View的宽高。因此，可以正常在onResume方法中拿到View的宽高。



[WMS添加Window过程](https://github.com/zhpanvip/AndroidNote/wiki/WMS%E6%A0%B8%E5%BF%83%E5%88%86%E6%9E%90)

