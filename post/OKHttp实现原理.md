## OkHttp使用



```java
OkHttpClient client = new OkHttpClient();
Request request = new Request.Builder()
                .get()
                .url("https:www.baidu.com")
                .build();

Call call = client.newCall(request);

// 同步调用
Response response = call.execute();

// 或者异步调用
call.enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
    }

    @Override
    public void onResponse(Call call, final Response response) throws IOException {

    }
});
```



## 原理剖析



### 1.通过newCall获得RealCall对象

```kotlin
override fun newCall(request: Request): Call = RealCall(this, request, forWebSocket = false)
```

所有execute方法和enqueue调用的都是RealCall中的代码，如下：



```kotlin
// RealCall
override fun execute(): Response {
  check(executed.compareAndSet(false, true)) { "Already Executed" }

  timeout.enter()
  callStart()
  try {
    client.dispatcher.executed(this)
    return getResponseWithInterceptorChain()
  } finally {
    client.dispatcher.finished(this)
  }
}


override fun enqueue(responseCallback: Callback) {
  check(executed.compareAndSet(false, true)) { "Already Executed" }

  callStart()
  // 注意这里调用Dispatcher的enqueue方法时实例化了一个AsyncCall
  client.dispatcher.enqueue(AsyncCall(responseCallback))
}
```

RealCall中不管是execute方法还是enqueue方法，最终都是通过Dispatcher发起的。



### 2.分发器--Dispatcher

Dispatcher可以翻译为分发器，是OKHttp中非常重要的一个角色。它的主要作用是用来调配请求任务的，Dispatcher会根据情况决定任务是被放到ready队列还是放到running队列。同时，还会根据条件将任务从ready队列调入running队列。Dispatcher类的代码结构如下：



```kotlin
class Dispatcher constructor() {
  // Okhttp能够同时发起的最大请求数
  var maxRequests = 64
  // 同一个Host能同时发起的最大请求数
  var maxRequestsPerHost = 5
  // 线程池
  private var executorServiceOrNull: ExecutorService? = null


  /** 异步调用的ready任务队列 */
  private val readyAsyncCalls = ArrayDeque<AsyncCall>()

  /** 异步调用的running任务队列 */
  private val runningAsyncCalls = ArrayDeque<AsyncCall>()

  /** 同步调用的任务队列 */
  private val runningSyncCalls = ArrayDeque<RealCall>()

  constructor(executorService: ExecutorService) : this() {
    this.executorServiceOrNull = executorService
  }

}
```

接下来以异步调用enqueue方法为例，来分析源码

```kotlin
// Dispatcher

// 这里的AsyncCall是在RealCall的enqueue方法中实例化出来的
internal fun enqueue(call: AsyncCall) {
  synchronized(this) { // 保证线程安全
    // 先将任务加入准备队列
    readyAsyncCalls.add(call)
    if (!call.call.forWebSocket) {
      val existingCall = findExistingCallWithHost(call.host)
      if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
    }
  }
  // 执行任务的主流程
  promoteAndExecute()
}
```

在promoteAndExecute方法中会根据条件将要执行的任务从ready队列移到，代码如下：

```kotlin
// Dispatcher
private fun promoteAndExecute(): Boolean {
  this.assertThreadDoesntHoldLock()

  val executableCalls = mutableListOf<AsyncCall>()
  val isRunning: Boolean
  synchronized(this) {
    val i = readyAsyncCalls.iterator()
    while (i.hasNext()) {
      val asyncCall = i.next()
			// 正在执行的任务大于等于64，直接结束循环遍历，此时任务被添加到了等待队列中等待执行
      if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
      // 如果当前正在这个host的请求数量大于maxRequestsPerHost，则跳过该任务，遍历后边任务
      if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.

      i.remove()
      asyncCall.callsPerHost.incrementAndGet()
      executableCalls.add(asyncCall)
      // 通过限制条件后则将任务加入到running队列
      runningAsyncCalls.add(asyncCall)
    }
    isRunning = runningCallsCount() > 0
  }
  for (i in 0 until executableCalls.size) {
    val asyncCall = executableCalls[i]
    // 遍历executableCalls集合并调用RealCall的executeOn开始执行任务
    asyncCall.executeOn(executorService)
  }

  return isRunning
}  
```

可以看到，如果当前正在执行的任务大于maxRequests时，直接结束循环，意味着新发起的请求被添加到了ready队列中等待执行。

如果当前正在runing的请求的host的数量大于maxRequestsPerHost，那么这个任务也不会被添加到running队列。continue后继续遍历后边的任务。

通过上述两个限制条件后，任务最终会被添加到executableCalls和runningAsyncCalls中等待执行。

最后，遍历executableCalls集合并调用RealCall的executeOn开始执行任务。



RealCall的executeOn方法如下：



```kotlin
// RealCall
fun executeOn(executorService: ExecutorService) {
  client.dispatcher.assertThreadDoesntHoldLock()

  var success = false
  try {
    // 将RealCall交给线程池执行
    executorService.execute(this)
    success = true
  } catch (e: RejectedExecutionException) {
    val ioException = InterruptedIOException("executor rejected")
    ioException.initCause(e)
    noMoreExchanges(ioException)
    responseCallback.onFailure(this@RealCall, ioException)
  } finally {
    if (!success) {
      client.dispatcher.finished(this) // This call is no longer running!
    }
  }
}
```

上述方法中通过executorService将任务交给线程池来执行。而executorService是在Dispatcher中被初始化的：



```kotlin
// Dispatcher

@get:Synchronized
@get:JvmName("executorService") val executorService: ExecutorService
  get() {
    if (executorServiceOrNull == null) {
      executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
          SynchronousQueue(), threadFactory("$okHttpName Dispatcher", false))
    }
    return executorServiceOrNull!!
  }
```

可以看到在OkHttp中创建的这个线程池核心线程数是0，最大线程数是Int.MAX_VALUE,且传入了一个SynchronousQueue的阻塞队列。这样创建出来的线程池有一个特点，即：高并发、最大吞吐量。

这里还应该注意一下SynchronousQueue，SynchronousQueue是一个没有容量的容器，通过SynchronousQueue保证了所有任务不会被添加到等待队列中，而是创建新的线程立即执行任务。不会造成任务被阻塞到队列。

RealCall被加入线程池之后则会去执行RealCall的run方法，run方法的代码如下：

```kotlin
// RealCall
override fun run() {
    threadName("OkHttp ${redactedUrl()}") {
      var signalledCallback = false
      timeout.enter()
      try {
        // 发起请求
        val response = getResponseWithInterceptorChain()
        signalledCallback = true
        // 请求成功后回调结果
        responseCallback.onResponse(this@RealCall, response)
      } catch (e: IOException) {
        // ...
        responseCallback.onFailure(this@RealCall, e)
        throw t
      } finally {
        client.dispatcher.finished(this)
      }
    }
  }
}
```

接下来，请求被交给了getResponseWithInterceptorChain方法来执行

```kotlin
// RealCall
internal fun getResponseWithInterceptorChain(): Response {
  // Build a full stack of interceptors.
  val interceptors = mutableListOf<Interceptor>()
  // 将所有拦截器添加到集合中
  // 自定义的拦截器
  interceptors += client.interceptors
  // 重试和重定向拦截器
  interceptors += RetryAndFollowUpInterceptor(client)
  // 桥接拦截器
  interceptors += BridgeInterceptor(client.cookieJar)
  // 缓存拦截器
  interceptors += CacheInterceptor(client.cache)
  // 链接拦截器
  interceptors += ConnectInterceptor
  
  if (!forWebSocket) {
    // 自定义拦截器
    interceptors += client.networkInterceptors
  }
  // 请求拦截器
  interceptors += CallServerInterceptor(forWebSocket)

  // 构建interceptors的责任链
  val chain = RealInterceptorChain(
      call = this,
      interceptors = interceptors,
      index = 0,
      exchange = null,
      request = originalRequest,
      connectTimeoutMillis = client.connectTimeoutMillis,
      readTimeoutMillis = client.readTimeoutMillis,
      writeTimeoutMillis = client.writeTimeoutMillis
  )

  var calledNoMoreExchanges = false
  try {
    // 通过责任链依次执行拦截器的intercept方法，并返回请求结果
    val response = chain.proceed(originalRequest)
 
    return response
  } catch (e: IOException) {
    calledNoMoreExchanges = true
    throw noMoreExchanges(e) as Throwable
  } finally {
    if (!calledNoMoreExchanges) {
      noMoreExchanges(null)
    }
  }
}
```

getResponseWithInterceptorChain方法中的代码很有意思，首先创建了一个interceptors的集合，并将一系列的intercepter添加到了集合中，然后通过责任链模式依次执行所有intercepter的intercept方法。具体代码可以看RealInterceptorChain类的实现。



### 2.拦截器--Intercepter

上一节中Dispatcher将请求最终交给了Intercepter，最终的请求也是在Intercept中执行的。可见Intercepter在OkHttp中是另一个重要角色。本节就来详细的分析OkHttp的拦截器。



Interceptor是一个接口，内部有一个intercept方法，以及一个Chain的内部接口：

```kotlin
fun interface Interceptor {
  @Throws(IOException::class)
  fun intercept(chain: Chain): Response

  companion object {
    /**
     * Constructs an interceptor for a lambda. This compact syntax is most useful for inline
     * interceptors.
     *
     * ```
     * val interceptor = Interceptor { chain: Interceptor.Chain ->
     *     chain.proceed(chain.request())
     * }
     * ```
     */
    inline operator fun invoke(crossinline block: (chain: Chain) -> Response): Interceptor =
      Interceptor { block(it) }
  }

  interface Chain {
    fun request(): Request

    @Throws(IOException::class)
    fun proceed(request: Request): Response

    /**
     * Returns the connection the request will be executed on. This is only available in the chains
     * of network interceptors; for application interceptors this is always null.
     */
    fun connection(): Connection?

    fun call(): Call

    fun connectTimeoutMillis(): Int

    fun withConnectTimeout(timeout: Int, unit: TimeUnit): Chain

    fun readTimeoutMillis(): Int

    fun withReadTimeout(timeout: Int, unit: TimeUnit): Chain

    fun writeTimeoutMillis(): Int

    fun withWriteTimeout(timeout: Int, unit: TimeUnit): Chain
  }
}
```



OkHttp中除了自定义的拦截器外，内部有五大默认拦截器来负责请求的重试重定向、桥接、缓存、以及调用服务器等功能。不管是自定义拦截器还是默认拦截器都实现了Intercepter接口。

#### （1）RetryAndFollowUpInterceptor

RetryAndFollowUpInterceptor是第一个被添加的默认拦截器，它负责失败重试和重定向。它的执行流程如下：

1. 先看任务是否被取消，被取消了就释放资源，不再执行请求。
2. 调用下一拦截器执行后续请求，如果请求出现问题就要判断是否要重新执行请求，不重新执行的话就释放资源并抛出异常。
3. 请求成功的话会根据响应码判断是否需要重定向，不需要重定向的话就返回 Response。需要重定向的话就销毁旧连接并创建新连接，进行新一轮循环。最多可重定向20次。

#### （2）BridgeInterceptor

负责把用户请求转换为发送到服务器的请求，并把服务器的响应转化为用户需要的响应。它的执行步骤如下：

1. 将用户的 Request 构造为发送给服务器的 Reuquest，该过程会添加各种请求报头(包括 Host、Connection、cookie 等)
2. 构造完成后，将新的 Request 交给下一拦截器来处理
3. 得到服务器的 Response 后，先保存 cookies，接着将服务器的 Response 转换为用户需要的 Response 并返回（如果使用了 gzip 压缩并且服务器的 Response 有 body 的话，还要给用户的 Response 设置相应 body）

#### （3）CacheInterceptor

负责读取缓存、更新缓存。执行步骤如下：

1. 从 Cache 中得到 Request 对应的缓存，默认没有设置 Cache，需要用户自己配置
2. 得到缓存策略
3. 如果通过缓存策略没有得到缓存，则关闭缓存
4. 如果缓存策略设置了禁用网络，看得到的缓存是否为空，如果缓存为空，则构建一个返回码为 504 的 Response，说明返回失败。如果缓存不为空，则返回缓存
5. 如果可以使用网络，就交给下一拦截器执行请求，执行请求的过程发生异常，及时关闭缓存，并抛出异常，让上一拦截器处理。
6. 当缓存和网络返回的 Response 同时存在时，如果返回的状态码为 304（说明服务器的文件未更新，可以使用缓存），则返回缓存。否则更新缓存，并返回网络请求后的 Response

#### （4）ConnectInterceptor

负责和服务器建立连接。执行步骤如下：

1. 找到一个可用的 RealConnection， 再利用这个 RealConnection 的输入输出（BufferSource 和 BufferSink）创建 HttpCodec。（ HttpCodec 有两个实现：Http1Codec 和 Http2Codec，分别对应 HTTP/1.1 和 HTTP/2 版本）
2. 调用下一拦截器进行后续请求操作。

#### （5）CallServerInterceptor

负责向服务器发送数据，从服务器读取响应数据。执行步骤如下：

1. 向服务器写入请求头，如果请求头有 Expect: 100-continue，需要根据服务器返回的结果决定是否可以继续写入请求体。
2. 得到响应头并构建带有响应头的 Response，接着为 Response 构建响应体并返回。

https://mrfzh.github.io/2019/07/18/okhttp3%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%9A%E4%BA%94%E5%A4%A7%E6%8B%A6%E6%88%AA%E5%99%A8/

