## View事件分发
### 1.事件分发机制流程
Android系统中的输入事件为InputEvent,InputEvent又分为KeyEvent和MotionEvent。KeyEvent对应着键盘输入事件，而MotionEvent则对应着屏幕触摸事件。这两种事件都由InputManager统一分发。
在系统启动时，SystemServer会启动窗口管理服务WindowManagerService,在WindowManagerService中会起动输入管理器InputManager来负责监控键盘消息。
InputManager负责从硬件接受输入事件，并将事件分发给当前激活的窗口处理，而InputManager与Window之间的通信是通过ViewRootImpl类实现的。
ActivityThread负责启动Activity，在performaLaunchActivity流程中，ActivityThread会为Activity创建PhoneWindow和DocerView，然后在handleResumeActivity()中会将PhoneWindow和InputManagerService建立起链接，保证UI可见时能够对输入事件进行正确的分发。

当InputManager监控到硬件层的输入事件时，会通知ViewRootImpl对事件进行底层分发。

UI层级的事件分发 作为 完整事件分发流程的一部分，发生在ViewPostImeInputStage.processPointerEvent

```java
// ViewRootImpl
final class ViewPostImeInputStage extends InputStage {

  private int processPointerEvent(QueuedInputEvent q) {
    // 让顶层的View开始事件分发
    final MotionEvent event = (MotionEvent)q.mEvent;
    boolean handled = mView.dispatchPointerEvent(event);
    //...
  }
}
```
这个顶层的View其实就是DecorView，DecorView实际上就是Activity中Window的根布局，它是一个FrameLayout。
DocerView在接受到事件后首先分发给了Activity，Activity又将事件分发给了Window，最终Window又将事件交回给DocerView，在接下来就是UI层的事件分发了。

**（1)事件分发的流程**

对于VieGroup的事件分发来说其本质就是递归，将本身视为上游的ViewGroup需要自定义dispatchTouchEvent()，并调用child的dispatchTouchEvent()将事件分发给下游的子View。同时，在归流程中，本身视为一个View，需要调用View自身的方法（super.dispatchTouchEvent()）来决定是否消费该事件,并将结果返回上游，直至回归到View树的根节点。

对于事件分发整体流程，我们可以进行如下定义：
- 1.ViewGroup将事件分发给子View，当子View接受到事件后，若其有child，则通过dispatchTouchEvent()继续讲事件分发给child。
- 2.子View接受到事件后，通过自身的dispatchTouchEvent方法来判断事件是否被消费掉了。
  - 2.1 若该View消费事件，则响上层ViewGroup返回true,上层ViewGroup接收到true后则认为事件已被消费，继续向其上册ViewGroup返回true。
  - 2.2 若该View为消费事件，则向上层ViewGroup返回false，上层ViewGroup接收到false后调用自身的dispatchTouchEvent来决定是否消费事件，并最终将是否消费返回到上层View。

其伪代码如下：

```java
// ViewGroup.dispatchTouchEvent
public boolean dispatchTouchEvent(MotionEvent event) {
  boolean consume = false;
  // 1.将事件分发给Child
  if (hasChild) {
    consume = child.dispatchTouchEvent();
  }
  // 2.若Child不消费该事件,或者没有child，判断自身是否消费该事件
  if (!consume) {
    consume = super.dispatchTouchEvent();
  }
  // 3.将结果向上层传递
  return consume;
}
```
**构建事件序列分发链**

在事件分发过程中，如果都对View进行一次递归遍历，是一个很好性能的过程，因此对其进行了优化。当接收到ACTIVITY_DOWN事件时，以为着一个完整序列的开始。通过递归遍历找到View中真正对事件进行消费的View，并将其保存。折后接收到ACTIVITY_MOVE和ACTIVITY_UP事件时则跳过递归过程，将事件直接分发给Child。

在ViewGroup中有一个mFirstTouchTarget的成员变量，用来保存消费事件的View，代码如下：

```java
public abstract class ViewGroup extends View {
    // 指向下一级事件分发的`View`
    private TouchTarget mFirstTouchTarget;

    private static final class TouchTarget {
        public View child;
        public TouchTarget next;
    }
}
```
每个ViewGroup都持有一个mFirstTouchTarget, 当接收到一个ACTION_DOWN时，通过递归遍历找到View树中真正对事件进行消费的Child，并保存在mFirstTouchTarget属性中，依此类推组成一个完整的分发链。
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/bbed19c5fa877a9b5617d6bcc150c530.png#pic_center)
在这之后，当接收到同一事件序列的其它事件如ACTION_MOVE、ACTION_UP时，则会跳过递归流程，将事件直接分发给 分发链 下一级的Child：

```java
// ViewGroup.dispatchTouchEvent
public boolean dispatchTouchEvent(MotionEvent event) {
  boolean consume = false;
  // ...
  if (event.isActionDown()) {
    // 1.第一次接收到Down事件，递归寻找分发链的下一级，即消费该事件的View
    // 这里可以看到，递归深度搜索的算法只执行了一次
    mFirstTouchTarget = findConsumeChild(this);
  }

  // ...
  if (mFirstTouchTarget == null) {
    // 2.分发链下一级为空，说明没有子`View`消费该事件
    consume = super.dispatchTouchEvent(event);
  } else {
    // 3.mFirstTouchTarget不为空，必然有消费该事件的`View`，直接将事件分发给下一级
    consume = mFirstTouchTarget.child.dispatchTouchEvent(event);
  }
  // ...
  return consume;
}
```
**事件拦截机制**

ViewGroup中通过onInterceptTouchEvent来决定是否拦截该事件，

```java
// 伪代码实现
// ViewGroup.dispatchTouchEvent
public boolean dispatchTouchEvent(MotionEvent event) {
  // 1.若需要对事件进行拦截，直接中止事件向下分发，让自身决定是否消费事件，并将结果返回
  if (onInterceptTouchEvent(event)) {
    return super.dispatchInputEvent(event);
  }
  // ...
  // 2.若不拦截当前事件，开始事件分发流程
}
```
为了避免额外的开销，设计者根据 事件序列 为 事件拦截机制 做出了额外的优化处理，保证了 事件拦截的判断在一个事件序列中只处理一次，伪代码简单实现如下：

```java
// ViewGroup.dispatchTouchEvent
public boolean dispatchTouchEvent(MotionEvent event) {
  if (mFirstTouchTarget != null) {
    // 1.若需要对事件进行拦截，直接中止事件向下分发，让自身决定是否消费事件，并将结果返回
    if (onInterceptTouchEvent(event)) {
      // 2.确定对该事件序列拦截后，因此就没有了下一级要分发的Child
      mFirstTouchTarget = null;
      // 下一个事件传递过来时，最外层的if判断就会为false，不会再重复执行onInterceptTouchEvent()了
      return super.dispatchInputEvent(event);
    }
  }

  // ...
  // 3.若不拦截当前事件，开始事件分发流程
}
```
### 2.ViewGroup中的mFirstTouchTarget是一个什么东西，它有什么作用？
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

### 3.如果在ViewGroup中拦截了ACTION_DOWN事件会怎样？

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

当然，在大部分情况下这样的代码可能并不会有问题，因为事件可能就是需要这个ViewGroup来拦截并且消费的。但是，如果此时子View需要拦截事件就无能为力了。例如，很多情况下需要通过内部拦截法来处理滑动冲突，而此时由于子View的dispatchTouchEvent方法根本就没有被调用，内部拦截法自然也无能为力。

### 4.为什么设置了onTouchListener后onClickListener不会被调用？

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
### 5.为什么一个View设置了setOnTouchListener会有提示没有引用performClick方法的警告？

当你添加了一些点击操作，例如像setOnClickListener这样的，它会调用performClick才可以完成onClick方法的调用，但你重写了onTouch，就有可能使得performClick没有被调用，这样这些点击操作就没办法完成了，所以就会有了这个警告。

参考
[反思|Android 事件分发机制的设计与实现](https://juejin.cn/post/6844903926446161927)
[反思｜Android 事件拦截机制的设计与实现](https://juejin.cn/post/6844904128397705230)
[图解 Android 事件分发机制](https://www.jianshu.com/p/e99b5e8bd67b)



