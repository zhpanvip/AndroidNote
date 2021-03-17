## ThreadLoacal是什么？

ThreadLocal是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，且存储的数据只有在该线程中才能获取的到，其他线程无法获取。

### 1.ThreadLocal的基本使用：
```
    public static void main(String[] args) throws InterruptedException {

        ThreadLocal<Boolean> threadLocal=new ThreadLocal<>();

        new Thread(){
            @Override
            public void run() {
                super.run();
                threadLocal.set(true);
                System.out.println("在子线程中获得threadLocal中的值："+threadLocal.get());
            }
        }.start();
        Thread.sleep(100);

        System.out.println("在主线程中获得threadLocal中的值："+threadLocal.get());

    }
```
输出结果：

```
 在子线程中获得threadLocal中的值：true
 在主线程中获得threadLocal中的值：null
```

## ThreadLocal的实现原理

ThreadLocal是一个泛型类。它的重点是有一个ThreadLocalMap的静态内部类和两个核心方法，get和set。

### 1.ThreadLocalMap
ThreadLocalMap是一个存储(key,value)键值对的类，它的实现与HashMap有些类似，这里不再详细赘述。它比较特别的地方是它内部的节点Entry：

```

static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```
Entry是位于ThreadLocalMap中的静态内部类，表示Map的节点(key，value),并且它继承了WeakReference,WeakReference泛型为ThreadLocal。看Entry的构造方法有两个参数ThreadLocal与一个Object，分别表示key和value。这里注意，在构造方法中将ThreadLocal作为参数调用了父类的构造方法，也就是ThreadLocalMap的key是一个弱引用，而value是一个强引用。意味着，当没有引用指向key时，key会在下一次垃圾回收时被JVM回收。

那ThreadLocalMap在哪里使用了呢？其实就在线程Thread类中：
```
class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```
可以看到Thread类中维护了一个ThreadLocalMap的成员变量。


### 2.ThreadLocal的set方法分析

set方法的源码如下：
```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}
// 获取当前线程的ThreadLocalMap
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

// 为线程t创建ThreadLocalMap
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
set方法的源码很简单，就是以当前的ThreadLocal作为key将value放到ThreadLocalMap中，ThreadLocalMap是位于ThreadLocal中的一个静态内部类。


### 3.为什么要将ThreadLocalMap的key声明成弱引用呢？

这么做其实是为了有利于垃圾回收。想一下如果没有将ThreadLocal的key声明为弱引用会有什么问题？当使用完了ThreaLocal之后，解除了对它的引用，希望它能够被回收，但是此时JVM发现ThreadLocal无法被回收，因为在ThreadLocalMap中，ThreadLocal还作为key被引用，虽然ThreadLocal已经不再使用，但是仍然无法被回收。而如果将ThreadLocalMap的key声明为弱引用就不会存在无法回收ThreadLocal的问题。


### 4.ThreadLocal的get方法分析

get方法就是从ThreadLocal中取值，它的源码如下：

```
 public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
可以看到get方法的源码也不难，就是拿到当前线程中的ThreadLocalMap,然后将当前ThreadLocal作为key查找set进去的值。

了解了ThreadLocal的内部实现后，我们通过开篇的例题来分析以下ThreadLoca的存储过程：
- （1）threadLoca在子线程(称为threadA)中通过threadLocal的set方法将值设置为了true。在set方法中会首先获得threadA线程，然后拿到threadA中的ThreadLocalMap,并将threadLocal作为key，将true作为value存储到了ThreadLocalMap中；

- （2）当在threadA线程中调用threadLocal的get方法时，get方法同样会先获取到threadA，然后拿到threadA中的ThreadLocalMap，接着以threadLocal作为key来获取ThreadLocalMap中对应的值，此时的ThreadLocalMap与第（1）不中的ThreadLocalMap是同一个，且key都是threadLocal。因此，在threadLocal中拿到的值为true；

- （3）在主线程中调用threadLocal的get操作，此时get方法会首先获取到当前线程为主线程，然后拿到了主线程中的ThreadLocalMap，接着以threadLocal作为key来获取主线程的ThreadLocalMap中对应的值，此时的ThreadLocalMap与(1)步中的ThreadLocalMap并非同一个。因此，主线程中获取到的值是null。

ThreadLocal的示意图如下：
![](https://gitee.com/zhpanvip/images/raw/master/project/article/thread/threadlocal.png)
