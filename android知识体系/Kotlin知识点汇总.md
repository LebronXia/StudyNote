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