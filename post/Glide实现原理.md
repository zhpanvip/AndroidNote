
## 1.Glide.with方法分析
```
  RequestManager requestManager = Glide.with((Activity) this)
```
### （1）Glide生命周期管控概述
Glide的With方法会创建一个空白的Fragment来监听Activity的生命周期。因为空白Fragment绑定在Activity上，所有空白Fragment能够感知到Activity的生命周期变化，当Fragment感知到Activity生命周期变化后则会回调ActivityFragmentLifecycle中的生命周期方法，ActivityFragmentLifecycle的生命周期会遍历lifecycleListeners，并分别调用LifecycleListener的生命周期方法。而RequestManager实现了LifecycleListener，所以它的的生命周期方法会被调用。接着，在RequestManager中会调用TargetTracker的生命周期，而TargetTracker内部是一个Target的集合，Target实现了LifecycleListener,在TargetTracker的生命周期方法中会遍历集合执行所有LifecycleListener的生命周期，而在Glide中像ImageViewTarget等许多类都实现了Target接口，因此，这些类生命周期的方法都会被调用。Glide以此实现生命周期管控。

调用with方法时内部会根据是否是主线程进行不同方式的管理：
- 如果是子线程，那么作用域是Application，也就是Glide加载的图片不会跟随Activity的销毁而回收，而是会与APP的生命周期同步。
- 如果是主线程，因为with的重载方法比较多，因此又需要分两种情况讨论：

    - 如果with的参数是View/Activity/Framgnet,那么会生成空白的Fragment来监控Activity的生命周期;

    - 如果with的参数是Application或者ServiceContext，那么与在子线程中调用with一样，作用域都会是Application。

### (2)Glide内部的空白Fragment是如何生成的？
在Glide的with方法中最终都会调用RequestManagerRetriever的get方法
```
 // RequestManagerRetriever

 @NonNull
  public RequestManager get(@NonNull Activity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else if (activity instanceof FragmentActivity) {
      return get((FragmentActivity) activity);
    } else {
      assertNotDestroyed(activity);
      frameWaiter.registerSelf(activity);
      android.app.FragmentManager fm = activity.getFragmentManager();
      return fragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }

@NonNull
  public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) { // 子线程中不会初始化空白Fragment
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      frameWaiter.registerSelf(activity);
      FragmentManager fm = activity.getSupportFragmentManager();
      return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }

@NonNull
  private RequestManager supportFragmentGet(
      @NonNull Context context,
      @NonNull FragmentManager fm,
      @Nullable Fragment parentHint,
      boolean isParentVisible) {
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      // This is a bit of hack, we're going to start the RequestManager, but not the
      // corresponding Lifecycle. It's safe to start the RequestManager, but starting the
      // Lifecycle might trigger memory leaks. See b/154405040
      if (isParentVisible) {
        requestManager.onStart();
      }
      current.setRequestManager(requestManager);
    }
    return requestManager;
  }

@NonNull
  private SupportRequestManagerFragment getSupportRequestManagerFragment(
      @NonNull final FragmentManager fm, @Nullable Fragment parentHint) {
    SupportRequestManagerFragment current =
        (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
      current = pendingSupportRequestManagerFragments.get(fm);
      if (current == null) {
        current = new SupportRequestManagerFragment();
        current.setParentFragmentHint(parentHint);
        pendingSupportRequestManagerFragments.put(fm, current);
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    return current;
  }
```
可以看到最终在getSupportRequestManagerFragment方法中实例化了一个SupportRequestManagerFragment。SupportRequestManagerFragment中持有了Lifecycle，并且会在SupportRequestManagerFragment的生命周期方法中回到Lifecycle的生命周期方法：

```
public class SupportRequestManagerFragment extends Fragment {

  private final ActivityFragmentLifecycle lifecycle;

  public SupportRequestManagerFragment() {
    this(new ActivityFragmentLifecycle());
  }

  @VisibleForTesting
  @SuppressLint("ValidFragment")
  public SupportRequestManagerFragment(@NonNull ActivityFragmentLifecycle lifecycle) {
    this.lifecycle = lifecycle;
  }

  @Override
  public void onStart() {
    super.onStart();
    lifecycle.onStart();
  }

  @Override
  public void onStop() {
    super.onStop();
    lifecycle.onStop();
  }

  @Override
  public void onDestroy() {
    super.onDestroy();
    lifecycle.onDestroy();
    unregisterFragmentWithRoot();
  }
 // .... 省略无关代码
}
```

### （3）Glide如何保证在同一个Activity中多次调用with方法只生成一个Fragment的？
在RequestManagerRetriever的getSupportRequestManagerFragment方法中生成Fragment的时候采用了双重校验来保证在同一个Activity中只会生成一个空白Fragment，其实现代码如下：

```
@NonNull
  private SupportRequestManagerFragment getSupportRequestManagerFragment(
      @NonNull final FragmentManager fm, @Nullable Fragment parentHint) {
    SupportRequestManagerFragment current =
        (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
      current = pendingSupportRequestManagerFragments.get(fm);
      if (current == null) {
        current = new SupportRequestManagerFragment();
        current.setParentFragmentHint(parentHint);
        pendingSupportRequestManagerFragments.put(fm, current);
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    return current;
  }
```
首先，第一重校验会通过RequestManagerRetriever中的pendingSupportRequestManagerFragments集合去获取Fragment，如果不为空说明已经创建了Fragment，直接返回即可。如果是空的话则会实例化Fragment，并将其存储到pendingSupportRequestManagerFragments集合中，然后通过commitAllowingStateLoss提交Fragment事务。
但是由于commitAllowingStateLoss是通过Handler实现的，不会立马执行，因此经过第一重校验后还是可能会出现重复Fragment的情况，因此Glide又进行了第二重校验，即通过handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();这句代码让commitFragment立即执行，以此保证了Fragment不会生成多个。

### （4）Glide生命周期的管控：

定义了Lifecycle接口，通过Lifecycle来管理LifecycleListener。
```
 public interface Lifecycle{
   void addLifecycleListener(LifecycleListener listener);
   void removeLifecycleListener(LifecycleListener listener);
 }
```
ApplicationLifecycle与ActivityFragmentLifecycle都实现了Lifecycle接口。

ApplicationLifecycle表示生命周期作用域为Application，对应了子线程调用with与with参数为ApplicationContext/ServiceContext的情况。

```
// ApplicationLifecycle没有通过空白Fragment监听生命周期，如果是此类情况，则资源只有在APP被杀死的时候才会被释放
public class ApplicationLifecycle implements Lifecycle{
  @Override
  public void addListener(LifecycleListener listener){
     // App启动了,执行onStart
     listener.onStart();
  }

  @Override
  public void removeListener(LifecycleListener listener){
    // App被杀死，无需做任何事情
  }
}
```
由于这种情况资源的生命周期等同于APP的生命周期，因此过多此种情况必然引起APP性能问题，要避免在子线程中使用Glide或者with中传入AppcationContext/ServiceContext.

非Application作用域，这里会生成一个空白Fragment来监听生命周期
ActivityFragmentLifecycle也实现Lifecycle,它代表的作用域为Activity，ActivityFragmentLifecycle中维护了一个LifecycleListener的集合，同时还添加了几个生命周期方法，在生命周期方法中遍历lifecycleListeners集合并调用它的生命周期方法：
```
class ActivityFragmentLifecycle implements Lifecycle {
  private final Set<LifecycleListener> lifecycleListeners =
      Collections.newSetFromMap(new WeakHashMap<LifecycleListener, Boolean>());
  private boolean isStarted;
  private boolean isDestroyed;


  @Override
  public void addListener(@NonNull LifecycleListener listener) {
    lifecycleListeners.add(listener);

    if (isDestroyed) {
      listener.onDestroy();
    } else if (isStarted) {
      listener.onStart();
    } else {
      listener.onStop();
    }
  }

  @Override
  public void removeListener(@NonNull LifecycleListener listener) {
    lifecycleListeners.remove(listener);
  }

  void onStart() {
    isStarted = true;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onStart();
    }
  }

  void onStop() {
    isStarted = false;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onStop();
    }
  }

  void onDestroy() {
    isDestroyed = true;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onDestroy();
    }
  }
}
```

LifecycleListener同样也是一个接口，同样管理了三个生命周期方法：
```
 public interface LifecycleListener{
   void onStart();
   void onStop();
   void onDestory();
 }
```

实现LifecycleListener的类是RequestManager，RequestManager中部分代码如下：
```
/**
   * Lifecycle callback that registers for connectivity events (if the
   * android.permission.ACCESS_NETWORK_STATE permission is present) and restarts failed or paused
   * requests.
   */
  @Override
  public synchronized void onStart() {
    resumeRequests();
    targetTracker.onStart();
  }

  /**
   * Lifecycle callback that unregisters for connectivity events (if the
   * android.permission.ACCESS_NETWORK_STATE permission is present) and pauses in progress loads.
   */
  @Override
  public synchronized void onStop() {
    pauseRequests();
    targetTracker.onStop();
  }

  /**
   * Lifecycle callback that cancels all in progress requests and clears and recycles resources for
   * all completed requests.
   */
  @Override
  public synchronized void onDestroy() {
    targetTracker.onDestroy();
    for (Target<?> target : targetTracker.getAll()) {
      clear(target);
    }
    targetTracker.clear();
    requestTracker.clearRequests();
    lifecycle.removeListener(this);
    lifecycle.removeListener(connectivityMonitor);
    Util.removeCallbacksOnUiThread(addSelfToLifecycle);
    glide.unregisterRequestManager(this);
  }
```
在这三个方法中，生命周期转移到了TargetTracker，TargetTracker是一个集合类，内部维护了一个Target的集合，Target实现了LifecycleListener接口，TargetTracker的源码如下：

```
public final class TargetTracker implements LifecycleListener {
  private final Set<Target<?>> targets =
      Collections.newSetFromMap(new WeakHashMap<Target<?>, Boolean>());

  public void track(@NonNull Target<?> target) {
    targets.add(target);
  }

  public void untrack(@NonNull Target<?> target) {
    targets.remove(target);
  }

  @Override
  public void onStart() {
    for (Target<?> target : Util.getSnapshot(targets)) {
      target.onStart();
    }
  }

  @Override
  public void onStop() {
    for (Target<?> target : Util.getSnapshot(targets)) {
      target.onStop();
    }
  }

  @Override
  public void onDestroy() {
    for (Target<?> target : Util.getSnapshot(targets)) {
      target.onDestroy();
    }
  }

  @NonNull
  public List<Target<?>> getAll() {
    return Util.getSnapshot(targets);
  }

  public void clear() {
    targets.clear();
  }
}
```
由此生命周期的方法最终交给了Target，实现Target的类有ImageViewTarget、CustomViewTarget等，


## Glide#load方法分析

```
RequestBuilder<Drawable> requestBuilder = requestManager.load("url");
```
Glide的load方法很简单，

```
// RequestManager

public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
}
```

```
// RequestBuilder 

public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
}

private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    if (isAutoCloneEnabled()) {
      return clone().loadGeneric(model);
    }
    this.model = model;
    isModelSet = true;
    // return自身
    return selfOrThrowIfLocked();
  }
```

## Glide#into方法
```
requestBuilder.into(imageView);
```
Glide的into方法非常复杂，这里无法详细分析，只能写出大致流程，
以load(String url)为例：

- 1.into方法会将ImageView封装成一个DrawableImageViewTarget；
- 2.构建Request(SingleRequest)对象，并调用request.begin；
- 3.在begin方法中通过Engine#load方法载入图片；
- 4.根据图片URL、width、height、签名等信息生成一个key，作为缓存的key，并从缓存中查找对应的资源。
- 5.在发起请求之前先通过Engine缓存机制获取缓存的资源，通过key查找活动缓存和内存缓存有没有缓存的资源，如果有则直接回调反馈。
- 6.如果没有缓存则在EngineJob中构建一个异步任务;
- 7.执行Request之前，先检测DiskCache中有没有本地缓存;
- 8.如果没有则通过HttpUrlConnection网络请求,返回输入流;
- 9.解析输入流InputStream进行采样压缩，最终拿到Bitmap;
- 10.对Bitmap转换成Drawable;
- 11.构建磁盘缓存
- 12.构建内存缓存
- 13.最终回到ImageViewTarget的子类DrawableImageViewTarget显示图片

## Glide缓存机制

Glide采用四级缓存机制，分别为活动缓存、LRU内存缓存、LRU磁盘缓存、访问HTTP/本地图片。Glide在缓存图片时会根据图片的width、height、url以及签名等信息经过base64编码后生成一个EngineKey，作为缓存查找的key.Glide加载一个图片的流程如下：

1.当加载一个图片时，会首先通过EngineKey查找活动缓存是否有该图片，如果有，则直接显示图片；

2.如果活动缓存没有该图片，则通过EngineKey查找LRU内存缓存，如果LRU有该图片，则将图片移动（剪切）到活动缓存，然后显示图片；

3.如果LRU中没有该图片，则会访问LRU磁盘缓存，并通过EngineKey查找图片资源，如果LRU缓存中有该图片，则将图片复制到活动缓存，然后显示图片；

4.如果磁盘缓存也没有找到，则通过HTTP/本地存储加载图片流，接着将图片保存到磁盘缓存，然后加载到活动缓存，之后显示图片；

5.当Activity结束的时候，会将活动缓存资源全部移动到LRU内存缓存。

## 什么是LRU算法？
LRU为最近最少使用算法，当put的时候超过了maxSize，则会移除最少的一个元素，并将当前元素插入。LRUCache内部使用LinkedHashMap实现。LinkedHashMap构造方法中有一个按照访问排序的参数，可以直接实现LRU。

## Glide已经有了LRU内存缓存，为什么还要在加一个活动缓存？
因为LRUCache会在内存不足时将最近最好使用的一个资源移除掉，如果Activity正在显示这个图片那么就会造成崩溃。因此，又添加了一个活动缓存。活动缓存是一个非LRU缓存，通过HashMap实现。且一个资源图片只会在活动缓存和LRU内存缓存仅保留一份。


