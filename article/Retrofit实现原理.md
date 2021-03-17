
## 1. 使用Retrofit的时候定义的是接口，那Retrofit是如何生成接口的实现类的呢？

在调用Retrofit的create方法的时候，会通过动态代理的方式去生成接口的实现类，当去调用接口方法时实际上会调用到InvocationHandler的invoke方法。create方法代码如下：

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
接下来loadServiceMethod方法会根据接口方法的信息将method封装成一个ServcieMethod,ServiceMethod是一个抽象类，它的唯一实现是HttpServiceMethod,在HttpServcieMethod类的invoke方法中生成Observable或者Call对象：

```java
  // HttpServiceMethod 
  @Override
  final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
      /**
       *  调用{@link CallAdapted#adapt}方法
       */
    return adapt(call, args);
  }
```

```java
// HttpServiceMethod$CallAdapter
@Override
    protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
      // 调用CallAdapter的adapt方法，比如{@link RxJava2CallAdapter#adapt}
      return callAdapter.adapt(call);
    }
```
以RxJava2CallAdapter为例：

```java
// RxJava2CallAdapter
@Override
  public Object adapt(Call<R> call) {
    Observable<Response<R>> responseObservable =
        isAsync ? new CallEnqueueObservable<>(call) : new CallExecuteObservable<>(call);

    Observable<?> observable;
    if (isResult) {
      observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
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
    return RxJavaPlugins.onAssembly(observable);
  }
// RxJavaPlugins 生成Observable
@SuppressWarnings({ "rawtypes", "unchecked" })
    public static <T> Observable<T> onAssembly(Observable<T> source) {
        Function<Observable, Observable> f = onObservableAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }
```
## 2.Retrofit是如何通过注解组装请求参数的呢？

在loadServiceMethod中通过ServiceMethod.parseAnnotations(this, method)将接口方法上的注解信息解析到ServiceMethod中。

```java
// ServiceMethod
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    // 将接口方法上的注解信息解析到RequestFactory中。
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
    // ... 省略部分代码
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
```

```java
// RequestFactory
static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
}

RequestFactory build() {
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }
   	  //...  省略部分代码
      return new RequestFactory(this);
    }

	// 将接口方法上的注解信息解析并拼接成Http的请求参数
    private void parseMethodAnnotation(Annotation annotation) {
      if (annotation instanceof DELETE) { // HTTP DELETE
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) { // HTTP GET
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
      } else if (annotation instanceof PATCH) {
        // ...省略代码
      }
    }

private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
      this.httpMethod = httpMethod;
      this.hasBody = hasBody;
      // 这里可能在URL中拼接了参数，将拼接的参数解析出来
      // Get the relative URL path and existing query string, if present.
      int question = value.indexOf('?');
      if (question != -1 && question < value.length() - 1) {
        // Ensure the query string does not have any named parameters.
        String queryParams = value.substring(question + 1);
        Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
  	 }
      this.relativeUrl = value;
      this.relativeUrlParamNames = parsePathParameters(value);
    }
```
## 3.为什么我们在接口方法的返回值可以是Call也可以是RxJava的Observable，Retrofit内部是如何实现的？
在HttpServiceMethod的parseAnnotations中会获取接口方法的返回值类型，根据返回类型取匹配对应的CallAdapter#Factory，并生成对应的CallAdapter。而CallAdapter#Factory是我们在初始化Retrofit的时候自行添加到Retrofit中的，CallAdapter#Factory会被保存到List集合中。
例如，在初始化Retrofit的时候，我们会为其添加RxJava2CallAdapterFactory:

```java
Retrofit retrofit =
        new Retrofit.Builder()
            .baseUrl(API_URL)
            .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
            .build();
```
而RxJava2CallAdapterFactory最终会被保存到callAdapterFactories中：

```java
// Retrofit 
 private final List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
 public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
      callAdapterFactories.add(Objects.requireNonNull(factory, "factory == null"));
      return this;
    }
```
在Retrofit解析注解信息的时候，会同时获得接口方法的返回值类型，并根据这个类型从callAdapterFactories中取出对应的CallAdapter#Factory并生成对应的CallAdapter

```java
// HttpServiceMethod,简化后的parseAnnotations
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    Annotation[] annotations = method.getAnnotations();
    Type adapterType;
   // 获取接口方法上定义的返回类型
      adapterType = method.getGenericReturnType();
    // 获取接口方法返回值类型对应的CallAdapter
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
 	// 注意CallAdapted继承了HttpServiceMethod
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
  }
```
接着会调用CallAdapted的invoke方法：

```java
 /**
   *  这个方法是就在{@link Retrofit#create}方法中调用的loadServiceMethod(method).invoke(args)
    */
  @Override
  final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
      /**
       *  调用{@link CallAdapted#adapt}方法
       */
    return adapt(call, args);
  }
```
而adapt实际上就是调用了RxJava2CallAdapter的adapt,来看实现：

```java
@Override
  public Object adapt(Call<R> call) {
    Observable<Response<R>> responseObservable =
        isAsync ? new CallEnqueueObservable<>(call) : new CallExecuteObservable<>(call);

    Observable<?> observable;
    if (isResult) {
      observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
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
    return RxJavaPlugins.onAssembly(observable);
  }
```
RxJavaPlugins#onAssembly
```java
   /**
     * Calls the associated hook function.
     * @param <T> the value type
     * @param source the hook's input value
     * @return the value returned by the hook
     */
    @SuppressWarnings({ "rawtypes", "unchecked" })
    public static <T> Observable<T> onAssembly(Observable<T> source) {
        Function<Observable, Observable> f = onObservableAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }
```
最终返回的就是一个Observable<T>

## 4.CallAdapter.Factory有什么用？
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