## 前言
大多数用户感知到的卡顿等性能问题的最主要根源都是因为渲染性能。Android系统每隔大概16.6ms发出VSYNC信号，触发对UI进行渲染，如果每次渲染都成功，这样就能够达到流畅的画面所需要的60fps，为了能够实现60fps，这意味着程序的大多数操作都必须在16ms内完成。

## 1.Android界面滑动卡顿主要有两个原因

- （1）UI线程（main）有耗时操作

- （2）视图渲染时间过长，导致卡顿

Android系统每隔16ms就会发送一个VSYNC信号（VSYNC：vertical synchronization 垂直同步，帧同步），触发对UI进行渲染，如果每次渲染都成功，这样就能够达到流畅的画面所需要的正常帧率：60fps。一旦这时候系统正在做大于16ms的耗时操作，系统就会无法响应VSYNC信号，执行渲染工作，导致发生丢帧现象。

关于Android屏幕刷新机制可以参考[Android屏幕刷新机制](https://github.com/zhpanvip/AndroidNote/wiki/%E5%B1%8F%E5%B9%95%E5%88%B7%E6%96%B0%E6%9C%BA%E5%88%B6)

用户容易在UI执行动画、ListView、RecyclerView滑动的时候感知到界面的卡顿与不流畅现象。所以开发者一定要注意在设计布局时不要嵌套太多层，多使用 include方法引入布局。同时不要让动画执行次数太多，导致CPU或者GPU负载过重。

## 2.UI优化的三个方案

### （1）布局优化
- 减少View的嵌套层级
   - LayoutInflater这个类会通过Pull解析器取解析xml文件，然后通过反射的方式来生成对应的View。而解析xml与反射都是一个相对消耗性能的操作。因此，在复杂的页面中，应该尽可能的减少View的嵌套层级，以此较少解析xml的性能消耗。
   - 使用merge减少一层嵌套
- 使用new来实例化View
对于简单的布局，比如只有一个View的情况，可以考虑直接使用new来实例化View，以此较少解析xml和反射带来的性能损耗

### （2）异步加载布局

对于布局特别复杂的页面，可以采用异步加载布局的方式来优化提升加载效率。Android为我们提供了 Asynclayoutinflater 把耗时的加载操作在异步线程中完成，最后把加载结果再回调给主线程。

```
dependencies { implementation "androidx.asynclayoutinflater:asynclayoutinflater:1.0.0" }
```

```
new AsyncLayoutInflater(this)
     .inflate(R.layout.activity_main, null, new AsyncLayoutInflater.OnInflateFinishedListener() { 
        @Override 
        public void onInflateFinished(@NonNull View view, int resid, @Nullable ViewGroup parent) { 
            setContentView(view); 
            //...... 
       } 
});
```
**需要注意：**

- 1、使用异步 inflate，那么需要这个 layout 的 parent 的 generateLayoutParams 函数是线程安全的；
- 2、所有构建的 View 中必须不能创建 Handler 或者是调用 Looper.myLooper；（因为是在异步线程中加载的，异步线程默认没有调用 Looper.prepare ）；
- 3、AsyncLayoutInflater 不支持设置 LayoutInflater.Factory 或者 LayoutInflater.Factory2； 
- 4、不支持加载包含 Fragment 的 layout；
- 5、如果 AsyncLayoutInflater 失败，那么会自动回退到UI线程来加载布局。

### （3）避免过度绘制

过渡绘制是指屏幕上某个像素在同一帧的时间内绘制了多次。在多层次的UI结构里面，如果不可见的UI也在做绘制操作，这就会导致某些像素区域被绘制了多次，这就是很大程度上浪费了CPU和GPU资源。最最常见的过度绘制，就是设置了无用的背景颜色！！！

对于Overdraw这个问题还是很容易发现的，我们可以通过以下步骤打开显示GPU过度绘制（Show GPU Overrdraw）选项

> 设置 -> 开发者选项 -> 调试GPU过度绘制 -> 显示GPU过度绘制

打开以后之后，你会发现屏幕上有各种颜色，此时你可以切换到需要检测的程序与界面，对于各个色块的含义，请看下图：
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzcyNzc5MC0yNzg4ZjdlOGY4MWQ4ZmI2LnBuZw?x-oss-process=image/format,png)

蓝色，淡绿，淡红，深红代表了4种不同程度的Overdraw情况：

- **蓝色**： 意味着overdraw 1倍。像素绘制了两次。大片的蓝色还是可以接受的（若整个窗口是蓝色的，可以摆脱一层）。 
- **绿色**： 意味着overdraw 2倍。像素绘制了三次。中等大小的绿色区域是可以接受的但你应该尝试优化、减少它们。 
- **淡红**： 意味着overdraw 3倍。像素绘制了四次，小范围可以接受。 
- **深红**： 意味着overdraw 4倍。像素绘制了五次或者更多。这是错误的，要修复它们。 

我们的目标就是尽量减少红色Overdraw，看到更多的蓝色区域。

#### OverDraw的解决方案

- 通过Show GPU Overdraw去检测Overdraw，最终可以通过移除不必要的背景。
- 通过Layout Inspector去检测渲染效率，去除不必要的嵌套；

## 3.卡顿分析常用工具

### （1）Systrace
Systrace 是Android平台提供的一款工具，用于记录短期内的设备活动。该工具会生成一份报告，其中汇总了Android 内核中的数据，例如 CPU 调度程序、磁盘活动和应用线程。Systrace主要用来分析绘制性能方面的问题。在发生卡顿时，通过这份报告可以知道当前整个系统所处的状态，从而帮助开发者更直观的分析系统瓶颈，改进性能。

在抓取systrace文件的时候，切记不要抓取太长时间，也不要太多不同操作。

![](https://gitee.com/zhpanvip/images/raw/master/project/article/performance/systrace1.png)

打开抓取的html文件，可以看到我们应用存在非常严重的掉帧，不借助工具直接用肉眼看应用UI是看不出来的。如果只是单独存在一个红色或者黄色的都是没关系的。关键是连续的红/黄色或者两帧间隔非常大那就需要我们去仔细观察。按“W” 放大视图，在UIThread（主线程）上面有一条很细的线，表示线程状态。

![](https://gitee.com/zhpanvip/images/raw/master/project/article/performance/systrace2.png)

Systrace 会用不同的颜色来标识不同的线程状态, 在每个方法上面都会有对应的线程状态来标识目前线程所处的状态。通过查看线程状态我们可以知道目前的瓶颈是什么, 是 CPU 执行慢还是因为 Binder 调用, 又或是进行 IO 操作,又或是拿不到 CPU 时间片。 通过查看线程状态我们可以知道目前的瓶颈是什么, 是 CPU 执行慢还是因为 Binder调用, 又或是进行 IO 操作, 又或是拿不到CPU事件片。

线程状态主要有下面几个：

- 绿色：表示正在运行；
  - 是否频率不够？（CPU处理速度）
  - 是否跑在了小核上？（不可控，但实际上很多手机都会有游戏模式，如果我们应用是手游，那系统会优先把手游中的任务放到大核上跑。）
- 蓝色：表示可以运行，但是CPU在执行其他线程；
  - 是否后台有太多的任务在跑？Runnable 状态的线程状态持续时间越长，则表示 cpu 的调度越忙，没有及时处理到这个任务
  - 没有及时处理是因为频率太低？

紫色：表示休眠，一般表示IO；

![](https://gitee.com/zhpanvip/images/raw/master/project/article/performance/systrace3.png)

- 橙色：不可中断的休眠
  - 线程在遇到 I/O 操作时被阻止或正在等待磁盘操作完成。

- 紫色：可中断的休眠
  - 线程在遇到另一项内核操作（通常是内存管理）时被阻止。
  - 但是实际从Android 9模拟器中拉取数据，遇到IO显示紫色，没有橙色状态显示。
- 白色：表示休眠，可能是因为线程在互斥锁上被阻塞 ，如Binder堵塞/Sleep/Wait等

### (2)Trace API
其实对于APP开发而言，使用systrace的帮助并不算非常大，大部分内容用于设备真机优化之类的系统开发人员观察。systrace无法帮助应用开发者定位到准确的错误代码位置，我们需要凭借很多零碎的知识点与经验来猜测问题原因。如果我们有了大概怀疑的具体的代码块或者有想了解的代码块执行时系统的状态，还可以结合 Trace API 打标签。

Android 提供了Trace API能够帮助我们记录收集自己应用中的一些信息 ： Trace.beginSection() 与 Trace.endSection();

```
public class MainActivity extends AppCompatActivity { 
    @Override protected void onCreate(Bundle savedInstanceState) { 
        super.onCreate(savedInstanceState); 
        TraceCompat.beginSection("enjoy_launcher"); //Trace.beginSection() 
        setContentView(R.layout.activity_main); 
        TraceCompat.endSection(); //Trace.endSection() 
} }
```

### BlockCanary
Android主线程更新UI。如果界面1秒钟刷新少于60次，即FPS小于60，用户就会产生卡顿感觉。简单来说，
Android使用消息机制进行UI更新，UI线程有个Looper，在其loop方法中会不断取出message，调用其绑定的
Handler在UI线程执行。如果在handler的dispatchMesaage方法里有耗时操作，就会发生卡顿。

只要检测 msg.target.dispatchMessage(msg) 的执行时间，就能检测到部分UI线程是否有耗时的操作。注意到这行
执行代码的前后，有两个logging.println函数，如果设置了logging，会分别打印出>>>>> Dispatching to和
<<<<< Finished to 这样的日志，这样我们就可以通过两次log的时间差值，来计算dispatchMessage的执行时
间，从而设置阈值判断是否发生了卡顿。

Looper 提供了 setMessageLogging(@Nullable Printer printer) 方法，所以我们可以自己实现一个Printer，在
通过setMessageLogging()方法传入即可

这种方式也就是 BlockCanary 原理。
