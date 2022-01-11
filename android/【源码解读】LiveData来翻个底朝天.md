## 前言

根据Jepack官方文档介绍：

> [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh_cn) 是一种可观察的数据存储器类。与常规的可观察类不同，LiveData 具有生命周期感知能力，意指它遵循其他应用组件（如 Activity、Fragment 或 Service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者。
>
> 如果观察者（由 [`Observer`](https://developer.android.com/reference/androidx/lifecycle/Observer?hl=zh_cn) 类表示）的生命周期处于 [`STARTED`](https://developer.android.com/reference/androidx/lifecycle/Lifecycle.State?hl=zh_cn#STARTED) 或 [`RESUMED`](https://developer.android.com/reference/androidx/lifecycle/Lifecycle.State?hl=zh_cn#RESUMED) 状态，则 LiveData 会认为该观察者处于活跃状态。LiveData 只会将更新通知给活跃的观察者。为观察 [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh_cn) 对象而注册的非活跃观察者不会收到更改通知。
>
> 您可以注册与实现 [`LifecycleOwner`](https://developer.android.com/reference/androidx/lifecycle/LifecycleOwner?hl=zh_cn) 接口的对象配对的观察者。有了这种关系，当相应的 [`Lifecycle`](https://developer.android.com/reference/androidx/lifecycle/Lifecycle?hl=zh_cn) 对象的状态变为 [`DESTROYED`](https://developer.android.com/reference/androidx/lifecycle/Lifecycle.State?hl=zh_cn#DESTROYED) 时，便可移除此观察者。这对于 Activity 和 Fragment 特别有用，因为它们可以放心地观察 [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh_cn) 对象，而不必担心泄露（当 Activity 和 Fragment 的生命周期被销毁时，系统会立即退订它们）。

现在我们知道了，LiveData使得数据的更新能以观察者模式被observer感知，且此感知只发生在 LifecycleOwner的活跃生命周期状态。

那么它是怎么感知生命周期变化的呢？需要取消注册吗？在设置相同的值时，订阅的观察者们会受到同样的值吗？还有一点，粘性事件是什么，又怎么防止数据倒灌？带着这些问题一起来看下源码实现。

## LiveData 怎么感知生命周期感知

在使用`LiveData.observe`订阅者的时候，也就是调用

```java
edPacketVm.grabRedPacketLiveData.observe(this, Observer {
            .....
        })
```

首先来看下`LiveData.observer`方法：

```java
### LiveData    
@MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            //在生命周期为DESTROYED，不进行订阅
            return;
        }
  			//用LifecycleBoundObserver包装对象
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
  			//LifecycleBoundObserver 是 ObserverWrapper 的子类或者实现类
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
  			//// 则抛出异常，不可以添加同一个 Observer 到两个不同的生命周期对象。
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
  		//拥有了感知生命周期的能力
        owner.getLifecycle().addObserver(wrapper);  
    }
```

可以看到当`LifecycleOwner`是`DESTROYED`状态，直接return。说明`DESTROYED`状态的组件是不允许注册的。

之后会新建LifecycleBoundObserver类，对observer进行包装，调用putIfAbsent方法，将owner和obseerver一键值对形式存储到mObservers中。最后通过该添加到Lifecycle中完成注册，这样当我们调用observer方法时，实际上是LiveData内部完成Lifecycle观察者的添加，这样LiveData会获取了观察组件生命周期的能力。

## Observer的事件回调

再来看下比较关键的类`LifecycleBoundObserver，`是如何进行回调的？

```java
//观察者的实现    
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
        @NonNull
        final LifecycleOwner mOwner;
  			boolean mActive;
        int mLastVersion = START_VERSION;

				LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
            super(observer);
            mOwner = owner;
        }
          
       @Override
        boolean shouldBeActive() {
          //判断当前传入的组件的状态是否是Active的
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }
          
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
```

LifecycleBoundObserver实现了LifecycleEventObserver方法，在onStateChanged方法里实现了生命周期状态的回调。当状态处于`DESTROYED`状态时，会移除观察者。**这也是为什么Livedata不需要做取消注册的原因。**所以当一个观察者处于`DESTROYED`状态时，将不会收到通知。

再来往下看`activeStateChanged`方法的具体实现：

```java
 private abstract class ObserverWrapper {
        final Observer<? super T> mObserver;
        boolean mActive;
        int mLastVersion = START_VERSION;

   			......

        void activeStateChanged(boolean newActive) {
          //第一次创建对象 mActive 没有赋值，则为默认值 false
        	// 刚刚调用 activeStateChanged 方法时，传入的值是 shouldBeActive() 返回的 true
            if (newActive == mActive) {
              //活跃状态 未发生变化时，不会处理。
                return;
            }
          
            mActive = newActive;
          //根据Active状态和处于Active状态的组件的数量,对Livedata进行扩展
            changeActiveCounter(mActive ? 1 : -1);
            if (mActive) {
              //观察者变为活跃，就进行数据分发
                dispatchingValue(this);
            }
        }
    }

//具体实现
 void changeActiveCounter(int change) {
        int previousActiveCount = mActiveCount;
        mActiveCount += change;
        if (mChangingActiveState) {
            return;
        }
        mChangingActiveState = true;
        try {
            while (previousActiveCount != mActiveCount) {
                boolean needToCallActive = previousActiveCount == 0 && mActiveCount > 0;
                boolean needToCallInactive = previousActiveCount > 0 && mActiveCount == 0;
                previousActiveCount = mActiveCount;
                if (needToCallActive) {
                  //活跃的观察者数量 由0变为1
                    onActive();
                } else if (needToCallInactive) {
                  //活跃的观察者数量 由1变为0
                    onInactive();
                }
            }
        } finally {
            mChangingActiveState = false;
        }
    }
```

`activityStateChange`方法处在`ObserverWrapper`中，也是LiveData的内部类。在内部会根据A处于Active状态的组件的数量，分别调用onActive和onInactive，属于Livedata的扩展使用的回调方法。

在观察者处于活跃时，就会调用`dispatchingValue`方法进行数据分发。

```java
 void dispatchingValue(@Nullable ObserverWrapper initiator) {
   //如果正在分发，则分发无效
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
    //标记正在分发
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                considerNotify(initiator);
                initiator = null;
            } else {
              //observerWrapper为空，遍历通知所有的观察者
                for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
   		////标记不处于分发状态
        mDispatchingValue = false;
    }
```

这里做了如果当前正在分发，则不会继续执行分发。走下来，不管`ObserverWrapper`是否为null，都会调用`considerNotify`方法。接着来看`considerNotify`方法。

```java
public LiveData() {
        mData = NOT_SET;
  			//初始赋值
        mVersion = START_VERSION;
    }    

private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {
            return;
        }
        
      //判断是活跃
  		//若当前observer对应owner非活跃，就会再调用activeStateChanged方法，并传入false，其内部会再次判断
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

如果非活跃的就会return掉，并再次调用`activeStateChanged`传入false，其内部会再次判断。都符合条件最终就会调用`onChange`方法，这也就是observer的事件回调。

这里我们看到，只是做了个version判断的处理，而没有做值是否相同的处理，所以**在设置相同的值时，订阅的观察者们还是会受到同样的值**。

## 数据更新  postValue/setValue

再来看下`postValue`和`setValue`是如何更新数据的。

```java
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

posetValue方法把mPostValueRunnable放到了主线程，在run方法里main最终还是调用了`setValue`方法。

```java
## setValue调用
protected void setValue(T value) {
      //检查当前线程是否在主线程
        assertMainThread("setValue");
      //version处理
        mVersion++;
        mData = value;
      //同样是分发值
        dispatchingValue(null);
    }
```

可以知道setValue是运行在主线程中的，内部调用了dispatchingValue方法。而这个也就是我们上面分析知道的，传到dispatchingValue方法里参数是null的情况，会遍历通知所有的观察者。

所以setValue和postValue方法都会调用dispatchingValue方法。

## observeForever怎么用

有没有不会被移除的观察者，答案也是有的。

```java
 public void observeForever(@NonNull Observer<? super T> observer) {
        assertMainThread("observeForever");
   			// 包装类 AlwaysActiveObserver   
        AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing instanceof LiveData.LifecycleBoundObserver) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        wrapper.activeStateChanged(true);
    }
```

方法内部直接调用了`wrapper.activeStateChanged(true)`，传入的true，保证了内部最后都会走`dispatchingValue`方法，同时用了不同的包装类`AlwaysActiveObserver`，再来看下`AlwaysActiveObserver`的代码：

```java
private class AlwaysActiveObserver extends ObserverWrapper {

        AlwaysActiveObserver(Observer<? super T> observer) {
            super(observer);
        }

        @Override
        boolean shouldBeActive() {
          //很简单，让它不依赖生命周期判断是否活跃，直接把返回值写死为true
          //                    
            return true;
        }
    }
```

很简单，让它不依赖生命周期判断是否活跃，直接把返回值写死为true。而我们上面也知道了调`considerNotify`方法时，内部要做一个是Active的状态判断`observer.shouldBeActive()`，所以这样写的话，会让组件的状态一直是Active的。

## 粘性事件是怎么发生的

完成了对LiveData源码的分析，现在我们分析下开发中会遇到的问题：

**为什么在 LiveData 有值时，调用 LiveData#observe 订阅观察者，会收到旧值？**

`粘性事件`指的是被观察者原本有值，当新注册观察者时，被观察者会将旧值发送给观察者，现在我们重新返回`considerNotify`方法开看下：

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

是不是很清楚了，因为新观察者订阅时，observer 是新创建的，所以obser.mLaseVersion还没被赋值，还是等于`START_VERSION`也就是-1。而mVersion不是第一次做分发了，所以 mVersion 肯定是大于初始值 START_VERSION -1 的，故此`observer.mLastVersion >= mVersion`出现了问题，才导致这粘性事件事件的异常出来。

具体的解决办法还是看大神的推荐篇[LiveData 数据倒灌：别问，问就是不可预期](https://zhuanlan.zhihu.com/p/161073021) 及参考 [《UnPeek-LiveData》](https://link.zhihu.com/?target=https%3A//github.com/KunMinX/UnPeek-LiveData) 源码。

最近找到一个方法：[**使用事件包装器**](https://juejin.cn/post/6844903623252508685#heading-7)来解决到倒灌

再来总结下有几种情况LiveData会分发值:

1. 调用 setValue 和 postValue 并且 LifecycleOwner 处于活跃状态时
2. LiveData 有值，并且处于活跃状态时，调用 LiveData#observe 订阅观察者
3. LiveData 有新值，也就是 ObserverWrapper 的 mLastVersion 小于 LiveData 的 mVersion，LifecycleOwner 从不活跃状态转为活跃状态时

## 总结

最后来个LiveData的关系类图：

![image-20211204232557940](../../../../Library/Application Support/typora-user-images/image-20211204232557940.png)

图片来自--[一文带你了解LiveData(原理篇）](https://juejin.cn/post/6844903982691794952)

分析源码的过程跟踪大部分关联的类都在里面了，结合这张图是不是又对LiveData有了更清晰的了解了。

**参考**

[LiveDataLiveData 概览](https://developer.android.com/topic/libraries/architecture/livedata?hl=zh_cn#work_livedata)

[Jetpack AAC完整解析（二）LiveData 完全掌握](https://juejin.cn/post/6903143273737814029#heading-7)

[一文带你了解LiveData(原理篇）](https://juejin.cn/post/6844903982691794952)

[**重学安卓：LiveData 数据倒灌 背景缘由全貌 独家解析**](https://xiaozhuanlan.com/topic/6719328450)

[[译] 在 SnackBar，Navigation 和其他事件中使用 LiveData（SingleLiveEvent 案例）](https://juejin.cn/post/6844903623252508685#heading-7)



