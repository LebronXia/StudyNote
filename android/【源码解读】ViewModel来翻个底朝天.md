## ViewModel背景

> ViewModel 类旨在以注重生命周期的方式存储和管理界面相关的数据。ViewModel类让数据可在发生屏幕旋转等配置更改后继续留存。
>
> 摘自 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel)概览

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

自定义ViewModel继承ViewMode，实现自定义ViewModel。

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

**Activity旋转屏幕后ViewModel可以恢复数据又是如何实现的？** 现在就让我们一起来分析下它的源码。

## 源码分析

### Viewmodel的创建

首先我们来看下Viewmodel是怎么被创建的。

从上面可以得知，创建Viewmodel的方式是：

```kotlin
 val mainViewModel = ViewModelProvider(this).get(TestViewModel::class.java)
```

传入class类型，然后得到了ViewModel。看下ViewModelProcider的构造函数：

```java
### ViewModelProvider 
public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
  		//得到了ViewModelStore
        this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
    }

//缓存了ViewModerStore
public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
        mFactory = factory;
        mViewModelStore = store;
    }
```

这里我们看到他得到了`ViewmodelStore`。可以是我们传进去的只是Activity实例，那`ViewModelStoreOwner`又是什么？

```java
//拥有 ViewModelStore 的范围。
//此接口的实现的职责是在配置更改期间保留拥有的 ViewModelStore，
//并在此范围将被销毁时调用 ViewModelStore.clear()。
public interface ViewModelStoreOwner {
    /**
     * Returns owned {@link ViewModelStore}
     *
     * @return a {@code ViewModelStore}
     */
    @NonNull
    ViewModelStore getViewModelStore();
}
```

ViewModelStoreOwner是个接口，再来看下`ComponentActivity`是不是对它进行了实现：

```java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        LifecycleOwner,
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory,
        SavedStateRegistryOwner,
        OnBackPressedDispatcherOwner {
          ......
            
      //实现了ViewModelStoreOwner， 重写了getViewModelStore方法
      public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
          //里面内容下面分析
         ....
        return mViewModelStore;
    }
}

```

原来ComponentActivity实现了ViewModelStoreOwner接口。

回到例子中，`ViewModelProvider(this).get(TestViewModel::class.java)`中的get方法又做了什么？

```java
### ViewModelProvider
   public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
  	//拿到key, 也就是ViewmodelStore中的Map用于存ViewModel的key
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }

public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
  	//获取到ViewModel实例
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            if (mFactory instanceof OnRequeryFactory) {
                ((OnRequeryFactory) mFactory).onRequery(viewModel);
            }
            //返回缓存的ViewModel对象
            return (T) viewModel;
        } else {
            if (viewModel != null) {
            }
        }
  	//使用工厂模式创建 ViewModel 实例
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
        } else {
            viewModel = (mFactory).create(modelClass);
        }
  	//将创建的 ViewModel 实例放进 mViewModelStore 缓存中
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    }
```

先尝试从ViewModelStore获取ViewModel实例，如果没有获取到，就使用Factory创建，创建出的ViewModel最后都会存放到mViewModelStore中，我们再来看下ViewModelStore这个类做了什么：

```java
### ViewModelStore
public class ViewModelStore {
   //内部由HashMap进行存储
    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }
  
  	//调用viewModel的clear方法，然后清除ViewModel
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```

在ViewModelStore中就做了将viewModel作为value存储到HashMap中。

看了VIewModel是怎么创建出来的，以及实例缓存到那里，再来看下ViewModelStore又是如何创建的。

### ViewmodelStore的创建

再次返回到上面getViewModelStore方法里面看下：

```java
### ComponentActiivty   
public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mViewModelStore == null) { 
          //获得 NonConfigurationInstances
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                // 从 NonConfigurationInstances 恢复 ViewModelStore
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }
```

先从`NonConfigurationInstances`中获取VIewModelStore实例，如果不存在，则会创建一个新的ViewModelSotre。这里我们知道了，创建出的`ViewModelStore`会缓存到`NonConfigurationInstances`中。那`ViewModelStore`又是什么时候缓存进去的。

来看下getLastNonConfigurationInstance方法：

```java
### Activity 
 //先前由 onRetainNonConfigurationInstance() 返回的对象
  /**
     * Retrieve the non-configuration instance data that was previously
     * returned by {@link #onRetainNonConfigurationInstance()}.  This will
     * be available from the initial {@link #onCreate} and
     * {@link #onStart} calls to the new instance, allowing you to extract
     * any useful dynamic state from the previous instance.
     *
     * <p>Note that the data you retrieve here should <em>only</em> be used
     * as an optimization for handling configuration changes.  You should always
     * be able to handle getting a null pointer back, and an activity must
     * still be able to restore itself to its previous state (through the
     * normal {@link #onSaveInstanceState(Bundle)} mechanism) even if this
     * function returns null.
     *
     * <p><strong>Note:</strong> For most cases you should use the {@link Fragment} API
     * {@link Fragment#setRetainInstance(boolean)} instead; this is also
     * available on older platforms through the Android support libraries.
     *
     * @return the object previously returned by {@link #onRetainNonConfigurationInstance()}
     */
public Object getLastNonConfigurationInstance() {
        return mLastNonConfigurationInstances != null
                ? mLastNonConfigurationInstances.activity : null;
    }
```

通过上面注释可以知道`mLastNonConfigurationInstances`是由`onRetainNonConfigurationInstance`返回的。

`onRetainNonConfigurationInstance`又是在什么时候被调用的？最终被我们找到了：

```java
###ActivityThread
void performDestroyActivity(ActivityClientRecord r, boolean finishing,
            int configChanges, boolean getNonConfigInstance, String reason) {
       ......
      if (getNonConfigInstance) {
                try {
                    r.lastNonConfigurationInstances = r.activity.retainNonConfigurationInstances();
                } catch (Exception e) {
                    if (!mInstrumentation.onException(r.activity, e)) {
                        throw new RuntimeException("Unable to retain activity "
                                + r.intent.getComponent().toShortString() + ": " + e.toString(), e);
                    }
                }
            }
        ......
}
```

在`performDestroyActivity`方法中会被调用，可以看到`onRetainNonConfigurationInstance`方法返回的Object会赋值给`ActivityClientRecord`的`lastNonConfigurationInstances`，这样子就保存了下来。再来看下`onRetainNonConfigurationInstance`方法。

```java
### ComponetActivity    
  //配置发生更改的时候调用
  //保留所有适当的非配置状态
public final Object onRetainNonConfigurationInstance() {
        Object custom = onRetainCustomNonConfigurationInstance();

        ViewModelStore viewModelStore = mViewModelStore;
        if (viewModelStore == null) {
            //没有人调用 getViewModelStore()，所以看看我们上一个 NonConfigurationInstance 中是否存在现					//有的 ViewModelStore
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                viewModelStore = nc.viewModelStore;
            }
        }

        if (viewModelStore == null && custom == null) {
            return null;
        }
				
  	//创建 NonConfigurationInstances 对象，并赋值 viewModelStore
        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.custom = custom;
        nci.viewModelStore = viewModelStore;
        return nci;
    }
```

到这里我们知道了，Activity在应为配置发生更改销毁重建的时候会调用`onRetainNonConfigurationInstance`将ViewModelStore实例保存到`NonConfigurationInstances`中。之后再重建的时候就可以通过`getLastNonConfigurationInstance`方法来获取之前缓存的ViewModelStore实例。

`NonConfigurationInstances`又是什么，怎么可以在销毁重建的时候还一直保留？来看下`NonConfigurationInstances`保存在哪里，先看下在哪里被赋值的。

```java
### Activity
  final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
            IBinder shareableActivityToken) {
  
  ......
    //在attach方法中会进行赋值
    mLastNonConfigurationInstances = lastNonConfigurationInstances;
  ......
}
```

`mLastNonConfigurationInstances`是在Activity的attach方法中赋值。


```java

### ActivityThread

  private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
			······
        //由ActivityClientRecord中获得
        activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken, r.shareableActivityToken);
  ......
}

```

在`Activity`的`performLaunchActivity`方法中，可以看到`lastNonConfigurationInstances`是保存在`ActivityClientRecord中`的。

又因为界面在销毁的时候调用`performDestroyActivity`方法，内部又会调用`Activity`的`retainNonConfigurationInstances`方法将`lastNonConfigurationInstances`缓存到ActivityClientRecord中，也就是存到应用本身的进程中了。

所以页面在销毁重建的时候，保存在`ActivityClientRecord`中的`lastNonConfigurationInstances`是不会受到影响的。

到这里我们也知道在Activity因配置更改重建后`ViewModel`依然存在的原因了。

### ViewModel的销毁

我们已经知道了，在`ViewmodelStore`中会调用viewmodle的clear方法。

```java
###ViewModelStore
public class ViewModelStore {

  ......
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```

遍历缓存中viewmodel依次调用，那ViewModelStore的`clear`方法又是什么时候调用的？

```java
### ComponentActivity
public ComponentActivity() {
  	......
      getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
              //activity生命周期处于destory状态
                if (event == Lifecycle.Event.ON_DESTROY) {
                  //非配置发生改变
                    if (!isChangingConfigurations()) {
                        getViewModelStore().clear();
                    }
                }
            }
        });
}
```

这里我们看到调用`ViewModelStore`中的clear方法被调用的条件首先要观察当前的生命周期处于`DESTORY`，另外检测因为Configurations的改变而销毁的。

所以在 **activity** 销毁时，判断如果是非配置改变导致的销毁， **getViewModelStore().clear()** 才会被调用。

### viewModelScope了解吗

`viewModelScope` 是一个 ViewModel 的 Kotlin 扩展属性。它能在`ViewModel`销毁时 (onCleared() 方法调用时) 退出。那它又是什么时候关闭的？

我们还是来看下它的实现：

```kotlin
//CoroutineScope 绑定到这个 ViewModel。当 ViewModel 被清除时，这个范围将被取消，即调用 //ViewModel.onCleared 这个范围绑定到 Dispatchers.Main.immediate
val ViewModel.viewModelScope: CoroutineScope
        get() {
            val scope: CoroutineScope? = this.getTag(JOB_KEY)
            if (scope != null) {
                return scope
            }
          // 未命中缓存，则通过 setTagIfAbsent() 添加到 ViewModel 中
            return setTagIfAbsent(JOB_KEY,
                CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate))
        }

internal class CloseableCoroutineScope(context: CoroutineContext) : Closeable, CoroutineScope {
    override val coroutineContext: CoroutineContext = context

    override fun close() {
        coroutineContext.cancel()
    }
}
```

根据注释可以得知，Viewmoel被清除的时候，就是调用ViewModel的onClear方法时，就会取消作用域。而ViewModel的清除又是为什么会让其关闭？

再让我们看下Viewmodel的clear方法：

```java
### ViewModel
 //当不再使用此ViewModel并将其销毁时，将调用此方法。
 protected void onCleared() {
    }

 final void clear() {
        mCleared = true;
        //由于 clear() 是最终的，因此仍会在模拟对象上调用此方法，并且在这些情况下，mBagOfTags 为空。它总是空
  //的，因为 setTagIfAbsent 和 getTag 不是最终的，所以我们可以跳过清除它
        if (mBagOfTags != null) {
            synchronized (mBagOfTags) {
                for (Object value : mBagOfTags.values()) {
                    // see comment for the similar call in setTagIfAbsent
                    closeWithRuntimeException(value);
                }
            }
        }
        onCleared();
    }
```

怎么onClear方法是空实现？又看到这个方法是被clear方法所调用，我们再来看下`mbagofTags`是什么东西？

```java
private final Map<String, Object> mBagOfTags = new HashMap<>();
```

这是个Map对象，再继续往下找可以看到调用setTagIfAbsent进行了赋值。

```java
 <T> T setTagIfAbsent(String key, T newValue) {
        T previous;
        synchronized (mBagOfTags) {
            previous = (T) mBagOfTags.get(key);
            if (previous == null) {
              //如果之前没有保存则放入map中
                mBagOfTags.put(key, newValue);
            }
        }
        T result = previous == null ? newValue : previous;
   			//// 如果此 viewModel 被标记清除
        if (mCleared) {
            closeWithRuntimeException(result);
        }
        return result;
    }
```

这个`setTagIfAbsent`方法不就是在viewModelScope第一次被创建的时候调用的，也就是在这里将对象添加到ViewModel中。

好了再来回顾下，viewModelScope 第一次被调用时，会调用 `setTagIfAbsent(JOB_KEY，CloseableCoroutineScope) `进行缓存。

看下` CloseableCoroutineScope` 类，实现了 Closeable 接口，并且在 close() 中进行了协程作用域` coroutineContext `对象的取消操作。

## 总结

我们先介绍了viewmodel的背景，了解到它的特点：因配置更改界面销毁重建后依然存在、不持有UI应用，也介绍了ViewModel的基本使用。最后大篇幅的分析了ViewModel几个关键的核心原理。

到这里，我们的分析之旅也结束了~

**参考**

[ViewModel官网](https://developer.android.com/reference/androidx/lifecycle/ViewModel)

[Android - ViewModel总结](https://blog.csdn.net/u012885461/article/details/118073987)

