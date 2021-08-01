## Android系统启动概述

Android 系统启动的过程，大致有以下几部分：


1. 启动电源以及系统启动

当电源按下时引导芯片代码从预定义的地方开始执行。加载引导程序 BootLoader 到 RAM，然后执行。

2. 引导程序 BootLoader

引导程序 BootLoader 是在 Android 操作系统开始运行前的一个小程序，它的主要作用是把系统 OS 拉起来运行。

3. Linux 内核启动

当内核启动时，设置缓存、存储器、计划列表、加载驱动。当内核完成系统设置时，它首先在系统文件中寻找 init.rc 文件，并启动 init 进程。

4. init 进程启动

初始化和启动属性服务，并且启动 Zygote 进程。

5. Zygote 进程启动

创建 Java 虚拟机并为 Java 虚拟机注册 JNI 方法，创建服务端 Socket，启动 SystemServer 进程。

6. SystemServer 进程启动

启动 Binder 线程池和 SystemServiceManager，并且启动各种系统服务。

7. Launcher 启动

被 SystemServer 进程启动的 AMS 会启动 Launcher，Launcher 启动后会将已安装的应用的快捷图标显示在界面上。

![](https://camo.githubusercontent.com/e30dad1128b7dad0e64f04ce844780bf08a6d6c78f6542857415d6efaa70bd4c/68747470733a2f2f692e6c6f6c692e6e65742f323031392f30342f32332f356362653561366362623131342e706e67)

## init进程启动
init 进程是 Android 系统中用户空间的第一个进程，进程号为 1。当我们按下电源时，系统启动后会加载引导程序，引导程序又启动 Linux 内核，在 Linux 内核加载完成后，第一件事就是要启动 init 进程。

init 进程主要做了以下三件事：

1. 创建和挂载启动所需的文件目录
2. 初始化和启动属性服务
3. 解析 init.rc 配置文件并启动 Zygote 进程

## Zygote 进程启动过程

在 Android 系统中，DVM 和 ART、应用程序进程以及运行系统的关键服务的 SystemServer 进程都是由 Zygote 进程来创建的，我们也将它称为孵化器。

Zygote 进程都是通过 fork 自身来创建子进程的。再通过 JNI 调用 ZygoteInit 的 main 方法后，Zygote 便进入了 Java 框架层，此前是没有任何代码进入 Java 框架层的，也就是说，Zygote 开创了 Java 框架层。

```java
    public static void main(String argv[]) {
        //...
        try {
						//...
          	//1. 创建一个 Server 端的 Socket，socketName 为 zygote
            zygoteServer.registerServerSocketFromEnv(socketName);
            
            if (!enableLazyPreload) {
                bootTimingsTraceLog.traceBegin("ZygotePreload");
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                    SystemClock.uptimeMillis());
              	//2. 预加载资源和类
                preload(bootTimingsTraceLog);
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                    SystemClock.uptimeMillis());
                bootTimingsTraceLog.traceEnd(); // ZygotePreload
            } else {
                Zygote.resetNicePriority();
            }
						//...
            if (startSystemServer) {
              	//3. 启动 SystemServer 进程
                Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
						//4. 等待 AMS 请求
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with exception", ex);
            throw ex;
        } finally {
            zygoteServer.closeServerSocket();
        }
    }
```
Zygote 进程主要做了以下几件事：

1. 创建 AppRuntime 并调用其 start 方法，启动 Zygote 进程
2. 创建 Java 虚拟机并为 Java 虚拟机注册 JNI 方法
3. 通过 JNI 调用 ZygoteInit 的 main 方法进入 Zygote 的 Java 框架层
4. 通过 registerZygoteSocket 方法创建服务端 Socket，并通过 runSelectLoop 方法等待 AMS 的请求来创建新的应用程序进程
5. 启动 SystemServer 进程

## SystemServer 

SystemServer 进程主要用于创建系统服务，比如 IMS、AMS、WMS、PMS 等都是由它来创建的。

SystemServer 进程被创建后，主要做了以下工作：

1. 启动 Binder 线程池，这样就可以与其他进程进行通信
2. 启动 SystemServiceManager，其用于对系统的服务进程创建、启动和生命周期管理
3. 启动各种系统服务

