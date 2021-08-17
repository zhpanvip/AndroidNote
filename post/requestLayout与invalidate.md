## requestLayout 与 invalidate执行效果

自定义CustomEmptyView，并在相关方法中打印日志，如下；

```java
public class CustomEmptyView extends View {
    public CustomEmptyView(Context context) {
        super(context);
    }
		public CustomEmptyView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    public CustomEmptyView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        Log.i("CustomEmptyView", "onMeasure");
    }

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed, left, top, right, bottom);
        Log.i("CustomEmptyView", "onLayout");
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Log.i("CustomEmptyView", "onDraw");
    }
}
```

调用CustomEmptyView的invalidate()方法，打印日志如下：

> 2019-03-26 17:32:34.739 8075-8075/com.example.myapplication I/CustomEmptyView: onDraw

调用CustomEmptyView的requestLayout()方法，打印日志如下：

> 2019-03-26 17:33:13.497 8075-8075/com.example.myapplication I/CustomEmptyView: onMeasure
> 2019-03-26 17:33:13.501 8075-8075/com.example.myapplication I/CustomEmptyView: onLayout
> 2019-03-26 17:33:13.503 8075-8075/com.example.myapplication I/CustomEmptyView: onDraw

从打印日志可以看出，调用invalidate方法只会执行View的onDraw方法，而调用requestLayout则会重新走View的绘制流程。

## requestLayout()原理分析

```java
@CallSuper
public void requestLayout() {
    if (mMeasureCache != null) mMeasureCache.clear();
		// 如果此时View的还处于layout之前的阶段，则直接调用ViewRootImpl的
    // requestLayoutDuringLayout方法将当前View进行传递
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
        ViewRootImpl viewRoot = getViewRootImpl();
        if (viewRoot != null && viewRoot.isInLayout()) {
            if (!viewRoot.requestLayoutDuringLayout(this)) {
                return;
            }
        }
        mAttachInfo.mViewRequestingLayout = this;
    }
		// 更新 mPrivateFlags 变量
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;

    // 向上递归调用 Parent 的 requestLayout方法，
    if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout();
    }
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
        mAttachInfo.mViewRequestingLayout = null;
    }
}
```

mParent.requestLayout()方法会一层一层的向上递归调用，最终调用ViewRootImpl的requestLayout，代码如下：

```java
// ViewRootImpl
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {\
        // 检查线程，如果不是UI线程则直接抛出异常
        checkThread();
        // 设置mLayoutRequested为true
        mLayoutRequested = true;
        // 调用这个方法，准备开始遍历View树
        scheduleTraversals();
    }
}
```

scheduleTraversals代码多次见到了，比较熟悉。

```java
@UnsupportedAppUsage
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        // mTraversalScheduled 设置为 true
        mTraversalScheduled = true;
        // 通过Handler 发送同步屏障，阻塞非异步消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 通过 mChoreographer post一个mTraversalRunnable，等待VSync信号的到来
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

当收到Vsync信号后则会执行mTraversalRunnable的run方法：

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        // 遍历View树
        doTraversal();
    }
}
```





```
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        // 移除同步屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

				// ...

        //  开启View的绘制流程
        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

在performTraversals方法中会一次调用performMeasure、performLayout、performDraw。进而调用View的measure、layout和draw方法。

在View的measure方法中，有如下代码：



```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // ...
    final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

		// ...

    if (forceLayout || needsLayout) {
        // first clears the measured dimension flag
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

        resolveRtlPropertiesIfNeeded();

        int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            long value = mMeasureCache.valueAt(cacheIndex);
            // Casting a long to int drops the high 32 bits, no mask needed
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
        // ...
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }
```

上述代码中通过mPrivateFlags & PFLAG_FORCE_LAYOUT来判断是否要强制布局，在调用requestLayout的时候，有这样一句代码：mPrivateFlags |= PFLAG_FORCE_LAYOUT; 因此，此时forceLayout为true。上述代码的最后一行  mPrivateFlags |= PFLAG_LAYOUT_REQUIRED; 标记了需要layout。

看下layout的代码：

```java
public void layout(int l, int t, int r, int b) {

    // ...
    // 布局是否发生改变
    boolean changed = isLayoutModeOptical(mParent) ? setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    // 发生改变或者标记了PFLAG_LAYOUT_REQUIRED才会走onLayout
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        // ...
    }
    // ...
}
```



## invalidate()原理分析

invalidate的意思是”作废，使无效“，在这里意味着调用invalidate方法会使之前的绘制的View失效。看下invalidate的源码：

```java
 // View

    /**
     * Invalidate the whole view. If the view is visible,
     * {@link #onDraw(android.graphics.Canvas)} will be called at some point in
     * the future.
     * <p>
     * This must be called from a UI thread. To call from a non-UI thread, call
     * {@link #postInvalidate()}.
     */
    public void invalidate() {
        invalidate(true);
    }

    /**
     * This is where the invalidate() work actually happens. A full invalidate()
     * causes the drawing cache to be invalidated, but this function can be
     * called with invalidateCache set to false to skip that invalidation step
     * for cases that do not need it (for example, a component that remains at
     * the same dimensions with the same content).
     *
     * @param invalidateCache Whether the drawing cache for this view should be
     *            invalidated as well. This is usually true for a full
     *            invalidate, but may be set to false if the View's contents or
     *            dimensions have not changed.
     * @hide
     */
    @UnsupportedAppUsage
    public void invalidate(boolean invalidateCache) {
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }
```

可以看到invalidate方法的注释：如果View是可见的，则使整个视图无效，onDraw方法将会在某一时刻被调用。在这个方法中仅仅是调用了invalidate的有参重载方法，并且传入了一个true。可见，invalidate实际操作都在这个重载方法中。

invalidate(boolean invalidateCache)方法的注释可以看到：这里是invalidate方法的工作实际执行的地方，一个完整的invalidate()方法会使绘图缓存失效，如果调用invalidate(boolean)时传入invalidateCache为false，那么就会跳过失效步骤。

接着看invalidateInternal方法源码：

```java
// View

void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
        boolean fullInvalidate) {
    if (mGhostView != null) {
        mGhostView.invalidate(true);
        return;
    }
		// 判断是否要跳过刷新
    if (skipInvalidate()) {
        return;
    }

    // Reset content capture caches
    mPrivateFlags4 &= ~PFLAG4_CONTENT_CAPTURE_IMPORTANCE_MASK;
    mContentCaptureSessionCached = false;

    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
            || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
            || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
            || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
        if (fullInvalidate) {
            mLastIsOpaque = isOpaque();
            mPrivateFlags &= ~PFLAG_DRAWN;
        }
				// 根据条件设置mPrivateFlags
        mPrivateFlags |= PFLAG_DIRTY;
        if (invalidateCache) {
            mPrivateFlags |= PFLAG_INVALIDATED;
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        }

        // Propagate the damage rectangle to the parent view.
        final AttachInfo ai = mAttachInfo;
        // 拿到mParent
        final ViewParent p = mParent;
        if (p != null && ai != null && l < r && t < b) {
            final Rect damage = ai.mTmpInvalRect;
            damage.set(l, t, r, b);
            // 调用parent的invalidateChild
            p.invalidateChild(this, damage);
        }

        // Damage the entire projection receiver, if necessary.
        if (mBackground != null && mBackground.isProjected()) {
            final View receiver = getProjectionReceiver();
            if (receiver != null) {
                receiver.damageInParent();
            }
        }
    }
}
```

在上述方法中首先判断是否要跳过刷新，skipInvalidate源码如下：

```java
private boolean skipInvalidate() {
    //如果ViewGroup没有正在执行动画，或者View不可见，并且当前View的动画是null,并且mParent不是ViewGroup
    return (mViewFlags & VISIBILITY_MASK) != VISIBLE && mCurrentAnimation == null &&
            (!(mParent instanceof ViewGroup) ||
                    !((ViewGroup) mParent).isViewTransitioning(this));
}
```

这里需要知道的是子View会通过mParent持有父View，可以一直向上追溯到DecorView，而在DecorView中mParent持有的是ViewRootImpl。

因此上述第一个中View的mParent不是ViewGroup那么这个View一定是DecorView,也就是说如果DecorView没有设置动画，并且不可见那么久跳过刷新。

接着看invalidateInternal方法，下边会为mPrivateFlags设置各种属性，然后调用了mParent的invalidateChild方法，即可以认为调用了ViewGroup的invalidateChild方法。看下ViewGroup中的实现：

```java
// ViewGroup

public final void invalidateChild(View child, final Rect dirty) {
    final AttachInfo attachInfo = mAttachInfo;
		// ...

    ViewParent parent = this;
    if (attachInfo != null) {
        // ...

        do {
						// ...
            parent = parent.invalidateChildInParent(location, dirty);
						// ...
        } while (parent != null);
    }
}

public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID)) != 0) {
        // either DRAWN, or DRAWING_CACHE_VALID
        if ((mGroupFlags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE))
                != FLAG_OPTIMIZE_INVALIDATE) {
            dirty.offset(location[CHILD_LEFT_INDEX] - mScrollX,
                    location[CHILD_TOP_INDEX] - mScrollY);
            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == 0) {
                dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
            }

            final int left = mLeft;
            final int top = mTop;

            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                if (!dirty.intersect(0, 0, mRight - left, mBottom - top)) {
                    dirty.setEmpty();
                }
            }

            location[CHILD_LEFT_INDEX] = left;
            location[CHILD_TOP_INDEX] = top;
        } else {

            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                dirty.set(0, 0, mRight - mLeft, mBottom - mTop);
            } else {
                // in case the dirty rect extends outside the bounds of this container
                dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
            }
            location[CHILD_LEFT_INDEX] = mLeft;
            location[CHILD_TOP_INDEX] = mTop;

            mPrivateFlags &= ~PFLAG_DRAWN;
        }
        mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        if (mLayerType != LAYER_TYPE_NONE) {
            mPrivateFlags |= PFLAG_INVALIDATED;
        }

        return mParent;
    }

    return null;
}
```

invalidateChild方法中通过do...while调动了自身的invalidateChildInParent方法,而在invalidateChildInParent方法中最终又return了mParent,即通过递归向上查找View，直到mParent为null。

也就是这里的调用链是 ViewGroup.invalidateChildInParent->ViewGroup.invalidateChildInParent->...->DecoreView.invalidateChildInParent->ViewRootImpl.invalidateChildInParent

即最终追溯到了ViewRootImpl的invalidateChildInParent方法中：

```java
@Override
public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
    // 检查线程
    checkThread();

		// ...

    invalidateRectOnScreen(dirty);

    return null;
}
```



```java
private void invalidateRectOnScreen(Rect dirty) {
    final Rect localDirty = mDirty;

    // ...

    if (!mWillDrawSoon && (intersected || mIsAnimating)) {
        // 开启遍历View树
        scheduleTraversals();
    }
}
```

这里与requestLayout不同的是mLayoutRequested没有被置为true。



参考：https://www.jianshu.com/p/ce30f200209e

https://juejin.cn/post/6844903808594624525

requestLayout()：该方法会递归调用父窗口的requestLayout()方法，直到触发ViewRootImpl的performTraversals()方法，此时mLayoutRequestede为true，会触发onMesaure()与onLayout()方法，不一定会触发onDraw()方法。

invalidate()：该方法递归调用父View的invalidateChildInParent()方法，直到调用ViewRootImpl的invalidateChildInParent()方法，最终触发ViewRootImpl的performTraversals()方法，此时mLayoutRequestede为false，不会触发onMesaure()与onLayout()方法，当时会触发onDraw()方法。

postInvalidate()：该方法功能和invalidate()一样，只是它可以在非UI线程中调用。一般说来需要重新布局就调用requestLayout()方法，需要重新绘制就调用invalidate()方法。