ANR(Application Not responding)，是指应用程序未响应，Android系统对于一些事件需要在一定的时间范围内完成，如果超过预定时间能未能得到有效响应或者响应时间过长，都会造成ANR

## 出现场景
- Service Timeout
- BroadcastQueue Timeout
- ContentProvider Timeout
- InputDispatching Timeout

## Timeout时长
- 对于前台服务，则超时为SERVICE_TIMEOUT = 20s；
- 对于后台服务，则超时为SERVICE_BACKGROUND_TIMEOUT = 200s
- 对于前台广播，则超时为BROADCAST_FG_TIMEOUT = 10s；
- 对于后台广播，则超时为BROADCAST_BG_TIMEOUT = 60s;
- ContentProvider超时为CONTENT_PROVIDER_PUBLISH_TIMEOUT = 10s;
- InputDispatching Timeout: 输入事件分发超时5s，包括按键和触摸事件。
- 主线程被IO操作（从4.0之后网络IO不允许在主线程中）阻塞。
- 主线程中存在耗时的计算
- 主线程中错误的操作，比如Thread.wait或者Thread.sleep等
Android系统会监控程序的响应状况，一旦出现下面两种情况，则弹出ANR对话框
- 应用在5秒内未响应用户的输入事件（如按键或者触摸）
- BroadcastReceiver未在10秒内完成相关的处理

## 2）如何避免

避免ANR的基本思路就是将IO操作在工作线程来处理，减少其他耗时操作和错误操作

- 使用Thread或者HandlerThread时，调用Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)设置优先级，否则仍然会降低程序响应，因为默认Thread的优先级和主线程相同。
- 使用Handler处理工作线程结果，而不是使用Thread.wait()或者Thread.sleep()来阻塞主线程。
- Activity的onCreate和onResume回调中尽量避免耗时的代码
- BroadcastReceiver中onReceive代码也要尽量减少耗时，建议使用IntentService处理。
- 避免主线程跟工作线程发生锁的竞争
- 减少系统耗时binder的调用
- 谨慎使用sharePreference
- 注意主线程执行provider query操作

## 如何改善

通常100到200毫秒就会让人察觉程序反应慢，为了更加提升响应，可以使用下面的几种方法 

如果程序正在后台处理用户的输入，建议使用让用户得知进度，比如使用ProgressBar控件。

程序启动时可以选择加上欢迎界面，避免让用户察觉卡顿。

使用Systrace和TraceView找出影响响应的问题。

如果开发机器上出现问题，我们可以通过查看/data/anr/traces.txt即可，最新的ANR信息在最开始部分。

## trace文件解读

### 1. 人为的收集trace.txt的命令
adb shell kill -3 888 //可指定进程pid
执行完该命令后traces信息的结果保存到文件/data/anr/traces.txt
### 2. trace文件解读


