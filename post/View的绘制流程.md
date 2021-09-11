### View绘制前相关流程概述

1. 在Activity被实例化后调用Activity的attach方法时会实例化PhoneWindow，并通过PhoneWindow的setWindowManager方法与WindowManager关联。
2. Activity的onCreate方法中会通过setContentView实例化DecorView，并将Activity中的布局文件添加到DecorView的content中。
3. ActivityThread的handleResumeActivity中，会将DecorView添加到WindowManager中，即执行[WMS添加Window的流程](https://github.com/zhpanvip/AndroidNote/wiki/WMS核心分析)
4. 在Window添加过程中会实例化ViewRootImpl,并且将DecorView传递给ViewRootImpl。
5. ViewRootImpl是一个Android视图层接口的顶部，是View和WindowManager的桥梁，ViewRootImpl与Choreographer协同完成View的绘制，也负责接收底层的触摸事件的中转分发。

### [Choreographer汇总](https://github.com/zhpanvip/AndroidNote/wiki/Choreographer详解)

### View的绘制流程概述

1. TraversalRunnable的run方法调用doTraversal方法开启View的绘制流程
2. doTraversal方法中首先会根据mTraversalScheduled的标记判断是否需要执行绘制，接着移除同步屏障，调用performTraversals开始绘制。
3. performTraversals中会根据Window的LayoutParams计算DecorView的MeasureSpec，然后分别执行performMeasure、performLayout、performDraw。
4. View的测量可分为View的测量和ViewGroup的测量，View的测量流程：
   - 调用View的measure方法执行公用的测量，然后执行onMeasure
   - onMeasure中会通过getDefaultSize确定并设置View自身的大小
   - getDefaultSize确定View的大小是根据父View传来的MeasureSpec参数确定的
5. ViewGroup除了测量本身大小，还会通过measureChildren遍历子View，并通过measureChild测量子View的大小：
   - measureChild中拿到子View的LayoutParams并计算子View的MeasureSpec然后调用child.measure方法
   - ViewGroup是一个抽象类，没有重写onMeasure方法，具体的测量逻辑是在其子View中根据子View的特性进行测量的
6. 当View的大小确定后，会调用performaLayout开始View的布局流程:
   - layout方法中通过setFrame来确定View的四个顶点，即确定View在父View中的位置，并得到一个boolean值表示是否需要重新布局
   - 如果需要重新布局则调用onLayout开始布局，onLayout方法的作用是父View确定子View的位置。View和ViewGroup中都没有onLayout的具体实现。需要子View根据自身特性进行布局。
7. 经过测量和布局流程后会确定View的大小及位置，接着调用performDraw->DecorView.draw开始View的绘制过程。
   - drawBackground绘制背景
   - onDraw 绘制自己
   - dispatchDraw 绘制child
   - onDrawScrollBars 绘制装饰

## 一 、View绘制流程前期准备

### 1. Activity的初始化与Window的添加

handleLaunchActivity中会首先调用performLaunchActivity来创建一个Activity，并且执行Activity的attach，接着通过Instrumentation的callActivityOnCreate方法调用了Activity的onCreate。代码如下：

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

        Activity activity = null;
        // ...
   			// 通过Instrumentation实例化Activity
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
						// 执行Activity的attach方法
            activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);
								// ... 
          
                // 调用Activity的onCreate方法
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
            }
            r.setState(ON_CREATE);
        } 
				// ...

        return activity;
    }
```

#### （1）PhoneWindow关联WindowManager

Activity在ActivityThread的performLaunchActivity中创建，创建完成后会首先执行attach方法，在attach方法中实例化PhoneWindow并关联WindowManager：

```
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
```

Activity的attach方法中首先实例化了一个PhoneWindow，然后调用PhoneWindow的setWindowManager去初始化WindowManager，可以看下setWindowManager的代码

```java
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

这里就是保证会创建一个WindowManagerImpl，并赋值给PhoneWindow的成员变量mWindowManager。WindowManagerImpl这个类在WMS中已经多次见到。

#### （2）初始化DecorView

`mInstrumentation.callActivityOnCreate` 这行代码会通过Instrumentation来调用Activity的onCreate方法。代码如下：

```java
// Instrumentation.java

public void callActivityOnCreate(Activity activity, Bundle icicle) {
  	调用Activity的performCreate
    activity.performCreate(icicle);
    // ...
}
// Activity.java 

final void performCreate(Bundle icicle) {
    performCreate(icicle, null);
}

final void performCreate(Bundle icicle, PersistableBundle persistentState) {
		// ...
  
    if (persistentState != null) {
        // 调用onCreate
        onCreate(icicle, persistentState);
    } else {
        onCreate(icicle);
    }
  
		// ...
}
```

可以看到，最终会调用Activity的onCreate方法，我们在onCreeate方法中会通过setContentView初始化DecorView,并将Activity的布局文件添加到DecorView的content布局中。在AppCompactActivity中还为DecorView设置了主题等布局。

#### （3）ViewRootImpl关联DecorView

在handleResumeActivity中，通过WindowManager的实现类WindowManagerImpl的addView方法将DecorView添加到了Window，这里其实就是一个Window的添加的过程了。

```java
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

Window的添加过程参考：[WindowManagerService](https://github.com/zhpanvip/AndroidNote/wiki/WMS核心分析)

SurfaceFlinger申请Surface：[SurfaceFlinger](https://github.com/zhpanvip/AndroidNote/wiki/SurfaceFlinger)

### 2. ViewRootImpl与Choreographer

ViewRootImpl 与 WMS、SurfaceFlinger 建立起连接后，此时 View 还没显示出来，所有的 UI 最终都要通过 Surface 来显示，那么 Surface 是什么时候创建的呢？

这就要回到前面所说的 ViewRootImpl 的 requestLayout 方法了，首先会 checkThread 检查是否是主线程，然后调用 scheduleTraversals 方法，scheduleTraversals 方法会先设置同步屏障，然后通过 Choreographer 类在下一帧到来时去执行 doTraversal 方法。简单来说，Choreographer 内部会接受来自 SurfaceFlinger 发出的 Vsync 垂直同步信号，这个信号周期一般是 16ms 左右。doTraversal 方法首先会先移除同步屏障，然后 performTraversals 真正进行 View 的绘制流程，即调用 performMeasure、performLayout、performDraw。不过在它们之前，会先调用 relayoutWindow 通过 WindowSession 与 WMS 进行交互，即把 Java 层创建的 Surface 与 Native 层的 Surface 关联起来。

在ViewRootImpl的setView中会先执行一次requestLayout，这次requestLayout的执行时机是向AMS成功添加窗口后，收到Input事件之前执行的，因为只有先完成测量布局绘制流程后，各种触摸事件才有一有。

同时，我们也可以自行调用View的requestLayout来发起View的测量、布局和绘制流程。看下requestLayout的代码：

```java
// ViewRootImpl.java

    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            // 检查是否是在主线程中，如果不是主线程则直接抛出异常
            checkThread();
            // mLayoutRequested标记设置为true，在同一个Vsync周期内，执行多次requestLayout的流程
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```

scheduleTraversals方法中会发出一个同步屏障消息，并且将这次requestLayout请求放到TraversalRunnable的run方法中。然后通过Choreographer将TraversalRunnable发送出去，代码如下:

```java
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
						// ...
        }
    }
```

Choreographer会通过FrameDisplayEventReceiver监听Vsync信号，等到Vsync到来时变化回调FrameDisplayEventReceiver的onVsync方法。并最终执行Choreographer中的doFrame方法,这个方法里边会去统计帧绘制的时间、真绘制信息、以及回调Callback,在ScheduleCallback中最终执行了TraversalRunnable的run方法。

```java
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

TraversalRunnable的run方法执行了doTraversal。doTraversal方法中会首先将同步屏障移除，然后调用performTraversals方法开启View的绘制流程。



## 二、View绘制流程

performTraversals方法中会依次执行performMeasure、performLayout和performDraw方法来开始View的测量、布局和绘制流程。

```java
// ViewRootImpl
private void performTraversals() {
     // 根据Window的宽高与DecorView的LayoutParams来计算DecorView的MeasureSpec
     int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
     int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
  
     // ...
  
    //measur过程
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
  
    // ...
  
    //layout过程
    performLayout(lp, desiredWindowWidth, desiredWindowHeight);
    // ...
  
    //draw过程
    performDraw();
}
```



### 1. MeasureSpec

MeasureSpec是一个32位的int值，高位代表测量模式,低30位代表测量大小,MeasureSpec将SpecMode和SpecSize封装成一个int值避免了过多的内存分配。

MeasureSpec类中主要是对MeasureSpec的int值进行打包和拆包的逻辑。核心代码如下：

```java
// View#MeasureSpec

public static class MeasureSpec {
    private static final int MODE_SHIFT = 30;
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

    @IntDef({UNSPECIFIED, EXACTLY, AT_MOST})
    @Retention(RetentionPolicy.SOURCE)
    public @interface MeasureSpecMode {}

    public static final int UNSPECIFIED = 0 << MODE_SHIFT;

    public static final int EXACTLY     = 1 << MODE_SHIFT;

    public static final int AT_MOST     = 2 << MODE_SHIFT;

  	// 将size与mode打包到一个int中
    public static int makeMeasureSpec( int size, int mode) {
        if (sUseBrokenMakeMeasureSpec) {
            return size + mode;
        } else {
            return (size & ~MODE_MASK) | (mode & MODE_MASK);
        }
    }

    /**
     * 从MeasureSpec的int值中解析出mode
     */
    @MeasureSpecMode
    public static int getMode(int measureSpec) {
        //noinspection ResourceType
        return (measureSpec & MODE_MASK);
    }
		/**
		 * 从MeasureSpec的int值中解析出size
		 */
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }

}
```

其中测量模式表示控件对应的宽高模式：

- **UNSPECIFIED**：父容器不对View做任何限制，要多大就多大，这种情况一般用于系统内部，表示一种测量状态。
 - **EXACTLY**：父容器已经检测出View所需要的 精确大小，这个时候View的最终大小就是SpecSize所指定的值。对应LayoutParams中的MATCH_PARENT和具体的数值两种模式。
- **AT_MOST**：父容器制定了一个可用大小的SpecSize，View的大小不能大于这个值。具体是什么值还要看不同的View的具体表现。对应LayoutParams中的WRAP_CONTENT。



performTraversals方法中首先通过getRootMeasureSpec根据Window的宽高与DecorView的LayoutParams来计算得到DecorView的MeasureSpec。代码如下;

```java
// ViewRootImpl
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
		// MATCH_PARENT对应EXACTLY模式    
    case ViewGroup.LayoutParams.MATCH_PARENT:
        // 将windowSize值与MeasureSpec.EXACTLY打包成一个int值MeasureSpec
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    // WRAP_CONTENT对应AT_MOST模式    
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // 将windowSize值与MeasureSpec.AT_MOST打包成一个int值MeasureSpec
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // 将DecorView的具体宽高值与MeasureSpec.EXACTLY打包成一个int值MeasureSpec
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

- LayoutParams.MATCH_PARENT：精确模式，大小就是窗口的大小
- LayoutParams.WRAP_CONTENT：最大模式，大小不确定，但是不能超过窗口的大小。
- 固定大小（例如100dp）：精确模式，大小为LayoutParams中指定的大小。

对于普通的View来说，它的MeasureSpec是由父View的MeasureSpec和自身的LayoutParmas决定的。ViewGroup中通过getChildMeasureSpec来计算子View的Measure，

```java
// ViewGroup.java

protected void measureChild(View child, int parentWidthMeasureSpec,
        int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();
		// 计算子View Width的MeasureSpec
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    // 计算子View Height的MeasureSpec
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);
		// 调用子View的measure开始子View的测量
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```



```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY: // 父View是EXACTLY模式
        if (childDimension >= 0) { // 子View的LayoutParams宽/高设置了精确值
            // 子View的大小就是该子View的LayoutParams中设置的值
            resultSize = childDimension;
            // 子View的模式设置为EXACTLY
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) { 
            // 子View的LayoutParams的宽/高设置了MATCH_PARENT
            // 子View的size设置为父View的size
            resultSize = size;
            // 子View的模式设置为EXACTLY
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // 子View的LayoutParams的宽/高设置了WRAP_CONTENT
            // 子View的size设置为父View的size
            resultSize = size;
            // 子View的模式设置为AT_MOST
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST: // 父View是AT_MOST模式
        if (childDimension >= 0) {
            // 子View的大小就是该子View的LayoutParams中设置的值
            resultSize = childDimension;
            // 子View的模式设置为EXACTLY
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // 子View的size设置为父View的size
            resultSize = size;
            // 子View的模式设置为AT_MOST
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // 子View的size设置为父View的size
            resultSize = size;
            // 子View的模式设置为AT_MOST
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
		// 不讨论这种情况
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let them have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

对于上面的代码可以归纳为如下表格：


| ParentSpecMode<br />childLayoutParams | EXACTLY                 | AT_MOST                 | UNSPECIFIED             |
| :------------------------------------ | ----------------------- | ----------------------- | ----------------------- |
| 固定大小 dp/px                        | EXACTLY<br /> childSize | EXACTLY<br /> childSize | EXACTLY<br /> childSize |
| match_parent                          | EXACTLY<br />parentSize | AT_MOST<br />parentSize | UNSPECIFIED<br />0      |
| wrap_content                          | AT_MOST<br />parentSize | AT_MOST<br />parentSize | UNSPECIFIED<br />0      |





### 2. View的测量流程

#### （1）View的测量

首先根据Window的宽高与LayoutParamters来生成MeasureSpec，然后执行performMeasure流程。performMeasure方法会去调用DecorView的measure方法，mesaure是一个final修饰的方法，开发者无法重写measure，在measure会进行一些公用的测量，然后会调用onMeasure，并将MeasureSpec传递给onMeasure.



```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
      if (mView == null) {
          return;
      }
      try {
          mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
      } finally {
          Trace.traceEnd(Trace.TRACE_TAG_VIEW);
      }
}
```

通过mView.measure将绘制流程交给了DecorView,即执行View中的measure方法。measure方法中会进行一些公用的逻辑处理，接着就会调用onMeasure方法

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
		onMeasure(widthMeasureSpec, heightMeasureSpec);  
}
```

而onMeasure的逻辑也非常简单，仅仅是设置了View的默认大小

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

getSuggestedMinimumWidth/Height会查看是否给View设置了背景和最小宽高，然后取最大值作为建议宽高

```java
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

getDefaultSize中只有会根据measureSpec拿到测量模式和测量宽高，可以看到在AT_MOST和EXACTLY时都返回了MeasureSpec的specSize。即默认情况下要么是固定值要么是match_parent.

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

由上述可以看出对于View的测量流程而言，其本身宽高直接受限于父View的 布局要求，举例来说，父View被限制宽度为40px,子View的最大宽度同样也需受限于这个数值。因此，在测量子View之时，子View必须已知父View的布局要求，这个 布局要求，  Android中通过使用 MeasureSpec 类来进行描述。

#### （2）ViewGroup的测量

对于ViewGroup除了完成自身的测量外，还需要调用所有子元素的measure,各个子元素再递归执行测量。在ViewGroup是一个抽象类，它没有重写onMeasure方法，但提供了一个叫measureChildren的方法：

```java
// ViewGroup
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}

```

从代码中看到，ViewGroup会对每一个子元素进行Measure。measureChild的方法如下：

```java
protected void measureChild(View child, int parentWidthMeasureSpec,
        int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

mesaureChild中取出子元素的LayoutParams然后通过getChildMeasureSpec来创建子元素的MeasureSpec，接着将MeasureSpec传递个子View进行测量。

ViewGroup中并没有实质的测量流程，具体的实现是在其子类中，因为子类需要根据自己的特性来进行测量。以LinearLayout的竖直布局为例：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}
```

measureVertical方法中的一部分代码：

```java
// LinearLayout
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // ...
    for (int i = 0; i < count; ++i) {
          final View child = getVirtualChildAt(i);
          measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                  heightMeasureSpec, usedHeight);

          final int childHeight = child.getMeasuredHeight();
          if (useExcessSpace) {
              // Restore the original height and record how much space
              // we've allocated to excess-only children so that we can
              // match the behavior of EXACTLY measurement.
              lp.height = 0;
              consumedExcessSpace += childHeight;
          }

          final int totalLength = mTotalLength;
          mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                 lp.bottomMargin + getNextLocationOffset(child));

          if (useLargestChild) {
              largestChildHeight = Math.max(childHeight, largestChildHeight);
          }
    }
  
    // 测量所有子View的总大小在加上上下padding，确定自身的高度。
    mTotalLength += mPaddingTop + mPaddingBottom; 
    heightSize = Math.max(heightSize, getSuggestedMinimumHeight());

    int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
    heightSize = heightSizeAndState & MEASURED_SIZE_MASK; 
  
    // ...
    
    // 测量自View完成后设置自身大小
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                heightSizeAndState);
}  

// LinearLayout
void measureChildBeforeLayout(View child, int childIndex,
        int widthMeasureSpec, int totalWidth, int heightMeasureSpec,
            int totalHeight) {
    measureChildWithMargins(child, widthMeasureSpec, totalWidth,
            heightMeasureSpec, totalHeight);
}

// ViewGroup
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}


```

遍历所有的子View，然后调用measureChildBeforeLayout对子View进行测量。并将测量子View的高度累加存储在mTotalLength中。测量完毕后将测量最终高度加上上下padding作为测量值来设置自身高度。当然，这里自身高度跟LinearLayout的高度参数也有关系，即可能是match_parent/wrap_content或者是固定值。而这是通过resolveSizeAndState方法实现的：

```java
// View
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    final int specMode = MeasureSpec.getMode(measureSpec);
    final int specSize = MeasureSpec.getSize(measureSpec);
    final int result;
    switch (specMode) {
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
                result = specSize | MEASURED_STATE_TOO_SMALL;
            } else {
                result = size;
            }
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            result = size;
    }
    return result | (childMeasuredState & MEASURED_STATE_MASK);
}
```



### 3.View的布局流程

Layout的作用是ViewGroup用来确定子元素的位置，当ViewGroup的位置被确定后，它在onLayout中会遍历所有子元素并调用其layout方法，在layout方法中onLayout方法又会被调用。layout方法确定View本身的位置，而onLayout方法会确定所有子元素的位置。

#### （1）View中的布局

```java
// 伪代码实现
public void layout(int l, int t, int r, int b) {
  // 1.决定是否需要重新进行测量流程（onMeasure）
  if(needMeasureBeforeLayout) {
    onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec)
  }

  // 先将之前的位置信息进行保存
  int oldL = mLeft;
  int oldT = mTop;
  int oldB = mBottom;
  int oldR = mRight;
  // 2.将自身所在的位置信息进行保存；
  // 3.判断本次布局流程是否引发了布局的改变；
  boolean changed = setFrame(l, t, r, b);

  if (changed) {
    // 4.若布局发生了改变，令所有子控件重新布局；
    onLayout(changed, l, t, r, b);
    // 5.若布局发生了改变，通知所有观察布局改变的监听发送通知
    mOnLayoutChangeListener.onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
  }
}
```

layout中首先会通过setFrame方法来设定View四个顶点的位置，View四个顶点的位置一旦确定那么View在父容器中的位置也就确定了。

#### （2）ViewGroup中的布局

在View确定自身位置后，接着调用onLayout方法，这个方法的用途是父View如容器确定子元素的位置。onLayout的具体实现和具体的布局有关，所以View和ViewGroup中都没有真正实现onLayout方法。以LinearLayout为例：

```java
// LinearLayout
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l, t, r, b);
    } else {
        layoutHorizontal(l, t, r, b);
    }
}
```

以竖直布局为例，来看layoutVertical方法

```java
void layoutVertical(int left, int top, int right, int bottom) {
	
    // ...
  
    final int count = getVirtualChildCount();

    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();

            final LinearLayout.LayoutParams lp =
                    (LinearLayout.LayoutParams) child.getLayoutParams();

            // ...

            if (hasDividerBeforeChildAt(i)) {
                childTop += mDividerHeight;
            }

            childTop += lp.topMargin;
            setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                    childWidth, childHeight);
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

            i += getChildrenSkipCount(child, i);
        }
    }
}
```

在这里该方法会遍历所有子元素并调用setChildFrame方法来为子元素指定对应的位置，其中childTop会逐步增大，意味着后边的子元素会被放在靠下的位置。这刚好是竖直方向LinearLayout的特性。至于setChildFrame仅仅是调用了子元素的layout方法而已，这样父元素在layout方法中完成自己的定位后，就通过onLayout方法调用子元素的layout方法，子元素又会通过自己的layout方法来确定自己的位置，这样一层一层的传递下去就完成了整个View树的layout过程。setFrameLayout代码如下：

```Java
private void setChildFrame(View child, int left, int top, int width, int height) {
    child.layout(left, top, left + width, top + height);
}
```

setFrame中的width和height就是子元素的测量宽高。而在layout方法中会通过setFrame取设置子元素四个顶点的位置，在setFrame中有如下赋值语句，这样一来子元素的位置就确定了：

```java
mLeft = left;
mTop = top;
mRight = right;
mBottom = bottom;
```

### 4. View的绘制流程

draw的过程是将View绘制到屏幕上，View的绘制过程遵循如下几步：

- 绘制背景 drawBackground(canvas)
- 绘制自己（onDraw)
- 绘制child(dispatchDraw)
- 绘制装饰(onDrawScrollBars)

```java
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */

    // Step 1, draw the background, if needed
    int saveCount;

    drawBackground(canvas);

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        drawAutofilledHighlight(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // Step 7, draw the default focus highlight
        drawDefaultFocusHighlight(canvas);

        if (debugDraw()) {
            debugDrawFocus(canvas);
        }

        // we're done...
        return;
    }

   // ...
}
```

View绘制过程的传递是通过dispatchDraw来实现的，dispatchDraw会通过遍历所有子元素的draw方法，如此draw事件就一层一层的



## 三、View绘制流程常见问题

### 1. View的getMeasureWidht与getWidth有什么区别?

View中getWidth与getHeight的代码如下：

```java
public final int getWidth() {
    return mRight - mLeft;
}

public final int getHeight() {
    return mBottom - mTop;
}
```

从getWidht和getHeightd 源码再结合mLeft、mRight、mTop和mBottom这四个遍历值的赋值来看，getWidth的返回值刚好就是View的测量值。因此，在View的默认实现汇总View的测量宽高和最终宽高是相等的，只不过测量宽高形成以Measure过程，而最终宽高形成与View的layout过程，即两者的赋值实际不同。测量宽高的赋值实际稍微早了一些。因此，在日常的开发中，我们可以认为View的测量宽高就等于最终宽高。

### 2. requestLayout()、invalidate()与postInvalidate()有什么区别？

requestLayout()：该方法会递归调用父窗口的requestLayout()方法，直到触发ViewRootImpl的performTraversals()方法，此时mLayoutRequestede为true，会触发onMesaure()与onLayout()方法，不一定会触发onDraw()方法。

invalidate()：该方法递归调用父View的invalidateChildInParent()方法，直到调用ViewRootImpl的invalidateChildInParent()方法，最终触发ViewRootImpl的performTraversals()方法，此时mLayoutRequestede为false，不会触发onMesaure()与onLayout()方法，当时会触发onDraw()方法。

postInvalidate()：该方法功能和invalidate()一样，只是它可以在非UI线程中调用。一般说来需要重新布局就调用requestLayout()方法，需要重新绘制就调用invalidate()方法。

 [View 工作原理](https://github.com/Omooo/Android-Notes/blob/master/blogs/Android/View%20%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.md)


 [反思|Android View机制设计与实现：测量流程](https://juejin.cn/post/6844903909320835080)


 [深入理解Android之View的绘制流程](https://www.jianshu.com/p/060b5f68da79)

