
## 1.View的绘制流程概述
**初始化PhoneWindow与WindowManager**

Activity在ActivityThread的performLaunchActivity中创建，创建完成后会首先执行attach方法，在attach方法中实例化PhoneWindow并关联WindowManager：

```java
// Activity#attach()：
   
	final void attach(...) {
        attachBaseContext(context);
		//初始化 PhoneWindow
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        
		//初始化 WindowManager
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        mWindowManager = mWindow.getWindowManager();
    }

	// Window#setWindowManager()：

    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        //...
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }

```
**初始化DecorView**

接下来在Activity的onCreate方法中会通过setContentView初始化DecorView,并将Activity的布局文件添加到DecorView的content布局中。在AppCompactActivity中还为DecorView设置了主题等布局。

**ViewRootImpl关联DecorView**

在handleResumeActivity中，通过WindowManager的实现类WindowManagerImpl的addView方法将DecorView添加到了Window，在addView方法中通过WindowManagerGlobal的实例去addView，并且会实例化一个ViewRootImpl,最后把DecorView传递给了ViewRootImpl的setView。ViewRootImpl是DecorView的管理者，负责View树的测量、布局、绘制，以及通过Choreographer来控制View的刷新。

```
// ActivityThread
@Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {

        // ... 省略无关代码

        final Activity a = r.activity;
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                // 此处调用WindowManagerImpl的addView将DecorView添加到了WindowManagerImpl中
                wm.addView(decor, l);
            } else {           
                a.onWindowAttributesChanged(l);
            }
    }
   // ... 省略无关代码
}
```

```
// WindowManagerImpl
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```
```
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {

        // ... 省略无关代码

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // 初始化ViewRootImpl
            root = new ViewRootImpl(view.getContext(), display);

            try {
                // 将DocerView添加到ViewRootImpl
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
            
            }
        }
    }
```

**PhoneWindow与WindowManagerService建立联系**

WMS是所有Window窗口的管理者，负责Window的添加和删除、Surface的管理和事件派发等，因此在Activity中的PhoneWindow对象需要显示等操作，就必须与WMS交互才能进行。

在ViewRootImpl的setView方法中，会调用requestLayout,并且通过WindowSession的addToDisplay与WMS进行交互。WMS会为每一个Window关联一个WindowStatus.

**建立与 SurfaceFlinger 的连接**

SurfaceFlinger 主要是进行 Layer 的合成和渲染。

在 WindowStatus 中，会创建 SurfaceSession，SurfaceSession 会在 Native 层构造一个 SurfaceComposerClient 对象，它是应用程序与 SurfaceFlinger 沟通的桥梁。

**申请 Surface**

经过步骤四和步骤五之后，ViewRootImpl 与 WMS、SurfaceFlinger 都已经建立起连接，但此时 View 还没显示出来，我们知道，所有的 UI 最终都要通过 Surface 来显示，那么 Surface 是什么时候创建的呢？

这就要回到前面所说的 ViewRootImpl 的 requestLayout 方法了，首先会 checkThread 检查是否是主线程，然后调用 scheduleTraversals 方法，scheduleTraversals 方法会先设置同步屏障，然后通过 Choreographer 类在下一帧到来时去执行 doTraversal 方法。简单来说，Choreographer 内部会接受来自 SurfaceFlinger 发出的 Vsync 垂直同步信号，这个信号周期一般是 16ms 左右。doTraversal 方法首先会先移除同步屏障，然后 performTraversals 真正进行 View 的绘制流程，即调用 performMeasure、performLayout、performDraw。不过在它们之前，会先调用 relayoutWindow 通过 WindowSession 与 WMS 进行交互，即把 Java 层创建的 Surface 与 Native 层的 Surface 关联起来。

```
    // ViewRootImpl
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```
```
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
    @UnsupportedAppUsage
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

    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }

    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            // 移除同步屏障
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }
            //  通过该方法开启View的绘制流程，会调用performMeasure方法、performLayout方法和performDraw方法。
            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```
**View的测量流程**

View的绘制是从performTraversals开始，进行测量、布局和绘制的流程。
首先根据DecorView的宽高来生成MeasureSpec，然后执行performMeasure流程。performMeasure方法会去调用DecorView的measure方法，mesaure是一个final修饰的方法，开发者无法重写measure，在measure会进行一些公用的测量，然后会调用onMeasure，并将MeasureSpec传递给onMeasure

View的测量机制本质非常简单，顾名思义，其目的便是 测量控件的宽高值，围绕该目的，View的设计者通过代码编织了一整套复杂的逻辑：

- 1、对于子View而言，其本身宽高直接受限于父View的 布局要求，举例来说，父View被限制宽度为40px,子View的最大宽度同样也需受限于这个数值。因此，在测量子View之时，子View必须已知父View的布局要求，这个 布局要求，  Android中通过使用 MeasureSpec 类来进行描述。

- 2、对于完整的测量流程而言，父控件必然依赖子控件宽高的测量；若子控件本身未测量完毕，父控件自身的测量亦无从谈起。Android中View的测量流程中使用了非常经典的 递归思想：对于一个完整的界面而言，每个页面都映射了一个View树，其最顶端的父控件测量开始时，会通过 遍历 将其 布局要求 传递给子控件，以开始子控件的测量，子控件在测量过程中也会通过 遍历 将其 布局要求 传递给它自己的子控件，如此往复一直到最底层的控件...这种通过遍历自顶向下传递数据的方式我们称为 测量过程中的“递”流程。而当最底层位置的子控件自身测量完毕后，其父控件会将所有子控件的宽高数据进行聚合，然后通过对应的 测量策略 计算出父控件本身的宽高，测量完毕后，父控件的父控件也会根据其所有子控件的测量结果对自身进行测量，这种从底部向上传递各自的测量结果，最终完成最顶层父控件的测量方式我们称为测量过程中的“归”流程，至此界面整个View树测量完毕。

**View的布局流程**

测量流程 的目的是 测量控件宽高 ，但只获取控件的宽高实际上是不够的，对于ViewGroup而言还需要一套额外的逻辑，负责对所有子控件进行对应策略的布局，这就是 布局流程（layout）。

- 1.对于叶子节点的View而言，其本身没有子控件，因此一般情况下仅需要记录自己在父控件的位置信息，并不需要处理为子控件布局的逻辑；
- 2.对于整体的布局流程而言，子控件的位置必然交由父控件布置，和 测量流程 一样，Android中布局流程中也使用了递归思想：对于一个完整的界面而言，每个页面都映射了一个View树，其最顶端的父控件开始布局时，会通过自身的布局策略依次计算出每个子控件的位置——值得一提的是，为了保证控件树形结构的 内部自治性，每个子控件的位置为 相对于父控件坐标系的相对位置 ，而不是以屏幕坐标系为准的绝对位置。位置计算完毕后，作为参数交给子控件，令子控件开始布局；如此往复一直到最底层的控件，当所有控件都布局完毕，整个布局流程结束。

## 2.MeasureSpec是什么？
MeasureSpec是一个32位的int值，高位代表测量模式,低30位代表测量大小,MeasureSpec将SpecMode和SpecSize封装成一个int值避免了过多的内存分配。其中测量模式表示控件对应的宽高模式：

- **UNSPECIFIED**：父元素不对子元素施加任何束缚，子元素可以得到任意想要的大小；日常开发中自定义View不考虑这种模式，可暂时先忽略；
 - **EXACTLY**：父元素决定子元素的确切大小，子元素将被限定在给定的边界里而忽略它本身大小；这里我们理解为控件的宽或者高被设置为 match_parent 或者指定大小，比如20dp；
- **AT_MOST**：子元素至多达到指定大小的值；这里我们理解为控件的宽或者高被设置为wrap_content。



对于模式和大小值的获取，只需要通过位运算即可。例如，通过mode和size相加就可以得到MeasureSpec:
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/a1ceee4d75304d8fe10442039c417e74.png#pic_center)
当需要获得Mode的时候只需要用measureSpec与MODE_TASK相与即可:
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/4a9d2c83362652f7a7c8f4d013d15a7e.png#pic_center)
想获得size的话只需要只需要measureSpec与~MODE_TASK相与即可，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/8efd96d45db0e23ffb73f2bb47ade95c.png#pic_center)


[View 工作原理](https://github.com/Omooo/Android-Notes/blob/master/blogs/Android/View%20%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.md)
[反思|Android LayoutInflater机制的设计与实现](https://juejin.cn/post/6844903919286485000)
 [反思|Android View机制设计与实现：测量流程](https://juejin.cn/post/6844903909320835080)
 [反思|Android View机制设计与实现：布局流程](https://juejin.cn/post/6844903912315551757)

 [深入理解Android之View的绘制流程](https://www.jianshu.com/p/060b5f68da79)

## 3.requestLayout()、invalidate()与postInvalidate()有什么区别？

requestLayout()：该方法会递归调用父窗口的requestLayout()方法，直到触发ViewRootImpl的performTraversals()方法，此时mLayoutRequestede为true，会触发onMesaure()与onLayout()方法，不一定会触发onDraw()方法。

invalidate()：该方法递归调用父View的invalidateChildInParent()方法，直到调用ViewRootImpl的invalidateChildInParent()方法，最终触发ViewRootImpl的performTraversals()方法，此时mLayoutRequestede为false，不会触发onMesaure()与onLayout()方法，当时会触发onDraw()方法。

postInvalidate()：该方法功能和invalidate()一样，只是它可以在非UI线程中调用。一般说来需要重新布局就调用requestLayout()方法，需要重新绘制就调用invalidate()方法。