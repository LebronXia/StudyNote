## ViewModel背景

> ViewModel 类旨在以注重生命周期的方式存储和管理界面相关的数据。ViewModel类让数据可在发生屏幕旋转等配置更改后继续留存。
>
> 摘自Viewmodel概览

详细讲，ViewModel有如下几个特点：

1. 对于简单数据，Activity被销毁的时候，可以使用onSaveInstanceState()方法从onCreate中恢复其绑定数据，但不适用数量较大的数据，如用户列表或位图。**而Viewmodel支持大量数据，也不需要序列化和反序列化操作。**
2. 视图控制器经常需要进程可能需要一些时间才能返回的异步调用。界面控制器需要管理这些调用，并确保系统在其销毁后清理这些调用以避免潜在的内存泄漏。**而Viewmodel可以很好的避免内存泄漏。**
3. 如果要求界面控制器也负责从数据库或网络加载数据，那么会使类越发膨胀。为界面控制器分配过多的责任可能会导致单个类尝试自己处理应用的所有工作，而不是将工作委托给其他类。**而Viewmodel可以有效的将视图数据逻辑和视图控制器分离开来。**

可以看出来VIewmodel是以感知生命周期的形式来存储和管理视图等相关的数据。

## 基本使用

对Viewmodel的特点有了初步了解后，开始简单操作下。

自定义数据获取类

```kotlin
class TestRepository {
    suspend fun getNameList(): List<String> {
        return withContext(Dispatchers.IO) {
            listOf("哈哈", "呵呵")
        }
    }
}
```

自定义ViewModel*继承ViewMode，实现自定义ViewModel。

```kotlin
class TestViewModel: ViewModel() {
    private val nameList = MutableLiveData<List<String>>()
    val nameListResult: LiveData<List<String>> = nameList
    private val testRepository = TestRepository()
    
    fun getNames() {
        viewModelScope.launch {
            nameList.value = testRepository.getNameList()
        }
    }
}
```

创建了MutableLiveData，并通过MutableLiveDat的setVale方法来更新数据。

然后就可以在Activity中使用TestViewModel了。

```kotlin
class MainActivity : AppCompatActivity() {
    // 创建 ViewModel 方式 1
    // 通过 kotlin 委托特性创建 ViewModel
    // 需添加依赖 implementation 'androidx.activity:activity-ktx:1.2.3'
    // viewModels() 内部也是通过 创建 ViewModel 方式 2 来创建的 ViewModel
    private val mainViewModel: TestViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate  (savedInstanceState)
        setContentView(R.layout.activity_main)
        // 创建 ViewModel 方式 2
        val mainViewModel = ViewModelProvider(this).get(TestViewModel::class.java)
        mainViewModel.nameListResult.observe(this, {
            Log.i("MainActivity", "mainViewModel: nameListResult: $it")
        })
        mainViewModel.getNames()
    }
}
```

通过ViemodelProvider配合就可以获得TestViewModel对象，并配合LiveData来观察其变化。通过日志打印我们得到了：

```java
maiinViewModel: nameListResult: [哈哈, 呵呵]
```

我们再旋转下手机，发现打印出来的信息仍然存在。

```java
maiinViewModel: nameListResult: [哈哈, 呵呵]
```

说明即使MainActivity被重建了，而ViewModel的实例对象依然存在，内部的LiveData也没有变，是不是很神奇!!!

**那这个又是如何实现的？**现在就让我们一起来分析下它的源码。

## ViewModel分析

首先我们来看下Viewmodel是怎么被创建的。

```kotlin
 val mainViewModel = ViewModelProvider(this).get(TestViewModel::class.java)
```

