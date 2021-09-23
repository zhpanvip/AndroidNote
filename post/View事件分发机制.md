事件在传递到DecorView之前的流程，请参考[ViewRootImpl与事件分发](https://github.com/zhpanvip/AndroidNote/wiki/ViewRootImpl#%E4%B8%89viewrootimpl%E4%B8%8E%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91)

## 事件分发流程

### 1.事件在DecorView中的分发过程

ViewRootImpl在接收到输入事件后最终会调用DecorView的dispatchPointerEvent，由于DecorView没有重写dispatchPointerEvent，所以调用的是View的dispatchPointerEvent，而View的dispatchPointerEvent又中会调用dispatchTouchEvent，这个方法在DecorView中被重写了，代码如下：

```java
// DecorView
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}
```

上述代码中的Callback即为Activity。也就是说事件分发最开始是传递给 DecorView 的，DecorView 的 dispatchTouchEvent 是传给 Window.Callback接口方法 dispatchTouchEvent，而 Activity 实现了 Window.Callback 接口。

紧接着Activity 的 dispatchTouchEvent方法里，是调到 Window 的 superDispatchTouchEvent，

```java
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

Window 的唯一实现类 PhoneWindow 又会把这个事件回传给 DecorView，DecorView 在它的 superDispatchTouchEvent 把事件转交给了 ViewGroup。

```java
    // PhoneWindow
    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
    // DecorView
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```

至此，触摸事件正式进入ViewGroup中，开启事件的分发流程。总结一下事件分发的流程是：

> DecorView -> Activity -> PhoneWindow -> DecorView -> ViewGroup -> View

不难看出，事件的传递过程是一个典型的责任链模式。

### 2.事件分发涉及到的三个方法

- **dispatchTouchEvent**
  事件从Activity经Window，最终会调用到DecorView(即ViewGroup）的dispatchTouchEvent。这个方法的作用是进行事件分发，只要事件能到达ViewGroup那么dispatchTouchEvent方法必定会被调用。dispatchTouchEvent方法返回一个boolean值，代表事件是否被消费。

- **onInterceptTouchEvent**
  该方法是ViewGroup独有的方法，在ViewGroup的dispatchTouchEvent中被调用，返回一个boolean值，用来判断当前ViewGroup是否要拦截事件。如果返回true，则表示该ViewGroup要拦截事件。事件交由ViewGrroup处理。

- **onTouchEvent**
  onTouchEvent方法会在dispatchTouchEvent中调用，作用是处理事件。返回结果表示是否消费当前事件。

View的dispatchTouchEvent事件

以上三个方法的关系可以用以下伪代码表示：

```java
// ViewGroup.dispatchTouchEvent
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean consume = false;
    if (onInterceptTouchEvent(event)) {
        consume = onTouchEvent(event);
    } else {
        consume = child.dispatchTouchEvent(event);
    }
    return consume;
}
```

通过上边的伪代码想要完整的了解事件分发是不现实的。想要完整的理解事件分发就必须深入到View跟ViewGroup内部一探究竟。


### 3.View中对事件的处理

View中dispatchTouchEvent方法实现逻辑比较简单，简化后代码如下：

```java
     public boolean dispatchTouchEvent(MotionEvent event) {

        //... 省略了对滑动等部分的处理逻辑代码
        boolean result = false;
        // 如果View是enable状态，并且设置了TouchListener,则调用TouchListener的onTouch,
        // 如果onTouchEvent方法返回true,则将true作为dispatchTouchEvent的返回值，表示事件
        // 被当前View消费掉了
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
        // 如果View是disable状态或者没有设置TouchListener或者TouchListener的onTouch方法反回了false
        // 那么就调用View自身的onTouchEvent来处理事件，onTouchEvent返回true表示当前View消费了事件。
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    return result;
    }
```

View中的onTouchEvent方法会对事件进行兜底处理，比如在ACTION_UP中调用performClick方法，进而执行View的OnClick方法

```java
public boolean onTouchEvent(MotionEvent event) {
     // 判断View是否是可以点击状态，即View设置了clickable或者longClickable或者contextClickable
     final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
                || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
         switch (action) {
           case MotionEvent.ACTION_UP:
           // 伪代码，这里省略了所有其他逻辑
           performClick();

           break;
        }
    }
}

public boolean performClick() {

    final boolean result;
    final ListenerInfo li = mListenerInfo;
    // 如果设置了OnClickListener，则调用OnClickListener的onClick方法
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }
    return result;
}
```

### 4.ViewGroup中对事件的分发与处理

首先由于ViewGroup是继承View的，那么它的dispatchTouchEvent有两条路可以走，

- （1）可以调用super.dispatchTouchEvent

如果调用super.dispatchTouchEvent意味着调用了View的dispatchTouchEvent中的逻辑，即事件交由自身处理。

- （2）调用自身的dispatchTouchEvent。

如果调用自身的dispatchTouchEvent，即走事件的分发流程，向下分发事件。

至于这两条路是如何选择的，就要详细分析ViewGroup中的dispatchTouchEvent方法了。下面贴一下dispatchTouchEvent简化后的代码：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
   
    boolean handled = false;
        // ACTION_DOWN事件认为是时间序列分发的开始
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // 重置mFirstTouchTarget状态为null
            cancelAndClearTouchTargets(ev);
            // 重置是否允许该ViewGroup拦截事件的标记，意味着disallowIntercept对DOWN事件无效
            resetTouchState();
        }
	// （1）如果是DOWN事件，条件一定成立，先询问自身是否要拦截事件
	// （2）mFirstTouchTarget不为null，说明有子View消费事件
        if (actionMasked == MotionEvent.ACTION_DOWN|| mFirstTouchTarget != null) {
               // 是有有子View禁止当前ViewGroup拦截事件
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;              
                if (!disallowIntercept) {
                    // 如果没有子View禁止当前ViewGroup拦截事件，则询问自身是否拦截事件
                    intercepted = onInterceptTouchEvent(ev);
                } else {
                    // 子View禁止了ViewGroup拦截事件，当前ViewGroup不拦截
                    intercepted = false;
                }
        } else {
            // 能走到此处说明一定不是DOWN事件。
            // 且mFirstTouchTarget为null，即没有子View消费事件
            intercepted = true;
        }

        TouchTarget newTouchTarget = null;
        // 事件是否已经分发过了的标记，用来避免后边重复分发
        boolean alreadyDispatchedToNewTouchTarget = false;
        
        
        // 注意此处的条件!!!!，此处的intercepted是理解事件分发的关键因素,根据上边分析的条件思考一下，
        // intercepted什么时候是false，什么时候是true？ 可以来分情况讨论：
        // (1)如果是ACTION_DOWN事件，并且拦截了ACTION_DOWN事件，则intercepted为true，否则为false
        // (2)如果非ACTION_DOWN事件，且mFirstTouchTarget不为null，拦截了事件则为true，否则为false
        // (3)如果不是ACTION_DOWN事件，且mFirstTouchTarget为null，那么intercepted一定为true。
        //    此时会跳过此处的if语句,直接将事件交给自身处理。之所以会出现这种情况是因为前面拦截了
        //     ACTION_DOWN事件，致使mFirstTouchTarget无法被赋值。
        // 对上边的三种情况以此执行下边的代码，分析会有什么样的结果
        if (!intercepted) {
            // 别管前面的条件如何，能走到这里来，意味着当前ViewGroup不拦截事件，即走上述的第二条路，向子View分发事件  
            
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // 注意此处，只有ACTION_DOWN事件才能进来
                
                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
     
                    final View[] children = mChildren;
                    // 遍历子View,查找消费事件的View/ViewGroup
                    for (int i = childrenCount - 1; i >= 0; i--) {
                      // 按顺序查找子View
                      final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);
                     // 检查子View是否能接收事件，即事件的坐标在子View内，并且View没有在执行动画
                     if (!child.canReceivePointerEvents()
                              || !isTransformedTouchPointInView(x, y, child, null)) {
                          ev.setTargetAccessibilityFocus(false);
                          // 不满足条件，则跳过该子View,继续下一个子View
                          continue;
                      }   
                      // 代码能执行到此处说明已经查找到了符合条件的子View
                      
                      
                      // 查找到目标View之后，通过dispatchTransformedTouchEvent开始向子View分发事件,
                      // 即可以理解为调用child的dispatchTouchEvent方法，并将child的dispatchTouchEvent返回值返回
                      // 那么dispatchTransformedTouchEvent的返回值就意味着子View是否消费了事件。                      
                      if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // 走到这里说明子View中消费了事件，那么就将该View保存到mFirstTouchTarget中
                            // 该保存操作是在addTouchTarget方法中完成
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            // 将alreadyDispatchedToNewTouchTarget置为true，标记已经分发了事件
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                    }
                }
            }
        }

        // mFirstTouchTarget为null，说明没有子View消费事件
        if (mFirstTouchTarget == null) {
            // 调用dispatchTransformedTouchEvent方法,注意这里child的参数是null，
            // 即走上述中的第一条路，调用super.dispatchTouchEvent方法把事件交给自身处理
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // 分析什么情况下能走到此处？该ViewGroup没有拦截ACTION_DOWN事件，并且DOWN事件中遍历子View找到了消费事件的子View，
            // 那么此时会为mFirstTouchTarget赋值，并且结束ACTION_DOWN事件的分发流程（注意，此时DOWN事件不会执行到此处的代码）
            // 接下来会有一系列的MOVE事件，如果此时没有拦截ACTION_MOVE事件，那么ACTION_MOVE事件就会跳过上边的if语句，即跳过事件分发的流程
            // 直接将MOVE事件分发给mFirstTouchTarget中的child，即跳过了遍历子View查找消费事件View的流程，节省了性能。
            TouchTarget target = mFirstTouchTarget;
            // 注意到此处是一个While循环，因为TouchTarget是一个链表，由于多指触控的支持，会形成了一个TouchTarget链表
            // 多个TouchTarget代表就有多个触控点，因此，此处会循环遍历链表，然后将多个触控点分发到child View.
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    // TouchTarget中的child成员才是真正消费事件的View,调用dispatchTransformedTouchEvent将事件交给child.        
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                }
                target = next;
            }
        }
    // 如果子View或者自身消费了事件则返回true，否则返回false
    return handled;
}


private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        // dispatchTransformedTouchEvent的大致逻辑是这样的，这里简化了代码，理解就好。
        if (child == null) {
            // 调用自身的dispatchTouchEvent方法
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            // 调用子View的dispatchTouchEvent方法
            handled = child.dispatchTouchEvent(transformedEvent);
        }
}

```

### 5. ACTION_CANCEL事件的处理

上述对于ViewGroup的dispatchTouchEvent方法的分析忽略了ACTION_CANCEL的处理。ACTION_CANCEL的调用时机是什么时候？看一个具体的例子。

在一个自定义的ViewGroup里边嵌套一个Button。自定义的ViewGroup重写onInterceptTouchEvent,并拦截一次ACTION_MOVE事件。

```java
class CustomViewGroup @JvmOverloads constructor(
  context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : ConstraintLayout(context, attrs, defStyleAttr) {
  var hasInterceptMoveEvent = false
  override fun onInterceptTouchEvent(ev: MotionEvent?): Boolean {
      // 只拦截一次ACTION_MOVE事件
      if (!hasInterceptMoveEvent && ev?.action == MotionEvent.ACTION_MOVE) {
          hasInterceptMoveEvent = true
          return true
      }
      return false
  }

  override fun dispatchTouchEvent(ev: MotionEvent?): Boolean {
      // 打印事件
      Log.d("TAG", "${getAction(ev?.action)}:CustomViewGroup dispatchTouchEvent")
      return super.dispatchTouchEvent(ev)

  }
}  
```

接下来自定义一个Button，并在Button中的dispatchTouchEvent方法中打印事件，如下：

```java
class CusButton @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : Button(context, attrs, defStyleAttr) {
    override fun dispatchTouchEvent(event: MotionEvent?): Boolean {
        Log.d("TAG", "${getAction(event?.action)}:CusButton dispatchTouchEvent")
        return true
    }
}
```

在布局文件中用CustomViewGroup嵌套一个CusButton，接下来，手指按住Button，然后移动，直到手指移除Button外，然后再松开。

此时会收到如下日志

> 04-23 15:13:48.553 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_DOWN:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.553 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_DOWN:CusButton dispatchTouchEvent
> 04-23 15:13:48.574 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.574 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_CANCEL:CusButton dispatchTouchEvent
> 04-23 15:13:48.591 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.607 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.624 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.642 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.658 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.674 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.691 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.708 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.724 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.741 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.758 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.775 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.791 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.808 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.825 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.841 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.859 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.866 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
> 04-23 15:13:48.867 22928-22928/com.lee.myapplication D/TAG: MotionEvent.ACTION_UP:CustomViewGroup dispatchTouchEvent
>
>

可以看到Button收到了一个ACTION_CANCEL事件。在收到这个ACTION_CANCEL事件之后，Button就再也没有收到其它事件。

接下来修改一下上边的例子，CustomViewGroup中不再拦截ACTION_MOVE事件，其它保持不变。重复上边的操作得到如下日志：

>04-23 15:29:24.089 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_DOWN:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.089 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_DOWN:CusButton dispatchTouchEvent
>04-23 15:29:24.104 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.105 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.121 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.121 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.139 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.139 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.154 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.155 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.171 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.171 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.188 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.188 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.205 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.205 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.221 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.221 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.233 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.233 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.250 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.250 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.268 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.268 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.275 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.276 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_MOVE:CusButton dispatchTouchEvent
>04-23 15:29:24.276 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_UP:CustomViewGroup dispatchTouchEvent
>04-23 15:29:24.276 24484-24484/com.lee.myapplication D/TAG: MotionEvent.ACTION_UP:CusButton dispatchTouchEvent

即Button的ACTION_CANCEL事件没有被回调，且抬起手指时CusButton收到了ACTION_UP。

可以总结一下，如果一个子View处理了DOWN事件，那么随之而来的MOVE事件及UP事件也会交给这个View处理。但是在交给它处理之前父View是可以拦截这个事件的。如果父View拦截了这个事件，那么这个子View就会收到一个CANCEL事件。并且后续大MOVE与UP事件都不会再传递给这个View。




```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {  
  boolean handled = false;
  final int action = ev.getAction();
  final int actionMasked = action & MotionEvent.ACTION_MASK;

  final boolean intercepted;
  if (actionMasked == MotionEvent.ACTION_DOWN
      || mFirstTouchTarget != null
  ) {
     // 此处，如果子View没有禁止该View拦截事件，则调用onInterceptTouchEvent方法看自身是否要拦截。
      final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
      if (!disallowIntercept) {
          // 如果拦截了MOVE事件，那么intercepted为true
          intercepted = onInterceptTouchEvent(ev);
          ev.setAction(action); 
      } else {
          intercepted = false;
      }
  } else {
      intercepted = true;
  }
  final boolean canceled = resetCancelNextUpFlag(this)
          || actionMasked == MotionEvent.ACTION_CANCEL;

  TouchTarget newTouchTarget = null;
  boolean alreadyDispatchedToNewTouchTarget = false;
  // intercepted为true，则这个if语句不会被执行，直接跳过
  if (!canceled && !intercepted) {
      // ...
  }

  if (mFirstTouchTarget == null) {
      // mFirstTouchTarget不为空，所以不会走到这里来
  } else {
      // intercepted为true时会执行到这里，这里显示是对mFirstTouchTarget链表的遍历
      TouchTarget predecessor = null;
      TouchTarget target = mFirstTouchTarget;
      while (target != null) {
          final TouchTarget next = target.next;
          if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
              // 事件已经分发到了新的TouchTarget
              handled = true;
          } else {
              // 正常情况这个MOVE事件应该走到这里来，此时由于intercepted为true，那么cancelChild就一定为true
              final boolean cancelChild = resetCancelNextUpFlag(target.child)
                      || intercepted;
              // 看dispatchTransformedTouchEvent源码，此时cancelChild为true
              if (dispatchTransformedTouchEvent(
                      ev, cancelChild,
                      target.child, target.pointerIdBits
                  )
              ) {
                  handled = true;
              }

              // CANCEl事件交给子View之后，这里判断如果子View被cancel了，那么就将这个View从mFirstTouchTarget链表中移除
              // 意味着后续的MOVE事件与UP事件这个View就再也不会收到
              if (cancelChild) {
                  if (predecessor == null) {
                      mFirstTouchTarget = next;
                  } else {
                      predecessor.next = next;
                  }
                  target.recycle();
                  target = next;
                  continue;
              }
          }
          predecessor = target;
          target = next;
      }
  }

  return handled;
}  
```



```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;

    final int oldAction = event.getAction();
    // cancel为true
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        // 将事件设置为ACTION_CANCEL
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            // 子View为null，则将这个事件交给自身
            handled = super.dispatchTouchEvent(event);
        } else {
            // 将这个事件分发给子View，那么子View就会收到一个ACTION_CANCEL事件
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
     // 。。。
}
```

ACTION_MOVE小节参考：https://blog.csdn.net/cufelsd/article/details/89471402

## 事件分发常见面试题


### 1.ViewGroup中的mFirstTouchTarget是一个什么东西，它有什么作用？

在ViewGroup中有一个类型为TouchTrarget的mFirstTouchTarget的成员变量，它是用来保存消费事件的子View的信息的。代码如下：

```java
private static final class TouchTarget {
        @UnsupportedAppUsage
        public View child;
        public TouchTarget next;
       // ...省略无关代码
    }
```

可以看到TouchTarget内部保存了一个View和一个类型为TouchTarget的next成员变量，也就是说TouchTarget是一个链表结构。为什么是链表结构呢？主要是因为Android系统是支持多点触控的，所以TouchTarget设计成了链表。

设计mFirstTouchTarget的目的是为了避免在所有的事件序列中都去递归查找要消费事件的View，只需要在ACTION_DOWN中递归查找消费的View，并将View封装后赋值为mFirstTouchTarget，避免了后续一系列事件的查找。

mFirstTouchTarget会在ACTION_DOWN的时候被赋值，查找是否有能够消费事件的子View，如果有则将这个View包装成TouchTarget赋值给mFirstTouchTarget，否则mFirstTouchTarget为null。

接下来的一系列ACTION_MOVE事件都会根据mFirstTouchTarget是否为null和onInterceptTouchEvent来判断是否要拦截事件。所以mFirstTouchTarget在事件分发的流程中占了非常重要的作用。

### 2.如果在ViewGroup中拦截了ACTION_DOWN事件会怎样？

首先来看ViewGroup的dispatchTouchEvent方法的伪代码：

```java
// ViewGroup
public boolean dispatchTouchEvent(MotionEvent ev) {
        boolean handled = false;
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
             // 调用自身的onInterceptTouchEvent方法来判断是否拦截事件
            intercepted = onInterceptTouchEvent(ev);
        } else {
            intercepted = true;
        }
        // 如果在ACTION_DOWN中拦截了事件，那么intercepted恒为true，则无法给mFirstTouchTarget赋值
        if (!canceled && !intercepted) {
            mFirstTouchTarget = findConsumeChild(this);
        }
        if (mFirstTouchTarget == null) {  
        	// 则调用自身的dispatchTouchEvent方法分发事件             
            handled = super.dispatchTouchEvent(event);
        } else{
        	// 则调用子View的dispatchTouchEvent方法
			handled = mFirstTouchTarget.child.dispatchTouchEvent(event);
		}
        return handled;
    }
```

从上面的代码中可以看到，如果在onInterceptTouchEvent方法中拦截ACTION_DOWN事件，则intercepted为true，而intercepted为true直接导致了mFirstTouchTarget无法被赋值。

接下来，一系列ACTION_MOVE以及ACTION_UP等事件都无法再调用onInterceptTouchEvent方法，也就是intercepted恒为true，且mFirstTouchTarget恒为null。

再往下，由于mFirstTouchTarget恒为null，就导致了所有的Motion事件（也包括ACTION_DOWN事件）只能交由自身处理，无法再将事件分发给子View。

这一点在事件分发流程中非常重要，通过Down事件确定了要处理该事件的View，接下来所有的Move事件序列就不会再走递归，而是直接交给这个View来处理。



### 3.为什么设置了onTouchListener后onClickListener不会被调用？

```java
// View
public boolean dispatchTouchEvent(MotionEvent event) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        return result;
    }
```

在View的dispatchTouchEvent中如果li.mOnTouchListener不为null，则调用li.mOnTouchListener.onTouch，而如果li.mOnTouchListener.onTouch返回了true，则下边的onTouchEvent就不会被调用，而onClickListener就是在onTouchEvent方法中才被调用的。

```java
// 伪代码
public boolean onTouchEvent(MotionEvent event) {
	public boolean onTouchEvent(MotionEvent event) {
		case MotionEvent.ACTION_UP:
			li.mOnClickListener.onClick(this);
		break;
	}
}
```

### 4.为什么一个View设置了setOnTouchListener会有提示没有引用performClick方法的警告？

当你添加了一些点击操作，例如像setOnClickListener这样的，它会调用performClick才可以完成onClick方法的调用，但你重写了onTouch，就有可能使得performClick没有被调用，这样这些点击操作就没办法完成了，所以就会有了这个警告。