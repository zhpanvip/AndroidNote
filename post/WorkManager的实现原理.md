WorkManager的实现原理

### 作用

WorkManager是Android jetpack提供执行后台任务的管理组件。使用WorkManager运行的任务，即使退出杀掉APP，也可以保证任务能够正常执行。
如果应用正在运行时执行任务，WorkManager可以在新线程中执行，如果应用为运行，WorkManager会选择一种合适的方式来执行后台任务。使用哪种方式取决于API和包依赖项。WorkManager可能会使用JobScheduler,Firebase JobDispatcher或者AlarmManager

确保重要的后台任务一定会执行（非及时性的任务，如上传日志、同步数据等）

### 简单使用

1.编写执行任务的Worker
```
public class MainWorker1 extends Worker {

    private final static String TAG = MainWorker1.class.getSimpleName();

    public MainWorker1(@NonNull Context context, @NonNull WorkerParameters workerParams) {
        super(context, workerParams);
    }

    // 后台任务 并且 异步的
    @NonNull
    @Override
    public Result doWork() {
        Log.d(TAG, "MainWorker1 doWork: run started ... ");
        try {
            Thread.sleep(8000); // 睡眠
        } catch (InterruptedException e) {
            e.printStackTrace();
            return Result.failure(); // 本次任务失败
        } finally {
            Log.d(TAG, "MainWorker1 doWork: run end ... ");
        }
        return Result.success(); // 本次任务成功
    }
}

```

2.执行任务

```
OneTimeWorkRequest oneTimeWorkRequest =
        new OneTimeWorkRequest.Builder(MainWorker1.class).build();

WorkManager.getInstance(this).enqueue(oneTimeWorkRequest);
```

3.添加约束条件
```

        // 约束条件，必须满足条件，才能执行后台任务 （在连接网络，插入电源 并且 处于空闲时）
    Constraints constraints = new Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED) // 网络链接中...
            /*.setRequiresCharging(true) // 充电中..
            .setRequiresDeviceIdle(true) // 空闲时.. (例如：你没有玩游戏，你没有看片)*/
            .build();

    OneTimeWorkRequest request = new OneTimeWorkRequest.Builder(MainWorker3.class)
            .setConstraints(constraints) // Request关联 约束条件
            .build();

    // 加入队列
    WorkManager.getInstance(this).enqueue(request);
```
4.执行周期任务
OneTimeWorkRequest的任务只能执行一次，如果是周期性的任务可以使用 PeriodicWorkRequest。

```
    //  制定一个每隔1小时执行一次的Request 
    PeriodicWorkRequest periodicWorkRequest
            = new PeriodicWorkRequest.Builder(MainWorker3.class, 1, TimeUnit.HOURS).build();

    // 监听状态
    WorkManager.getInstance(this).getWorkInfoByIdLiveData(periodicWorkRequest.getId())
            .observe(this, new Observer<WorkInfo>() {
                @Override
                public void onChanged(WorkInfo workInfo) {
                    Log.d(MainWorker2.TAG, "状态：" + workInfo.getState().name()); // 这里只会是状态只会是ENQUEUE，因为是周期任务。
                    if (workInfo.getState().isFinished()) {
                        Log.d(MainWorker2.TAG, "状态：isFinished=true 后台任务已经完成了...");
                    }
                }
            });

    WorkManager.getInstance(this).enqueue(periodicWorkRequest);
```


### 实现原理

WorkManager会将任务保存到Room数据库中，系统通过JobScheduler或者AlarmManager来执行。

#### WorkManager初始化过程

1.WorkManager会在清单文件中生成一个ContentProvider，在第一次打开APP的时候便会执行WorkManager.getInstance进行单利的初始化工作。
```
        <provider
            android:name="androidx.work.impl.WorkManagerInitializer"
            android:exported="false"
            android:multiprocess="true"
            android:authorities="com.zhangpan.myapplication.workmanager-init"
            android:directBootAware="false" />
```
```
public class WorkManagerInitializer extends ContentProvider {
    @Override
    public boolean onCreate() {
        // Initialize WorkManager with the default configuration.
        WorkManager.initialize(getContext(), new Configuration.Builder().build());
        return true;
    }
}

```
2.WorkManager中initialize方法调用WorkManagerImpl的initialize，并实例化WorkManagerImpl。

```
    public static void initialize(@NonNull Context context, @NonNull Configuration configuration) {
        synchronized (sLock) {
            if (sDelegatedInstance == null) {
                context = context.getApplicationContext();
                if (sDefaultInstance == null) {
                    sDefaultInstance = new WorkManagerImpl(
                            context,
                            configuration,
                            new WorkManagerTaskExecutor(configuration.getTaskExecutor()));
                }
                sDelegatedInstance = sDefaultInstance;
            }
        }
    }

```

3.WorkManagerImpl构造方法中会创建Room数据库与任务执行器Schedulers
```
    public WorkManagerImpl(
            @NonNull Context context,
            @NonNull Configuration configuration,
            @NonNull TaskExecutor workTaskExecutor,
            @NonNull WorkDatabase database) {
        Context applicationContext = context.getApplicationContext();
        Logger.setLogger(new Logger.LogcatLogger(configuration.getMinimumLoggingLevel()));
        // 创建任务执行器
        List<Scheduler> schedulers = createSchedulers(applicationContext, workTaskExecutor);
        Processor processor = new Processor(
                context,
                configuration,
                workTaskExecutor,
                database,
                schedulers);
        internalInit(context, configuration, workTaskExecutor, database, schedulers, processor);
    }
```
```
    public List<Scheduler> createSchedulers(
            @NonNull Context context,
            @NonNull TaskExecutor taskExecutor) {
        // 创建立即后台的任务执行器与立即执行器
        return Arrays.asList(
                Schedulers.createBestAvailableBackgroundScheduler(context, this),
                // Specify the task executor directly here as this happens before internalInit.
                // GreedyScheduler creates ConstraintTrackers and controllers eagerly.
                new GreedyScheduler(context, taskExecutor, this));
    }
```

createBestAvailableBackgroundScheduler方法中会根据SDK版本创建对应的后台执行器
```
    @NonNull
    static Scheduler createBestAvailableBackgroundScheduler(
            @NonNull Context context,
            @NonNull WorkManagerImpl workManager) {

        Scheduler scheduler;

        if (Build.VERSION.SDK_INT >= WorkManagerImpl.MIN_JOB_SCHEDULER_API_LEVEL) {
            scheduler = new SystemJobScheduler(context, workManager);
            setComponentEnabled(context, SystemJobService.class, true);
            Logger.get().debug(TAG, "Created SystemJobScheduler and enabled SystemJobService");
        } else {
            scheduler = tryCreateGcmBasedScheduler(context);
            if (scheduler == null) {
                scheduler = new SystemAlarmScheduler(context);
                setComponentEnabled(context, SystemAlarmService.class, true);
                Logger.get().debug(TAG, "Created SystemAlarmScheduler");
            }
        }
        return scheduler;
    }
```

#### WorkManager执行过程

1.以WorkManager.getInstance(this).enqueue(oneTimeWorkRequest)为入口最终会追踪到WorkManagerImpl的enqueue方法
```
    public Operation enqueue(
            @NonNull List<? extends WorkRequest> workRequests) {

        // This error is not being propagated as part of the Operation, as we want the
        // app to crash during development. Having no workRequests is always a developer error.
        if (workRequests.isEmpty()) {
            throw new IllegalArgumentException(
                    "enqueue needs at least one WorkRequest.");
        }
        return new WorkContinuationImpl(this, workRequests).enqueue();
    }
```

2.WorkContinuationImpl的enqueue方法中将EnqueueRunnable加入线程池
```
    @Override
    public @NonNull Operation enqueue() {
        // Only enqueue if not already enqueued.
        if (!mEnqueued) {
            // The runnable walks the hierarchy of the continuations
            // and marks them enqueued using the markEnqueued() method, parent first.
            EnqueueRunnable runnable = new EnqueueRunnable(this);
            mWorkManagerImpl.getWorkTaskExecutor().executeOnBackgroundThread(runnable);
            mOperation = runnable.getOperation();
        } else {
            Logger.get().warning(TAG,
                    String.format("Already enqueued work ids (%s)", TextUtils.join(", ", mIds)));
        }
        return mOperation;
    }
```
3. 线程池执行EnqueueRunnable的run方法
```
    @Override
    public void run() {
        try {
            // 将WorkRequest.WorkSpec加入到数据库
            boolean needsScheduling = addToDatabase();
            if (needsScheduling) {
                // Enable RescheduleReceiver, only when there are Worker's that need scheduling.
                final Context context =
                        mWorkContinuation.getWorkManagerImpl().getApplicationContext();
                PackageManagerHelper.setComponentEnabled(context, RescheduleReceiver.class, true);
                // 后台执行任务
                scheduleWorkInBackground();
            }
            mOperation.setState(Operation.SUCCESS);
        } catch (Throwable exception) {
            mOperation.setState(new Operation.State.FAILURE(exception));
        }
    }

```
Schedulers最终会循环遍历执行Scheduler
```
    public void scheduleWorkInBackground() {
        WorkManagerImpl workManager = mWorkContinuation.getWorkManagerImpl();
        Schedulers.schedule(
                workManager.getConfiguration(),
                workManager.getWorkDatabase(),
                workManager.getSchedulers());
    }

```

以GreedyScheduler为例

```
public void schedule(@NonNull WorkSpec... workSpecs) {

        registerExecutionListenerIfNeeded();

        List<WorkSpec> constrainedWorkSpecs = new ArrayList<>();
        List<String> constrainedWorkSpecIds = new ArrayList<>();
        for (WorkSpec workSpec : workSpecs) {
            if (workSpec.state == WorkInfo.State.ENQUEUED
                    && !workSpec.isPeriodic()
                    && workSpec.initialDelay == 0L
                    && !workSpec.isBackedOff()) {
                if (workSpec.hasConstraints()) { // 判断是否添加了约束条件
                    if (SDK_INT >= 23 && workSpec.constraints.requiresDeviceIdle()) {
                        // Ignore requests that have an idle mode constraint.
                        Logger.get().debug(TAG,
                                String.format("Ignoring WorkSpec %s, Requires device idle.",
                                        workSpec));
                    } else if (SDK_INT >= 24 && workSpec.constraints.hasContentUriTriggers()) {
                        // Ignore requests that have content uri triggers.
                        Logger.get().debug(TAG,
                                String.format("Ignoring WorkSpec %s, Requires ContentUri triggers.",
                                        workSpec));
                    } else {
                        // 添加到约束任务中，等到执行
                        constrainedWorkSpecs.add(workSpec);
                        constrainedWorkSpecIds.add(workSpec.id);
                    }
                } else {
                    // 执行任务
                    mWorkManagerImpl.startWork(workSpec.id);
                }
            }
        }

        // onExecuted() which is called on the main thread also modifies the list of mConstrained
        // WorkSpecs. Therefore we need to lock here.
        synchronized (mLock) {
            if (!constrainedWorkSpecs.isEmpty()) {
                Logger.get().debug(TAG, String.format("Starting tracking for [%s]",
                        TextUtils.join(",", constrainedWorkSpecIds)));
                mConstrainedWorkSpecs.addAll(constrainedWorkSpecs);
                mWorkConstraintsTracker.replace(mConstrainedWorkSpecs);
            }
        }
    }

```
WorkManagerImpl中开始执行任务
```

    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public void startWork(@NonNull String workSpecId) {
        startWork(workSpecId, null);
    }

    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public void startWork(
            @NonNull String workSpecId,
            @Nullable WorkerParameters.RuntimeExtras runtimeExtras) {
        // 在线程池中开始执行任务
        mWorkTaskExecutor
                .executeOnBackgroundThread(
                        new StartWorkRunnable(this, workSpecId, runtimeExtras));
    }


```

线程池调用StartWorkRunnable的run方法

```
    @Override
    public void run() {
        mWorkManagerImpl.getProcessor().startWork(mWorkSpecId, mRuntimeExtras);
    }
```
执行Processor的startWork，在这个方法中执行WorkerWrapper的run方法：

```
    @WorkerThread
    @Override
    public void run() {
        mTags = mWorkTagDao.getTagsForWorkSpecId(mWorkSpecId);
        mWorkDescription = createWorkDescription(mTags);
        runWorker();
    }
```
runWorker方法中最终会执行Worker的startWork方法，这个方法即使用WorkManager中自定义的Worker。
