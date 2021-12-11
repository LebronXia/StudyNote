

# ViewModel

-  viewModelScope 什么时候关闭的
-  **为什么Activity旋转屏幕后ViewModel可以恢复数据**
-  **ViewModel 的实例缓存到哪儿了**
-  **什么时候 ViewModel#onCleared() 会被调用**

**ViewModel** 对象存在了 **ComponentActivity** 的 **mViewModelStore** 对象中。

Activity 在因配置更改而销毁重建过程中会先调用 onRetainNonConfigurationInstance 保存 viewModelStore 实例。
在重建后可以通过 getLastNonConfigurationInstance 方法获取之前的 viewModelStore 实例。

在 **activity** 销毁时，判断如果是非配置改变导致的销毁， **getViewModelStore().clear()** 才会被调用。

```Java
 public ComponentActivity() {
        ... 
        getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_DESTROY) {
                    // Clear out the available context
                    mContextAwareHelper.clearAvailableContext();
                    // And clear the ViewModelStore
                    if (!isChangingConfigurations()) {
                        getViewModelStore().clear();
                    }
                }
            }
        });
        ...
    }

public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
```



viewModelScope 第一次被调用时，会调用 setTagIfAbsent(JOB_KEY，CloseableCoroutineScope) 进行缓存。
看下 CloseableCoroutineScope 类，实现了 Closeable 接口，并且在 close() 中进行了协程作用域 coroutineContext 对象的取消操作。

**并且在 ViewModel 被清除时 viewModelScope 会被取消。**

```java
   final void clear() {
    	// 标记当前 ViewModel 已经被清除
        mCleared = true;
        // 判断 mBagOfTags 是否为空，不为空则遍历 map 的 value 并且调用了 closeWithRuntimeException() 方法
        if (mBagOfTags != null) {
            synchronized (mBagOfTags) {
              //mBagOfTags保存了CloseableCoroutineScope类
                for (Object value : mBagOfTags.values()) {
                    // see comment for the similar call in setTagIfAbsent
                    closeWithRuntimeException(value);
                }
            }
        }
        onCleared();
    }

internal class CloseableCoroutineScope(context: CoroutineContext) : Closeable, CoroutineScope {
    override val coroutineContext: CoroutineContext = context

    override fun close() {
        coroutineContext.cancel()
    }
}
```



[Jetpack ： ViewModel 必知的几个问题](https://blog.csdn.net/u012885461/article/details/118073987)

[关于 ViewModel 的一个疑问](https://mp.weixin.qq.com/s/om56J31MIza_0BM6XQCqWg)

[viewModelScope 什么时候关闭的？](https://blog.csdn.net/u012885461/article/details/118056917)

[ViewModel原理分析](https://www.jianshu.com/p/e5c363255617)

# LiveData

- **LiveData 怎么感知生命周期感知？需要取消注册吗？**
- **setValue 和 postValue 有什么区别**
- **设置相同的值，订阅的观察者们会收到同样的值吗**
- **粘性事件原理，怎么防止数据倒灌**
- **observeForever怎么用**

调用 observe 方法时，会调用 owner.getLifecycle().addObserver 已达到感知生命周期的目的。

`LifecycleBoundObserver`,onStateChanged 方法中注释如果是已销毁状态，则调用 LiveData#removeObserver 方法进行移除 mObserver (mObserver 是 LifecycleBoundObserver 的父类 ObserverWrapper 的属性

setValue 只能在主线程使用，而 postValue 不限制线程。

**设置相同的值，订阅的观察者们会收到同样的值吗** 在分发值的过程中没有判断过新值是否等于老值的代码出现，只出现了一个判断 version，通过 demo 尝试也会发现，会收到同样的值的哟

有几种情况 LiveData 会分发值。

1. 调用 setValue 和 postValue 并且 LifecycleOwner 处于活跃状态时
2. LiveData 有值，并且处于活跃状态时，调用 LiveData#observe 订阅观察者
3. LiveData 有新值，也就是 ObserverWrapper 的 mLastVersion 小于 LiveData 的 mVersion，LifecycleOwner 从不活跃状态转为活跃状态时

## 源码

在使用LiveData.observe 订阅者的时候，也就是调用

```java
edPacketVm.grabRedPacketLiveData.observe(this, Observer {
            .....
        })
```

现在来看下它的源码实现

```java
 @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
          //在生命周期为DESTROYED，不进行订阅
            return;
        }
       //用LifecycleBoundObserver包装对象
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
      //LifecycleBoundObserver 是 ObserverWrapper 的子类或者实现类
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
      // 则抛出异常，不可以添加同一个 Observer 到两个不同的生命周期对象。
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
      //生命周期进行感知
        owner.getLifecycle().addObserver(wrapper);
    }
```

可以看到两个重要的类：

```java
//观察者的实现    
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
        @NonNull
        final LifecycleOwner mOwner;

				......
          
        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
          //获取生命中周期
            Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
            if (currentState == DESTROYED) {
              //如果是销毁状态，则调用LiveData.removeObserver  进行移除
                removeObserver(mObserver);
                return;
            }
            Lifecycle.State prevState = null;
          //两次状态变化，也是会回调
            while (prevState != currentState) {
                prevState = currentState;
              //下面说明
                activeStateChanged(shouldBeActive());
                currentState = mOwner.getLifecycle().getCurrentState();
            }
        }

        @Override
        boolean isAttachedTo(LifecycleOwner owner) {
            return mOwner == owner;
        }

        @Override
        void detachObserver() {
            mOwner.getLifecycle().removeObserver(this);
        }
    }

//ObserverWrapper.java
void activeStateChanged(boolean newActive) {
  //第一次创建对象 mActive 没有赋值，则为默认值 false
        // 刚刚调用 activeStateChanged 方法时，传入的值是 shouldBeActive() 返回的 true
            if (newActive == mActive) {
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            mActive = newActive;
            changeActiveCounter(mActive ? 1 : -1);
            if (mActive) {
              //对消息进行传递
                dispatchingValue(this);
            }
        }
```

## setValue

后面看setValue和postValue实现了什么

```java
  @MainThread
    protected void setValue(T value) {
      //检查当前线程是否在主线程
        assertMainThread("setValue");
      //version处理
        mVersion++;
        mData = value;
      //同样是分发值
        dispatchingValue(null);
    }

//postValue调用
    protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
      //子线程执行后，可重新执行
        if (!postTask) {
            return;
        }
      //最终调用MainHandler.post(runnable)，做线程切换
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }

	//子线程实现
  private final Runnable mPostValueRunnable = new Runnable() {
        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            Object newValue;
            synchronized (mDataLock) {
                newValue = mPendingData;
                mPendingData = NOT_SET;
            }
          //最终还是调用setValue
            setValue((T) newValue);
        }
    };

```

看到`postValue`同样是操作了`setValue`赋值，再往下看最后是调`c`方法

```java
    @SuppressWarnings("unchecked")
    private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {
            return;
        }
        
      //判断是活跃
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
      
      //mLastVersion刚开始未初始值
      //mVersion 每次调用setValue方法时+1
      //vesion判断
        if (observer.mLastVersion >= mVersion) {
            return;
        }
      //mLastVersion只在这里赋值
        observer.mLastVersion = mVersion;
      //这里终于调用了我们 LiveData.observe 方法传入的 Observer 对象的 onChanged 方法。
        observer.mObserver.onChanged((T) mData);
    }
```

看到这里基本把LiveData的基本原理看明白了。

## 粘性事件

**粘性事件**又是怎么一回事？

为什么在 LiveData 有值时，调用 LiveData#observe 订阅观察者，会收到旧值？

是因为LiveData有值的时候，并且处于活跃状态，第一次调用`Livedata.observer订阅观察者`。进行版本判断出错

```java
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
 
   ......
     
    // 因为 observer 是新创建的 所以这里的 observer.mLastVersion 应为初始值 LiveData#START_VERSION 为 -1 
    // 而 mVersion 因为 LiveData 不是第一次分发值，所以 mVersion 肯定是大于初始值 START_VERSION -1 的
    // 故此条件不成立, 导致异常
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    // 将 LiveData 的 mVersion 赋值给 observer.mLastVersion
    observer.mLastVersion = mVersion;
    // 调用了 onChanged 也就是我们在 LiveData.observe 传入的 Observer 的 Observer 对象的 onChanged
    observer.mObserver.onChanged((T) mData);
}

```

**observeForever**被调用时，用了另外的观察者

```java
 private class AlwaysActiveObserver extends ObserverWrapper {

        AlwaysActiveObserver(Observer<? super T> observer) {
            super(observer);
        }

   //一直未true,不受生命周期活跃控制
   //没有了removeObserver,需要自己手动处理
        @Override
        boolean shouldBeActive() {
            return true;
        }
    }
```





[Android 面试总结 - LiveData](https://juejin.cn/post/6991168529454088228)

[Android 面试总结 - LiveData(二)](https://juejin.cn/post/6991497263457501221)

[**重学安卓：LiveData 数据倒灌 背景缘由全貌 独家解析**](https://xiaozhuanlan.com/topic/6719328450)

# LifeCycle

## 原理

Activity是通过ReportFragment代理了LifecycleOwner的实现

我们可以看到在对应的生命周期调用了dispatch方法，而在dispatch方法中又调用了一个多参的dispatch方法。在此方法中会判断activity是属于LifecycleRegistryOwner还是LifecycleRegistry。最终都会调到LifecycleRegistry的handleLifecycleEvent方法中去。

最终它都会走到observer.dispatchEvent(lifecycleOwner, event);方法中。
并且最终会调用mLifecycleObserver.onStateChanged(owner, event);也就是LifecycleEventObserver的方法。我们看基于LifecycleEventObserver的实现类ReflectiveGenericLifecycleObserver。

lifeCycle基于观察者模式，LifecycleOwner被观察者，LifecycleObserver观察者。

## 如何进行生命中周期分发

```java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        LifecycleOwner,
        ViewModelStoreOwner,
        SavedStateRegistryOwner,
        OnBackPressedDispatcherOwner {
          .....
                private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
          ....
             @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
          
              @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mSavedStateRegistryController.performRestore(savedInstanceState);
        ReportFragment.injectIfNeededIn(this);
        if (mContentLayoutId != 0) {
            setContentView(mContentLayoutId);
        }
    }

          ....
        }
```

`getLifeCycle()`得到了LifecycleRegistry，并在`onCreate()`中写入`reportFragment`作为生命周期的观察

```java
public class ReportFragment extends Fragment {
  ...
    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
        // just want to be sure that we won't leak reference to an activity
        mProcessListener = null;
    }

    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
}
```

fragment在这里的作用是对生命周期进行分发，真正实现的是`getLifecycle().handleLifecycleEvent(event)`

```java
 public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }

 private void moveToState(State next) {
        if (mState == next) {
            return;
        }
       ....
         sync();
        mHandlingEvent = false;
    }

 private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        .....
        while (!isSynced()) {
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
              //这里调用observer.dispatchEvent(lifecycleOwner, event); 进行分发，也就是调用ObserverWithState里的方法
              backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
              // //这里调用observer.dispatchEvent(lifecycleOwner, event); 进行分发  
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
```

`LifecycleRegistry`内部则是实现了`Lifecycle`的抽象方法，完成了观察者具体实现。当生命周期改变时调用handleLifecycleEvent方法通知LifecycleObserver，并调用相关方法。

## LifecycleObserver

回到`LifecycleRegistry`代码，当我们调用`getLifecycle().addObserver(myLocationListener);`内部实现：

```java
public class LifecycleRegistry extends Lifecycle {
  private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
              new FastSafeIterableMap<>();
  ....
        @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    //都会被包装成对应的Observer
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    //将观察者放入map中，后续作为分发
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }
  
}
```

`ObserverWithState`中写了什么

```java
static class ObserverWithState {
        State mState;
        LifecycleEventObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
```

在`Lifecycling.lifecycleEventObserver(observer)`方法中会返回`LifecycleEventObserver`对象

```java
//Lifecycling.java
    @NonNull
    static LifecycleEventObserver lifecycleEventObserver(Object object) {
        boolean isLifecycleEventObserver = object instanceof LifecycleEventObserver;
        boolean isFullLifecycleObserver = object instanceof FullLifecycleObserver;
        if (isLifecycleEventObserver && isFullLifecycleObserver) {
            return new FullLifecycleObserverAdapter((FullLifecycleObserver) object,
                    (LifecycleEventObserver) object);
        }
        if (isFullLifecycleObserver) {
            return new FullLifecycleObserverAdapter((FullLifecycleObserver) object, null);
        }

        if (isLifecycleEventObserver) {
            return (LifecycleEventObserver) object;
        }

        final Class<?> klass = object.getClass();
      //这份方法通过klass获取注解，进行记录
        int type = getObserverConstructorType(klass);
        if (type == GENERATED_CALLBACK) {
            List<Constructor<? extends GeneratedAdapter>> constructors =
                    sClassToAdapters.get(klass);
            if (constructors.size() == 1) {
                GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                        constructors.get(0), object);
                return new SingleGeneratedAdapterObserver(generatedAdapter);
            }
            GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
            for (int i = 0; i < constructors.size(); i++) {
                adapters[i] = createGeneratedAdapter(constructors.get(i), object);
            }
            return new CompositeGeneratedAdaptersObserver(adapters);
        }
        return new ReflectiveGenericLifecycleObserver(object);
    }

//getObserverConstructorType(klass)，最终调用的方法
boolean hasLifecycleMethods = ClassesInfoCache.sInstance.hasLifecycleMethods(klass);
```

返回到`ObserverWithState`中，获取到观察者后状态发生变化会调用`mLifecycleObserver.onStateChanged(owner, event);`其中有三个FullLifecycleObserverAdapter`、 `CompositeGeneratedAdaptersObserver`、`ReflectiveGenericLifecycleObserver`。

现在去`FullLifecycleObserverAdapter`分下下，看下内部怎么实现

```java
class FullLifecycleObserverAdapter implements LifecycleEventObserver {

    private final FullLifecycleObserver mFullLifecycleObserver;
    private final LifecycleEventObserver mLifecycleEventObserver;

    FullLifecycleObserverAdapter(FullLifecycleObserver fullLifecycleObserver,
            LifecycleEventObserver lifecycleEventObserver) {
        mFullLifecycleObserver = fullLifecycleObserver;
        mLifecycleEventObserver = lifecycleEventObserver;
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        switch (event) {
            case ON_CREATE:
                mFullLifecycleObserver.onCreate(source);
                break;
            case ON_START:
                mFullLifecycleObserver.onStart(source);
                break;
            case ON_RESUME:
                mFullLifecycleObserver.onResume(source);
                break;
            case ON_PAUSE:
                mFullLifecycleObserver.onPause(source);
                break;
            case ON_STOP:
                mFullLifecycleObserver.onStop(source);
                break;
            case ON_DESTROY:
                mFullLifecycleObserver.onDestroy(source);
                break;
            case ON_ANY:
                throw new IllegalArgumentException("ON_ANY must not been send by anybody");
        }
        if (mLifecycleEventObserver != null) {
            mLifecycleEventObserver.onStateChanged(source, event);
        }
    }
}
```

可以看到当onStateChanged被调用时，根据event回调我们实现的DefaultLifecycleObserver里的各种生命周期方法。

[Android Jetpack--lifecycle全解析](https://blog.csdn.net/verymrq/article/details/89356843)

# Glide

- Glide中怎么实现图片的加载进度条，Glide的缓存是怎么设计的？为什么要用弱引用
- Glide怎么绑定生命周期
- LruCache原理

OkhttpIncepter ResponedBody content_length

## **Glide优势**

1. 多种图片格式的缓存，适用于更多的内容表现形式（如Gif、WebP、缩略图、Video）
2. 生命周期集成（根据Activity或者Fragment的生命周期管理图片加载请求）
3. 高效处理Bitmap（bitmap的复用和主动回收，减少系统回收压力）
4. 高效的缓存策略，灵活（Picasso只会缓存原始尺寸的图片，Glide缓存的是多种规格），加载速度快且内存开销小（默认Bitmap格式的不同，使得内存开销是Picasso的一半）

## **Key的生成**
这个id会结合signature、width、height等等10个参数一起传入到EngineKeyFactory的buildKey()方法当中，从而构建出了一个EngineKey对象，这个EngineKey也就是Glide中的缓存Key了。

## **为什么用弱引用**

如果找到了，我们从LruResourceCache中获取到缓存图片之后会将它从缓存中移除，然后将这个缓存图片存储到activeResources当中，以保护这些图片不会被LruCache算法回收掉。如果在LruResourceCache中没有找到，再去activeResources中进行查找。如果二者均找不到符合资源，再开启子线程进行加载图片资源
在Glide的内存缓存机制中，LruResourceCache的优先级在activeResources之前。

## **LRUcache算法**

内部存在一个LinkedHashMap和maxSize，把最近使用的对象用强引用存储在 LinkedHashMap中，给出来put和get方法，每次put图片时计算缓存中所有图片总大小，跟maxSize进行比较，大于maxSize，就将最久添加的图片移除；反之小于maxSize就添加进来。

## **LinkedHashMap**

LinkHashMap 的 put方法和get方法最后会调用`trimToSize`方法，**LruCache 重写`trimToSize`方法，判断内存如果超过一定大小，则移除最老的数据**

1. LruCache中维护了一个集合LinkedHashMap，该LinkedHashMap是以访问顺序排序的。
2. 当调用put()方法时，就会在结合中添加元素，并调用trimToSize()判断缓存是否已满，如果满了就用LinkedHashMap的迭代器删除队首元素，即近期最少访问的元素。
3. 当调用get()方法访问缓存对象时，就会调用LinkedHashMap的get()方法获得对应集合元素，同时会更新该元素到队尾。

LruCache里存的是软引用对象，那么当内存不足的时候，Bitmap会被回收，也就是说通过SoftReference修饰的Bitmap就不会导致OOM

Glide的图片加载过程中会调用两个方法来获取内存缓存，loadFromCache()和loadFromActiveResources()。这两个方法中一个使用的就是LruCache算法，另一个使用的就是弱引用。

当我们从LruResourceCache中获取到缓存图片之后会将它从缓存中移除，然后在第16行将这个缓存图片存储到activeResources当中。activeResources就是一个弱引用的HashMap，用来缓存正在使用中的图片，我们可以看到，loadFromActiveResources()方法就是从activeResources这个HashMap当中取值的。使用activeResources来缓存正在使用中的图片，可以保护这些图片不会被LruCache算法回收掉。

这里首先会将缓存图片从activeResources中移除，然后再将它put到LruResourceCache当中。这样也就实现了正在使用中的图片使用弱引用来进行缓存，不在使用中的图片使用LruCache来进行缓存的功能。

## **RequestManager感应生命周期**

在Activity/fragment 销毁的时候，取消图片加载任务

这其中采用的监控方法就是在当前activity中添加一个没有view的fragment，这样在activity发生onStart onStop onDestroy的时候，会触发此fragment的onStart onStop onDestroy。

```java
public void onDestroy() {
    targetTracker.onDestroy();
    for (Target<?> target : targetTracker.getAll()) {
      //清理任务
      clear(target);
    }
    targetTracker.clear();
    requestTracker.clearRequests();
    lifecycle.removeListener(this);
    lifecycle.removeListener(connectivityMonitor);
    mainHandler.removeCallbacks(addSelfToLifecycle);
    glide.unregisterRequestManager(this);
  }
```

## **设计图片加载框架**

1. 异步加载：最少两个线程池

2. 切换到主线程：Handler

3. 缓存：LruCache、DiskLruCache，涉及到LinkHashMap原理

4. 防止OOM：软引用、LruCache、图片压缩没展开讲、Bitmap像素存储位置源码分析、Fresco部分源码分析

5. 内存泄露：注意ImageView的正确引用，生命周期管理

6. 列表滑动加载的问题：加载错乱用tag、队满任务存在则不添加

[Android图片加载框架最全解析（七），实现带进度的Glide图片加载功能](https://blog.csdn.net/guolin_blog/article/details/78357251)

[Android图片加载框架最全解析（三），深入探究Glide的缓存机制](https://blog.csdn.net/guolin_blog/article/details/54895665)

[**Android 图片加载之Glide缓存策略**](https://www.imooc.com/article/279266)

[面试官：简历上最好不要写Glide，不是问源码那么简单](https://juejin.cn/post/6844903986412126216#heading-10)

# Okhttp

- OKHttp有哪些拦截器，分别起什么作用
- OkHttp怎么实现连接池
- OkHttp里面用到了什么设计模式
- 怎么取消一个请求
- 断点续传

(以下还没解决)

- 1.Okhttp源码流程,线程池

- 2.Okhttp拦截器,addInterceptor 和 addNetworkdInterceptor区别

- 3.Okhttp责任链模式

- 4.Okhttp缓存怎么处理

- 5.Okhttp连接池和socket复用

  

1）首先，`ConectionPool`中维护了一个双端队列`Deque`，也就是两端都可以进出的队列，用来存储连接。
 2）然后在`ConnectInterceptor`，也就是负责建立连接的拦截器中，首先会找可用连接，也就是从连接池中去获取连接，具体的就是会调用到`ConectionPool`的get方法。也就是遍历了双端队列，如果连接有效，就会调用acquire方法计数并返回这个连接。

3）如果没找到可用连接，就会创建新连接，并会把这个建立的连接加入到双端队列中，同时开始运行线程池中的线程，其实就是调用了`ConectionPool`的put方法。其实这个线程池中只有一个线程，是用来清理连接的，也就是上述的`cleanupRunnable`

这个`runnable`会不停的调用cleanup方法清理线程池，并返回下一次清理的时间间隔，然后进入wait等待。

也就是当如果空闲连接`maxIdleConnections`超过5个或者keepalive时间大于5分钟，则将该连接清理掉。

同时通过对`StreamAllocation`的引用计数实现自动回收。

**断点续传**

step 1：判断检查本地是否有下载文件，若存在，则获取已下载的文件大小 downloadLength，若不存在，那么本地已下载文件的长度为 0

step 2：获取将要下载的文件总大小（HTTP 响应头部的 content-Length)

step 3：比对已下载文件大小和将要下载的文件总大小（contentLength），判断要下载的长度

step 4：再即将发起下载请求的 HTTP 头部中添加即将下载的文件大小范围（Range: bytes = downloadLength - contentLength)

[谈谈OKHttp的几道面试题](https://www.jianshu.com/p/7cb9300c6d71)

[android 快速请求取消,Android OkHttp + Retrofit 取消请求的方法](https://blog.csdn.net/weixin_35439783/article/details/117777344)

[Android Okhttp 断点续传面试解析](https://juejin.cn/post/6844903854115389447)

# LeakCanary

- 如何确定内存泄露的对象

- 如何确定从GC root到泄露对象的引用链

- 如何将这些生命周期对象纳入监测

在确定待检测对象与时机后，查看ObjectWatcher的expectWeaklyReachable方法，可以得知如何实现将泄露对象从待检测对象（默认即上节我们分析的那些有生命周期的类对象）挑出来的。确定内存泄露对象的原理是我们常用的WeakReference，其双参数构造函数支持传入一个ReferenceQueue，当其关联的对象回收时，会将WeakReference加入ReferenceQueue中。LeakCanary的做法是继承ReferenceQueue，增加一个值为UUID的属性key，同时将每个需要监测的对象WeakReference以此UUID作为键加入一个map中。这样，在GC过后，removeWeaklyReachableObjects方法通过遍历ReferenceQueue，通过key值删除map中已回收的对象，剩下的对象就基本可以确定发生了内存泄露。

在确定内存泄露的对象后，就需要其他手段来确定泄露对象引用链了，这一过程开始于checkRetainedObjects方法，跟踪调用可以看到启动了前台服务HeapAnalyzerService，这在我们使用LeakCanary时可以在通知栏看到。服务中调用了HeapAnalyzer的analyze方法进行堆内存分析，由Shark库实现该功能，就不再进行追踪。

官方的原理简单来解释就是这样的：**在一个Activity执行完onDestroy()之后，将它放入WeakReference中，然后将这个WeakReference类型的Activity对象与ReferenceQueque关联。这时再从ReferenceQueque中查看是否有没有该对象，如果没有，执行gc，再次查看，还是没有的话则判断发生内存泄露了。最后用HAHA这个开源库去分析dump之后的heap内存。**

[LeakCanary检测内存泄漏原理](http://www.androidchina.net/11559.html)

[WeakReference应用-LeakCanary检测内存泄漏](https://blog.csdn.net/inwhites/article/details/93141379)

[看完这篇 LeakCanary 原理分析，又可以虐面试官了](https://zhuanlan.zhihu.com/p/73675401)

# Arouter

## 原理

我们在代码里加入的@Route注解，会在编译时期通过apt生成一些存储path和activityClass映射关系的类文件，然后app进程启动的时候会拿到这些类文件，把保存这些映射关系的数据读到内存里(保存在map里)，然后在进行路由跳转的时候，通过build()方法传入要到达页面的路由地址，ARouter会通过它自己存储的路由表找到路由地址对应的Activity.class(activity.class = map.get(path))，然后new Intent()，当调用ARouter的withString()方法它的内部会调用intent.putExtra(String name, String value)，调用navigation()方法，它的内部会调用startActivity(intent)进行跳转，这样便可以实现两个相互没有依赖的module顺利的启动对方的Activity了。

## 源码

[探索Android路由框架-ARouter之深挖源码（二）](https://www.jianshu.com/p/5b35309e9bb2)

`_ARouter init()`

```java
public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
  
  .....
    try {
            long startInit = System.currentTimeMillis();
            //billy.qi modified at 2017-12-06
            //load by plugin first
            loadRouterMap();
            if (registerByPlugin) {
                logger.info(TAG, "Load router map by arouter-auto-register plugin.");
            } else {
                Set<String> routerMap;

                // It will rebuild router map every times when debuggable.
                if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
                    logger.info(TAG, "Run with debug mode or new install, rebuild router map.");
                    // These class was generated by arouter-compiler.
                    routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
                    if (!routerMap.isEmpty()) {
                        context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
                    }

                    PackageUtils.updateVersion(context);    // Save new version name when router map update finishes.
                } else {
                    logger.info(TAG, "Load router map from cache.");
                    routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
                }

                logger.info(TAG, "Find router map finished, map size = " + routerMap.size() + ", cost " + (System.currentTimeMillis() - startInit) + " ms.");
                startInit = System.currentTimeMillis();

                for (String className : routerMap) {
                    if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                        // This one of root elements, load root.
                        ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                        // Load interceptorMeta
                        ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                        // Load providerIndex
                        ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                    }
                }
            }
}
```

1. 初始化操作，内部调用`LogisiticsCenter init`帮我们管理逻辑
2. 第一次加载，`ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);`先通过`com.alibaba.android.arouter.routes`包名，扫描下面包含的所有ClassName,后面对其进行存储Sp。第二次加载就直接从Sp中获取。
3. 然后遍历、匹配，满足条件的添加到具体的集合中，按照文件的前缀不同，将他们太假到路哟偶映射表中Warehouse的groupsIndex、interceptorsIndex、providersIndex 中

`_ARouter.getInstance().build(path)`

```java
  protected Postcard build(String path) {
        if (TextUtils.isEmpty(path)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                path = pService.forString(path);
            }
            return build(path, extractGroup(path));
        }
    }
```

根据`PathReplaceService`获得预处理路径，这个接口是`Iprovider`的子类

extractGroup(path)这个方法，这个方法主要是获取分组名称。切割path字符串，默认为path中第一部分为组名。这就证明了如果我们不自定义分组，默认就是第一个分号的内容。

build方法，最终返回的是一个Postcard对象。

`_ARoter navigation(Class<? extends T> service)`会调用`LogisticsCenter.completion(postcard)`。

1. 首先，根据path在Warehouse.routes映射表中查找对应的RouteMeta。但是，第一次加载的时候，是没有数据的。(而第一次加载是在LogisticsCenter.init()中，上面也说了。因此只有`Warehouse`的`groupsIndex、interceptorsIndex、providersIndex` 有数据)，因此这个时候routeMeta=null。所以，这个时候会先从Warehouse.groupsIndex中取出类名前缀为com.alibaba.android.arouter.routes.ARouter$$Group$$group的文件
2. 接着，将我们添加@Route注解的类映射到Warehouse.routes中；
3. 然后将已经加载过的组从Warehouse.groupsIndex中移除，这样也避免了重复添加进Warehouse.routes；注意，这个时候Warehouse.routes已经有值了，所以重新调用本方法（completion(postcard)）执行了else代码块。

> 完成了对Warehouse.providers、Warehouse.routes的赋值。

那么Warehouse.interceptors又是在哪里赋值的呢？

```java
//LogisticsCenre.java

provider = providerMeta.getConstructor().newInstance();
                            provider.init(mContext);
                            Warehouse.providers.put(providerMeta, provider);
                            instance = provider;

//InterceptorServiceImpl.java
 public void init(final Context context) {
   .....
      IInterceptor iInterceptor = interceptorClass.getConstructor().newInstance();
                            iInterceptor.init(context);
                            Warehouse.interceptors.add(iInterceptor);
 }
```

## ARouter缺陷

`ARouter`的缺陷就在于拿到这个`Map`的过程，我们在使用`ARouter`时都需要初始化，`ARouter`所做的即是在初始化时利用反射扫描指定包名下面的所有`className`，然后再添加`map`中

```java
//源码代码，插桩前
private static void loadRouterMap() {
	//registerByPlugin一直被置为false
    registerByPlugin = false;
}
//插桩后反编译代码
private static void loadRouterMap() {
    registerByPlugin = false;
    register("com.alibaba.android.arouter.routes.ARouter$$Root$$modulejava");
    register("com.alibaba.android.arouter.routes.ARouter$$Root$$modulekotlin");
    register("com.alibaba.android.arouter.routes.ARouter$$Root$$arouterapi");
    register("com.alibaba.android.arouter.routes.ARouter$$Interceptors$$modulejava");
    register("com.alibaba.android.arouter.routes.ARouter$$Providers$$modulejava");
    register("com.alibaba.android.arouter.routes.ARouter$$Providers$$modulekotlin");
    register("com.alibaba.android.arouter.routes.ARouter$$Providers$$arouterapi");
}

```

默认通过扫描`dex`的方式进行加载,通过`gradle`插件进行自动注册可以缩短初始化时间,同时解决应用加固导致无法直接访问`dex`文件，初始化失败的问题

[ARouter原理与缺陷解析](https://shenzhen2017.github.io/blog/2021/03/aruter-good-bad.html)







