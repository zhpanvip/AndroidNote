### 1.面临的问题

由于TM平台对于APP性能做出了要求，只有性能符合他们的标准才能在该平台上线。主要要求有一下几个方面：

- APP包体积不能超过30M
- CPU的最高使用率不能超过70%
- 运行时的内存占用不超过200M

  上线前项目情况：

- 包体积大小200+M；
- 在AI录播课页面存在CPU使用率超过70%的情况；
- APP运行占用内存有优化的空间
- 启动速度太慢，APP在debug模式下的启动时间在8s(机器性能差)左右，有大幅的优化空间。

### 2.启动优化

#### （1）检测启动时间

##### 查看APP启动时间

使用adb命令查看启动时间：

> adb shell am start -S -W [packageName]/[ packageName. AppstartActivity]

```
Stopping: com.example.app
Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.example.app/.MainActivity }
Status: ok
LaunchState:COLD
Activity: com.example.app/.MainActivity
ThisTime: 1059
TotalTime: 1059
WaitTime: 1073
Complete
```

LaunchState 为 COLD，代表的是冷启动。ThisTime为最后一个Activity启动耗时， TotalTime 所有Activity启动耗时，WaitTime为AMS启动Activity的总耗时。一般查看得到的TotalTime，即应用的启动时间，包括创建进程 + Application初始化 + Activity初始化到界面显示的过程。

##### 查看每个方法执行时间

- 使用startMethodTracing

```java
// 开启方法追踪
Debug.startMethodTracing(new File(getExternalFilesDir(""),"trace").getAbsolutePath(),8*1024*1024,1_000);
// 停止方法追踪
Debug.stopMethodTracing()
  
```

通过上述方法会在data/data/package下边生成trace文件，记录每个方法的时间，CPU信息。但是对运行时性能有较大影响。

- 使用startMethodTracingSampling

```java
// 开启方法采样追踪
Debug.startMethodTracingSampling(new File(getExternalFilesDir(""),"trace").getAbsolutePath(),8*1024*1024,1_000);
// 停止方法追踪
Debug.stopMethodTracing();
```

相比于Trace Java Methods会记录每个方法的时间、CPU信息，它会在应用的Java代码执行期间频繁捕获应用的调用堆栈，对运行时性能的影响比较小，能够记录更大的数据区域。



#### （2）启动优化方案

##### 子线程启动

将不紧急的第三方库或者一些初始化工作放入到子线程中启动。例如分享库、录音SDK、推送SDK等第三方库的初始化工作可以放到子线程中执行。另外，在启动时候执行检查APP内存等操作均可放入子线程中执行。

##### ARouter的启动优化

通过监测，发现项目中ARouter启动时耗时特别长，达到了4秒左右，故将启动优化重点放在ARouter上。

研究ARouter源码发现，由于ARouter会在编译期间通过APT生成路由表相关类，在APP启动时候回扫描APP中的所有类，然后找到编译时生成的路由表去做一些初始化的工作。启动耗时主要浪费在了扫描APP下的所有类。

通过研究ARouter的源码，发现在ARouter扫描类前有一个hook点，可以通过字节码插装的方式禁止ARouter全盘扫描类文件，然后自行指定要加载的类。这一操作可以通过一个第三方库来实现-- [AutoRegister](https://github.com/luckybilly/AutoRegister)。



### 3.包体积优化

(包体积优化)[https://github.com/zhpanvip/AndroidNote/wiki/%E5%8C%85%E4%BD%93%E7%A7%AF%E4%BC%98%E5%8C%96]

### 4.CPU优化

可以通过Profiler或者shell命令来监控APP运行时CPU的占用率。

#### CPU优化方向

##### （1）UI绘制优化

复杂的UI绘制会占用一定的内存，因此是一个优化方向。通过Profiler工具Trace System Call记录页面加载和绘制过程中各个方法的耗时，通过对方高耗时的方法进行分析，排查问题所在，比如是不是布局界面过于复杂，是否有可以懒加载的View。当然，通过这种方法也可以查到页面是不是在主线程中执行了耗时操作。根据自己的情况优化即可。

##### （2）AI互动课页面优化

该页面是有视频播放器+WebView构成。占用CPU主要是播放视频与WebView的互动，因此客户端能做的只是对于视频播放的优化，webview需要前端人员优化处理。降低视频分辨率，代码上可以降低视频码率、帧率、采样率等优化手段。

