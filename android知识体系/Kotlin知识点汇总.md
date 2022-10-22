# Kotlin

- kotlin扩展函数底层原理
- Kotlin inner 关键字

其实 Kotlin 的扩展函数并没有修改原有的 String 类，而是在自己的类中生成了一个静态的方法,当我们在 Kotlin 中调用扩展函数时,编译器将会调用自动生成的函数并且把当前的对象（String）传入。

# 委托

- by lazy是如何实现延迟加载的

Lazy`是一个接口，其中包括一个值`value`，即我们在`by lazy{}`中的返回值和一个用来判断当前值是否已经被初始化过的方法`isInitialized()

```kotlin
private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    // final field is required to enable safe publication of constructed instance
    private val lock = lock ?: this

    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }

            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }

    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE
}
```



[by lazy是如何实现延迟加载的](https://www.jianshu.com/p/68962ad7986f)

[委托模式（Delegate）和委托属性（Delegate Properties）](https://www.jianshu.com/p/f54ff17425b2)

# 协变和逆变



### 协变应用

- 在 Java 中用通配符 `? extends` 表示协变
- 在 Kotlin 中关键字 `out` 表示协变

### 逆变应用

- 在 Java 中使用通配符 `? super` 表示逆变
- 在 Kotlin 中使用关键字 `in` 表示逆变

# 协程

作用域

线程和协程的区别

# Flow

flowOnLifecycle

[Kotlin协程之Flow使用](https://juejin.cn/post/7034381227025465375#heading-1) Flow的操作符介绍

[官方推荐 Flow 取代 LiveData,有必要吗？](https://juejin.cn/post/6986265488275800072)



