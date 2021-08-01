### 1.简单使用

```
SharedPreferences sp = Context.getSharedPreferences(String fileName, int mode)
SharedPreferences.Editor edit = sp.edit();
edit.putString("Value","test");
edit.apply(); // 或者edit.commit();

sp.getString("Value","");
```

### 2.初始化与加载

getSharedPreferences方法是在ContextImpl中实现的，源码如下：

```java
// ContextImpl 
public SharedPreferences getSharedPreferences(String name, int mode) {
        // ...
    
        File file;
        synchronized (ContextImpl.class) {
            if (mSharedPrefsPaths == null) {
                mSharedPrefsPaths = new ArrayMap<>();
            }
            file = mSharedPrefsPaths.get(name);
            if (file == null) {
                file = getSharedPreferencesPath(name);
                mSharedPrefsPaths.put(name, file);
            }
        }
        return getSharedPreferences(file, mode);
    }

    @Override
    public SharedPreferences getSharedPreferences(File file, int mode) {
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            sp = cache.get(file);
            if (sp == null) {
                checkMode(mode);
                if (getApplicationInfo().targetSdkVersion >= android.os.Build.VERSION_CODES.O) {
                    if (isCredentialProtectedStorage()
                            && !getSystemService(UserManager.class)
                                    .isUserUnlockingOrUnlocked(UserHandle.myUserId())) {
                        throw new IllegalStateException("SharedPreferences in credential encrypted "
                                + "storage are not available until after user is unlocked");
                    }
                }
                sp = new SharedPreferencesImpl(file, mode);
                cache.put(file, sp);
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }


    private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
        if (sSharedPrefsCache == null) {
            sSharedPrefsCache = new ArrayMap<>();
        }

        final String packageName = getPackageName();
        ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
        if (packagePrefs == null) {
            packagePrefs = new ArrayMap<>();
            sSharedPrefsCache.put(packageName, packagePrefs);
        }

        return packagePrefs;
    }

```



- 找到对应name的文件。
- 加载对应文件到内存中SharedPreference。
- 一个xml文件对应一个ShredPreferences**单例**。

sSharedPrefsCache存储的是File和SharedPreferencesImpl键值对，当对应File的SharedPreferencesImpl加载之后就会一支存储于sSharedPrefsCache中。类似的mSharedPrefsPaths存储的是name和File的对应关系。

当通过name最终找到对应的File之后，就会实例化一个SharedPreferencesImpl对象。在SharedPreferences构造方法中开启一个子线程加载磁盘中的xml文件。



SP持久化的本质是在本地磁盘记录了一个xml文件，这个文件所在的文件夹shared_prefs



### 3.从SP中取数据

取数据时应保证SP的初始化已经完成，以getString为例：

```java
// SharedPreferencesImpl
@Override
@Nullable
public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}

private void awaitLoadedLocked() {
    if (!mLoaded) {
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
        try {
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
    if (mThrowable != null) {
        throw new IllegalStateException(mThrowable);
    }
}


```

使用awaitLoadedLocked()方法检测，是否已经加载完成，如果没有加载完成，就等待堵塞。等加载完成之后，继续执行；

### 4.编辑数据

调用sharedPreferences.edit()返回一个EditorImpl对象，操作数据之后调用apply()或者commit()。

#### 4.1commit流程

```java
// SharedPreferencesImpl
@Override
public boolean commit() {

    MemoryCommitResult mcr = commitToMemory();// 提交到内存

    SharedPreferencesImpl.this.enqueueDiskWrite( // 写入磁盘
        mcr, null /* sync write on this thread okay */);
    try {
        mcr.writtenToDiskLatch.await(); // 等待写入磁盘完毕
    } catch (InterruptedException e) {
        return false;
    } finally {}
    notifyListeners(mcr);//通知监听
    return mcr.writeToDiskResult;
}

private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    final boolean isFromSyncCommit = (postWriteRunnable == null);
    //如果postWriteRunnable为空表示来自commit()方法调用
    final Runnable writeToDiskRunnable = new Runnable() {
            @Override
            public void run() {
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr, isFromSyncCommit);//蒋数据写入磁盘
                }
                synchronized (mLock) {
                    mDiskWritesInFlight--;
                }
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

   //如果是commit提交，且mDiskWritesInFlight为1的时候，直接在当前所在线程执行写入磁盘操作
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (mLock) {
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            writeToDiskRunnable.run();
            return;
        }
    }

    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}



```



![image](https://user-gold-cdn.xitu.io/2019/12/16/16f0d8b3eeefb909?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



- 首先将编辑的结果同步到内存中。

- enqueueDiskWrite（）将这个结果同步到磁盘中，enqueueDiskWrite()的第二个参数postWriteRunnable传入空。通常情况下也就是mDiskWritesInFlight（正在执行的写入磁盘操作的数量）为1的时候，直接在当前所在线程执行写入磁盘操作。否则还是异步到QueuedWork中去执行。**commit()时，写入磁盘操作会发生在当前线程的说法是不准确的**。

- 执行mcr.writtenToDiskLatch.await(); MemoryCommitResult 中有个一个CountDownLatch 成员变量，他的具体作用可以查阅其他资料。总的来说，当前线程执行会堵塞在这，直到mcr.writtenToDiskLatch满足了条件。也就是当写入磁盘成功之后，会继续执行下面的操作。

所以，**commit提交之后会有返回结果，同步堵塞直到有返回结果**。


#### 4.1 apply()流程



```java
// SharedPreferencesImpl
@Override
public void apply() {
    final long startTime = System.currentTimeMillis();

    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }

                if (DEBUG && mcr.wasWritten) {
                    Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                            + " applied after " + (System.currentTimeMillis() - startTime)
                            + " ms");
                }
            }
        };

    QueuedWork.addFinisher(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
            @Override
            public void run() {
                awaitCommit.run();
                QueuedWork.removeFinisher(awaitCommit);
            }
        };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    notifyListeners(mcr);
}


```

- 加入到QueuedWork中，是一个单线程的操作。
- 没有返回结果。
- 默认会有100ms的延迟

#### 4.3  QueuedWork

```java
// QueuedWork
public static void queue(Runnable work, boolean shouldDelay) {
    Handler handler = getHandler();

    synchronized (sLock) {
        sWork.add(work);

        if (shouldDelay && sCanDelay) {
            handler.sendEmptyMessageDelayed(QueuedWorkHandler.MSG_RUN, DELAY);
        } else {
            handler.sendEmptyMessage(QueuedWorkHandler.MSG_RUN);
        }
    }
}


```

- 当apply()方式提交的时候，默认消息会延迟发送100毫秒，避免频繁的磁盘写入操作。
- 当commit()方式，调用QueuedWork的queue()时，会立即向handler()发送Message。

#### 主线程ANR问题

```java
public static void waitToFinish() {
    // ...
    processPendingWork();
    // ...
}

private static void processPendingWork() {
    long startTime = 0;

    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }

    synchronized (sProcessingWork) {
        LinkedList<Runnable> work;

        synchronized (sLock) {
            work = (LinkedList<Runnable>) sWork.clone();
            sWork.clear();

            // Remove all msg-s as all work will be processed now
            getHandler().removeMessages(QueuedWorkHandler.MSG_RUN);
        }

        if (work.size() > 0) {
            for (Runnable w : work) {
                w.run();
            }

            if (DEBUG) {
                Log.d(LOG_TAG, "processing " + work.size() + " items took " +
                        +(System.currentTimeMillis() - startTime) + " ms");
            }
        }
    }
}
```

waitToFinish()会将储存在QueuedWork的操作一并处理掉。什么时候呢？在Activiy的 onPause()、BroadcastReceiver的onReceive()以及Service的onStartCommand()方法之前都会调用waitToFinish()。大家知道这些方法都是执行在主线程中，一旦waitToFinish()执行超时，就会跑出ANR。



### 5.常见问题

commit()方法和apply()方法的区别?



commit()方法是同步的有返回结果，同步保证使用Countdownlatch，即使同步但不保证往磁盘的写入是发生在当前线程的。apply()方法是异步的具体发生在QueuedWork中，里面维护了一个单线程去执行磁盘写入操作。

commit()和apply()方法其实都是Block主线程。commit()只要在主线程调用就会堵塞主线程;apply（）方法磁盘写入操作虽然是异步的，但是当组件(Activity Service BroadCastReceiver)这些系统组件特定状态转换的时候，会把QueuedWork中未完成的那些磁盘写入操作放在**主线程**执行，且如果比较耗时会产生ANR，手动可怕。







https://juejin.cn/post/6844904022063710221

