相信大家平时经常用到`Lifecycle`，对它怎么使用应该已经相当熟悉了吧，所以今天省略这一块内容。

**想个问题，它解决了什么痛点？**

在真实的应用中，最终会有太多管理界面和其他组件的调用，以响应生命周期的当前状态。管理多个组件会在生命周期方法（如 onStart() 和 onStop()）中放置大量的代码，这使得它们难以维护。同时也无法保证组件会在 Activity/Fragment停止后不执行启动。

那`Lifecycle`是怎么解决这些问题的呢，我们直接进入分析源码正题吧。

## Lifecycle类

分析之前先看下Lifecycle内部做了什么事情？

```java
public abstract class Lifecycle {
    //添加 LifecycleObserver，当 LifecycleOwner 更改状态时将收到通知。
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);
    //从观察者列表中删除给定的观察者。
    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);
    //返回生命周期的当前状态
    public abstract State getCurrentState();

//生命周期事件，对应Activity生命周期方法
    public enum Event {
        ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
      //可以响应任意一个事件
        ON_ANY  
    }
    
    //生命周期状态. （Event是进入这种状态的事件）
    public enum State {
        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;

        //判断至少是某一状态
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
```

可以看到提供了两种枚举来关联组件的生命周期状态：

- `Event`：生命周期事件，对应Activity/Fragment生命周期方法。
- `State`：生命周期状态，而Event是指你进入一种状态的事件。

好了，我们现在正式开始来分析了。

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
      //标注
        ReportFragment.injectIfNeededIn(this);
        if (mContentLayoutId != 0) {
            setContentView(mContentLayoutId);
        }
    }

          ....
        }
```

在Android Support Library 26.1.0 及其之后的版本，Activity和Fragment已经默认实现了LifecycleOwner接口，LifecycleOwner可以理解为被观察者。这里看到`getLifeCycle()`得到了`LifecycleRegistry`，这里的`LifecycleRegistry`是`LifeCyle`的具体实现。并在`onCreate()`中创建了`ReportFragment`来作为生命周期的观察，是不是感觉豁然开朗，原来是`ReportFragment`在帮我们事件分发。

```java
public class ReportFragment extends Fragment {
  
  public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
          //在API 29及以上，可以直接注册回调 获取生命周期
            activity.registerActivityLifecycleCallbacks(
                    new LifecycleCallbacks());
        }
        android.app.FragmentManager manager = activity.getFragmentManager();
     	//API29以前，使用fragment 获取生命周期
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            manager.executePendingTransactions();
        }
    }
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
              //使用LifecycleRegistry的handleLifecycleEvent方法处理事件
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
}
```

这里`injectIfNeededIn`给activity添加了`ReportFragment`，并且是没有布局的。

注意到`fragment`在这里的作用是对生命周期进行分发，里面都走了`dispatch()`方法，而它内部真正实现的是`LifecycleRegistry`的`handleLifecycleEvent`方法处理事件。

### LifecycleRegistry事件分发

```java
### LifecycleRegistry
  
  public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
  //根据event生命周期方法的相应的状态
        State next = getStateAfter(event);
  			//移动到下面的状态
        moveToState(next);
    }

 private void moveToState(State next) {
   //状态不一致则不处理
        if (mState == next) {
            return;
        }
   //设置新状态
       mState = next;
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            // we will figure out what to do on upper level.
            return;
        }
        mHandlingEvent = true;
   //同步生命周期分发
       sync();
        mHandlingEvent = false;
    }

 private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        .....
          //所有观察者都同步完了
        while (!isSynced()) {
            mNewEventOccurred = false;
            // mObserverMap储存的观察者们
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
              //这里调用observer.dispatchEvent(lifecycleOwner, event); 进行分发，也就是调用ObserverWithState里的方法
              //mState比最老观察者状态小
              backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
              //mState比最新观察者状态大  
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }

static State getStateAfter(Event event) {
        switch (event) {
            case ON_CREATE:
            case ON_STOP:
                return CREATED;
            case ON_START:
            case ON_PAUSE:
                return STARTED;
            case ON_RESUME:
                return RESUMED;
            case ON_DESTROY:
                return DESTROYED;
            case ON_ANY:
                break;
        }
        throw new IllegalArgumentException("Unexpected event value " + event);
    }
```

处理分发时，首先要获得event所对应的状态，之后`movetoState`移动到新状态，最后把生命周期状态同步到所有的观察者。`backwardPass`和`forwardPass`基本差不多，就拿`forwardPass`来看。

```java
private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
              //调用了observer.dispatchEvent对事件处理
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }
```

看到了调用了`observer.dispatchEvent`对事件处理，不在这里分析，先看完下面在分析。

所有的观察者都存放在`mObserverMap`，我们再来看下是怎么存放进去的。

回到`LifecycleRegistry`代码，当我们调用`getLifecycle().addObserver(myLocationListener)`内部实现：

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
    //通过while循环，把新的观察者的状态 连续地 同步到最新状态mState。
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
          //继续深入
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            sync();
        }
        mAddingObserverCounter--;
    }
  
}
```

主要是将观察者加入到`mObserverMap`，然后通过循环找到最新的观察者并把状态同步过去，可以把之前的事件一个个都分发过去，具有粘性。

接着来看下`ObserverWithState`，如何让加了对应注解的方法执行的?

沿着`statefulObserve.dispatchEvent()`方法继续往下看。

```java
### LifecycleRegister/ObserverWithState
static class ObserverWithState {
        State mState;
        LifecycleEventObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
          //防止重复
            mState = min(mState, newState);
          //事件通知观察者
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
```

`ObserverWithState`是`LifecycleRegister`的内部类，在这里`mState`的作用是新事件时机重新赋值，防止重复通知。

在`Lifecycling.lifecycleEventObserver(observer)`方法中会返回`LifecycleEventObserver`对象。

```java
public interface LifecycleEventObserver extends LifecycleObserver {
    //当状态转换事件发生时调用
    void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event);
}
```

再来看下`Lifechcling`做了什么操作：

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

返回到`ObserverWithState`中，获取到观察者后状态发生变化会调用`mLifecycleObserver.onStateChanged(owner, event);`其中有三个FullLifecycleObserverAdapter`、 `CompositeGeneratedAdaptersObserver`、`ReflectiveGenericLifecycleObserver`。而我们关注的是`ComponentActivity`，所以`LifecycleEventObserver`的实现类是`ReflectiveGenericLifecycleObserver`。

### LifecycleObserver方法执行

```java
### ReflectiveGenericLifecycleObserver
class ReflectiveGenericLifecycleObserver implements LifecycleEventObserver {
    private final Object mWrapped;
    private final CallbackInfo mInfo;

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Event event) {
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```

这里看到调用了`CallbackInfo`的`invokeCallbacks`来进行分发。我们先要了解下`CallbackInfo`是怎么产生的呢，那就要从`ClassesInfoCache`内部看了。

```java
### ClassInfoCache
  
CallbackInfo getInfo(Class<?> klass) {
        CallbackInfo existing = mCallbackMap.get(klass);
        if (existing != null) {
            return existing;
        }
        existing = createInfo(klass, null);
        return existing;
    }

private CallbackInfo createInfo(Class<?> klass, @Nullable Method[] declaredMethods) {
        Class<?> superclass = klass.getSuperclass();
        Map<MethodReference, Lifecycle.Event> handlerToEvent = new HashMap<>();
        if (superclass != null) {
            CallbackInfo superInfo = getInfo(superclass);
            if (superInfo != null) {
                handlerToEvent.putAll(superInfo.mHandlerToEvent);
            }
        }

        Class<?>[] interfaces = klass.getInterfaces();
  //不断的遍历各个方法，获取方法上的名为OnLifecycleEvent的注解，这个注解正是实现LifecycleObserver接口时用到的
        for (Class<?> intrfc : interfaces) {
            for (Map.Entry<MethodReference, Lifecycle.Event> entry : getInfo(
                    intrfc).mHandlerToEvent.entrySet()) {
                verifyAndPutHandler(handlerToEvent, entry.getKey(), entry.getValue(), klass);
            }
        }

        Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);
        boolean hasLifecycleMethods = false;
        for (Method method : methods) {
            OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
            if (annotation == null) {
                continue;
            }
            hasLifecycleMethods = true;
            Class<?>[] params = method.getParameterTypes();
            int callType = CALL_TYPE_NO_ARG;
            if (params.length > 0) {
                callType = CALL_TYPE_PROVIDER;
                if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
                    throw new IllegalArgumentException(
                            "invalid parameter type. Must be one and instanceof LifecycleOwner");
                }
            }
            Lifecycle.Event event = annotation.value();

            if (params.length > 1) {
                callType = CALL_TYPE_PROVIDER_WITH_EVENT;
                if (!params[1].isAssignableFrom(Lifecycle.Event.class)) {
                    throw new IllegalArgumentException(
                            "invalid parameter type. second arg must be an event");
                }
                if (event != Lifecycle.Event.ON_ANY) {
                    throw new IllegalArgumentException(
                            "Second arg is supported only for ON_ANY value");
                }
            }
            if (params.length > 2) {
                throw new IllegalArgumentException("cannot have more than 2 params");
            }
          //新建了一个MethodReference，其内部包括了使用了该注解的方法
          //这个method就是最后要调用invoke的方法
            MethodReference methodReference = new MethodReference(callType, method);
          //用于将MethodReference和对应的Event存在类型为Map<MethodReference, Lifecycle.Event> 的handlerToEvent中
            verifyAndPutHandler(handlerToEvent, methodReference, event, klass);
        }
  			//并将handlerToEvent传进去
        CallbackInfo info = new CallbackInfo(handlerToEvent);
        mCallbackMap.put(klass, info);
        mHasLifecycleMethods.put(klass, hasLifecycleMethods);
        return info;
    }
```

了解了之后，再回过来看下`invokeCallbacks`方法：

```java
### ClassesInfoCache/CallbackInfo
  
   static class CallbackInfo {
        final Map<Lifecycle.Event, List<MethodReference>> mEventToHandlers;
        final Map<MethodReference, Lifecycle.Event> mHandlerToEvent;

        CallbackInfo(Map<MethodReference, Lifecycle.Event> handlerToEvent) {
            mHandlerToEvent = handlerToEvent;
            mEventToHandlers = new HashMap<>();
          //循环的意义在于将handlerToEvent进行数据类型转换，转化为一个HashMap，key的值为事件，value的值为MethodReference。
            for (Map.Entry<MethodReference, Lifecycle.Event> entry : handlerToEvent.entrySet()) {
                Lifecycle.Event event = entry.getValue();
                List<MethodReference> methodReferences = mEventToHandlers.get(event);
                if (methodReferences == null) {
                    methodReferences = new ArrayList<>();
                    mEventToHandlers.put(event, methodReferences);
                }
                methodReferences.add(entry.getKey());
            }
        }

        @SuppressWarnings("ConstantConditions")
        void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
          //这里会传入mEventToHandlers.get(event)，也就是事件对应的MethodReference的集合。
            invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
            invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,
                    target);
        }

        private static void invokeMethodsForEvent(List<MethodReference> handlers,
                LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
            if (handlers != null) {
              //会遍历MethodReference的集合，调用MethodReference的invokeCallback方法。
                for (int i = handlers.size() - 1; i >= 0; i--) {
                    handlers.get(i).invokeCallback(source, event, mWrapped);
                }
            }
        }
    }
```

主要做了执行对应event的方法，最后会遍历MethodReference的集合，调用MethodReference的invokeCallback方法。

```java
    static final class MethodReference {
        final int mCallType;
        final Method mMethod;

        MethodReference(int callType, Method method) {
            mCallType = callType;
            mMethod = method;
            mMethod.setAccessible(true);
        }

        void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
            //noinspection TryWithIdenticalCatches
            try {
                switch (mCallType) {
                    case CALL_TYPE_NO_ARG:
                    //没有参数的
                        mMethod.invoke(target);
                        break;
                    case CALL_TYPE_PROVIDER:
                    //一个参数的：LifecycleOwner
                        mMethod.invoke(target, source);
                        break;
                    case CALL_TYPE_PROVIDER_WITH_EVENT:
                    ////两个参数的：LifecycleOwner，Event
                        mMethod.invoke(target, source, event);
                        break;
                }
            } catch (InvocationTargetException e) {
                throw new RuntimeException("Failed to call observer method", e.getCause());
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
        }
       ......
    }
```

在`MethodReference`根据callType的类型通过invoke对方法进行反射。

最后一张图来概括下今天所看的内容：

![4B3E630D01764C4F068FF09A18916011](../../../../Library/Application Support/typora-user-images/4B3E630D01764C4F068FF09A18916011.jpg)

​								图片来自--[Android Jetpack架构组件（三）一文带你了解Lifecycle（原理篇](https://juejin.cn/post/6844903977788817416)

**参考**

[Lifecycle官方文档](https://developer.android.com/topic/libraries/architecture/lifecycle)

[Android Jetpack架构组件（三）一文带你了解Lifecycle（原理篇](https://juejin.cn/post/6844903977788817416)

[Jetpack AAC完整解析（一）Lifecycle 完全掌握！](https://juejin.cn/post/6893870636733890574)

[Android Jetpack--lifecycle全解析](https://blog.csdn.net/verymrq/article/details/89356843)

