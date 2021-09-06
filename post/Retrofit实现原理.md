## Retrofit 概述

1. Retrofit的创建使用建造者模式，通过Retrofit#Builder可以配置baseUrl、ConverterFactory、CallAdapterFactory等参数，最后通过build生成Retrofit实例。
2. 通过Retrofit实例调用create方法使用动态代理生成ApiService的代理对象。
3. create方法中通过loadServiceMethod方法解析ApiService中接口的注解，并将解析后的信息保存到RequestFactory中，然后传递给HttpServiceMethod，并且，这个create方法最终返回的是一个继承了HttpServiceMethod的CallAdapted实例，然后调用CallAdapted的invoke方法。
4. CallAdapted的invoke方法中调用了adapt方法，adapt方法又调用了CallAdapter 的adapt方法来发起网络请求。
5. CallAdapter是通过Retrofit初始化时传入的CallAdapterFactory创建的，如果不传的话会创建一个默认的CallAdapterFactory，在Retrofit中维护了一个CallAdapterFactory的集合。具体使用哪一个CallAdapterFactory是由ApiService中的接口方法的返回值确定的，即在HttpServiceMethod的parseAnnotations时会获取接口方法的返回值Type，然后根据Type来匹配对应的CallAdapterFactory，并得到CallAdapter实例。
6. CallAdapter持有OkHttpCall，真正发起网络请求的是CallAdapter的实现类，比如RxJava2CallAdapter。
7. 在请求服务器数据成功后会通过ConverterFactory将response解析成APIService中接口方法定义的Response类型。
8. ConverterFactory同样也是在Retrofit初始化时可以配置的参数（例如GsonConvertFactory），Retrofit中维护了一个ConverterFactory集合。
9. 由于有多个ConverterFactory，解析式选择哪一个ConverterFactory的实现逻辑也是在HttpServiceMethod的parseAnnotations。最终根据responseType选取合适的ConvertFactory来解析数据。

## 一、Retrofit 使用

### 1.定义接口

首先，定义一个名为GitHub的接口，并在接口中添加一个API方法，如下：

```java
public interface GitHub {
  @GET("/repos/{owner}/{repo}/contributors")
  Call<List<Contributor>> contributors(@Path("owner") String owner, @Path("repo") String repo);
}
```

### 2.创建Retrofit实例

Retrofit完美的封装让我们可以通过一个简单的链式调用，就可以生成一个Retrofit实例。代码如下：

```java
Retrofit retrofit =
    new Retrofit.Builder()
        .baseUrl(API_URL)
        .addConverterFactory(GsonConverterFactory.create())
        .build();
```

### 3. 生成API接口代理

使用Retrofit实例生成GitHub接口的代理实例，代码如下：

```java
GitHub github = retrofit.create(GitHub.class);
```

### 4. 发起网络请求

使用GitHub接口的代理，调用contributors方法发起网络请求，代码如下：

```java
github.contributors("square", "retrofit").enqueue(new Callback<List<Contributor>>() {
  @Override
  public void onResponse(Call<List<Contributor>> call, Response<List<Contributor>> response) {

  }

  @Override
  public void onFailure(Call<List<Contributor>> call, Throwable t) {

  }
});
```

在上述代码中，我们仅仅是定义了一个GitHub接口，Retrofit内部究竟是怎么生成 GitHub 实例的呢？其实这里就是用到了动态代理来实现的。

## 二、Retrofit源码分析



### 1. 生成代理对象

在调用Retrofit的create方法的时候，会通过动态代理的方式去生成接口的代理类，当去调用接口方法时实际上会调用到InvocationHandler的invoke方法，这里的InvocationHandler匿名内部类可以看做是实现了接口方法的类，也就是被代理类。create方法代码如下：

```java
public <T> T create(final Class<T> service) {
    validateServiceInterface(service);
    return (T)
        Proxy.newProxyInstance(
            service.getClassLoader(),
            new Class<?>[] {service},
            new InvocationHandler() {
              private final Platform platform = Platform.get();
              private final Object[] emptyArgs = new Object[0];

              @Override
              public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                  throws Throwable {
                // 判断方法是Object方法则直接调用
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                args = args != null ? args : emptyArgs;
                return platform.isDefaultMethod(method) // 是否为JDK8中接口的Default方法
                    ? platform.invokeDefaultMethod(method, service, proxy, args)
                    : loadServiceMethod(method).invoke(args); // 我们在API Service接口中声明的方法，重点只需要看这里即可。
              }
            });
  }
```

create 方法中两个重要的点：

1. 通过动态代理，使用ApiService 接口生成了一个实现该接口的代理对象。
2. 调用ApiService接口的方法，实际上会执行InvocationHandler 的invoke()方法，相当于是对原接口方法的具体实现。这个方法会返回一个Call对象或者Observable对象。
3. 处理ApiService接口方法的实现是在 loadServiceMethod 中

看下loadServiceMethod方法的实现

```java
// Retrofit
private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();

ServiceMethod<?> loadServiceMethod(Method method) {
  // 先从缓存中尝试获取 ServiceMethod
  ServiceMethod<?> result = serviceMethodCache.get(method);
  if (result != null) return result;
  synchronized (serviceMethodCache) {
    result = serviceMethodCache.get(method);
    if (result == null) {
      // 如果没有缓存，则通过ServiceMethod解析接口方法上的注解信息，并将结果封装成ServiceMethod.
      result = ServiceMethod.parseAnnotations(this, method);
      serviceMethodCache.put(method, result);
    }
  }
  return result;
}
```

在这个方法中，先从缓存中尝试获取ServiceMethod，如果没有缓存，那么通过然后通过`ServiceMethod.parseAnnotations(this, method);` 来得到ServiceMethod，ServiceMethod是一个抽象类，它的parseAnnotations方法是一个静态方法，源码如下：

```java
// ServiceMethod
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
  // 将接口方法上的注解信息解析到RequestFactory中。
  RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

	// ...

  return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
}
```

在 ServiceMethod 的 parseAnnotations 方法中，又通过RequestFactory的parseAnnotations来解析方法上的注解信息，并将注解信息封装到了RequestFactory中，关于这个方法这里就不贴代码了，无非就是解析方法上的注解，如GET、POST等，还有解析方法参数上的注解，如Query、Body等。得到RequestFactory后，将RequestFactory等参数传递给 HttpServiceMethod 的 parseAnnotations方法最终得到了ServiceMethod。HttpServiceMethod是ServiceMethod的实现类，它自身也是一个抽象方法，HttpServiceMethod.parseAnnotations最终返回的也是一个继承了HttpServiceMethod的CallAdapted实例。在得到CallAdapted的实例后最终在Retrofit的create方法中调用了CallAdapted的invoke方法来发起网络请求。

### 2. HttpServiceMethod

HttpServiceMethod 是 Retrofit 中很重要的一个类，它的核心方法是parseAnnotations，上一小节中已经提到了这个方法，它会返回一个CallAdapted的实例，并且这个CallAdapted继承自HttpServiceMethod。先来看下 `HttpServiceMethod.parseAnnotations`的实现，代码如下：

```java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {
  boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
	// ...
	// 获取方法的注解
  Annotation[] annotations = method.getAnnotations();
  Type adapterType;
  if (isKotlinSuspendFunction) { // 对Kotlin挂起函数的支持
    // ... 省略处理挂起函数的逻辑
  } else {
    // 获取API接口方法上定义的返回类型,因为ApiService中的方法返回类型可能是Call、
    // Observable等所以需要知道方法的返回类型，然后根据返回类型获取CallAdapter
    adapterType = method.getGenericReturnType();
  }
  // 根据方法返回值类型获取对应的CallAdapter
  CallAdapter<ResponseT, ReturnT> callAdapter =
      createCallAdapter(retrofit, method, adapterType, annotations);
  // 获取response类型
  Type responseType = callAdapter.responseType();
  // ... 省略 responseType 的校验
  
  //  根据responseType获取Response转换器，ResponseConverter是可以自定义的Response转换器，比如GsonConverterFactory
  Converter<ResponseBody, ResponseT> responseConverter =
      createResponseConverter(retrofit, method, responseType);
  // 获取OkHttp3的Factory
  okhttp3.Call.Factory callFactory = retrofit.callFactory;
  if (!isKotlinSuspendFunction) {
    // 注意CallAdapted继承了HttpServiceMethod
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
  }
  // ... 省略对kotlin挂起函数的处理
}
```

这个方法中的逻辑可以分为三部分：

- **获取API接口方法的返回类型。**不考虑这个方法对kotlin挂起函数的至此，简化后的逻辑其实并不复杂。首先，通过method得到Api接口方法的返回类型adapterType。然后调用createCallAdapter根据adapterType去得到对应的CallAdapter。

- **获取API接口方法的Response类型。** 拿到callAdapter后通过responseType方法拿到responseType，即返回类型，这个返回类型指的是API接口方法中返回参数的泛型，例如Call\<T> 或者 Observable\<T> 中的泛型类型。因为通过OkHttp获取到服务器返回的数据后是需要将数据转换成对应的接收类型的。那么有了这个类型后又会通过createResponseConverter来获取响应数据的转换器，通过这个转换器将OkHttp拿到的Response转换成需要的类型。

- **构造CallAdapted实例。** 有了以上参数后，那么就可以发起网络请求了。即通过构造方法实例化CallAdapted，并将这些参数传递个CallAdapted。最终调用CallAdapted的invoke发起网络请求。

因此，接下来我们还需要分析CallAdapted、CallAdapter.Factory以及ResponseConverter的具体实现。

### 3. CallAdapted

关于CallAdapted这个类上一节已经看到多次，它是HttpServiceMethod的静态内部类，并且继承自HttpServcieMethod。Retrofit的create方法中调用loadServiceMethod方法返回的实例就是CallAdapted，在得到CallAdapted的实例对象后便调用了invoke方法。因此，先来看下 CallAdapted 方法中对 invoke 的实现,翻阅源码发现CallAdapted中并没有实现invoke方法，而是在它的父类HttpServiceMethod中。代码如下：

```java
// HttpServiceMethod
@Override
final @Nullable ReturnT invoke(Object[] args) {
  Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
  // 调用自身的adapt方法
  return adapt(call, args);
}

protected abstract @Nullable ReturnT adapt(Call<ResponseT> call, Object[] args);
```

invoke方法中实例化了一个OkHttpCall对象，OKHttp的网络请求就是由这个对象发起的，然后调用自身的adapt方法并将OkHttpCall参数传入。HttpServiceMethod的adapt方法是一个抽象方法，具体实现是在CallAdapted中，CallAdapted的源码如下：

```java
static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
  private final CallAdapter<ResponseT, ReturnT> callAdapter;

  CallAdapted(
      RequestFactory requestFactory,
      okhttp3.Call.Factory callFactory,
      Converter<ResponseBody, ResponseT> responseConverter,
      CallAdapter<ResponseT, ReturnT> callAdapter) {
    super(requestFactory, callFactory, responseConverter);
    // 注意这里传入的callAdapter参数
    this.callAdapter = callAdapter;
  }

  @Override
  protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
    // 调用CallAdapter的adapt方法，比如{@link RxJava2CallAdapter#adapt}
    return callAdapter.adapt(call);
  }
}
```

CallAdapted中的adapt实现的就很简单了，将OKHttpCall作为参数，然后调用了callAdapter的adapt方法。CallAdapter在本章第二节中已经有所了解，它是通过CallAdapter.Factory创建出来的。那么接下来详细的看下 CallAdapter.Factory的实现。

### 4. CallAdapter.Factory

在本章第2节中HttpServiceMethod的parseAnnotations方法中，首先通过createCallAdapter来获取CallAdapter。这个方法的代码如下：

```java
// HttpServiceMethod

// 通过接口方法上的返回类型查找对应的CallAdapter，比如如果返回的是Observable<T>,则会找到RxJava2CallAdapter
private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
    Retrofit retrofit, Method method, Type returnType, Annotation[] annotations) {
  try {
    //noinspection unchecked
    return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(method, e, "Unable to create call adapter for %s", returnType);
  }
}
```

在这个方法中又去调用了retrofit.callAdapter方法来获取CallAdapter。所以，继续来看Retrofit的callAdapter方法的代码：

```java
// Retrofit

final List<CallAdapter.Factory> callAdapterFactories;

public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
  return nextCallAdapter(null, returnType, annotations);
}

public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
    Annotation[] annotations) {

  int start = callAdapterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
    CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
    if (adapter != null) {
      return adapter;
    }
  }
	// ... 省略异常校验
}
```

callAdapter中又调用了nextCallAdapter方法，在这个方法里边会去遍历callAdapterFactories集合，并根据returnType查找对应的CallAdapter.Factory。找到对应的CallAdapter.Factory后调用get方法便得到了一个CallAdapter

callAdapterFactories集合中的对象是什么时候被添加进去的呢？其实就是在构建Retrofit实例的时候，Retrofit中有一个 `addCallAdapterFactory` 方法，源码如下：

```java
public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
  callAdapterFactories.add(checkNotNull(factory, "factory == null"));
  return this;
}
```

在Retrofit#Builder的build方法中会为callAdapterFactories添加默认的CallAdapterFactorie，代码如下：

```java
// Retrofit#Builder

public Retrofit build() {
  if (baseUrl == null) {
    throw new IllegalStateException("Base URL required.");

	// ...
  添加默认的 CallAdapter.Factory
  callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

	// ...
  return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
      unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
}
```

在Platform中创建了 ExecutorCallAdapterFactory，代码如下：

```java
List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
    @Nullable Executor callbackExecutor) {
  if (callbackExecutor != null) {
    return singletonList(new ExecutorCallAdapterFactory(callbackExecutor));
  }
  return singletonList(DefaultCallAdapterFactory.INSTANCE);
}
```

ExecutorCallAdapterFactory 对应的是API接口中方法返回 Call\<T> 的情况。而如果我们想要返回Observable\<T>，则需要在初始化Retrofit#Builder时自行添加 RxJava2CallAdapterFactory，代码如下：

```java
Retrofit retrofit =
    new Retrofit.Builder()
        .baseUrl(API_URL)
  			.addCallAdapterFactory(RxJava2CallAdapterFactory.create()) // 添加RxJava2CallAdapterFactory
        .addConverterFactory(GsonConverterFactory.create())
        .build();
```

只有添加了添加RxJava2CallAdapterFactory之后API接口中方法的返回值才支持Observable\<T>类型。

在项目中大部分情况都是结合RxJava来使用的，因此，这里就拿RxJava2CallAdapterFactory来分析。由于在Retrofit的nextCallAdapter方法中根据returnType获取到对应的CallAdapter#Factory后，通过调用CallAdapter#Factory的get方法得到了CallAdapter。所以，先来看RxJava2CallAdapterFactory的get方法。代码如下：

```java
public @Nullable CallAdapter<?, ?> get(
    Type returnType, Annotation[] annotations, Retrofit retrofit) {
  Class<?> rawType = getRawType(returnType);

  if (rawType == Completable.class) {
    // Completable 的返回类型表示Response是空的，或者不关心返回数据，因此无需responseType.
    // 故可以直接实例化RxJava2CallAdapter，responseType传入Void.class即可。
    return new RxJava2CallAdapter(
        Void.class, scheduler, isAsync, false, true, false, false, false, true);
  }

	// ...
	
  // 接下来通过returnType来得到responseType
  Type responseType;

  Type observableType = getParameterUpperBound(0, (ParameterizedType) returnType);
  Class<?> rawObservableType = getRawType(observableType);
  if (rawObservableType == Response.class) {
		// ...
    responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
  } else if (rawObservableType == Result.class) {
    // ...
    responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
    isResult = true;
  } else {
    responseType = observableType;
    isBody = true;
  }
	// 实例化RxJava2CallAdapter
  return new RxJava2CallAdapter(
      responseType, scheduler, isAsync, isResult, isBody, isFlowable, isSingle, isMaybe, false);
}
```

可以看到get方法中的逻辑仅仅是实例化了RxJava2CallAdapter。上一节中最后调用了CallAdapter的adapt方法，RxJava2CallAdapter实现了CallAdapter，因此上一节中的adapt方法实际上会调用到RxJava2CallAdapter的adapt方法。RxJava2CallAdapter的adapt代码如下：

```java
// RxJava2CallAdapter
final class RxJava2CallAdapter<R> implements CallAdapter<R, Object> {
 // ...
  
  @Override
  public Object adapt(Call<R> call) {
    // 实例化 CallEnqueueObservable 或者 CallExecuteObservable，区别是一个异步，一个同步
    Observable<Response<R>> responseObservable =
        isAsync ? new CallEnqueueObservable<>(call) : new CallExecuteObservable<>(call);

    Observable<?> observable;
    // 根据isResult构造Observable
    if (isResult) {
      // 包含Result的Observable，持有CallEnqueueObservable
      observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
      // 只有Body的Observable
      observable = new BodyObservable<>(responseObservable);
    } else {
      observable = responseObservable;
    }

    if (scheduler != null) {
      observable = observable.subscribeOn(scheduler);
    }

    if (isFlowable) {
      return observable.toFlowable(BackpressureStrategy.LATEST);
    }
    if (isSingle) {
      return observable.singleOrError();
    }
    if (isMaybe) {
      return observable.singleElement();
    }
    if (isCompletable) {
      return observable.ignoreElements();
    }
    // 可以理解为 return observable 
    return RxJavaPlugins.onAssembly(observable);
  }
}
```

RxJava2CallAdapter的adapt中首先根据是否是异步调用来创建CallEnqueueObservable或者CallExecuteObservable。CallExecuteObservable是一个Observable，待会儿看源码，接下来，我们以请求会返回Response为例，即实例化了一个ResultObservable，它持有了CallExecuteObservable。最终这个方法返回的是这个ResultObservable。也就是我们调用接口最终拿到的是这个ResultObservable，ResultObservable源码如下：

```java
final class ResultObservable<T> extends Observable<Result<T>> {
  private final Observable<Response<T>> upstream;

  ResultObservable(Observable<Response<T>> upstream) {
    this.upstream = upstream;
  }

  @Override
  protected void subscribeActual(Observer<? super Result<T>> observer) {
    upstream.subscribe(new ResultObserver<T>(observer));
  }

  private static class ResultObserver<R> implements Observer<Response<R>> {
    private final Observer<? super Result<R>> observer;

    ResultObserver(Observer<? super Result<R>> observer) {
      this.observer = observer;
    }
		
    @Override
    public void onSubscribe(Disposable disposable) {
      // 调用了observer的onSubscribe
      observer.onSubscribe(disposable);
    }

    @Override
    public void onNext(Response<R> response) {
      observer.onNext(Result.response(response));
    }

    @Override
    public void onError(Throwable throwable) {
      try {
        observer.onNext(Result.<R>error(throwable));
      } catch (Throwable t) {
        try {
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
        return;
      }
      observer.onComplete();
    }

    @Override
    public void onComplete() {
      observer.onComplete();
    }
  }
}
```

调用它的subscribe方法最终触发了subscribeActual，这个方法找那个又调用了CallEnqueueObservable的subscribe方法。因此看下CallEnqueueObservable的实现：

```java
final class CallEnqueueObservable<T> extends Observable<Response<T>> {
  private final Call<T> originalCall;

  CallEnqueueObservable(Call<T> originalCall) {
    this.originalCall = originalCall;
  }

  @Override
  protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    CallCallback<T> callback = new CallCallback<>(call, observer);
    observer.onSubscribe(callback);
    if (!callback.isDisposed()) {
      // 这里是通过OKHttpCall调用了enqueue发起请求，enqueue是异步方法。
      call.enqueue(callback);
    }
  }
	// OKHttpCall的回调类
  private static final class CallCallback<T> implements Disposable, Callback<T> {
    private final Call<?> call;
    private final Observer<? super Response<T>> observer;
    private volatile boolean disposed;
    boolean terminated = false;

    CallCallback(Call<?> call, Observer<? super Response<T>> observer) {
      this.call = call;
      this.observer = observer;
    }
		// 请求成功
    @Override
    public void onResponse(Call<T> call, Response<T> response) {
      if (disposed) return;

      try {
        observer.onNext(response);

        if (!disposed) {
          terminated = true;
          observer.onComplete();
        }
      } catch (Throwable t) {
        Exceptions.throwIfFatal(t);
        if (terminated) {
          RxJavaPlugins.onError(t);
        } else if (!disposed) {
          try {
            observer.onError(t);
          } catch (Throwable inner) {
            Exceptions.throwIfFatal(inner);
            RxJavaPlugins.onError(new CompositeException(t, inner));
          }
        }
      }
    }
		// 请求失败
    @Override
    public void onFailure(Call<T> call, Throwable t) {
      if (call.isCanceled()) return;

      try {
        observer.onError(t);
      } catch (Throwable inner) {
        Exceptions.throwIfFatal(inner);
        RxJavaPlugins.onError(new CompositeException(t, inner));
      }
    }

    @Override
    public void dispose() {
      disposed = true;
      call.cancel();
    }

    @Override
    public boolean isDisposed() {
      return disposed;
    }
  }
}
```

可以看到CallEnqueueObservable继承了Observable，内部通过OkHttpCall的equeue发起网络请求。CallCallback是接收请求结果的回调。



## 4. ConverterFactory

ConverterFactory 主要用于对象的序列化和反序列化，比如说将请求的对象转成 JSON ，然后传递给服务器，获取将服务器返回的 JSON 转成我们需要的对象。在初始化Retrofit的时候我们会通过addConverterFactory添加转换器，例如常用的GsonConverterFactory。Retrofit内部同样会将ConverterFactory保存到一个List集合。代码如下：

```java
	// Retrofit
    private final List<Converter.Factory> converterFactories = new ArrayList<>();
    /** Add converter factory for serialization and deserialization of objects. */
    public Builder addConverterFactory(Converter.Factory factory) {
      converterFactories.add(Objects.requireNonNull(factory, "factory == null"));
      return this;
    }
```

而CallAdapter的获取同样是在HttpServiceMethod的parseAnnotations中：

```java
 //  根据responseType获取Response转换器，ResponseConverter是可以自定义的Response转换器，比如GsonConverterFactory
    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);
```

最终ConverterFactory会被添加到OkHttpCall中，在调用equeue发起请求后，将reponse使用ConverterFactory进行解析：

```java
public void enqueue(final Callback<T> callback) {
    Objects.requireNonNull(callback, "callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
	    call.enqueue(
	        new okhttp3.Callback() {
	          @Override
	          public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
	            Response<T> response;
	           response = parseResponse(rawResponse);
	          }
  	}
  }
```

```java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
    try {
     // 此处即为设置的Converter
      T body = responseConverter.convert(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```

GosnResponseBodyConverter的实现如下：

```java
final class GsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
  private final Gson gson;
  private final TypeAdapter<T> adapter;

  GsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter) {
    this.gson = gson;
    this.adapter = adapter;
  }

  @Override
  public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      T result = adapter.read(jsonReader);
      if (jsonReader.peek() != JsonToken.END_DOCUMENT) {
        throw new JsonIOException("JSON document was not fully consumed.");
      }
      return result;
    } finally {
      value.close();
    }
  }
}
```

