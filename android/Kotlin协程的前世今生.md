协程中有几个概念: `CoroutineScope`, `Job`,`CoroutineContext`

###  CoroutineScope 协程作用域

异步作用域函数

创建作用域有两种创建方式，常用的`launch`只是CoroutineScope的扩展函数

```kotlin
//没有返回结果
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job
//有返回结果
public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T>
```

其中有个概念结构化并发，无论你再任何页面启动协程，并控制其生命周期，都应该创建 `CoroutineScop`。 在顶层协程代码块中可以创建子Coroutine执行，当所有的子协程执行完毕并且返回的时候，代码块才执行完毕。

在Android中引入KTX类，已经未特定的生命周期提供了`CoroutineScope`, 如`viewModelScope`和`lifecycleScope`。

```kotlin
val job = GlobalScope.launch { // 启动一个新协程并保持对这个作业的引用
    delay(1000L)
    println("World!")
}
println("Hello,")
job.join() // 等待直到子协程执行结束

fun main() = runBlocking<Unit> { // 开始执行主协程
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(1000L)
        println("World!")
    }
    println("Hello,") // 主协程在这里会立即执行
    delay(2000L)      // 延迟 2 秒来保证 JVM 存活
}
```

`runBlocking`和`coroutineScope`类似，都是等待协程体以及子协程结束。区别在于`runBlocking`方法会阻塞当前线程等待（常规函数），而`coroutineScope`只是挂起，会释放底层线程用于其他（挂起函数）。

### CoroutineContext 协程上下文

协程上下文是各种不同元素的集合，其中主要元素是协程中的Job。

而定义协程上下文的元素，有下面几个部分：

- **Job**，管理协程的生命周期
- **CoroutineDispatcher**，分发任务到合适的线程
- **CoroutineName**，协程的名称，用于调试
- **CoroutineExceptionHandler**，处理未捕获的异常

> 其中一些元素具有默认值：`CoroutineDispatcher` 的默认值是 `Dispatchers.Default` ，`CoroutineName`的默认值是 `coroutine`

```kotlin
val scope = CoroutineScope(Job() + Dispatchers.Main)
 
val job = scope.launch {
    // 这里的新协程的父亲是 scope
    val result = async {
        // 这里的新协程的父亲是上面的 scope.launch 启动的协程
    }.await()
}
```

**父协程的职责**：一个父协程总是等待所有的子协程执行结束。

在协程的继承结构中，每一个协程都会有一个父协程，子协程创建的时候继承的 `CoroutineContext` 是父亲的 CoroutineContext，**传递到协程构建器的参数优先于继承上下文的参数**。

![image-20210818221837988](../../../../Library/Application Support/typora-user-images/image-20210818221837988.png)

最终的`CoroutineContext` 的协程调度器是 `Dispatchers.IO`，因为它被协程构建器中的参数覆盖了。

#### Job

job代表了一个协程。可以对协程进行`join()`和`cancel()`，管理协程的生命周期

- Join（）:等待执行子线程执行结束，阻塞当前线程
- canenl（）：取消协程的执行
- cancelAndJoin（）：它合并了对 [cancel](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html) 以及 [join](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html) 的调用

> 所有 `kotlinx.coroutines` 中的挂起函数都是 *可被取消的* 。它们检查协程的取消， 并在取消时抛出 `CancellationException`，如果协程正在执行计算任务，并且没有检查取消的话，那么它是不能被取消的。

#### 组合上下文中的元素

如果在一个协程中定义多个元素，可以用`+`操作符来实现。由于 `CoroutineContext` 包含一系列元素，当创建新的 `CoroutineContext` 时，“+” 右侧的元素将会覆盖左侧的元素。

```kotlin
//我们可以显式指定一个调度器来启动协程并且同时显式指定一个命名：
launch(Dispatchers.Default + CoroutineName("test")) {
    println("I'm working in thread ${Thread.currentThread().name}")
}

//打印结果：I'm working in thread DefaultDispatcher-worker-1 @test#2
```

```kotlin
//SupervisorJob ，它会改变协程作用域的异常处理
val a = CoroutineScope(SupervisorJob() + coroutineContext).launch(handler) {
  delay(1000)
  System.err.println("(Main.kt:51)    ${Thread.currentThread()}")
}
```

#### 调度器

用来指定协程代码块在哪个线程中执行。kotlin提供了几个默认的协程调度器，分别是`Default`、`Main`、`Unconfined`

```
Unconfined            : I'm working in thread main
Default               : I'm working in thread DefaultDispatcher-worker-1
newSingleThreadContext: I'm working in thread MyOwnThread
main runBlocking      : I'm working in thread main
```

#### 协程取消和超时

协程体如果已经执行实际上属于不可取消的, 在协程体中通过检查`job.isActive`或者`ensureActive`判断协程是否处于活跃中，通过取消函数的参数指定异常CancellationException可以自定义异常对象

```kotlin
fun Job.ensureActive(): Unit {
    if (!isActive) {
         throw getCancellationException()
    }
}
```

```kotlin
while (i < 5) {
    ensureActive()
    …
}
```

**使用`yield()`取消协程**

`yield` 会进行的第一个工作就是检查任务是否完成，如果 Job 已经完成的话，就会抛出 `CancellationException` 来结束协程。`yield` 应该在定时检查中最先被调用，就像前面提到的 `ensureActive` 一样。

`Deferred` 是一种 `Job`，它也是可以被取消的。对已经被取消的 deferred 调动 `await` 方法会抛出 `JobCancellationException`。

```kotlin
val deferred = async { … }
 
deferred.cancel()
val result = deferred.await() // throws JobCancellationException!
```

> 协程的取消需要代码配合实现，所以确保你在代码中检测了取消，以避免额外的无用工作。

**运行不能取消的代码块**

当你需要挂起一个被取消的协程，你可以将相应的代码包装在 `withContext(NonCancellable) {……}` 中，并使用 函数以`withContext`及`NonCancellable`上下文。

```kotlin
val job = launch {
    try {
        repeat(1000) { i ->
            println("job: I'm sleeping $i ...")
            delay(500L)
        }
    } finally {
        withContext(NonCancellable) {
            println("job: I'm running finally")
            delay(1000L)
            println("job: And I've just delayed for 1 sec because I'm non-cancellable")
        }
    }
}
delay(1300L) // 延迟一段时间
println("main: I'm tired of waiting!")
job.cancelAndJoin() // 取消该作业并等待它结束
println("main: Now I can quit.")


job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm running finally
job: And I've just delayed for 1 sec because I'm non-cancellable
main: Now I can quit.
```

**超时**

在实践中绝大多数取消一个协程的理由是它有可能超时。

```kotlin
//withContext
 //引用并启动了一个单独的协程在延迟后取消追踪 会抛出TimeoutCancellationException
fun main() = runBlocking {
    try {
        withTimeout(2000L) {
            repeat(100_000) {
                println("launch_____$it")
                delay(1000L)
            }
        }
    } catch (error: TimeoutCancellationException) {
        println("error:$error")
    }
    println("Game is over")
}
```

```kotlin
//withTimeoutOrNull 通过返回 null 来进行超时操作，从而替代抛出一个异常：
fun main() = runBlocking {
    try {
        val result = withTimeoutOrNull(2000L) {
            repeat(100_000) {
                println("launch_____$it")
                delay(1000L)
            }
            "Done"
        }
        println("result____$result")
    } catch (error: TimeoutCancellationException) {
        println("error:$error")
    }
    println("Game is over")
}
```

#### 协程异常

用`supervisorjob`,子线程的失败不会影响其他的子协程，此外，`SupervisorJob` 也不会传播异常，而是让子协程自己处理。

```kotlin
val scope = CoroutineScope(SupervisorJob())
scope.launch {
    // Child 1
}
scope.launch {
    // Child 2
}
```

```kotlin
val scope = CoroutineScope(Job())
scope.launch {
    supervisorScope {
        launch {
            // Child 1
        }
        launch {
            // Child 2
        }
    }
}
```

这两种情况下，`child#1` 失败了，`scope` 和 `child#2` 都不会被取消。

> `SupervisorJob` 仅在属于下面两种作用域时才起作用：使用 **supervisorScope** 或者 **CoroutineScope(SupervisorJob())** 创建的作用域。

如果`SupervisorJob` 是父协程通过 `scope.launch` 创建的，`SupervisorJob` 是不会发挥任何作用。

在我们处理异常的时候，通常是调用`try/catch`捕获异常，针对下面情况：

```kotlin
supervisorScope {
    val deferred = async {
        codeThatCanThrowExceptions()
    }
    try {
        deferred.await()
    } catch(e: Exception) {
        // SupervisorJob 让协程自己处理异常，可以捕获到异常
    }
}

coroutineScope {
    try {
        val deferred = async {
            codeThatCanThrowExceptions()
        }
        deferred.await()
    } catch(e: Exception) {
        // 由其他协程创建的协程如果发生了异常，也将会自动传播到父协程，无论你的协程构建器是什么
        //所以没有捕获到
    }
}

```

**CoroutineExceptionHandler**

协程异常处理器 CoroutineExceptionHandler 是 `CoroutineContext` 中的一个可选元素，它可以帮助你 **处理未捕获异常** 。

```kotlin
val handler = CoroutineExceptionHandler {
    context, exception -> println("Caught $exception")
}

//handel处理要放到父协程上才能捕获异常
val scope = CoroutineScope(Job())
scope.launch(handler) {
    launch {
        throw Exception("Failed coroutine")
    }
}

//supervisorScope 由子协程处理异常
supervisorScope {
        val child = launch(handler) {
            println("The child throws an exception")
            throw AssertionError()
        }
        println("The scope is completing")
    }
```

捕获条件：

- 是被可以自动抛异常的协程抛出的（`launch`，而不是 `async`）
- 在 CoroutineScope 或者根协程的协程上下文中（`CoroutineScope` 的直接子协程或者 `supervisorScope`）

 **参考**

- [Kotlin官网](https://www.kotlincn.net/docs/reference/coroutines/coroutines-guide.html)
- [Coroutines : First things first](https://blog.csdn.net/sunluyao_/article/details/105872818?spm=1001.2014.3001.5501)
- [最全面的Kotlin协程](https://juejin.cn/post/6844904037586829320#heading-17)



