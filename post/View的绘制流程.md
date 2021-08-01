## 一 View的绘制流程概述

### 1.Framework层与绘制流程相关的知识点

#### （1）初始化PhoneWindow与WindowManager

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

#### （2）初始化DecorView

接下来在Activity的onCreate方法中会通过setContentView初始化DecorView,并将Activity的布局文件添加到DecorView的content布局中。在AppCompactActivity中还为DecorView设置了主题等布局。

#### （3）ViewRootImpl关联DecorView

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

#### （4）PhoneWindow与WindowManagerService建立联系

WMS是所有Window窗口的管理者，负责Window的添加和删除、Surface的管理和事件派发等，因此在Activity中的PhoneWindow对象需要显示等操作，就必须与WMS交互才能进行。

在ViewRootImpl的setView方法中，会调用requestLayout,并且通过WindowSession的addToDisplay与WMS进行交互。WMS会为每一个Window关联一个WindowStatus.

#### （5）建立与 SurfaceFlinger 的连接

SurfaceFlinger 主要是进行 Layer 的合成和渲染。

在 WindowStatus 中，会创建 SurfaceSession，SurfaceSession 会在 Native 层构造一个 SurfaceComposerClient 对象，它是应用程序与 SurfaceFlinger 沟通的桥梁。

#### （6）申请 Surface

经过步骤四和步骤五之后，ViewRootImpl 与 WMS、SurfaceFlinger 都已经建立起连接，但此时 View 还没显示出来，我们知道，所有的 UI 最终都要通过 Surface 来显示，那么 Surface 是什么时候创建的呢？

这就要回到前面所说的 ViewRootImpl 的 requestLayout 方法了，首先会 checkThread 检查是否是主线程，然后调用 scheduleTraversals 方法，scheduleTraversals 方法会先设置同步屏障，然后通过 Choreographer 类在下一帧到来时去执行 doTraversal 方法。简单来说，Choreographer 内部会接受来自 SurfaceFlinger 发出的 Vsync 垂直同步信号，这个信号周期一般是 16ms 左右。doTraversal 方法首先会先移除同步屏障，然后 performTraversals 真正进行 View 的绘制流程，即调用 performMeasure、performLayout、performDraw。不过在它们之前，会先调用 relayoutWindow 通过 WindowSession 与 WMS 进行交互，即把 Java 层创建的 Surface 与 Native 层的 Surface 关联起来。

```java
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

### 2.View及ViewGroup的绘制流程

View的绘制是从performTraversals开始，进行测量、布局和绘制的流程。

```java
// ViewRootImpl
private void performTraversals() {
     // 根据Window的宽高与LayoutParams来计算DecorView的MeasureSpec
     int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
     int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
     ...............
    //measur过程
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
     ...............
    //layout过程
    performLayout(lp, desiredWindowWidth, desiredWindowHeight);
     ...............
    //draw过程
    performDraw();
}
```



```java
// ViewRootImpl
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {

    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

```java
public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                  @MeasureSpecMode int mode) {
    if (sUseBrokenMakeMeasureSpec) {
        return size + mode;
    } else {
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
```

#### MeasureSpec是什么？

上述代码中用到了MeasureSpec，MeasureSpec是一个32位的int值，高位代表测量模式,低30位代表测量大小,MeasureSpec将SpecMode和SpecSize封装成一个int值避免了过多的内存分配。

##### MeasureSpec的创建规则

MeasureSpec是有Parent的SpecMode与子View的LayoutParams共同确定的，其计算规则如下表：


| childLayoutParams/ParentSpecMode | EXACTLY                 | AT_MOST                 | UNSPECIFIED             |
| -------------------------------- | ----------------------- | ----------------------- | ----------------------- |
| dp/px                            | EXACTLY<br /> childSize | EXACTLY<br /> childSize | EXACTLY<br /> childSize |
| match_parent                     | EXACTLY<br />parentSize | AT_MOST<br />parentSize | UNSPECIFIED<br />0      |
| wrap_content                     | AT_MOST<br />parentSize | AT_MOST<br />parentSize | UNSPECIFIED<br />0      |

#### MeasureSpec的计算方式

其中测量模式表示控件对应的宽高模式：

- **UNSPECIFIED**：父元素不对子元素施加任何束缚，子元素可以得到任意想要的大小；日常开发中自定义View不考虑这种模式，可暂时先忽略；
 - **EXACTLY**：父元素决定子元素的确切大小，子元素将被限定在给定的边界里而忽略它本身大小；这里我们理解为控件的宽或者高被设置为 match_parent 或者指定大小，比如20dp；
- **AT_MOST**：子元素至多达到指定大小的值；这里我们理解为控件的宽或者高被设置为wrap_content。



对于模式和大小值的获取，只需要通过位运算即可。例如，通过mode和size相加就可以得到MeasureSpec:
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/a1ceee4d75304d8fe10442039c417e74.png#pic_center)
当需要获得Mode的时候只需要用measureSpec与MODE_TASK相与即可:
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/4a9d2c83362652f7a7c8f4d013d15a7e.png#pic_center)
想获得size的话只需要只需要measureSpec与~MODE_TASK相与即可，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/8efd96d45db0e23ffb73f2bb47ade95c.png#pic_center)



#### （1）View的测量流程

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






####  （2）ViewGroup的测量流程

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

#### （3）View的布局流程

Layout的作用是ViewGroup用来确定子元素的位置，当ViewGroup的位置被确定后，它在onLayout中会遍历所有子元素并调用其layout方法，在layout方法中onLayout方法又会被调用。layout方法确定View本身的位置，而onLayout方法会确定所有子元素的位置。

#### View中的layout

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

#### （4）ViewGroup中的onLayout

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



#### （5）View的Draw过程

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



## 二、View绘制流程常见问题

### 1.View的getMeasureWidht与getWidth有什么区别?

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

### 2.requestLayout()、invalidate()与postInvalidate()有什么区别？

requestLayout()：该方法会递归调用父窗口的requestLayout()方法，直到触发ViewRootImpl的performTraversals()方法，此时mLayoutRequestede为true，会触发onMesaure()与onLayout()方法，不一定会触发onDraw()方法。

invalidate()：该方法递归调用父View的invalidateChildInParent()方法，直到调用ViewRootImpl的invalidateChildInParent()方法，最终触发ViewRootImpl的performTraversals()方法，此时mLayoutRequestede为false，不会触发onMesaure()与onLayout()方法，当时会触发onDraw()方法。

postInvalidate()：该方法功能和invalidate()一样，只是它可以在非UI线程中调用。一般说来需要重新布局就调用requestLayout()方法，需要重新绘制就调用invalidate()方法。





 [View 工作原理](https://github.com/Omooo/Android-Notes/blob/master/blogs/Android/View%20%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.md)


 [反思|Android View机制设计与实现：测量流程](https://juejin.cn/post/6844903909320835080)


 [深入理解Android之View的绘制流程](