## 一、 Fragment的使用

### 1. 添加Fragment

```kotlin
val homeFragment = HomeFragment()
val fragmentTransaction = supportFragmentManager.beginTransaction()
fragmentTransaction.add(R.id.cl_fragment, homeFragment)
fragmentTransaction.commit()
// fragmentTransaction.commitNow()
```

通过SupportFragmentManager.beginTransaction开启事务，然后通过add方法将Fragment添加到布局文件中。使用fragmentTransaction.commit或者fragmentTransaction.commitNow提交事务。

### 2.移除Fragment

```kotlin
val fragmentTransaction = supportFragmentManager.beginTransaction()
fragmentTransaction.remove(homeFragment)
fragmentTransaction.commit()
```

### 3. 替换Fragment

```kotlin
val projectFragment = ProjectFragment()
val fragmentTransaction = supportFragmentManager.beginTransaction()
fragmentTransaction.replace(R.id.cl_fragment, projectFragment)
fragmentTransaction.commit()
```

### 4. 显示或隐藏Fragment

```kotlin
val fragmentTransaction = supportFragmentManager.beginTransaction()
fragmentTransaction.hide(homeFragment)
// fragmentTransaction.show(homeFragment)
fragmentTransaction.commit()
```

## 二、原理分析



### 1. FragmentManager获取流程

通过getSupportFragmentManager获得FragmentManager，源码如下：

```java
// FragmentActivity
public FragmentManager getSupportFragmentManager() {
    return mFragments.getSupportFragmentManager();
}
```

mFragments是在FragmentActivity中的FragmentController类型的成员变量，它是在FragmentActivity中被初始化的，如下：

```java
// FragmentActivity
final FragmentController mFragments = FragmentController.createController(new HostCallbacks());
```

通过createController并传进去一个HostCallbacks的参数实例化出了mFragments，createController的源码如下：

```java
// FragmentController
public static FragmentController createController(@NonNull FragmentHostCallback<?> callbacks) {
    return new FragmentController(checkNotNull(callbacks, "callbacks == null"));
}
```

checkNotNull检测如果callbacks不为空，则返回这个callback，在FragmentController的构造方法中会将callbacks赋值给一个mHost的成员变量，如下：

```java
// FragmentController
private final FragmentHostCallback<?> mHost;
private FragmentController(FragmentHostCallback<?> callbacks) {
    mHost = callbacks;
}
```

接着看FragmentController的getSupportFragmentManager的源码：

```java
// FragmentController
public FragmentManager getSupportFragmentManager() {
    return mHost.mFragmentManager;
}
```

这里通过mHost.mFragmentManager获取到了FragmentManager，mHost就是HostCallbacks，HostCallbacks继承了FragmentHostCallback，mFragmentManager在mFragmentManager中被实例化，代码如下：

```java
// FragmentHostCallback
final FragmentManager mFragmentManager = new FragmentManagerImpl();
```

即在Activity中通过getSupportFragmentManager实际获得的是FragmentManagerImpl的实例。



### 2. FragmentManager

点进FragmentManagerImpl的源码发现FragmentManagerImpl是一个空类，仅仅继承了FragmentManager。因此，beginTransaction是在FragmentManager中实现的。其源码如下:

```java
// FragmentManager
public FragmentTransaction beginTransaction() {
    return new BackStackRecord(this);
}
```

这里仅仅是返回了一个BackStackRecord实例，BackStackRecord实例化时会将FragmentManager自身作为参数传递给BackStackRecord，BackStackRecord继承自FragmentTransaction。add方法、remove方法都在FragmentTransaction中因此，这就意味着添加Fragment、移除Fragment、显示或隐藏Fragment都是由FragmentTransaction来完成的。



### 3. FragmentTransaction的add过程

上节说到通过 `supportFragmentManager.beginTransaction()` 实际上拿到的是一个BackStackRecord实例，所以调用add方法添加Fragment实际上调用的是BackStackRecord

```java
static final int OP_ADD = 1;
public FragmentTransaction add(@IdRes int containerViewId, @NonNull Fragment fragment) {
    // 注意第三个参数，OP_ADD标记是添加Fragment
    doAddOp(containerViewId, fragment, null, OP_ADD);
    return this;
}
```

add方法直接调用了FragmentTransaction重载的doAddOp方法，如下：

```java
void doAddOp(int containerViewId, Fragment fragment, @Nullable String tag, int opcmd) {
    final Class<?> fragmentClass = fragment.getClass();
    final int modifiers = fragmentClass.getModifiers();
    // ... 省略校验代码

    if (tag != null) {
				// ... 省略校验代码
        fragment.mTag = tag;
    }

    if (containerViewId != 0) {
        // ... 省略校验代码
        fragment.mContainerId = fragment.mFragmentId = containerViewId;
    }

    addOp(new Op(opcmd, fragment));
}


```

这个方法中省略校验相关代码后代码逻辑其实很简单，即为Fragment设置Tag和ContainerId，然后将OP_ADD与Fragment封装成一个Op对象，传入addOp方法。addOp方法的源码如下：

```java
ArrayList<Op> mOps = new ArrayList<>();
void addOp(Op op) {
    mOps.add(op);
    op.mEnterAnim = mEnterAnim;
    op.mExitAnim = mExitAnim;
    op.mPopEnterAnim = mPopEnterAnim;
    op.mPopExitAnim = mPopExitAnim;
}
```

可见mOps是一个List，用来存放所有被添加进来的Fragment。而Op是FragmentTransaction中的静态内部类，存储的是Fragment的信息：

```java
static final class Op {
    int mCmd;
    Fragment mFragment;
    int mEnterAnim;
    int mExitAnim;
    int mPopEnterAnim;
    int mPopExitAnim;
    Lifecycle.State mOldMaxState;
    Lifecycle.State mCurrentMaxState;
		// ...
}
```

Fragment真正加载到Activity是在调用commit之后。

### 4. FragmentTransaction的commit过程

commit方法是在FragmentTransaction的子类BackStackRecord中实现的。源码如下：

```java
// BackStackRecord
public int commit() {
    return commitInternal(false);
}
```

```java 
int commitInternal(boolean allowStateLoss) {
  // 多次commit抛出异常
  if (mCommitted) throw new IllegalStateException("commit already called");
   // ...
    mCommitted = true;
    if (mAddToBackStack) {
        mIndex = mManager.allocBackStackIndex();
    } else {
        mIndex = -1;
    }
    // 通过FragmentManager将请求任务加入队列，注意第二个参数是上边传来的，值为false
    mManager.enqueueAction(this, allowStateLoss);
    return mIndex;
}
```

这里的核心其实就是 mManager.enqueueAction ，mManager是位于BackstackRecord中FragmentManager类型的成员变量。enqueueAction这个方法的第一个参数接收一个OpGenerator，第二个参数是有commit传过来的一个false。

```java
interface OpGenerator {
    boolean generateOps(@NonNull ArrayList<BackStackRecord> records,
            @NonNull ArrayList<Boolean> isRecordPop);
}
```

而BackStackRecord实现了OpGenerator，并重写了 generateOps 

```java
// BackStackRecord 中实现了 generateOps 方法
@Override
public boolean generateOps(@NonNull ArrayList<BackStackRecord> records,
        @NonNull ArrayList<Boolean> isRecordPop) {
    if (FragmentManager.isLoggingEnabled(Log.VERBOSE)) {
        Log.v(TAG, "Run: " + this);
    }

    records.add(this);
    isRecordPop.add(false);
    if (mAddToBackStack) {
        mManager.addBackStackState(this);
    }
    return true;
}
```

接着继续看enqueueAction的源码：

```java
// FragmentManager

private final ArrayList<OpGenerator> mPendingActions = new ArrayList<>();

void enqueueAction(@NonNull OpGenerator action, boolean allowStateLoss) {
    // ...
    synchronized (mPendingActions) {
        // ...
        mPendingActions.add(action);
        scheduleCommit();
    }
}

```

enqueueAction方法中的核心就是将action加入到了mPendingActions这个List集合中，然后调用scheduleCommit方法。

```java
// FragmentManager
void scheduleCommit() {
    synchronized (mPendingActions) {
        boolean postponeReady =
                mPostponedTransactions != null && !mPostponedTransactions.isEmpty();
        boolean pendingReady = mPendingActions.size() == 1;
        if (postponeReady || pendingReady) {
            mHost.getHandler().removeCallbacks(mExecCommit);
            mHost.getHandler().post(mExecCommit);
            updateOnBackPressedCallbackEnabled();
        }
    }
}

private Runnable mExecCommit = new Runnable() {
    @Override
    public void run() {
        execPendingActions(true);
    }
};
```

可以看到在scheduleCommit方法中判断如果真备好了则使用Handlerpost出来一个mExecCommit。那么接下来Handler执行到mExecCommit这个Message的时候会执行mExecCommit的run方法，进而执行execPendingActions。

```java
boolean execPendingActions(boolean allowStateLoss) {
    ensureExecReady(allowStateLoss);

    boolean didSomething = false;
    while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
        mExecutingActions = true;
        try {
            // 这里是主线
            removeRedundantOperationsAndExecute(mTmpRecords, mTmpIsPop);
        } finally {
            cleanupExec();
        }
        didSomething = true;
    }

    updateOnBackPressedCallbackEnabled();
    doPendingDeferredStart();
    mFragmentStore.burpActive();

    return didSomething;
}
```

在removeRedundantOperationsAndExecute方法中又会调用executeOpsTogether方法，executeOpsTogether源码如下：

```java
private void executeOpsTogether(@NonNull ArrayList<BackStackRecord> records,
        @NonNull ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
    final boolean allowReordering = records.get(startIndex).mReorderingAllowed;
    boolean addToBackStack = false;
    if (mTmpAddedFragments == null) {
        mTmpAddedFragments = new ArrayList<>();
    } else {
        mTmpAddedFragments.clear();
    }
    mTmpAddedFragments.addAll(mFragmentStore.getFragments());
    Fragment oldPrimaryNav = getPrimaryNavigationFragment();
    for (int recordNum = startIndex; recordNum < endIndex; recordNum++) {
        final BackStackRecord record = records.get(recordNum);
        final boolean isPop = isRecordPop.get(recordNum);
        if (!isPop) {
            oldPrimaryNav = record.expandOps(mTmpAddedFragments, oldPrimaryNav);
        } else {
            oldPrimaryNav = record.trackAddedFragmentsInPop(mTmpAddedFragments, oldPrimaryNav);
        }
        addToBackStack = addToBackStack || record.mAddToBackStack;
    }
    mTmpAddedFragments.clear();

    if (!allowReordering) {
        // 开启事务
        FragmentTransition.startTransitions(this, records, isRecordPop, startIndex, endIndex,
                false, mFragmentTransitionCallback);
    }
    executeOps(records, isRecordPop, startIndex, endIndex);

    int postponeIndex = endIndex;
    if (allowReordering) {
        ArraySet<Fragment> addedFragments = new ArraySet<>();
        addAddedFragments(addedFragments);
        postponeIndex = postponePostponableTransactions(records, isRecordPop,
                startIndex, endIndex, addedFragments);
        makeRemovedFragmentsInvisible(addedFragments);
    }

    if (postponeIndex != startIndex && allowReordering) {
        // 开启事务，绑定Fragment
        FragmentTransition.startTransitions(this, records, isRecordPop, startIndex,
                postponeIndex, true, mFragmentTransitionCallback);
        // 修改Fragment的生命周期状态
        moveToState(mCurState, true);
    }

    for (int recordNum = startIndex; recordNum < endIndex; recordNum++) {
        final BackStackRecord record = records.get(recordNum);
        final boolean isPop = isRecordPop.get(recordNum);
        if (isPop && record.mIndex >= 0) {
            record.mIndex = -1;
        }
        record.runOnCommitRunnables();
    }
    if (addToBackStack) {
        reportBackStackChanged();
    }
}
```





















