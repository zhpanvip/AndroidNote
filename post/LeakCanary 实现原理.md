
## 1. ReferenceQueue

ReferenceQueue是一个链表结构的存储队列，节点是reference，节点之间通过next连接。ReferenceQueue的意义在于能够在外部对queue中的引用进行监控。当引用中的对象没回收后可以对引用对象本身继续做一些清理操作。使用ReferenceQueue结合弱引用（或者软引用）可以监测弱引用中的对象是否被回收。如果弱引用中的对象被回收，那么弱引用会被加入到这个关联的ReferenceQueue中。



```Java
// 引用队列
ReferenceQueue<Object> queue = new ReferenceQueue<>();
Object obj = new Object()
// 将queue与WeakReference关联
WeakReference weakRef = new WeakReference<Object>(obj,queue);
obj = null;
// 调用GC进行垃圾回收
System.gc();
// 从队列中取出 reference
Reference<?> reference = queue.remove();
if (reference != null){
    System.out.println("对象已被回收: "+ reference.get());  // 对象为null
}
```

上述代码中将ReferenceQueue与WeakReference关联，如果WeakReference中的obj对象被回收了，那么WeakReference就会被加入到这个ReferenceQueue中。

因此，可以使用该方法来检测对象是否被回收。



## 2.LeakCanary原理

LeakCanary的核心原理即使用ReferenceQueue对Activity进行监测。当Activity执行完onDestory后，就将Activity放入到WeakReference中。然后将这个WeakReference类型的Activity与ReferenceQueque关联，此时查看ReferenceQueque中是否有这个WeakReference对象。如果没有，则执行GC，再次查看，如果还没有则证明这个Activity发生了内存泄漏。然后用HaHa这个开源库去分析dump之后的heap内存。




