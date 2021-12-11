前文说到，Activity启动过程在`ATMS`中绕了一大圈，最终还是通过调用`ClientTransaction`的`schedule`方法，回到了`ApplicationThread`中。那我们就接着往下看启动过程。

## ActivityThread启动Activity

我们来看下`ApplicadtionThread`的`scheduleTransaction`方法：

```java
### ActivityThread/ApplicationThread
  public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
```

实际上最终实现调用的是在`ActivityThread`的父类`ClientTransactionHandler`中：

```java
### ClientTransactionHandler 
void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
```

方法很简单，就是发送一个启动消息交由Handler处理，这个handler有着一个很简单的名字：H。在这里地调用了`sendMessage`方法向H类发送类型为`EXECUTE_TRANSACTION`消息，sendMessage方法如下所示：

```java
### ActivityThread
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) {
            Slog.v(TAG,
                    "SCHEDULE " + what + " " + mH.codeToString(what) + ": " + arg1 + " / " + obj);
        }
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
```

这里mH指的是H，它是ActivityThread的内部类并继承Handler，是应用程序进程中`主线程的消息管理类`。

**再想下这个消息是从哪里发送出来的？**我们知道，服务器的Binder方法运行在Binder的线程池中，也就是说系统进行跨进程调用ApplicationThread的scheduleTransaction就是执行在Binder的线程池中的了。

接着来看下Handler H对消息的处理，如下所示：

```java
### ActivityThread
final H mH = new H();

class H extends Handler {
        public static final int BIND_APPLICATION        = 110;
       ......
        public static final int EXECUTE_TRANSACTION = 159;
        public static final int RELAUNCH_ACTIVITY = 160;

       ......
         
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                case EXIT_APPLICATION:
                    if (mInitialApplication != null) {
                        mInitialApplication.onTerminate();
                    }
                    Looper.myLooper().quit();
                    break;
                
               .......
                 
                case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        //系统进程内的客户端事务在客户端而不是 ClientLifecycleManager 被回收，以避免在处理此消息之前被清除。
                        transaction.recycle();
                    }
                    // 回收本地预定的事务。
                    break;
                case RELAUNCH_ACTIVITY:
                    handleRelaunchActivityLocally((IBinder) msg.obj);
                    break;
                .......
            }
            .......
        }
    }
```

查看H的` handleMessage`方法中对`EXECUTE_TRANSACTION`的处理。取出`ClentTransaction`实例，调用`execute`方法。

```java
### TransactionExector
public void execute(ClientTransaction transaction) {
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Start resolving transaction");

        final IBinder token = transaction.getActivityToken();
       
        ......

        executeCallbacks(transaction);

        executeLifecycleState(transaction);
        mPendingActions.clear();
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "End resolving transaction");
    }
```

接着来看`executeCallbacks`方法：

```java
### TransactionExector
      public void executeCallbacks(ClientTransaction transaction) {
        final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
        ......

        final int size = callbacks.size();
        for (int i = 0; i < size; ++i) {
            final ClientTransactionItem item = callbacks.get(i);
            if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Resolving callback: " + item);
            final int postExecutionState = item.getPostExecutionState();
            final int closestPreExecutionState = mHelper.getClosestPreExecutionState(r,
                    item.getPostExecutionState());
            if (closestPreExecutionState != UNDEFINED) {
                cycleToPath(r, closestPreExecutionState, transaction);
            }

          //关键代码
            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
            if (r == null) {
                // Launch activity request will create an activity record.
                r = mTransactionHandler.getActivityClient(token);
            }

           ......
        }
    }
```

可以看到遍历了`Callbacks`，调用了`execute`方法。那这里的`item`又是什么？

```java
### ActivityStackSupervisor.realStartActivityLocked
clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                        r.assistToken, r.createFixedRotationAdjustmentsIfNeeded()));
```

从上篇文章知道，放入callbacks里的其实是`ClientTransactionItem`的子类`LaunchActivityItem`，所以我们关注下`launchActivityItem`中的`execute`方法。

```java
### launchActivityItem
public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client, mAssistToken, mFixedRotationAdjustments);
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
```

这里`ClientTransactionHandler`是`ActivityThread`的父类。又将启动Activity的参数封装成ActivityClientRecord，最后调用了`handleLaunchActivity`方法。

## Activity启动核心实现

> 下面开始启动过程代码基本Android`7.0`,`8.0`,`9.0`,`10.0`一样

```java
### ActivityThread
      public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
      
        ......

        // 在创建活动之前初始化
        if (!ThreadedRenderer.sRendererDisabled
                && (r.activityInfo.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            HardwareRenderer.preload();
        }
        WindowManagerGlobal.initialize();

        GraphicsEnvironment.hintActivityLaunch();

  			//启动Activity
        final Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            if (!r.activity.mFinished && pendingActions != null) {
                pendingActions.setOldState(r.state);
                pendingActions.setRestoreInstanceState(true);
                pendingActions.setCallOnPostCreate(true);
            }
        } else {
            try {
              //停止Activity启动
                ActivityTaskManager.getService()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

        return a;
    }
```

`performLaunchActivity`方法最终完成了`Activity`对象的创建和启动过程。我们再来看下`performActivity`方法做了什么。

```java
### ActivityThread
//核心实现
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
  			//从ActivityClientRecord中后去待启动的Activity的组件信息
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
          //获取APK文件的描述类LoadedApk
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

  			//ComponentName类中保存了该Activity的包名和类名
        ComponentName component = r.intent.getComponent();  
        ......

  			//创建要启动Activity的上下文环境
        //Context中的大部分逻辑都是由ContextImpl来完成
        ContextImpl appContext = createBaseContextForActivity(r);  
  
  		//通过Instrumentation的newActivity方法使用类加载创建Activity对象。
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
          //根据ComponentName中存储的Activity类名，用类加载器来创建该Activity的实例
            activity = mInstrumentation.newActivity(  
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
          //创建Application，makeApplication方法内部会调用Application的onCreate方法
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);   //注释1

            .......

            if (activity != null) {
                .......
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }

                // 必须使用与应用程序上下文相同的加载器来初始化活动资源。
                appContext.getResources().addLoaders(
                        app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

                appContext.setOuterContext(activity);
              	// 初始化Activity
                //attach方法中会创建Window对象（PhoneWindow）并与Activity自身进行关联。
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                .......
                  
                activity.mCalled = false;
                if (r.isPersistable()) {
                  //调用Instrumentation的callActivityOnCreate方法来启动Activity
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);      //注释2
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
               .......
                 
                r.activity = activity;
                mLastReportedWindowingMode.put(activity.getActivityToken(),
                        config.windowConfiguration.getWindowingMode());
            }
            r.setState(ON_CREATE);
            synchronized (mResourcesManager) {
                mActivities.put(r.token, r);
            }

        } catch (SuperNotCalledException e) {
            throw e;

        } 
 			 ........

        return activity;
    }
```

再来看下注释1 处，**Application是怎么被创建出来的**? ` makeApplication`方法，如下所示：

```java
### LoadApk  
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }
        .......
          
       Application app = null;

       String appClass = mApplicationInfo.className;
  			if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }
       .......
       
        try {
            final java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                        "initializeJavaContextClassLoader");
                initializeJavaContextClassLoader();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
          ......

            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            NetworkSecurityConfigProvider.handleNewApplication(appContext);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
           .......
        }  
         

 			  mActivityThread.mAllApplications.add(app);
        mApplication = app;
        if (instrumentation != null) {
            try {
              //通过Instrumentation来完成
                instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!instrumentation.onException(app, e)) {
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        }

        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        return app;
    }
```

Applicatiion对象的创建也是通过`Instrumentation`来完成的，这个过程和Actiivty对象的创建一样，都是通过类加载器来实现的。

注释2处会调用Instrumentation的callActivityOnCreate方法来启动Activity。

```java
### Instrumenttation
  //执行活动的 Activity.onCreate 方法的调用。
   public void callActivityOnCreate(Activity activity, Bundle icicle,
            PersistableBundle persistentState) {
        prePerformCreate(activity);
        activity.performCreate(icicle, persistentState);
        postPerformCreate(activity);
    }
```

最终还是调用到了`Activity`的`performCreate`方法。

```java
### Activity
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        dispatchActivityPreCreated(icicle);
        ......
          
        if (persistentState != null) {
            onCreate(icicle, persistentState);
        } else {
            onCreate(icicle);
        }
        ......
        dispatchActivityPostCreated(icicle);
    }
```

由于`Activity`的`onCreate`方法已经被调用，这也意味着`Activity`已经完成了整个启动过程。

总结下：`performLaunchActivity`完成了下面的事情。

1. 从ActivityClientRecord获取待启动的Activity的组件信息
2. 创建ContextImpl对象并通过activity.attach方法对重要数据初始化，关联了Context的具体实现ContextImpl，attach方法内部还完成了window创建，这样Window接收到外部事件后就能传递给Activity了
3. 通过LoadedApk的makeApplication方法创建Application对象，内部也是通过mInstrumentation使用类加载器，创建后就调用了instrumentation.callApplicationOnCreate方法，也就是Application的onCreate方法。
4. 调用Activity的onCreate方法，是通过 mInstrumentation.callActivityOnCreate方法完成。

## Activity的onResume何时调用

走了一圈，都没看到onStart和onResume在哪里被调用，是不是被我们忽略了？

再回过头看下`ActivityStackSuperviso`r`的realStartActivityLocked`方法:

```java
### ActivityStackSupervisor
  boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
					......
                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                //这里ResumeActivityItem
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);
  
  			.......
}
```

惊喜的发现了`ResumeActivityItem`和`PauseActivityItem`实例。

首先我们要知道这些代表的含义：

- LaunchActivityItem 远程App端的onCreate生命周期事务

- ResumeActivityItem 远程App端的onResume生命周期事务

- PauseActivityItem 远程App端的onPause生命周期事务

- StopActivityItem 远程App端的onStop生命周期事务

- DestroyActivityItem 远程App端onDestroy生命周期事务

现在我们来关注先`ResumeActivityItem`

```java
### ClientTrasaction
public void setLifecycleStateRequest(ActivityLifecycleItem stateRequest) {
  //事务后客户端活动应处于的最终生命周期状态。
        mLifecycleStateRequest = stateRequest;
    }
```

被添加进了`mLifecycleStateRequest`中，`mLifecycleStateRequest`表示事务后客户端活动应处于的最终生命周期状态。

继续看下哪里被使用到？我们返回处理`ActivityThread.H.EXECUTE_TRANSACTION`的消息处理，也就是`TransactioonExcutor`的`execute`方法。

```java
  ### TransactionExcutor
  public void execute(ClientTransaction transaction) {
       ......

        executeCallbacks(transaction);

        executeLifecycleState(transaction);
        mPendingActions.clear();
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "End resolving transaction");
    }
```

之前看的是`excuteCallbacks`方法，现在我们来看下`executeLifecycleState`方法：

```java
### TransactioonExcutor
private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        if (lifecycleItem == null) {
            // No lifecycle request, return early.
            return;
        }

        final IBinder token = transaction.getActivityToken();
  			final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
        .....

        //执行最终的状态
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
```

在调用它的`excuter`方法，实际上是调用`ResumeActivityItem`方法。

```java
public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
        client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
                "RESUME_ACTIVITY");
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
```

最后还是走了`ActivityThread`的`handleResumeActivity`方法,来调用被启动Acitivity的`onResume`这一生命周期方法。

`handleResumeActivity`主要做了以下事情：

1. 调用生命周期：通过performResumeActivity方法，内部调用生命周期onStart、onResume方法
2. 设置视图可见：通过activity.makeVisible方法，添加window、设置可见。（所以视图的真正可见是在onResume方法之后）

在`performResumeActivity`内部又会调用了Activity的`performResume`的方法，最后的最后我们来看一眼：

```java
   ### Activity
   final void performResume(boolean followedByPause, String reason) {
        dispatchActivityPreResumed();
        performRestart(true /* start */, reason);

        ......
        // 运行onResume方法
        mInstrumentation.callActivityOnResume(this);
        EventLogTags.writeWmOnResumeCalled(mIdent, getComponentName().getClassName(), reason);
        
        .......

        // 运行Fragment的onResume方法
        mFragments.dispatchResume();
        mFragments.execPendingActions();

       	.....
        dispatchActivityPostResumed();
    }
```

在里面先后调用了`performRestart()`，是最后的`Activity的onStart()`了，然后调用了`mInstrumentation.callActivityOnResume`，是最后的`Activity的onResume()`方法。

好了，这次真的结束了。

## 总结

在app启动过程中会涉及到4个进程分别是Zygote进程、Launcher进程、AMS所在进程（SyetemServer进程）、应用程序进程，如图所示：

![image-20211128183323403](../../../../Library/Application Support/typora-user-images/image-20211128183323403.png)

图片来自--[Android8.0 根Activity启动过程](http://liuwangshu.cn/framework/component/7-activity-start-2.html)

首先Launcher进程向AMS请求创建根Activity，AMS会判断根Activity所需的应用程序进程是否存在并启动，如果不存在就会请求Zygote进程创建应用程序进程。应用程序进程准备就绪后会通知AMS，AMS会请求应用程序进程创建根Activity。上图中步骤2采用的是Socket通信，步骤1和步骤4采用的是Binder通信。

> Binder里有很多线程在跑。fork会把进程里面当前线程复制过去，当线程里某个资源被其他资源锁住时，当fork后线程信息丢失了（fork原理），最后没有开锁的钥匙了导致死锁。socket会把其他线程停掉，fork后是干净的



App启动全过程：

1. 点击app图标，Launche进程会调用startActivitySafely方法，内部会调用startActivity方法。
2. startActivity方法实际会调用Insturmention的execStartActivity方法，然后通过Binder方式发送给了ATMS，经过ActivityStart执行处理Intent和Flag信息，然后再 交给ActivityStack处理Activity进栈相关流程。
3. 之后通过ActivityStackSupvisor判断根Activity所需的应用程序进程是否存在并启动，如果不存在则会调用AMS中的startProcessLocked，内部会调用process.start，并通过连接调用Zygote的native方法fork出一个进程，也就是ActivityThread。
4. 在ActivityThread内部调用main方法，会开启一个loop循环执行。在attach内部去会发起发起attchApplication任务。在AMS中做一些配置工作，然后让ActivityThread执行handlebindApplication操作，Application被创建初始化。
5. 之后通过ATMS调用attachApplication方法，内部遍历activity栈，然后调用mStackSupervisor.realStartActivityLocked方法，将通过Binder方式调用ApplicationThread把启动信息以H对象发送给主线程，发送消息的类型是EXECUTE_TRANSACTION，消息体是ClientTransaction，实际上是LaunchActivityItem。拿到启动消息后，最终调用performLaunchActivity方法，调用Instrumentation类的newActivity方法，其内通过ClassLoader创建Activity2实例。
6. 通知Activity2去performCreate。

> 也就是说，正常情况下Application的初始化是在handleBindApplication方法中的，并且是创建进程后调用的。performLaunchActivity中只是做了一个检测，异常情况Application不存在时才会创建。

**参考**

《Android开发艺术探索》

[Android深入四大组件（六）Android8.0 根Activity启动过程（前篇）](http://liuwangshu.cn/framework/component/6-activity-start-1.html)

[Android深入四大组件（七）Android8.0 根Activity启动过程（后篇）](http://liuwangshu.cn/framework/component/7-activity-start-2.html)

[Activity的启动过程详解（基于Android10.0）](https://juejin.cn/post/6847902222294990862#heading-2)

[进程内Activity启动过程源码研究，基于 android 9.0](https://www.jianshu.com/p/3985a358a92d)
