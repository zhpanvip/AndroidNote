## App启动流程分析

AndroidManifest文件中配置默认启动的Activity：	

```xml
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
```

系统在启动时会启动PMS服务，解析所有APP的AndroidManifest文件，在Launcher启动的时候就会将APP的图标及启动信息封装，在点击桌面上的APP图标时则出发APP的启动流程。APP的启动流程时序图如下：

![](https://gitee.com/zhpanvip/images/raw/master/project/article/framework/app_start1.webp)

## Launcher

Launcher是Android的桌面应用，在framework下有一个Launcher类，它集成子StateFulActivity，从本质上讲Launcher其实本身是一个Activity。点击APP的图标后，则会执行Launcher的startActivitySafely方法，代码如下：

```java
// packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
public class Launcher extends StatefulActivity<LauncherState> implements LauncherExterns,
        Callbacks, InvariantDeviceProfile.OnIDPChangeListener, PluginListener<OverlayPlugin> {
          
    // ...
          
    @Override
    public boolean startActivitySafely(View v, Intent intent, ItemInfo item,
            @Nullable String sourceContainer) {
				// 调用父类的startActivitySafely
        boolean success = super.startActivitySafely(v, intent, item, sourceContainer);
        if (success && v instanceof BubbleTextView) {
            BubbleTextView btv = (BubbleTextView) v;
            btv.setStayPressed(true);
            addOnResumeCallback(btv);
        }
        return success;
    }       
}
```

Launcher的startActivitySafely直接调用到了其父类的startActivitySafely：

```java
// packages/apps/Launcher3/src/com/android/launcher3/BaseDraggingActivity.java
public boolean startActivitySafely(View v, Intent intent, @Nullable ItemInfo item,
        @Nullable String sourceContainer) {
    if (mIsSafeModeEnabled && !PackageManagerHelper.isSystemApp(this, intent)) {
        Toast.makeText(this, R.string.safemode_shortcut_error, Toast.LENGTH_SHORT).show();
        return false;
    }

    Bundle optsBundle = (v != null) ? getActivityLaunchOptionsAsBundle(v) : null;
    UserHandle user = item == null ? null : item.user;

    // 注意这里给Intent设置了FLAG_ACTIVITY_NEW_TASK，意味着新启动的Activity位于一个新任务栈
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    if (v != null) {
        intent.setSourceBounds(getViewBounds(v));
    }
    try {
   
        if (isShortcut) {
            // 支持3D Touch功能
            startShortcutIntentSafely(intent, optsBundle, item, sourceContainer);
        } else if (user == null || user.equals(Process.myUserHandle())) {
            // 启动Activity
            startActivity(intent, optsBundle);
            AppLaunchTracker.INSTANCE.get(this).onStartApp(intent.getComponent(),
                    Process.myUserHandle(), sourceContainer);
        } else {
            getSystemService(LauncherApps.class).startMainActivity(
                    intent.getComponent(), user, intent.getSourceBounds(), optsBundle);
            AppLaunchTracker.INSTANCE.get(this).onStartApp(intent.getComponent(), user,
                    sourceContainer);
        }
        getUserEventDispatcher().logAppLaunch(v, intent, user);
        if (item != null) {
            InstanceId instanceId = new InstanceIdSequence().newInstanceId();
            logAppLaunch(item, instanceId);
        }
        return true;
    } catch (NullPointerException|ActivityNotFoundException|SecurityException e) {
        Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
        Log.e(TAG, "Unable to launch. tag=" + item + " intent=" + intent, e);
    }
    return false;
}
```



可以看到BaseDraggingActivity的startActivitySafely中调用了startActivity方法，而这个方法位于Activity中，源码如下：

```java
public void startActivity(Intent intent, @Nullable Bundle options) {
    // ...
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}
```

接着调用了startActivityForResult：

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
			  // ...
    } else {
				// ...
    }
}
```

mInstrumentation是Activity中的成员变量，用于监视应用程序和系统交互。mMainThread是ActivityThread类型的成员变量。通过 mMainThread.getApplicationThread() 获得 ApplicationThread，它是 ActivityThread 的一个内部类，是一个 Binder 对象。AMS 会通过 Application 来和 ActivityThread 进行进程间通信。

Instrumentation 的 execStartActivity 方法如下：

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
  	// 使用AIDL进行进程间通信，实现类为ActivityThread内部的ApplicationThread
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    // ...
    try {
        intent.migrateExtraStreamToClipData(who);
        intent.prepareToLeaveProcess(who);
        int result = ActivityTaskManager.getService().startActivity(whoThread,
                who.getBasePackageName(), who.getAttributionTag(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```





```java
// frameworks/base/core/java/android/app/ActivityTaskManager.java
public static IActivityTaskManager getService() {
    return IActivityTaskManagerSingleton.get();
}

@UnsupportedAppUsage(trackingBug = 129726065)
private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
        new Singleton<IActivityTaskManager>() {
            @Override
            protected IActivityTaskManager create() {
                // 通过ServiceManager获取远程服务ActivityTaskManagerService
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                // 简历Launcher与ATMS的链接
                return IActivityTaskManager.Stub.asInterface(b);
            }
};
```

ServiceManager获取ActivityTaskManagerService，底层调用的是c++层的ServiceManager,它是Binder的守护服务，通过它能够在Android系统启动时注册系统服务，其中就包含提到的ATMS。接着通过IActivityTaskManager简历与ATMS的联系，这样回到execStartActivity方法，Launcher就通过调用ATMS的startActivity方法将启动Activity的数据交给了我ATMS服务来处理。

Launcher到ActivityTaskManagerService的时序图如下：



![](https://user-gold-cdn.xitu.io/2020/4/1/17133beb378c39b0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)









## ActivityTaskManagerService 调用ApplicationThread

在Android10之中ActivityManagerService用于管理Activity及其容器的系统服务分离出来放到了ATMS中，ActivityTaskManagerService代码如下：

```java
    // frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java

    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }

    public int startActivityAsUser(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);
    }


    private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
        assertPackageMatchesCallingUid(callingPackage);
        enforceNotIsolatedCaller("startActivityAsUser");

        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
      	// 构造ActivityStarter
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setCallingFeatureId(callingFeatureId)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setUserId(userId)
                .execute();

    }
```



可以发现，ActivityManagerService中startActivity() 方法调用了ActivityTaskManagerService的 startActivityAsUser() 方法，最后调用了 ActivityStartController.obtainStarter() 方法：

```java
// ActivityStartController
ActivityStarter obtainStarter(Intent intent, String reason) {
    return mFactory.obtain().setIntent(intent).setReason(reason);
}
```

根据注释得知 ActivityController 是以委托的方式把启动请求交给了一个包含所有启动信息的 ActivityStarter 对象并调用 execute 方法执行，ActivityStarter 的 execute() 方法如下：

```java
// frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java
int execute() {
        try {
            // TODO(b/64750076): Look into passing request directly to these methods to allow
            // for transactional diffs and preprocessing.
            if (mRequest.mayWait) { // 注释1
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                        mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            }
        } finally {
            onExecutionComplete();
        }
    }




```

由以上代码看到注释1，前面构造ActivityStarter已经传入了用户id，所以这里判断条件成立，则继续调用startActivityMayWait方法



```java
// frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java
private int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, int requestRealCallingPid, int requestRealCallingUid,
            Intent intent, String resolvedType, IVoiceInteractionSession voiceSession,
            IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
            int userId, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup,
            PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
            ......  
  final ActivityRecord[] outRecord = new ActivityRecord[1];//1
        int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,
                allowBackgroundActivityStart);//2
      ......
        return res;
    }
}	





```

由以上代码，可以看到注释1处创建了一个ActivityRecord数组，ActivityRecord代表一个Activity,接着调用了startActivity方法，



```java
// frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup,
            PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
        mSupervisor.getActivityMetricsLogger().notifyActivityLaunching(intent);
        int err = ActivityManager.START_SUCCESS;
        // Pull the optional Ephemeral Installer-only bundle out of the options early.
        final Bundle verificationBundle
                = options != null ? options.popAppVerificationBundle() : null;

    WindowProcessController callerApp = null;
    if (caller != null) {//1
        callerApp = mService.getProcessController(caller);//2
        if (callerApp != null) {
            callingPid = callerApp.getPid();
            callingUid = callerApp.mInfo.uid;
        } else {
            Slog.w(TAG, "Unable to find app for caller " + caller
                    + " (pid=" + callingPid + ") when starting: "
                    + intent.toString());
            err = ActivityManager.START_PERMISSION_DENIED;
        }
    }
   .......

    ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
            callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
            resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
            mSupervisor, checkedOptions, sourceRecord);
    if (outActivity != null) {
        outActivity[0] = r;//3
    }

   ......

    final int res = startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
            true /* doResume */, checkedOptions, inTask, outActivity, restrictedBgActivity);//4
    .....
    return res;
}




```



startActivity里面有很多的逻辑代码，这里只看一些重点的逻辑代码：

- 注释1处判断IApplicationThread是否为空，前面第一小节我们就已经提到过，它代表的就是Launcher进程的ApplicationThread，

- 注释2通过与即将要启动的应用程序进程建立联系，应用程序进程的是fork到Zyote进程，这里先不进行展开了,先专注Activity启动流程。

- 注释3创建ActivityRecord代表即将要启动的Activity，包含了Activity的所有信息，并赋值给上一步骤中创建的ActivityRecord类型的outActivity，

- 注释4则继续调用startActivity方法



```java
// frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                ActivityRecord[] outActivity, boolean restrictedBgActivity) {
        int result = START_CANCELED;
        final ActivityStack startedActivityStack;
        try {
            mService.mWindowManager.deferSurfaceLayout();
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity, restrictedBgActivity);//1
        } 
        ........
      return result;

}

private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity, boolean restrictedBgActivity) {
        ......    
        final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                ? mSourceRecord.getTaskRecord() : null;
        // Should this be considered a new task?
        int result = START_SUCCESS;
        if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
                && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) { //1
            newTask = true;
            result = setTaskFromReuseOrCreateNewTask(taskToAffiliate); //2
        } 
        ........
        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTaskRecord().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                
              mTargetStack.ensureActivitiesVisibleLocked(mStartActivity, 0, !PRESERVE_WINDOWS);
                mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
            } else {
                // If the target stack was not previously focusable (previous top running activity
                // on that stack was not visible) then any prior calls to move the stack to the
                // will not update the focused stack.  If starting the new activity now allows the
                // task stack to be focusable, then ensure that we now update the focused stack
                // accordingly.
                if (mTargetStack.isFocusable()
                        && !mRootActivityContainer.isTopDisplayFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront("startActivityUnchecked");
                }
                mRootActivityContainer.resumeFocusedStacksTopActivities(
                        mTargetStack, mStartActivity, mOptions);//3
            }
        }



```

注释1，在前面第一节Launcher部分中有提到过设置了Flag为**FLAG_ACTIVITY_NEW_TASK**，所以注意判断条件成立，则调用setTaskFromReuseOrCreateNewTask，它内部会创建的TaskRecord（代表Activity的任务栈），并将传入的TaskRecord对象设置给代表启动的Activity的ActivityRecord，

注释3调用了RootActivityContainer的resumeFocusedStacksTopActivities方法，RootActivityContainer 将一些东西从ActivityStackSupervisor中分离出来。目的是将其与RootWindowContainer合并，作为统一层次结构的一部分，接着看它的resumeFocusedStacksTopActivities方法

```java
 
// frameworks/base/services/core/java/com/android/server/wm/RootActivityContainer.java
boolean resumeFocusedStacksTopActivities(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
     // ...
    boolean result = false;
    if (targetStack != null && (targetStack.isTopStackOnDisplay()
            || getTopDisplayFocusedStack() == targetStack)) { 
      	// 调用ActivityStack的resumeTopActivityUncheckedLocked
        result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);//1
    }

    .......

    return result;
}




```



```java
// frameworks/base/services/core/java/com/android/server/wm/ActivityStack.java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    if (mInResumeTopActivity) {
        // Don't even start recursing.
        return false;
    }

    boolean result = false;
    try {
        // 防止递归
        mInResumeTopActivity = true;
        result = resumeTopActivityInnerLocked(prev, options); //1
        ........
    } finally {
        mInResumeTopActivity = false;
    }

    return result;
}

//确保栈顶 activity 为Resume
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mInResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // 防止递归
            mInResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options); //1
            ........
        } finally {
            mInResumeTopActivity = false;
        }

        return result;
}

private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    
   // ....
   // 调用ActivityStackSupervisor的startSpecificActivityLocked启动Activity
   mStackSupervisor.startSpecificActivityLocked(next, true, true);
  
   // ...
}





```



### ActivityStackSupervisor 启动Activity

```java
// frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java
void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);

    boolean knownToBeDead = false;
    if (wpc != null && wpc.hasThread()) {
        try {
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
        knownToBeDead = true;
    }

    // Suppress transition until the new activity becomes ready, otherwise the keyguard can
    // appear for a short amount of time before the new process with the new activity had the
    // ability to set its showWhenLocked flags.
    if (getKeyguardController().isKeyguardLocked()) {
        r.notifyUnknownVisibilityLaunched();
    }

    try {
        if (Trace.isTagEnabled(TRACE_TAG_ACTIVITY_MANAGER)) {
            Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "dispatchingStartProcess:"
                    + r.processName);
        }
        // Post message to start process to avoid possible deadlock of calling into AMS with the
        // ATMS lock held.
        final Message msg = PooledLambda.obtainMessage(
                ActivityManagerInternal::startProcess, mService.mAmInternal, r.processName,
                r.info.applicationInfo, knownToBeDead, "activity", r.intent.getComponent());
        mService.mH.sendMessage(msg);
    } finally {
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
}


  boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {

                // ...

                // 创建ClientTransaction，包含一系列消息容器，发送到客户端，客户端即为要启动的应用的Activity
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.appToken);

                final DisplayContent dc = r.getDisplay().mDisplayContent;
    						// 将启动Activity的参数封装成LaunchActivityItem请求request对象。
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.icicle, r.persistentState, results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                                r.assistToken));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);
                // 调用ClientLifecycleManager的scheduleTransaction方法
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
               // ...

        return true;
    }

```

### ClientLifecycleManager处理ClientTransaction

```java
// frameworks/base/services/core/java/com/android/server/wm/ClientLifecycleManager.java
void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    // 获取要启动的应用程序进程的IAppClicationThread
    final IApplicationThread client = transaction.getClient();
    // 调用ClientTransaction的schedule方法
    transaction.schedule();//2
    if (!(client instanceof Binder)) {
        // If client is not an instance of Binder - it's a remote call and at this point it is
        // safe to recycle the object. All objects used for local calls will be recycled after
        // the transaction is executed on client in ActivityThread.
        transaction.recycle();
    }
}


```

```java
// frameworks/base/core/java/android/app/servertransaction/ClientTransaction.java
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);  
}
```

mClient则代表要启动的应用程序进程的IApplicationThread，而当前还处于ATMS服务的进程，也就是SystemServer进程，这时ATMS要与即将启动的应用程序进程通信则通过IApplicationThread来执行AIDL，IApplicationThread实现为ApplicationThread，它是ActivityThread的内部类，所以前面也说过ApplicationThread为进程间通信的桥梁，接下来则相当于是IApplicationThread.scheduleTransaction，并将包含要启动Activity信息的ClientTransaction传递到了应用程序进程，下一节就从IApplicationThread讲起。

### AMTS调用到ApplicationThread时序图

![](https://user-gold-cdn.xitu.io/2020/4/1/17133beb3e0359e2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



## ActivityThread启动Activity



ApplicationThread的scheduleTransaction方法



```java
// frameworks/base/core/java/android/app/ActivityThread$ApplicationThread
private class ApplicationThread extends IApplicationThread.Stub {
    // ...
    @Override
    public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        ActivityThread.this.scheduleTransaction(transaction);
    }
}
```

上面代码调用了ActivityThread的scheduleTransaction方法，ActivityThread继承了ClientTransactionHandler，scheduleTransaction在里面实现：



```java
// frameworks/base/core/java/android/app/ClientTransactionHandler.java
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```



```java
class H extends Handler {
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {                    case EXECUTE_TRANSACTION:
                final ClientTransaction transaction = (ClientTransaction) msg.obj;
                mTransactionExecutor.execute(transaction);
                if (isSystem()) {
                    transaction.recycle();
                }
                break;
         }
    }
}
```

调用TransactionExecutor的execute方法

```java
	// frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java
    public void execute(ClientTransaction transaction) {
        executeCallbacks(transaction);
				// ...
    }

    public void executeCallbacks(ClientTransaction transaction) {
        final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
        if (callbacks == null || callbacks.isEmpty()) {
            // No callbacks to execute, return early.
            return;
        }
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Resolving callbacks in transaction");

        final IBinder token = transaction.getActivityToken();
        ActivityClientRecord r = mTransactionHandler.getActivityClient(token);

        // In case when post-execution state of the last callback matches the final state requested
        // for the activity in this transaction, we won't do the last transition here and do it when
        // moving to final state instead (because it may contain additional parameters from server).
        final ActivityLifecycleItem finalStateRequest = transaction.getLifecycleStateRequest();
        final int finalState = finalStateRequest != null ? finalStateRequest.getTargetState()
                : UNDEFINED;
        // Index of the last callback that requests some post-execution state.
        final int lastCallbackRequestingState = lastCallbackRequestingState(transaction);

        final int size = callbacks.size();
        for (int i = 0; i < size; ++i) {
            final ClientTransactionItem item = callbacks.get(i);
            if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Resolving callback: " + item);
            final int postExecutionState = item.getPostExecutionState();
            final int closestPreExecutionState = mHelper.getClosestPreExecutionState(r,
                    item.getPostExecutionState());
            if (closestPreExecutionState != UNDEFINED) {
                cycleToPath(r, closestPreExecutionState, transaction);
            }
						// LaunchActivityItem的execute
            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
            if (r == null) {
                // Launch activity request will create an activity record.
                r = mTransactionHandler.getActivityClient(token);
            }

            if (postExecutionState != UNDEFINED && r != null) {
                // Skip the very last transition and perform it by explicit state request instead.
                final boolean shouldExcludeLastTransition =
                        i == lastCallbackRequestingState && finalState == postExecutionState;
                cycleToPath(r, postExecutionState, shouldExcludeLastTransition, transaction);
            }
        }
    }
```



获取的ClientTransactionItem则为第二小节中提到过的LaunchActivityItem对象，它继承了ClientTransactionItem，并保存这需要启动的Activity数据，接着看到注释2 LaunchActivityItem的execute方法。

```java
@Override
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    // 恢复要启动的Activity的数据
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client, mAssistToken);
    // ActivityThread的handleLaunchActivity
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

恢复了要启动的Activity的数据，ActivityClientRecord是ActivityThread的内部类，这里的client为ClientTransactionHandler，而前面已经说过ActivityThread继承ClientTransactionHandler，所以这里的注释2处调用的就是ActivityThread的handleLaunchActivity方法

```java
// frameworks/base/core/java/android/app/ActivityThread.java
@Override
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
  	// ...
  
    final Activity a = performLaunchActivity(r, customIntent);
  
    return a;
}
```



```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // 1.获取了前面保存启动应用程序信息的ActivityClientRecord中的应用程序信息，包括应用程序在清单文件中注册了哪些四大组件，启动的根Activity是什么
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            // 2.通过getPackageInfo方法获取LoadedApk描述对应Apk文件资源
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
        // 3.ComponentName类获取则对应启动Activity的包名和类名
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        // 4.应用程序Context的创建
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
        
            java.lang.ClassLoader cl = appContext.getClassLoader();
            // 5.通过类加载器和Intent对象实例化了Activity对象
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
        
           // 6.根据上面获取到的Apk描述对象LoadedApk创建了应用程序的Application对象，
           // 并在makeApplication方法中调用了它的OnCreate方法,此时Application启动
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

           ......

            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                // 7.通过Activity的 attach 方法将 context等各种数据与Activity绑定，初始化Activity
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken); 

                ......
                if (r.isPersistable()) {
                    // 8.Instrumentation对象的callActivityOnCreate方法
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ......
                r.activity = activity;
            }
            r.setState(ON_CREATE);

           .......
        }

        return activity;
}
```

### Activity的 OnCreate方法调用

```java
// frameworks/base/core/java/android/app/Instrumentation.java
public void callActivityOnCreate(Activity activity, Bundle icicle) {
    prePerformCreate(activity);
    // 调用Activity的performCreate
    activity.performCreate(icicle);
    postPerformCreate(activity);
}

```



```java
// frameworks/base/core/java/android/app/Activity.java
final void performCreate(Bundle icicle) {
    performCreate(icicle, null);
}

@UnsupportedAppUsage
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    dispatchActivityPreCreated(icicle);
    mCanEnterPictureInPicture = true;
    restoreHasCurrentPermissionRequest(icicle);
    if (persistentState != null) {
        // Activity已经实例化
        onCreate(icicle, persistentState);
    } else {
        // 调用Activity的onCreate方法
        onCreate(icicle);
    }
    writeEventLog(LOG_AM_ON_CREATE_CALLED, "performCreate");
    mActivityTransitionState.readState(icicle);

    mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
            com.android.internal.R.styleable.Window_windowNoDisplay, false);
    mFragments.dispatchActivityCreated();
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    dispatchActivityPostCreated(icicle);
}
```

### ActivityThread启动Activity的时序图

![](https://user-gold-cdn.xitu.io/2020/4/1/17133beb3c7a9e75?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



[深入理解Android 之 Activity启动流程（Android 10）](