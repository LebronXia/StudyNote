委托模式是实现继承的一个很好的替代方式，也是Kotlin语言的一种特性，可以很优雅的实现委托模式。在开发过程中也少不了使用它，但常常都会被低估。所以今天就让它得到重视，去充分的掌握kotlin委托特性以及原理。

## 一、委托类

我们先完成一个委托类，常常用于实现类的`委托模式`，它的关键是通过`by`关键字：

```kotlin
interface Base{
  fun print()
}

class BaseImpl(val x: Int): Base{
  override fun print() { print(x) }
}

class Derived(b: Base): Base by b

fun main(){
  val b = BaseImpl(10)
  Deriived(b).print()
}

//最后输出了10
```

在这个委托模式中Derived相当于是个包装，虽然也实现了base，但并不关心它怎么实现，通过by这个关键字，将接口的实现委托给了它的参数db。

相当于Java代码的结构：

```java
class Derived implements Base{
  Base b;
  public Derived(Base b){ this.b = b}
}
```

## 二、 委托属性

前面讲到Kotlin委托类是委托的是接口方法，委托属性委托的则是吧属性的getter和setter方法。kotlin支持的委托属性语法：

```kotlin
class Example {
    var prop: String by Delegate()
}
```

属性对应的`get()`和`set()`会被委托给它的`getValue`和`setValue`方法。当然属性的委托不是随便写的，它必须要提供一个`getValue`函数，如果是var属性的则要提供`setValue`属性。先来看个官方提供的委托属性`Delegate`：

```kotlin
class Delegate {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "$thisRef, thank you for delegating '${property.name}' to me!"
    }
 
    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("$value has been assigned to '${property.name}' in $thisRef.")
    }
}
```

我们可以看到对于var修饰的属性，必须要有`getValue`和`setValue`方法，同时这两个方法必须有`operator`关键字的修饰。

再来看第一个参数`thisRef`,它的类型是要这个属性所有者的类型，或者是它的父类。当我们不确定属性会属于哪个类，就可以将thisRef的类型定义为`Any？`了。

接着看另一个参数`property`，它的类型是必须要`KProperty<*>`或其超类型，它的值则是前面的字段的名字`prop`。

最后一个参数，它的类型必须是**委托属性**的类型，或者是它的父类。也就是说例子中的 `value: String` 也可以换成 `value: Any`。

我们来测试下到底是不是这样的：

```kotlin
fun main() {
    println(Test().prop)
    Test().prop = "Hello, World"
}
```

则会看到输出：

```javascript
Example@5197848c, thank you for delegating 'prop' to me!
Hello, World has been assigned to 'prop' in Example@17f052a3.
```

### 2.1 自定义委托

在知道了委托属性怎么写之后，也可以根据需求来实现自己的属性委托。但是每次写都要写那么多模板代码，也是很麻烦的，所以官方也提供了接口类给我们快速实现：

```kotlin
interface ReadOnlyProperty<in R, out T> {
    operator fun getValue(thisRef: R, property: KProperty<*>): T
}

interface ReadWriteProperty<in R, T> {
    operator fun getValue(thisRef: R, property: KProperty<*>): T
    operator fun setValue(thisRef: R, property: KProperty<*>, value: T)
}
```

现在被委托的类只要实现这个接口的其中一个就可以了。对于 val 变量使用 `ReadOnlyProperty`，而 var 变量实现`ReadWriteProperty`。我们现在就用`ReadWriteProperty `接口来实现一个自定义委托：

```kotlin
class Owner {
  var text: String by StringDelegate()
}


class StringDelegate(private var s: String = "Hello"): ReadWriteProperty<Owner, String> {
    override operator fun getValue(thisRef: Owner, property: KProperty<*>): String {
        return s
    }
    override operator fun setValue(thisRef: Owner, property: KProperty<*>, value: String) {
        s = value
    }
}
```

## 三、委托进阶

### 3.1 懒加载委托

懒加载委托，也就是我们再对一些资源进行操作的时候，希望它在被访问的时候采取触发，避免不必要的消耗。官方已经帮我们提供了一个`lazy()`方法来快速创建懒加载委托：

```kotlin
val lazyData: String by lazy {
    request()
}

fun request(): String {
    println("执行网络请求")
    return "网络数据"
}

fun main() {
    println("开始")
    println(lazyData)
    println(lazyData)
}

//结果：
开始
执行网络请求
网络数据
网络数据
```

可以看到只有第一次调用，才会执行`lambda`表达式里的逻辑，后面再调用只会返回`lambda`表达式的最终结果。

**那么懒加载委托又是怎么实现的呢？**现在来看你下它的源代码：

```kotlin
public actual fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)

public actual fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
        LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
        LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
    }
```

在这个里面`lazy()`方法会接收一个`LazyThreadSafetyMod`类型的参数，如果不传这个参数的话，就会默认使用`SynchronizedLazyImpl`方式。看解释就可以知道它是用来多线程同步的，而另外两个则不是多线程安全的。

- `LazyThreadSafetyMode.PUBLICATION`：初始化方法可以被多次调用，但是值只是第一次返回时的返回值，也就是只有第一次的返回值可以赋值给初始化的值。

- `LazyThreadSafetyMode. NONE`：如果初始化将总是发生在与属性使用位于相同的线程，这种情况下可以使用，但它

  没有同步锁。

我们现在主要来看下`SynchronizedLazyImpl`里面做了什么事情：

```kotlin
private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    // final field is required to enable safe publication of constructed instance
    private val lock = lock ?: this

    override val value: T
        get() {
            val _v1 = _value
            //判断是否已经初始化过，如果初始化过直接返回，不在调用高级函数内部逻辑
          	//如果这两个值不相同，就说明当前的值已经被加载过了，直接返回
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }

            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                  //调用高级函数获取其返回值
                    val typedValue = initializer!!()
                  //将返回值赋值给_value,用于下次判断时，直接返回高级函数的返回值
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }
			......
}
```

通过上面代码，可以发现`SynchronizedLazyImpl`覆盖了lazy接口的返回值，并且重写了属性的访问器，具体逻辑是与Java的双重校验类似的。但`Lazy`接口又是怎么变成委托属性的？

在`Lazy.kt`文件中发现它声明了Lazy接口的getValue扩展属性，也就在最终赋值的时候会被调用，而我们在自定义委托中说过，对于val属性，我们需要提供一个getValue函数。

```kotlin
## Lazy.kt
//此扩展允许使用 Lazy 的实例进行属性委托
public inline operator fun <T> Lazy<T>.getValue(thisRef: Any?, property: KProperty<*>): T = value
```

有了这个懒加载委托后，我们实现单例会变的更加简单：

```kotlin
class SingletonDemo private constructor() {
    companion object {
        val instance: SingletonDemo by lazy{
        SingletonDemo() }
    }
}
```

### 3.2 Delegates.observable 观察者委托

如果你要观察一个属性的变化过程，可以将属性委托给`Delegates.observable`，它有三个参数：被赋值的属性、旧值和新值：

```kotlin
var name: String by Delegates.observable("<no name>") {
        prop, old, new ->
        println("$old -> $new")
    }
```

返回了一个`ObservableProperty` 对象，继承自 `ReadWriteProperty`。再来看下它的内部实现：

```kotlin
public inline fun <T> observable(initialValue: T, crossinline onChange: (property: KProperty<*>, oldValue: T, newValue: T) -> Unit):
            ReadWriteProperty<Any?, T> =
        object : ObservableProperty<T>(initialValue) {
            override fun afterChange(property: KProperty<*>, oldValue: T, newValue: T) = onChange(property, oldValue, newValue)
        }
```

`initialValue`是初始值，而另外个参数`onChange是属性值被修改时的回调处理器。

### 3.3 by map 映射委托 

一个常见的用例是在一个映射（map）里存储属性的值，它可以使用`Map/MutableMap`来实现属性委托：

```kotlin
class User(val map: Map<String, Any?>) {
    val name: String by map
}

fun main(args: Array<String>) {
    val map = mutableMapOf(
        "name" to "哈哈"
    )
    val user = User(map)
    println(user.name)
    map["name"] = "LOL"
    println(user.name)
}

//输出：
哈哈
LoL
```

不过在使用过程中会有个问题，如果Map中不存在委托属性名的映射值的时候，会再取值时抛异常：`Key $key is missing in the map`：

```kotlin
## MapAccessors.kt
public inline operator fun <V, V1 : V> MutableMap<in String, out @Exact V>.getValue(thisRef: Any?, property: KProperty<*>): V1 = (getOrImplicitDefault(property.name) as V1)

@kotlin.internal.InlineOnly
public inline operator fun <V> MutableMap<in String, in V>.setValue(thisRef: Any?, property: KProperty<*>, value: V) {
    this.put(property.name, value)
}

## MapWithDefault.kt
internal fun <K, V> Map<K, V>.getOrImplicitDefault(key: K): V {
    if (this is MapWithDefault)
        return this.getOrImplicitDefault(key)

    return getOrElseNullable(key, { throw NoSuchElementException("Key $key is missing in the map.") })
}
```

所以在使用的时候要注意，必须要要有映射值。

### 3.4 两个属性之间的直接委托

从 Kotlin 1.4 开始，我们可以直接在语法层面将“属性 A”委托给“属性 B”，就如下示例：

```kotlin
class Item {
    var count: Int = 0
    var total: Int by ::count
}
```

上面代码total的值与count完全一致，因为我们把total这个属性的getter和setter都委托给了count。可以用代码来解释下具体的逻辑：

```kotlin
class Item {
    var count: Int = 0

    var total: Int
        get() = count

        set(value: Int) {
            count = value
        }
}
```

> 在写法上，委托名称可以使用":"限定符，比如`this::delegate` 或`MyClass::delegate`。

这种用法在字段发生改变，又要保留原有的字段时非常有用。可以定义一个新字段，然后将其委托给原来的字段，这样就不用担心新老字段数值不一样的问题了。

### 3.5 提供委托

如果要在绑定属性委托之前再做一些额外的判断工作要怎么办？我们可以定义`provideDelegate`来实现：

```kotlin
class StringDelegate(private var s: String = "Hello") {                                                     
    operator fun getValue(thisRef: Owner, property: KProperty<*>): String {
        return s
    }                       
    operator fun setValue(thisRef: Owner, property: KProperty<*>, value: String) {
            s = value
    }
}


class SmartDelegator {

    operator fun provideDelegate(
        thisRef: Owner,
        prop: KProperty<*>
    ): ReadWriteProperty<Owner, String> {
				//根据属性委托的名字传入不同的初始值
        return if (prop.name.contains("log")) {
            StringDelegate("log")
        } else {
            StringDelegate("normal")
        }
    }
}

class Owner {
    var normalText: String by SmartDelegator()
    var logText: String by SmartDelegator()
}

fun main() {
    val owner = Owner()
    println(owner.normalText)
    println(owner.logText)
}

//结果：
normal
log
```

这里我们创建了一个新的`SmartDelegator`,通过对成员方法`provideDelegate`再套了一层，然后在里面进行一些逻辑判断，最后才把属性委托get`StringDelegate`。

这种拦截属性与其委托之间的绑定的能力，大大缩短了要实现相同功能，还要必须传递属性名的逻辑。

## 四、委托栗子

### 4.1 简化Fragment / Activity 传参

平时在针对Fragment传参，每次都要写一大段代码是不是很烦，现在有了委托这个法包就来一起简化它，正式模式如下：

```kotlin
class BookDetailFragment : Fragment(R.layout.fragment_book_detail) {

    private var bookId: Int? = null
    private var bookType: Int? = null

    companion object {

        const val EXTRA_BOOK_ID = "bookId"
        const val EXTRA_BOOK_TYPE = "bookType";

        fun newInstance(bookId: Int, bookType: Int?) = BookDetailFragment().apply {
            Bundle().apply {
                putInt(EXTRA_BOOK_ID, bookId)
                if (null != bookType) {
                    putInt(EXTRA_BOOK_TYPE, bookType)
                }
            }.also {
                arguments = it
            }
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        arguments?.let {
            bookId = it.getInt(EXTRA_book_ID, 123)
            bookType = it.getInt(EXTRA_BOOK_TYPE, 1)
        }
    }
}

```

写了那么一大段，终于写好了传参的基本方法，在获取值的时候还要处理参数为空的情况，现在我们就抽取委托类用属性委托的方式重新实现上面功能：

```kotlin
class BookDetailFragment : Fragment(R.layout.fragment_book_detail) {

    private var bookId: Int by argument()

    companion object {
        fun newInstance(bookId: Int, bookType: Int) = BookDetailFragment().apply {
            this.bookId = bookId
        }
    }

    override fun onViewCreated(root: View, savedInstanceState: Bundle?) {
      Log.d("tag", "BOOKID:" + bookId);
    }
}
```

看上去减少了大量代码，是不是很神奇，下面实现思路如下所示：

```kotlin
class FragmentArgumentProperty<T> : ReadWriteProperty<Fragment, T> {

    override fun getValue(thisRef: Fragment, property: KProperty<*>): T {
      ////对Bunndle取值还要进行单独处理
        return thisRef.arguments?.getValue(property.name) as? T
            ?: throw IllegalStateException("Property ${property.name} could not be read")
    }

    override fun setValue(thisRef: Fragment, property: KProperty<*>, value: T) {
        val arguments = thisRef.arguments ?: Bundle().also { thisRef.arguments = it }
        if (arguments.containsKey(property.name)) {
            // The Value is not expected to be modified
            return
        }
      	//对Bunndle设值还要进行单独处理
        arguments[property.name] = value
    }
}

fun <T> Fragment.argument(defaultValue: T? = null) = FragmentArgumentProperty(defaultValue)
```

### 4.2 简化SharedPreferences存取值

如果我们现在存取值可以这样做是不是很方便：

```kotlin
private var spResponse: String by PreferenceString(SP_KEY_RESPONSE, "")

// 读取，展示缓存
display(spResponse)

// 更新缓存
spResponse = response
```

答案是可以的，还是用委托属性来改造，下面就是具体的实现示例：

```kotlin
class PreDelegate<T>(
        private val name: String,
        private val default: T,
        private val isCommit: Boolean = false,
        private val prefs: SharedPreferences = App.prefs) {

    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return getPref(name, default) ?: default
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        value?.let {
            putPref(name, value)
        }
    }

    private fun <T> getPref(name: String, default: T): T? = with(prefs) {
        val result: Any? = when (default) {
            is Long -> getLong(name, default)
            is String -> getString(name, default)
            is Int -> getInt(name, default)
            is Boolean -> getBoolean(name, default)
            is Float -> getFloat(name, default)
            else -> throw IllegalArgumentException("This type is not supported")
        }

        result as? T
    }

    private fun <T> putPref(name: String, value: T) = with(prefs.edit()) {
        when (value) {
            is Long -> putLong(name, value)
            is String -> putString(name, value)
            is Int -> putInt(name, value)
            is Boolean -> putBoolean(name, value)
            is Float -> putFloat(name, value)
            else -> throw IllegalArgumentException("This type is not supported")
        }

        if (isCommit) {
            commit()
        } else {
            apply()
        }
    }
}
```

### 4.3 数据与View的绑定

有了委托之后，在不用到`DataBinding，`数据与View之间也可以进行绑定。

```kotlin
operator fun TextView.provideDelegate(value: Any?, property: KProperty<*>) = object : ReadWriteProperty<Any?, String?> {
    override fun getValue(thisRef: Any?, property: KProperty<*>): String? = text
    override fun setValue(thisRef: Any?, property: KProperty<*>, value: String?) {
        text = value
    }
}
```

给TextView写一个扩展函数，让它支持了String属性的委托。

```kotlin
val textView = findViewById<textView>(R.id.textView)

var message: String? by textView

textView.text = "Hello"
println(message)

message = "World"
println(textView.text)

//结果：
Hello
World
```

我们通过委托的方式，将 message 委托给了 textView。这意味着，message 的 getter 和 setter 都将与 TextView 关联到一起。

## 五、小结

主要讲解了 Kotlin 委托的用法和本质，总共有两类委托类和委托属性，特别是属性委托要值得重视。在开发中其实有很多场景可以用委托来进行简化，减少很多重复的样板代码，可以说跟扩展比起来也毫不逊色。

**参考**

[kotlin官网委托介绍](https://www.kotlincn.net/docs/reference/delegation.html)

[Kotlin Jetpack 实战 | 07. Kotlin 委托](https://juejin.cn/post/6859172099680894989#heading-9)

[Kotlin 委托的本质以及 MMKV 的应用](https://juejin.cn/post/7043843490366619685)

[Kotlin | 委托机制 & 原理 & 应用](https://juejin.cn/post/6958346113552220173#heading-15)

