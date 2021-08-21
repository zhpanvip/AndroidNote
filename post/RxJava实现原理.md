了解RxJava前可以先了解[观察者模式](https://github.com/zhpanvip/AndroidNote/wiki/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F)，观察者模式一般是有多个观察者和一个被观察者组成，观察者订阅被观察者，当被观察者发生改变时，通知观察者。而RxJava是基于变形的观察者模式实现的。RxJava中的观察者模式的特殊点在于它有多个被观察者和一个观察者。

## 一、RxJava使用

```java
  Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override public void subscribe(@NotNull ObservableEmitter<Integer> emitter)
        throws Exception {
      emitter.onNext(1);
    }
  }).subscribe(new Consumer<String>() {
    @Override public void accept(String s) throws Exception {
      LogUtils.e(s);
    }
  });
}
```

上述代码中通过 Observable.create 创建了一个被观察者 ObservableCreate，然后通过 subscribe 方法实现了观察者的订阅。

ObservableCreate 继承了 Observable，Observable是一个抽象类，它的内部定义了RxJava的操作符，例如 create、map、flatMap、zip等等。另外最重要的是作为被观察者，需要有一个subscribe方法来让观察者订阅，subscribe方法可以接受一个Consumer，也可接受一个Observer.

下面以create操作符和map操作符为例进行分析。

## 二、RxJava的全局Hook点

RxJava的被观察者的操作符中都调用了RxJavaPlugins.onAssembly，以 create 和 map 为例，代码如下：

```java
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```

```java
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}
```

可以看到，create 方法中实例化了一个 ObservableCreate 作为参数然后调用 RxJavaPlugins.onAssembly，map方法与其类似，不同的是实例化了一个 ObservableMap 参数。

在 RxJavaPlugins 的 onAssembly 方法中对 onObservableAssembly 进行判空，如果不为空，则调用 apply 方法并返回，否则直接返回 Observable，

```java
// RxJavaPlugins
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    Function<? super Observable, ? extends Observable> f = onObservableAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```

通常情况下，上述代码中的 onObservableAssembly 都是空，除非我们主动地为其设置了值。也就是说正常情况下onAssembly方法中什么都没做，仅仅是直接返回了source。

如果onObservableAssembly不为空，则在apply方法中调用了onObservableAssembly 的apply方法，这个方法接受的参数就是上边的source，如下代码：

```java
static <T, R> R apply(@NonNull Function<T, R> f, @NonNull T t) {
    try {
        return f.apply(t);
    } catch (Throwable ex) {
        throw ExceptionHelper.wrapOrThrow(ex);
    }
}
```

onObservableAssembly 是RxJavaPlugins中的静态成员变量，通过 setOnObservableAssembly 方法被赋值，代码如下：

```java
public final class RxJavaPlugins {
  static volatile Function<? super Observable, ? extends Observable> onObservableAssembly;

  public static void setOnObservableAssembly(@Nullable Function<? super Observable, ? extends Observable> onObservableAssembly) {
    if (lockdown) {
        throw new IllegalStateException("Plugins can't be changed anymore");
    }
    RxJavaPlugins.onObservableAssembly = onObservableAssembly;
  }
}
```

在使用RxJava时可以调用 setOnObservableAssembly 这个方法为 onObservableAssembly 赋值，如下代码：

```java
RxJavaPlugins.setOnObservableAssembly(new Function<Observable, Observable>() {
  @Override public Observable apply(Observable observable) {
    Log.e("RxJava",observable.getClass().getSimpleName() + observable.hashCode() + "被调用了");
    return observable;
  }
});
```

调用第1节中的代码，在apply方法中打印了一下日志，然后返回observable，这里的observable就是上文的source。打印结果如下：

```
2021-08-21 17:51:20.811 13698-13698/com.zhangpan.rxjavatest E/RxJava: ObservableCreate189516865被调用了
2021-08-21 17:51:20.812 13698-13698/com.zhangpan.rxjavatest E/RxJava: ObservableMap46659046被调用了
```

可以看出来，这里的onObservableAssembly其实做了一个全局拦截，设置了onObservableAssembly后RxJava的所有操作都会走到这里。

## 三、RxJava中的观察者模式



### 1.创建被观察者

可以通过Observable.create来创建一个Observable，create方法接受一个ObservableOnSubscribe<T>的参数类型。创建Observable的代码如下：

```java
ObservableCreate observableCreate = Observable.create(new ObservableOnSubscribe<Integer>() {
  @Override public void subscribe(ObservableEmitter<Integer> emitter)
      throws Exception {
    emitter.onNext(1);
  }
})
```

上一章中已经分析过了，这个过程仅仅是实例化了一个ObservableCreate，ObservableCreate构造方法接受一个ObservableOnSubscribe的参数，ObservableCreate类的结构如下：

```java
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

    // 需要搞清楚这个方法是在哪里调用的
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        // 创建一个CreateEmitter
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        // 观察者的onSubscribe方法首先被调用
        observer.onSubscribe(parent);

        try {
            // 调用subscribe，并将CreateEmitter作为参数
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
}
```

可以看到，通过构造方法将 source 赋值给了 ObservableCreate 的成员变量。接着，在 subscribeActual 方法中先调用了 source 的onSubscribe方法，接着调用了 subscribe 方法。subscribe 方法被调用意味着 ` emitter.onNext(1);`这句代码会被执行，即执行CreateEmitter的onNext方法。CreateEmitter 的onNext代码如下：

```java
@Override
public void onNext(T t) {
    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    if (!isDisposed()) {
        // 执行观察者的onNext方法
        observer.onNext(t);
    }
}
```

到这里，执行流程其实已经很清楚了，但是重点是 subscribeActual 方法是在哪里调用的？

### 2.订阅观察者

上一小节中通过Observable的create方法创建了被观察者ObservableCreate，有了被观察者就需要有观察者来订阅监听，所以，可以通过ObservableCreate发起订阅，如下代码：

```java
observableCreate.subscribe(new Observer<Integer>() {
  @Override public void onSubscribe(@org.jetbrains.annotations.NotNull Disposable d) {

  }

  @Override public void onNext(@org.jetbrains.annotations.NotNull Integer integer) {

  }

  @Override public void onError(@org.jetbrains.annotations.NotNull Throwable e) {

  }

  @Override public void onComplete() {

  }
});
```

observableCreate.subscribe方法中接收了一个Observer，者便是观察者。接下来需要看下subscribe方法的实现，这个方法位于Observable中，源码如下：

```java
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
				// ...

				// 调用 subscribeActual
        subscribeActual(observer);

    }
    // ... 省略catch相关代码
}
```

subscribe 方法中，最重要的一行代码是调用了 subscribeActual 方法，还记得这个方法把？上一节中已经见过了。在ObservableCreate的subscribeActual方法中首先调用了observer的onSubscribe，接着又调用了source.subscribe(parent)，从而emitter.onNext(1);这行代码被调用了，emitter.onNext方法中又会去调用观察者observer的onNext方法。

至此，走完了整个Rxjava的发布订阅流程。



## 三、RxJava的链式调用

第二章的内容，其实就是一个标准的观察者模式。但是RxJava的功能远不止这么简单。链式调用Observable的操作符才是Rxjava的精髓。如何实现的？



Observable的create会创建一个ObservableCreate的被观察者，它继承了Observable。而RxJava中的所有操作符都被定义在Observable中。例如map、flatMap、zip等。因此ObservableCreate拥有Observable的所有操作符。即可以通过ObservableCreate继续调用map、flatMap等操作符。

```java
ObservableCreate observableCreate = Observable.create(new ObservableOnSubscribe<Integer>() {
  @Override public void subscribe(ObservableEmitter<Integer> emitter)
      throws Exception {
    emitter.onNext(1);
  }
})
// 继续调用map操作符
ObservableMap observableMap = observableCreate.map(new Function<Integer, String>() {
  @Override public String apply(Integer integer) throws Exception {
    return "Convert to String :" + integer;
  }
})
```

而map操作符返回的是一个ObservableMap的被观察者，它同样是继承自Observable，因此ObservableMap也同样拥有Observable中的所有操作符。即可以继续调用map、zip、flatMap等操作，当然也可以发起订阅，如下：

```java
observableMap.subscribe(new Consumer<String>() {
  @Override public void accept(String s) throws Exception {
    Log.e("RxJava", s);
  }
});
```

上一章已经分析了ObservableCreate的订阅代码，ObservableMap与其如出一辙，这里不再赘述。







