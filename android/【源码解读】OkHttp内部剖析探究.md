OkHttp内部关键在于拦截器的处理来实现，把网络请求封装到各个拦截器来实现，实现了各层的解耦。

我们首先发起一个请求：

```java
//创建okHttpClient对象
OkHttpClient okHttpClient = new OkHttpClient.Builder()
                    .connectTimeout(6, TimeUnit.SECONDS)
                    .readTimeout(6, TimeUnit.SECONDS)
                    .build();

//创建一个Request
Request request = new Request.Builder().url(strUrl).build();
//发起同步请求
try {
            Response response = client.newCall(request).execute();
            return response.body().string();
        } catch (IOException e) {
            e.printStackTrace();
        }
```

## 内部请求流程

最终实现的代码是OkHttp会调用newCall，返回一个RealCall对象，并调用execute同步方法。其中`RealCall`是管理网络请求的类。

```kotlin
## RealCall   
override fun execute(): Response {
    synchronized(this) {
      check(!executed) { "Already Executed" }
      executed = true
    }
    //开始计时，发送
    transmitter.timeoutEnter()
    transmitter.callStart()
    try {
      //适配器请求
      client.dispatcher.executed(this)
      //调用拦截器处理
      return getResponseWithInterceptorChain()
    } finally {
      //适配器结束
      client.dispatcher.finished(this)
    }
  }
```

知道了这个三个方法后。我们一个个来看~

先看这里调用了`dispatcher`的`excuted`方法，那`Dispatcher`在里面起到什么作用？可以看到有三个数组队列进行处理

```kotlin
### Dispatcher
	//准备异步调用队列
  private val readyAsyncCalls = ArrayDeque<AsyncCall>()

  //运行异步调用队列
  private val runningAsyncCalls = ArrayDeque<AsyncCall>()

 	//运行同步调用队列
  private val runningSyncCalls = ArrayDeque<RealCall>()
```

在我们上一步过程中，调用了

```kotlin
### Dispatcher

//同步
@Synchronized internal fun executed(call: RealCall) {
  //放进正在运行的队列
    runningSyncCalls.add(call)
  }

//异步
internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
      readyAsyncCalls.add(call)

      // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
      // the same host.
      if (!call.get().forWebSocket) {
        val existingCall = findExistingCallWithHost(call.host())
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
      }
    }
    promoteAndExecute()
  }


//Dispath 核心逻辑
//执行enqueue异步时被调用  
//将符合条件的调用从 [readyAsyncCalls] 提升到 [runningAsyncCalls] 并在执行程序服务上运行它们。不得与同步调用，因为执行调用可以调用用户代码。 @return true 如果调度程序当前正在运行调用。
  private fun promoteAndExecute(): Boolean {
    assert(!Thread.holdsLock(this))

    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    synchronized(this) {
      val i = readyAsyncCalls.iterator()
      while (i.hasNext()) {
        val asyncCall = i.next()

        if (runningAsyncCalls.size >= this.maxRequests) break // 大于最大请求request数量
        if (asyncCall.callsPerHost().get() >= this.maxRequestsPerHost) continue // 大于最大容量
//当一个取出来后，从readyAsyncCalls中删除
        i.remove()
        asyncCall.callsPerHost().incrementAndGet()
        executableCalls.add(asyncCall)
        //移入到runningAsyncCalls中
        runningAsyncCalls.add(asyncCall)
      }
      isRunning = runningCallsCount() > 0
    }

    //将executableCalls可执行的调用提交到任务线程池
    for (i in 0 until executableCalls.size) {
      val asyncCall = executableCalls[i]
      asyncCall.executeOn(executorService)
    }

    return isRunning
  }

```

这个方法`promoteAndExecute`会在入队列和上一个任务执行结束后调用。我们再来看一下里面的`executorService`线程池的用法。

```kotlin
### Dispatcher
@get:Synchronized
  @get:JvmName("executorService") val executorService: ExecutorService
    get() {
      if (executorServiceOrNull == null) {
        //用到了一个CachedThreadPool线程池，快速处理大量耗时较短的任务
        executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
            SynchronousQueue(), threadFactory("OkHttp Dispatcher", false))
      }
      return executorServiceOrNull!!
    }
```

`SynchronousQueue`同步队列，这个队列类似于一个接力棒，入队出队必须同时传递，因为CachedThreadPool线程创建无限制，不会有队列等待，所以使用SynchronousQueue。

分析好后，再看第二个方法getResponseWithInterceptorChain()`

```kotlin
### RealCall  
@Throws(IOException::class)
  fun getResponseWithInterceptorChain(): Response {
    // 创建拦截器数组
    val interceptors = mutableListOf<Interceptor>()
    //添加自定义拦截器
    interceptors += client.interceptors】
    //添加重试和重定向拦截器
    interceptors += RetryAndFollowUpInterceptor(client)
    //添加桥拦截器
    interceptors += BridgeInterceptor(client.cookieJar)
    //添加缓存拦截器
    interceptors += CacheInterceptor(client.cache)
    //添加连接池拦截器
    interceptors += ConnectInterceptor
    //添加网络拦截器
    if (!forWebSocket) {
      interceptors += client.networkInterceptors
    }
    //添加网络请求拦截器
    interceptors += CallServerInterceptor(forWebSocket)

    //创建拦截器链，所有拦截器的最终调用者
    val chain = RealInterceptorChain(interceptors, transmitter, null, 0, originalRequest, this,
        client.connectTimeoutMillis, client.readTimeoutMillis, client.writeTimeoutMillis)

    var calledNoMoreExchanges = false
    try {
      //启动拦截器链
      val response = chain.proceed(originalRequest)
      .....
      return response
    } 
    ......
  }
```

可以看到`addInterceptor(interceptor)`所设置的拦截器会在所有其他`Intercept`处理之前运行。后面就是默认的五个拦截器。

| 拦截器 | 作用 |
|  ----  | ----  |
| 应用拦截器 | 处理header头信息， |
| RetryAndFollowUpInterceptor | 负责出错重试，重定向 |
| BridgeInterceptor | 填充http请求协议中的head头信息 |
| CacheInterceptor | 缓存拦截器，如果命中缓存则不会发起网络请求。 |
| ConnectInterceptor | 连接池拦截器，Okhttp中的核心 |
| networkInterceptors | 自定义网络拦截器，用于监控网络传输数据 |
| CallServerInterceptor | 负责和网络的收发 |

拦截器待会分析，把流程先走完。最终会执行`chain.proceed(originalRequest)`,看下内部实现

```kotlin
  ### RealInterceptorChain
  @Throws(IOException::class)
  fun proceed(request: Request, transmitter: Transmitter, exchange: Exchange?): Response {
    if (index >= interceptors.size) throw AssertionError()

    //当前拦截器调用proceed的次数
    calls++

    //exchange 传输单个 HTTP 请求和响应对
    // 确认传入的请求正在调用,之前的网络拦截器对url或端口进行了修改，
    check(this.exchange == null || this.exchange.connection()!!.supportsUrl(request.url)) {
      "network interceptor ${interceptors[index - 1]} must retain the same host and port"
    }

    // 确认chain.proceed()是唯一调用,在connectInteceptor及其之后的拦截器最多只能调用一次proceed!!
    check(this.exchange == null || calls <= 1) {
      "network interceptor ${interceptors[index - 1]} must call proceed() exactly once"
    }

    // 创建下个拦截链处理
    val next = RealInterceptorChain(interceptors, transmitter, exchange,
        index + 1, request, call, connectTimeout, readTimeout, writeTimeout)
    
    //取出下标为index的拦截器，并调用其intercept方法，将新建的链传入。
    val interceptor = interceptors[index]

    //责任链设计，依次调用拦截器
    @Suppress("USELESS_ELVIS")
    val response = interceptor.intercept(next) ?: throw NullPointerException(
        "interceptor $interceptor returned null")

    // 保证在ConnectInterceptor及其之后的拦截器至少调用一次proceed!!
    check(exchange == null || index + 1 >= interceptors.size || next.calls == 1) {
      "network interceptor $interceptor must call proceed() exactly once"
    }

    check(response.body != null) { "interceptor $interceptor returned a response with no body" }

    return response
  }
```

责任链在不同拦截器对于proceed的调用次数有不同限制，`ConnectInterceptor`拦截器及其之后的拦截器能且只能调用一次。因为网络握手、连接、发送请求的工作发生在这些拦截器内，表示正式发出了一次网络请求。而在这之前的拦截器可以执行多次proceed。

其中主要实现了创建下一级责任链，然后取出当前拦截器，调用`intercept`方法并传入创建的责任链，经过拦截链一级一级的调用，最终执行到`CallServerInterceptor的`intercept`返回`Response`对象。

## 缓存拦截器

要了解缓存拦截器的实现原理，首先就要先知道Http缓存的知识点。Http缓存分为两类（强制缓存和对比缓存）。

[彻底弄懂HTTP缓存机制及原理](https://www.cnblogs.com/chenqf/p/6386163.html)

**强制缓存**

强制缓存是指网络请求响应header标识了Expires或Cache-Control带了max-age信息，而此时客户端计算缓存并未过期，则可以直接使用本地缓存内容，而不用真正的发起一次网络请求。

**对比缓存**

浏览器第一次请求数据时，服务器会将缓存标识与数据一起返回给客户端，客户端将二者备份至缓存数据库中。
再次请求数据时，客户端将备份的缓存标识发送给服务器，服务器根据缓存标识进行判断，判断成功后，返回304状态码，通知客户端比较成功，可以使用缓存数据。

在对比缓存中，缓存标识的传递需要重点理解下。

- Last-Madified: 服务器在响应请求时，告诉浏览器资源的最后修改时间。
- If-Modified-Since：再次请求服务器时，通过此字段通知服务器上次请求时，服务器返回的资源最后修改时间。
- Etag：（优先级高于Last-Modified / If-Modified-Since）服务器响应请求时，告诉浏览器当前资源在服务器的唯一标识。
- If-None-Match:再次请求时，通过此字段通知服务器客户端缓存数据的唯一标识。

对于强制缓存，服务器通知浏览器一个缓存时间，在缓存时间内，下次请求，直接用缓存，不在时间内，执行比较缓存策略。
对于比较缓存，将缓存信息中的Etag和Last-Modified通过请求发送给服务器，由服务器校验，返回304状态码时，浏览器直接使用缓存。

![image-20211113221256993](../../../../Library/Application Support/typora-user-images/image-20211113221256993.png)

在`CacheInterceptor`会调用`intercept`方法

```kotlin
###CacheInterceptor

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    //用requestd的url 从缓存中获取响应 
    val cacheCandidate = cache?.get(chain.request())

    val now = System.currentTimeMillis()

    //根据request 候选Response 获取缓存策略
    //缓存策略 将决定是否使用缓存
    //strategy.networkRequest为null，不使用网络；
    //strategy.cacheResponse为null，不使用缓存
    val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
    val networkRequest = strategy.networkRequest
    val cacheResponse = strategy.cacheResponse

    //处理是命中网络还是本地缓存
    //根据缓存策略更新统计指标：请求次数、网络请求次数、使用缓存次数
    cache?.trackResponse(strategy)

    if (cacheCandidate != null && cacheResponse == null) {
      // 有缓存 但不能用，关闭
      cacheCandidate.body?.closeQuietly()
    }

    // 网络和缓存都不能用  返回504
    if (networkRequest == null && cacheResponse == null) {
      return Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(HTTP_GATEWAY_TIMEOUT)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build()
    }

    // 如果不用网络 cacheResponse肯定就不为null，那么就使用缓存
    if (networkRequest == null) {
      return cacheResponse!!.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build()
    }

    //networkRequest 不为null要进行网络请求了，所以 拦截器链 就继续 往下处理了
    var networkResponse: Response? = null
    try {
      networkResponse = chain.proceed(networkRequest)
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        cacheCandidate.body?.closeQuietly()
      }
    }

    // 如果网络请求返回304 标识服务端资源没有修改 
    // 根据网络响应和缓存响应，更新缓存
    if (cacheResponse != null) {
      if (networkResponse?.code == HTTP_NOT_MODIFIED) {
        val response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers, networkResponse.headers))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build()

        networkResponse.body!!.close()

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache!!.trackConditionalCacheHit()
        cache.update(cacheResponse, response)
        return response
      } else {
        //如果不是304,说明服务端资源有更新 就关闭缓存body
        cacheResponse.body?.closeQuietly()
      }
    }

    //将网络数据和缓存传入
    val response = networkResponse!!.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build()

    if (cache != null) {
      //网络响应可缓存 （CacheStrategy.isCacheable对响应处理，根据response的code和response.cacheControl.noStore）
      if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
        // 写入缓存
        val cacheRequest = cache.put(response)
        return cacheWritingResponse(cacheRequest, response)
      }

      //OkHttp默认只会对get请求进行缓存，因为get请求的数据一般是比较持久的，而post一般是交互操作，没太大意义进行缓存
	  //不是get请求就移除缓存
      if (HttpMethod.invalidatesCache(networkRequest.method)) {
        try {
          cache.remove(networkRequest)
        } catch (_: IOException) {
          // The cache cannot be written.
        }
      }
    }

    return response
  }

```

用到了使用缓存策略`CacheStrategy`来确定是否使用缓存。

可以理解为：

1. 网络和缓存都不能用 ，返回504
2. 网络networkResponse为null, cacheResponse肯定就不为null，那么就使用缓存
3. networkResponse不为null，不管cacheResponse是否为null，直接去请求网络
4. cacheResponse 不为null，如果网络请求返回304 标识服务端资源没有修改
5. 如果不是304,说明服务端资源有更新，将网络数据和缓存传入
6. 如果网络响应可缓存，返回`cacheWritingResponse`
7. 最后一步，不是get请求就移除缓存

```kotlin
//根据request 候选Response 获取缓存策略
val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
```

再来看下`CacheStrategy`中处理了什么？

```kotlin
###CacheStrategy
//给定一个请求和缓存的响应，这将确定是使用网络、缓存还是两者兼而有之。选择缓存策略可能会向请求添加条件（如条件 GET 的“If-Modified-Since”标头）或向缓存响应添加警告（如果缓存数据可能过时）。
//初始化内部
  class Factory(
    private val nowMillis: Long,
    internal val request: Request,
    private val cacheResponse: Response?
  ) {
    //服务缓存响应的服务器时间
    private var servedDate: Date? = null
    private var servedDateString: String? = null

    //缓存响应的最后修改时间
    private var lastModified: Date? = null
    private var lastModifiedString: String? = null

    //缓存响应的到期时间
    private var expires: Date? = null

    //设置的指定  第一次发起缓存的Http请求时的时间戳
    private var sentRequestMillis = 0L

    //第一次收到缓存响应的时间戳
    private var receivedResponseMillis = 0L

    //缓存响应的Etag
    private var etag: String? = null

    //缓存响应的存活时间
    private var ageSeconds = -1

    /**
     * Returns true if computeFreshnessLifetime used a heuristic. If we used a heuristic to serve a
     * cached response older than 24 hours, we are required to attach a warning.
     */
    private fun isFreshnessLifetimeHeuristic(): Boolean {
      return cacheResponse!!.cacheControl.maxAgeSeconds == -1 && expires == null
    }

    init {
      if (cacheResponse != null) {
        //请求时间、响应时间、过期时长、修改时间、资源标记，都是从缓存响应中获取
        this.sentRequestMillis = cacheResponse.sentRequestAtMillis
        this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis
        val headers = cacheResponse.headers
        for (i in 0 until headers.size) {
          val fieldName = headers.name(i)
          val value = headers.value(i)
          when {
            fieldName.equals("Date", ignoreCase = true) -> {
              servedDate = value.toHttpDateOrNull()
              servedDateString = value
            }
            fieldName.equals("Expires", ignoreCase = true) -> {
              expires = value.toHttpDateOrNull()
            }
            fieldName.equals("Last-Modified", ignoreCase = true) -> {
              lastModified = value.toHttpDateOrNull()
              lastModifiedString = value
            }
            fieldName.equals("ETag", ignoreCase = true) -> {
              etag = value
            }
            fieldName.equals("Age", ignoreCase = true) -> {
              ageSeconds = value.toNonNegativeInt(-1)
            }
          }
        }
      }
    }
  .....  
  }
```

再看下它怎么返回`CacheStrategy`

```kotlin
### CacheStrategy
//使用 cacheResponse 返回满足请求的策略。
fun compute(): CacheStrategy {
      val candidate = computeCandidate()

      //禁止使用网络  缓存不足
      if (candidate.networkRequest != null && request.cacheControl.onlyIfCached) {
        return CacheStrategy(null, null)
      }

      return candidate
    }

    //返回假设请求可以使用网络的策略。
    private fun computeCandidate(): CacheStrategy {
      //没有缓存响应
      if (cacheResponse == null) {
        return CacheStrategy(request, null)
      }

      // https 但没有握手，网络请求
      if (request.isHttps && cacheResponse.handshake == null) {
        return CacheStrategy(request, null)
      }

      // 网络响应不可缓存
      if (!isCacheable(cacheResponse, request)) {
        return CacheStrategy(request, null)
      }

      //缓存控制指令
      //不使用缓存
      //添加了头部参数If-Modified-Since  If-None-Match 
      val requestCaching = request.cacheControl
      if (requestCaching.noCache || hasConditions(request)) {
        return CacheStrategy(request, null)
      }

      val responseCaching = cacheResponse.cacheControl

      //缓存的年龄
      val ageMillis = cacheResponseAge()
      //缓存的有效期
      var freshMillis = computeFreshnessLifetime()

      
      if (requestCaching.maxAgeSeconds != -1) {
        freshMillis = minOf(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds.toLong()))
      }

      var minFreshMillis: Long = 0
      if (requestCaching.minFreshSeconds != -1) {
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds.toLong())
      }

      var maxStaleMillis: Long = 0
      if (!responseCaching.mustRevalidate && requestCaching.maxStaleSeconds != -1) {
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds.toLong())
      }

      // 如果响应头没有要求忽略本地缓存 且 整合后的缓存年龄 小于 整合后的过期时间，那么缓存就可以用
      if (!responseCaching.noCache && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        val builder = cacheResponse.newBuilder()
        if (ageMillis + minFreshMillis >= freshMillis) {
          builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"")
        }
        val oneDayMillis = 24 * 60 * 60 * 1000L
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
          builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"")
        }
        //缓存没过期返回
        return CacheStrategy(null, builder.build())
      }

      //缓存过期了
      // 找缓存里的Etag、lastModified、servedDate
      // 查找要添加到请求中的条件。如果满足条件，则不会发送响应正文。
      val conditionName: String
      val conditionValue: String?
      when {
        etag != null -> {
          conditionName = "If-None-Match"
          conditionValue = etag
        }

        lastModified != null -> {
          conditionName = "If-Modified-Since"
          conditionValue = lastModifiedString
        }

        servedDate != null -> {
          conditionName = "If-Modified-Since"
          conditionValue = servedDateString
        }

        //都没有就返回request
        else -> return CacheStrategy(request, null) // No condition! Make a regular request.
      }

    //有这些参数直接加到头部文件中，进行条件请求
      val conditionalRequestHeaders = request.headers.newBuilder()
      conditionalRequestHeaders.addLenient(conditionName, conditionValue!!)

      val conditionalRequest = request.newBuilder()
          .headers(conditionalRequestHeaders.build())
          .build()
      //conditionalRequest表示 条件网络请求： 有缓存但过期了，去请求网络 询问服务端，还能不能用。能用侧返回304，不能则正常执行网路请求。
      return CacheStrategy(conditionalRequest, cacheResponse)
    }
```

`CacheStrategy`处理了缓存请求过程：

1. 没有缓存、是https但没有握手、要求不可缓存、忽略缓存或手动配置缓存过期，都是 直接进行 网络请求。

2. 以上都不满足时，如果缓存没过期，那么就是用缓存（可能要添加警告）。

3. 如果缓存过期了，但响应头有Etag，Last-Modified，Date，就添加这些header 进行**条件网络请求**。

4. 如果缓存过期了，且响应头**没有**设置Etag，Last-Modified，Date，就进行网络请求。

还有个问题，缓存响应保存在哪里？

不废话了直接看，放在了`DiskLruCache`当中。

```kotlin
### CacheInterceptor
val cacheCandidate = cache?.get(chain.request())

### Cahce
  internal fun get(request: Request): Response? {
    val key = key(request.url)
    val snapshot: DiskLruCache.Snapshot = try {
      cache[key] ?: return null
    } catch (_: IOException) {
      return null // Give up because the cache cannot be read.
    }

   ......
    
    return response
  }
```

现在应该清楚`Okhttp`缓存的实现了吧。

## 连接池拦截器

终于到核心的拦截器了~

先来看看`ConnectInterceptor`做了什么

```kotlin
### ConnectInterceptor

object ConnectInterceptor : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val request = realChain.request()
    val transmitter = realChain.transmitter()

    // 需要网络来满足这个要求。用于验证是否满足不是 GET 这个条件
    val doExtensiveHealthChecks = request.method != "GET"
    //Exchange真正的IO操作： 写入请求 读取响应
    //Transmitter 是发射器， 是把其请求从应用端发送到网络层，它持有请求的连接、请求响应和流，一个请求对应一个Transmitter实例，一个数据流
    val exchange = transmitter.newExchange(chain, doExtensiveHealthChecks)

    return realChain.proceed(request, transmitter, exchange)
  }
}
```

下面来看下`transmitter.newExchange`做了什么操作

```kotlin
//返回一个新的交换来携带新的请求和响应  
internal fun newExchange(chain: Interceptor.Chain, doExtensiveHealthChecks: Boolean): Exchange {
    synchronized(connectionPool) {
      check(!noMoreExchanges) { "released" }
      check(exchange == null) {
        "cannot make a new request because the previous response is still open: " +
            "please call response.close()"
      }
    }

  //exchangeFinde负责真正的IO操作—写请求、读响应
    val codec = exchangeFinder!!.find(client, chain, doExtensiveHealthChecks)
  //管理IO操作，可以理解为数据流，是ExchangeCodec的包装，增加了事件回调，
  //一个请求对应一个Exchange实例。传给下个拦截器CallServerInterceptor使用
    val result = Exchange(this, call, eventListener, exchangeFinder!!, codec)

    synchronized(connectionPool) {
      this.exchange = result
      this.exchangeRequestDone = false
      this.exchangeResponseDone = false
      return result
    }
  }
```

### ExchangeFinder 

`exchangeFinder`是主要负责真正的IO操作 写请求、读响应，本质是为请求寻找一个TCP连接。

```kotlin
###  ExchangeFinder 
//查找连接并在它健康时返回它，如果它不健康，则重复该过程，直到找到健康的连接
@Throws(IOException::class)
  private fun findHealthyConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean,
    doExtensiveHealthChecks: Boolean
  ): RealConnection {
    while (true) {
      //查找连接
      val candidate = findConnection(
          connectTimeout = connectTimeout,
          readTimeout = readTimeout,
          writeTimeout = writeTimeout,
          pingIntervalMillis = pingIntervalMillis,
          connectionRetryEnabled = connectionRetryEnabled
      )

      // 如果这是一个全新的连接，可以跳过检查直接返回
      synchronized(connectionPool) {
        if (candidate.successCount == 0) {
          return candidate
        }
      }

      // 不健康的继续 查找
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        //标记不可用
        candidate.noNewExchanges()
        continue
      }

      return candidate
    }
  }
```

循环寻找连接。如果是不健康的连接，标记未不可用然后继续查找。

```kotlin
### ExchangeFinder  
//返回一个连接来承载一个新的流。
//如果存在，则优先选择现有连接，然后是池，最后建立新连接。
@Throws(IOException::class)
  private fun findConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean
  ): RealConnection {
    var foundPooledConnection = false
    var result: RealConnection? = null
    var selectedRoute: Route? = null
    var releasedConnection: RealConnection?
    val toClose: Socket?
    synchronized(connectionPool) {
      //连接请求被取消了
      if (transmitter.isCanceled) throw IOException("Canceled")
      hasStreamFailure = false // This is a fresh attempt.

      //给分配的真正的连接
      releasedConnection = transmitter.connection
      //有已分配的连接，但是标记了不可用，就尝试释放调
      toClose = if (transmitter.connection != null && transmitter.connection!!.noNewExchanges) {
        transmitter.releaseConnectionNoEvents()
      } else {
        null
      }

      if (transmitter.connection != null) {
        // 有一个分配好的连接
        result = transmitter.connection
        releasedConnection = null
      }

      if (result == null) {
        // 这里会尝试从连接池中进行获取（**********）
        if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, null, false)) {
          foundPooledConnection = true
          result = transmitter.connection
        } else if (nextRouteToTry != null) {
          //有可尝试的路由
          selectedRoute = nextRouteToTry
          nextRouteToTry = null
        } else if (retryCurrentRoute()) {
          //如果应该重试用于当前连接的路由，则返回 true，即使连接本身不健康。
          selectedRoute = transmitter.connection!!.route()
        }
      }
    }
   // 关闭待关闭的连接
    toClose?.closeQuietly()

    if (releasedConnection != null) {
      //回调连接释放事件
      eventListener.connectionReleased(call, releasedConnection!!)
    }
    if (foundPooledConnection) {
      //回调获取连接事件
      eventListener.connectionAcquired(call, result!!)
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result!!
    }

    // 做路由信息获取，阻塞操作
    var newRouteSelection = false
    if (selectedRoute == null && (routeSelection == null || !routeSelection!!.hasNext())) {
      newRouteSelection = true
      routeSelection = routeSelector.next()
    }

    var routes: List<Route>? = null
    synchronized(connectionPool) {
      if (transmitter.isCanceled) throw IOException("Canceled")

      if (newRouteSelection) {
        // 现在我们有了一组 IP 地址，再次尝试从池中获取连接。由于连接合并，这可能匹配。
        routes = routeSelection!!.routes
        if (connectionPool.transmitterAcquirePooledConnection(
                address, transmitter, routes, false)) {
          foundPooledConnection = true
          result = transmitter.connection
        }
      }

      //上面连接池没找到， 就会新建连接
      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          //路由增加
          selectedRoute = routeSelection!!.next()
        }

        // 创建连接并立即将其分配给此分配。这使得异步 cancel() 可以中断我们即将进行的握手
        result = RealConnection(connectionPool, selectedRoute!!)
        connectingConnection = result
      }
    }

    // 第二次找到了可用的连接
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result!!)
      return result!!
    }

    //如果第二次还没找到
    //就新建连接，进行TCP + TLS握手，与服务端建立连接
    result!!.connect(
        connectTimeout,
        readTimeout,
        writeTimeout,
        pingIntervalMillis,
        connectionRetryEnabled,
        call,
        eventListener
    )
    //根据Ip地址  从失败黑名单中删除
    connectionPool.routeDatabase.connected(result!!.route())

    var socket: Socket? = null
    synchronized(connectionPool) {
      connectingConnection = null
      // 连接合并的最后一次尝试，只有当我们尝试多个并发连接到同一主机时才会发生。（确保http2多路复用）
      if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, routes, true)) {
        // 如果获取到，就关闭我们创建的连接，返回获取的连接
        result!!.noNewExchanges = true
        socket = result!!.socket()
        result = transmitter.connection

        // 连接成功的路由可以作为 下次尝试的路由.
        nextRouteToTry = selectedRoute
      } else {
        connectionPool.put(result!!)
        transmitter.acquireConnectionNoEvents(result!!)
      }
    }
    socket?.closeQuietly()

    eventListener.connectionAcquired(call, result!!)
    return result!!
  }
```

寻找连接,经历了三次从连接池中寻找。

1. 如果有连接，直接用
2. 没有已分配的连接到连接池中找，`connectionPool.transmitterAcquirePooledConnection`有的也直接返回
3. 如果第一次没成功就获取路由信息`routeSelector.next()`，有了Ip地址后再次从连接池中获取，可能会因为连接合并而匹配
4. 如果第二次没成功，就新建连接,进行TCP+TLS握手，与服务器建立连接
5. 最后一次尝试获取，这里是确保Http2.0连接的多路复用，获取到就关闭刚才创建的链接
6. 如果第三次没成功，就把刚创建的新建连接存入连接池

### ConnectionPool

相同Address的http请求共享一个连接，`ConnectioonPool`就是实现了连接的复用。

```kotlin
 ###  ConnectionPool
//最大空闲连接数5，最大空闲时间5分钟
class ConnectionPool(
  maxIdleConnections: Int,
  keepAliveDuration: Long,
  timeUnit: TimeUnit
) {
  internal val delegate = RealConnectionPool(maxIdleConnections, keepAliveDuration, timeUnit)

  constructor() : this(5, 5, TimeUnit.MINUTES)

  /** Returns the number of idle connections in the pool. */
  fun idleConnectionCount(): Int = delegate.idleConnectionCount()

  /** Returns total number of connections in the pool. */
  fun connectionCount(): Int = delegate.connectionCount()

  /** Close and remove all idle connections in the pool. */
  fun evictAll() {
    delegate.evictAll()
  }
}
```

目前，该池最多可容纳 5 个空闲连接，这些连接将在 5 分钟不活动后被逐出。可以看到真正实现它的类是`RealConnectionPool`。这样就知道了它是okhttp内部真实管理连接的地方。

先看下连接放进去连接池做了什么操作：

```kotlin
###  RealConnectionPool
fun put(connection: RealConnection) {
    assert(Thread.holdsLock(this))
  //可以看到，放入连接池的同时，也会开启一个清理线程
    if (!cleanupRunning) {
      cleanupRunning = true
      executor.execute(cleanupRunnable)
    }
    connections.add(connection)
  }

 private val cleanupRunnable = object : Runnable {
    override fun run() {
      //循环清理
      while (true) {
        val waitNanos = cleanup(System.nanoTime())
        if (waitNanos == -1L) return
        try {
          //下一次清理之前等待
          this@RealConnectionPool.lockAndWaitNanos(waitNanos)
        } catch (ie: InterruptedException) {
          // Will cause the thread to exit unless other connections are created!
          evictAll()
        }
      }
    }
  }
```

这里为什么要开启一个清理线程，因为连接池有最大空闲连接数和最大空闲时间的限制，所以不满足的时候就要进行清理。接下来再看下`cleanup()`方法

```kotlin
###  RealConnectionPool
//对此池执行维护，如果连接超过保持活动限制或空闲连接限制，则驱逐空闲时间最长的连接。
//返回以nanos 为单位的睡眠持续时间，直到下一次预定调用此方法。
//如果不需要进一步清理，则返回 -1。
fun cleanup(now: Long): Long {
    var inUseConnectionCount = 0  //正在使用连接数
    var idleConnectionCount = 0		// 空闲连接数
    var longestIdleConnection: RealConnection? = null  //空闲时间最长的连接
    var longestIdleDurationNs = Long.MIN_VALUE  //最长的空闲时间

    synchronized(this) {
      // 遍历连接 找到待清理的链接，找到下一次要清理的连接数
      for (connection in connections) {
        // 如果连接正在被使用， 则继续搜索
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++
          continue
        }

        idleConnectionCount++

        // If the connection is ready to be evicted, we're done.
        val idleDurationNs = now - connection.idleAtNanos
         //赋值时间最长的空闲时间连接
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs
          longestIdleConnection = connection
        }
      }

      when {
        //如果最长空闲时间大于5分钟或者 空闲数大于5，就会移除它
        longestIdleDurationNs >= this.keepAliveDurationNs
            || idleConnectionCount > this.maxIdleConnections -> {
          
          connections.remove(longestIdleConnection)
        }
        //空闲连接存在，就返回 还剩多久时间来清理
        idleConnectionCount > 0 -> {
          // A connection will be ready to evict soon.
          return keepAliveDurationNs - longestIdleDurationNs
        }
        //在用的连接存在
        inUseConnectionCount > 0 -> {
          // 所有连接都在使用中。它至少是保持活动持续时间，直到我们再次运行。
          return keepAliveDurationNs
        }
        else -> {
          // 没有连接，不清理
          cleanupRunning = false
          return -1
        }
      }
    }

    longestIdleConnection!!.socket().closeQuietly()

    // 关闭移除红藕 立即清理
    return 0
  }
```

可以总结下

1. 如果最长的空闲时间大于5分钟或空闲连接大于5，就移除关闭这个最长的空闲时间。
2. 如果有空闲连接，就返回到5分钟的剩余时间
3. 没有空闲连接就等5分钟再次清理
4. 没有连接不清理

```kotlin
###  RealConnectionPool
// 判断是否有在使用的连接
private fun pruneAndGetAllocationCount(connection: RealConnection, now: Long): Int {
    val references = connection.transmitters
    var i = 0
    while (i < references.size) {
      val reference = references[i]

      if (reference.get() != null) {
        i++
        continue
      }

      // 如果TransmitterReference有泄漏的话  就进行移除
      val transmitterRef = reference as TransmitterReference
      val message = "A connection to ${connection.route().address.url} was leaked. " +
          "Did you forget to close a response body?"
      Platform.get().logCloseableLeak(message, transmitterRef.callStackTrace)

      references.removeAt(i)
      connection.noNewExchanges = true

      // 连接队列失控的  就重新设置空闲时间未5分钟
      if (references.isEmpty()) {
        connection.idleAtNanos = now - keepAliveDurationNs
        return 0
      }
    }

  //如果不为0，仍然在使用欧冠
    return references.size
  }
```

`transmitters size`大于1即表示多个请求复用此连接。

将连接从连接池中取出来怎么操做？

```kotlin
### RealConnectionPool
//从连接池中获取对应的address的连接。 如果routes不为空，可能会因为合并而获取Http/2的连接
fun transmitterAcquirePooledConnection(
    address: Address,
    transmitter: Transmitter,
    routes: List<Route>?,
    requireMultiplexed: Boolean
  ): Boolean {
    assert(Thread.holdsLock(this))
    for (connection in connections) {
      //是不是多路复用
      if (requireMultiplexed && !connection.isMultiplexed) continue
      if (!connection.isEligible(address, routes)) continue
      transmitter.acquireConnectionNoEvents(connection)
      return true
    }
    return false
  }



```

这里面有个方法要关注下  isEligible

```kotlin
### RealConnection
//用于判断连接是否可以指向address的数据流
  internal fun isEligible(address: Address, routes: List<Route>?): Boolean {
    // /连接不再接受新的数据流，false
    if (transmitters.size >= allocationLimit || noNewExchanges) return false

    // 匹配address中非host的部分
    if (!this.route.address.equalsNonHost(address)) return false

    // 匹配address的host，到这里也匹配的话，就return true
    if (address.url.host == this.route().address.url.host) {
      return true // This connection is a perfect match.
    }

    // At this point we don't have a hostname match. But we still be able to carry the request if
    // our connection coalescing requirements are met. See also:
    // https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding
    // https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/

    //  1. 连接须是 HTTP/2.
    if (http2Connection == null) return false

    // 2. IP 地址匹配
    if (routes == null || !routeMatchesAny(routes)) return false

    // 3. 证书匹配
    if (address.hostnameVerifier !== OkHostnameVerifier) return false
    if (!supportsUrl(address.url)) return false

    // 4. 证书 pinning 匹配.
    try {
      address.certificatePinner!!.check(address.url.host, handshake()!!.peerCertificates)
    } catch (_: SSLPeerUnverifiedException) {
      return false
    }

    return true // The caller's address can be carried by this connection.
  }
```

取的过程就是 遍历连接池，进行地址等一系列匹配。

到这里，okHttp大部分关键的都看完了。

## 带着问题看源码

1. Okhttp源码流程,线程池
2. Okhttp拦截器,addInterceptor 和 addNetworkdInterceptor区别
3. Okhttp责任链模式
4. Okhttp缓存怎么处理
5. Okhttp连接池和Tcp复用

参考：

[听说你熟悉OkHttp原理？](https://juejin.cn/post/6844904087788453896#heading-10)

[拦截器详解1：重试重定向、桥、缓存](https://juejin.cn/post/6844904176183410702#heading-2)

[OkHttp 原理 8 连问](https://juejin.cn/post/7020027832977850381#heading-5)



