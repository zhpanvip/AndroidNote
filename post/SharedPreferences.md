SharedPreferences是Android中轻量级的数据存储方式，适用于保存简单的数据类型。其内部是以xml结构保存在/data/data/{包名}/shared_prefs文件夹下的。数据以键值对的形式保存，如下：

```xml
<map>
    <float name="isFloat" value="1.5" />
    <string name="isString">Android</string>
    <int name="isInt" value="1" />
    <long name="isLong" value="1000" />
    <boolean name="isBoolean" value="true" />
    <set name="isStringSet">
        <string>element 1</string>
        <string>element 2</string>
        <string>element 3</string>
    </set>
</map>
```



### 1.简单使用

```java
// 通过Context获取SharedPreferencesImpl实例
SharedPreferences sp = Context.getSharedPreferences("my_sp", Context.MODE_PRIVATE);
// 通过SP获取EditorImpl实例
SharedPreferences.Editor edit = sp.edit();
// 存入key为value，值为text的数据。
edit.putString("value","test");
// 提交
edit.apply(); // 或者edit.commit();
// 从SP中读取value对应的值
sp.getString("value","");
```



### 2.获取SharedPreferences

通过ContextImpl的getSharedPreferences获取

```java
// 缓存了name到文件的映射
private ArrayMap<String, File> mSharedPrefsPaths;
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {
		// 查找name对应的SP文件
    File file;
    synchronized (ContextImpl.class) {
        if (mSharedPrefsPaths == null) {
            mSharedPrefsPaths = new ArrayMap<>();
        }
        // 根据name从缓存中取出SP文件
        file = mSharedPrefsPaths.get(name);
        if (file == null) {
            // 没有在缓存中则根据name创建xml文件
            file = getSharedPreferencesPath(name);
            // 保存name与创建的文件
            mSharedPrefsPaths.put(name, file);
        }
    }
  	// 真正获取SP
    return getSharedPreferences(file, mode);
}

// 调用makeFilename创建xml文件：
@Override
public File getSharedPreferencesPath(String name) {
    return makeFilename(getPreferencesDir(), name + ".xml");
}
```

mSharedPrefsPaths记录了所有SP文件，以文件名为key，具体文件为value。

```java 

@Override
public SharedPreferences getSharedPreferences(File file, int mode) {
    SharedPreferencesImpl sp;
    synchronized (ContextImpl.class) {
        // 获取缓存集合ArrayMap
        final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
        // 从缓存中读取SP
        sp = cache.get(file);
        if (sp == null) {
						// ...
          	// 实例化SP
            sp = new SharedPreferencesImpl(file, mode);
            // 放入缓存
            cache.put(file, sp);
            return sp;
        }
    }
		// ...
    return sp;
}

// sSharedPrefsCache:以包名为key, 二级key是以SP文件, 以SharedPreferencesImpl为value的嵌套map结构. 这里需要sSharedPrefsCache是静态类成员变量, 每个进程是保存唯一一份,
private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;

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

可以看到ContextImpl的getSharedPreferences最终获取到的是SharedPreferencesImpl实例对象。SharedPreferencesImpl的构造方法如下：

```java
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    // 创建备份文件
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    mThrowable = null;
    // 开启从磁盘加载
    startLoadFromDisk();
}
```



```java
@UnsupportedAppUsage
private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }
    // 开启子线程从磁盘加载数据
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}
```



```java
private void loadFromDisk() {
    // ...

    Map<String, Object> map = null;
    StructStat stat = null;
    Throwable thrown = null;
    try {
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
                // 读取xml文件流
                str = new BufferedInputStream(
                        new FileInputStream(mFile), 16 * 1024);
                // 将xml文件解析成map集合
                map = (Map<String, Object>) XmlUtils.readMapXml(str);
            } catch (Exception e) {
                Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    }  catch (Throwable t) {
        thrown = t;
    }

    synchronized (mLock) {
        mLoaded = true;
        mThrowable = thrown;
        try {
            if (thrown == null) {
                if (map != null) {
                    // 赋值给mMap
                    mMap = map;
                    mStatTimestamp = stat.st_mtim;
                    mStatSize = stat.st_size;
                } else {
                    mMap = new HashMap<>();
                }
            }
            // In case of a thrown exception, we retain the old map. That allows
            // any open editors to commit and store updates.
        } catch (Throwable t) {
            mThrowable = t;
        } finally {
            // 加载结束后唤醒所有线程
            mLock.notifyAll();
        }
    }
}
```

SharedPreferencesImpl的构造方法中会从磁盘上读取xml文件，并将该文件解析成一个Map集合，并赋值mMap这个成员变量。

### 2.从SP中读取数据

以getString为例：

```java
@Override
@Nullable
public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        // 这里会调用awaitLoadedLocked方法看是否已加载了xml文件
        // 如果没加载则阻塞主线程并加载xml文件
        awaitLoadedLocked();
      	// 加载完成后才会从内从中读取key对应的value
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}
```

getString中使用synchronized对代码块进行了加锁，并且首先会调用awaitLoadedLocked方法去加载xml文件。

```java
@GuardedBy("mLock")
private void awaitLoadedLocked() {
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
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

可以看到awaitLoadedLocked方法中会先判断是否已经从磁盘加载了xml文件，如果没有则会通过BlockGuard开启一个线程从磁盘中读取文件。没太搞明白BlockGuard是什么东西...它会通过某种方式调用了loadFromDisk方法？然后在loadFromDisk中加载文件完成后将Loaded置为true，并且 mLock.notifyAll()来唤起线程？不管怎样在while之前的代码是一个耗时操作，在mLoaded为false的时候会一直阻塞主线程，如果xml文件中存储数据过多会长时间阻塞主线程，造成一定性能问题。

### 3.Editor

接着调用了SharedPreferencesImpl的edit方法来获取Editor的实例，代码如下：

```java
// SharedPreferencesImpl
@Override
public Editor edit() {
    synchronized (mLock) {
        // 这里再次出现awaitLoadedLocked方法
        awaitLoadedLocked();
    }

    return new EditorImpl();
}
```

edit方法中也会先通过awaitLoadedLocked方法查看是否已经加载xml文件，如果没有加载则阻塞线程加载xml，加载成功后则会实例化一个EditorImpl实例对象。



```java
public final class EditorImpl implements Editor {
    private final Object mEditorLock = new Object();

    @GuardedBy("mEditorLock")
    private final Map<String, Object> mModified = new HashMap<>();

    @GuardedBy("mEditorLock")
    private boolean mClear = false;

    @Override
    public Editor putString(String key, @Nullable String value) {
        synchronized (mEditorLock) {
            mModified.put(key, value);
            return this;
        }
    }

		//... 省略了各种put方法

    @Override
    public Editor remove(String key) {
        synchronized (mEditorLock) {
            mModified.put(key, this);
            return this;
        }
    }

    @Override
    public Editor clear() {
        synchronized (mEditorLock) {
            mClear = true;
            return this;
        }
    }

    @Override
  public void apply() {
      final long startTime = System.currentTimeMillis();
			// 提交到内存
      final MemoryCommitResult mcr = commitToMemory();
      final Runnable awaitCommit = new Runnable() {
              @Override
              public void run() {
                  try {
                      // 写入磁盘文件
                      mcr.writtenToDiskLatch.await();
                  } catch (InterruptedException ignored) {
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

    @Override
  public boolean commit() {
      long startTime = 0;
			// 提交到内存
      MemoryCommitResult mcr = commitToMemory();
			// 写入磁盘
      SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null);
      try {
          mcr.writtenToDiskLatch.await();
      }
      // ...
      notifyListeners(mcr);
      // 返回文件操作结果
      return mcr.writeToDiskResult;
  }

}
```

可以看到，调用putXXX方法时仅仅是将数据存入了mModified的map集合中，只有调用apply或者commit方法的时候才会去commitToMemory和写入磁盘。



### 4. commit与apply

**commit方法**

```java
 @Override
 public boolean commit() {
      long startTime = 0;
			// 将数据更新到内存
      MemoryCommitResult mcr = commitToMemory();
			// 写入磁盘
      SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null);
      try {
          mcr.writtenToDiskLatch.await();
      }
      // ...
      notifyListeners(mcr);
      // 返回文件操作结果
      return mcr.writeToDiskResult;
  }
```



```java
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    final boolean isFromSyncCommit = (postWriteRunnable == null);

    final Runnable writeToDiskRunnable = new Runnable() {
            @Override
            public void run() {
                synchronized (mWritingToDiskLock) {
                    // 将数据写入文件
                    writeToFile(mcr, isFromSyncCommit);
                }
                synchronized (mLock) {
                    mDiskWritesInFlight--;
                }
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

    if (isFromSyncCommit) {
        // commit方法会进入此处
        boolean wasEmpty = false;
        synchronized (mLock) {
            //commitToMemory过程会加1,则wasEmpty=true
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

每次commit是把全部数据更新到文件, 所以每个文件的数据量必须保证足够精简.



**apply方法**

```java
public void apply() {
    final long startTime = System.currentTimeMillis();
		// 更新到内存
    final MemoryCommitResult mcr = commitToMemory();

    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    // 进入等待状态
                    mcr.writtenToDiskLatch.await();
                }
                // ...
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
		// 开始写入操作
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    notifyListeners(mcr);
}
```





```java
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    final boolean isFromSyncCommit = (postWriteRunnable == null);

    final Runnable writeToDiskRunnable = new Runnable() {
            @Override
            public void run() {
                synchronized (mWritingToDiskLock) {
                   //执行文件写入操作
                    writeToFile(mcr, isFromSyncCommit);
                }
                synchronized (mLock) {
                    mDiskWritesInFlight--;
                }
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

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
    // 往系统的队列中发送任务，然后在工作线程中执行任务
    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```

enqueueDiskWrite 方法中首先判断的postWriteRunnable 是否等于null，如果等于空了，就在当前调用的地方执行写入操作，如果不是就往QueuedWork 队列中发送任务.

在apply方法中调用enqueueDiskWrite方法的时候最后一个参数是不等于空的，也就是说我们要执行一个异步任务，最终这异步任务的执行是在QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit)方法中

### QueuedWork

QueuedWork就是android系统提供的一个执行异步任务的工具类，内部的实现逻辑的就是创建一个HandlerThread作为工作线程，然后QueuedWorkHandler和这个HandlerThread进行管理，每当有任务添加进来就在这个异步线程中执行，这个异步线程的名字queued-work-looper

```java
public class QueuedWork {


		// 创建handle的过程
    private static Handler getHandler() {
        synchronized (sLock) {
            if (sHandler == null) {
                HandlerThread handlerThread = new HandlerThread("queued-work-looper",
                        Process.THREAD_PRIORITY_FOREGROUND);
                handlerThread.start();

                sHandler = new QueuedWorkHandler(handlerThread.getLooper());
            }
            return sHandler;
        }
    }


    public static void addFinisher(Runnable finisher) {
        synchronized (sLock) {
            sFinishers.add(finisher);
        }
    }


    public static void removeFinisher(Runnable finisher) {
        synchronized (sLock) {
            sFinishers.remove(finisher);
        }
    }


    public static void waitToFinish() {
        // ... 省略，见下文
    }

    // 首先往sWork 添加一个任务，sWork是一个LinkedList，这个队列中数据最终在
    // queued-work-looper 线程中依次得到执行
    @UnsupportedAppUsage
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
                for (Runnable w
                     : work) {
                    w.run();
                }
            }
        }
    }

    private static class QueuedWorkHandler extends Handler {
        static final int MSG_RUN = 1;

        QueuedWorkHandler(Looper looper) {
            super(looper);
        }

        public void handleMessage(Message msg) {
            if (msg.what == MSG_RUN) {

                processPendingWork();
            }
        }
    }
}
```



## apply方法为什么会引起ANR？

 apply的中写入操作也是在异步线程执行，不会导致主线程卡顿，但是如果异步任务执行时间过长，当ActvityThread执行了handleStopActivity或者handleServiceArgs或者handlePauseActivity 等方法的时候都会调用QueuedWork.waitToFinish()方法,而此方法中会在异步任务执行完成前一直阻塞住主线程，所以卡顿问题就产生了

```java
public static void waitToFinish() {
    long startTime = System.currentTimeMillis();
    boolean hadMessages = false;

    Handler handler = getHandler();

    synchronized (sLock) {
        if (handler.hasMessages(QueuedWorkHandler.MSG_RUN)) {
            handler.removeMessages(QueuedWorkHandler.MSG_RUN);

            if (DEBUG) {
                hadMessages = true;
                Log.d(LOG_TAG, "waiting");
            }
        }

        sCanDelay = false;
    }

    StrictMode.ThreadPolicy oldPolicy = StrictMode.allowThreadDiskWrites();
    try {
        processPendingWork();
    } finally {
        StrictMode.setThreadPolicy(oldPolicy);
    }

    try {
        while (true) {
            Runnable finisher;

            synchronized (sLock) {
                // 关键点，读取sFinishers中的数据
                finisher = sFinishers.poll();
            }

            if (finisher == null) {
                break;
            }
						// 执行run方法
            finisher.run();
        }
    } finally {
        sCanDelay = true;
    }

    synchronized (sLock) {
        long waitTime = System.currentTimeMillis() - startTime;

        if (waitTime > 0 || hadMessages) {
            mWaitTimes.add(Long.valueOf(waitTime).intValue());
            mNumWaits++;
        }
    }
}
```

会从sFinishers队列中取出数据然后执行run方法，别忘了在apply的方法中，我们还添加了QueuedWork.addFinisher(awaitCommit); 这个awaitCommit 就得到执行了但是awaitCommit中的代码是阻塞的代码，等待写入线程执行完毕才能唤起此线程

```java
final Runnable awaitCommit = new Runnable() {
        public void run() {
            try {
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException ignored) {
            }
        }
    };
```

如果 apply中的写入代码不执行完，主线程就一直卡住了，也就出现了我们上面的问题



参考：https://www.jianshu.com/p/40e42da910e2
