## 前言

Activity的启动分为两种，一个是根Activity的启动过程，另一种是普通Activity的启动过程。而第一种就是指一个app启动的过程，普通Activity就是指在应用中调用`startActivity`的过程。

拿根Activity的启动来讲比较全面，也很好的理解Android的整个启动过程。可以分为三个部分`Launcher请求过程`、`AMS到ApplicationThread的调用过程`和`ActivityThread启动Activity`。

把android7、8、9的启动过程看了一遍，那我们现在就基于**android 11**源码再来分析一遍，要选就选最新的来分析，相信思想都是通的。

 ## Launcher启动过程

当点击app图标后，就会调用`launcher`的`startActivitySafely`方法，内部又会调用`startActivity`方法。

```java
## Launcher.java
public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
     ...
       //设置在新的任务栈中启动
     intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
     if (v != null) {
         intent.setSourceBounds(getViewBounds(v));
     }
     try {
         if (Utilities.ATLEAST_MARSHMALLOW
                 && (item instanceof ShortcutInfo)
                 && (item.itemType == Favorites.ITEM_TYPE_SHORTCUT
                  || item.itemType == Favorites.ITEM_TYPE_DEEP_SHORTCUT)
                 && !((ShortcutInfo) item).isPromise()) {
             startShortcutIntentSafely(intent, optsBundle, item);
         } else if (user == null || user.equals(Process.myUserHandle())) {
           //调用startActivity来启动页面
             startActivity(intent, optsBundle);//2
         } else {
             LauncherAppsCompat.getInstance(this).startActivityForProfile(
                     intent.getComponent(), user, intent.getSourceBounds(), optsBundle);
         }
         return true;
     } catch (ActivityNotFoundException|SecurityException e) {
         Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
         Log.e(TAG, "Unable to launch. tag=" + item + " intent=" + intent, e);
     }
     return false;
 }
```

这其中会设置intent的Flag为`Intent.FLAG_ACTIVITY_NEW_TASK`开启一个新的任务栈，真正调用的会再`activity`的`startActivity`方法。

```java
### Activity
      public void startActivity(Intent intent, @Nullable Bundle options) {
        ......
          
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            //我们希望通过此调用与可能已覆盖该方法的应用程序兼容。
            startActivityForResult(intent, -1);
        }
    }
```

都会走到`startActivityForResult`方法里，第二个参数为-1，表示Launcher不需要知道Activity启动的结果。

```java
### Activitu.java   
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
           ......

            cancelInputsAndStartExitTransition(options);
        } else {
            ......
        }
    }
```

由于是第一次创建`mParent = null`，内部会调用了`Instrumentation`的`execStartActivity`方法。`Instrumentation`具有跟踪application及生命周期的方法。

```java
### Instrumentation    
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ......
       
        try {
            intent.migrateExtraStreamToClipData(who);
            intent.prepareToLeaveProcess(who);
            int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getBasePackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

（ActivityTaskManager andrrod 10.0新增方法，getService这个是android 8.0后新增方法）首先会调用`ActivityTaskManager.getService()`获取AMS的代理对象，接着调用`startActivity`方法、那又是如何获取到AMS的代理对象的呢？

```java
### ActivityTaskManager
 public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
    }
    
 private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                  //获取到ACTIVITY_TASK_SERVICE的引用，也就是IBinder类型的ATMS的引用
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                  //要实现进程间通信，服务端也就是AMS只需要继承IActivityManager.Stub类并实现相应的方法就可以了。
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };
```

可以看到这里获取了一个跨进程的服务，也就是`**ActivityTaskManagerService**`，它继承于`IActivityTaskManager.Stub，是个Binder对象。`知道了这是什么后，就可以回到`Instrumentation`的`execStartActivity`方法往下看。

## ATMS到ApplicationThread过程

接着我们就进到了系统提供的服务`ATMS`中，接着是ATMS到ApplicationThread的调用流程，时序图如图：

![ATMS时序图 ](../../../../Downloads/ATMS时序图 .png)

接下来继续分析，已经走到了ATMS的`startActivity`方法。

```java
 ### ActivityTaskManagerService 
 public final int startActivity(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions) {
   //可以发现startActivityAsUser方法比startActivity方法多了一个参数UserHandle.getCallingUserId()，这个方法会获得调用者的UserId
   //AMS会根据这个UserId来确定调用者的权限。
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }

 public int startActivityAsUser(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);
    }

  private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
        assertPackageMatchesCallingUid(callingPackage);
    //判断调用者进程是否被隔离   
        enforceNotIsolatedCaller("startActivityAsUser");

    //检查调用者权限
        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // 获取ActivityStart对象，并在最后excute启动activity.
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setCallingFeatureId(callingFeatureId)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setUserId(userId)
                .execute();

    }
```

经过一层套一层地调用，来到了最后，通过`etActivityStartController().obtainStarter`获取ActivityStart对象，并在最后`excute`方法来启动activity。

```java
### ActivityStarter
  //执行开始活动的请求
int execute() {
        try {
            ......

            if (mRequest.activityInfo == null) {
                mRequest.resolveActivity(mSupervisor);
            }

            int res;
            synchronized (mService.mGlobalLock) {
                final boolean globalConfigWillChange = mRequest.globalConfig != null
                        && mService.getGlobalConfiguration().diff(mRequest.globalConfig) != 0;
                final ActivityStack stack = mRootWindowContainer.getTopDisplayFocusedStack();
              ......

                final long origId = Binder.clearCallingIdentity();

                res = resolveToHeavyWeightSwitcherIfNeeded();
                if (res != START_SUCCESS) {
                    return res;
                }
              
              //执行启动接界面请求
                res = executeRequest(mRequest);

                Binder.restoreCallingIdentity(origId);

                ......
                return getExternalResult(mRequest.waitResult == null ? res
                        : waitForResult(res, mLastStartActivityRecord));
            }
        } finally {
            onExecutionComplete();
        }
    }
```

（executeRequest是Android11新增方法）再来看下`executeRequest`方法，关注下主要的方法：

```java
### ActivityStarter.java
  
  //执行活动启动请求，开启活动启动之旅。首先是执行几个初步检查。正常的活动启动流程会经过
  private int executeRequest(Request request) {
  .....
  mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
                restrictedBgActivity, intentGrants);

        if (request.outActivity != null) {
            request.outActivity[0] = mLastStartActivityRecord;
        }

        return mLastStartActivityResult;
}
```

里面有调用`startActivityUnchecked`方法。

```java
###   ActivityStarter
//主要处理栈管理相关的逻辑
//初步检查并确认调用者拥有执行此操作所需的权限后开始一项活动
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, Task inTask,
                boolean restrictedBgActivity, NeededUriGrants intentGrants) {
        int result = START_CANCELED;
        final ActivityStack startedActivityStack;
        try {
            mService.deferWindowLayout();
            Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "startActivityInner");
            result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, restrictedBgActivity, intentGrants);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
            startedActivityStack = handleStartResult(r, result);
            mService.continueWindowLayout();
        }

        postStartActivityProcessing(r, result, startedActivityStack);

        return result;
    }
```

`startActivityUnchecked`主要处理栈管理相关的逻辑。之后调用了`startActivityInner`方法，最后调用到了`RootWindowContainer` 的`resumeFocusedStacksTopActivities`方法。

```java
### ActivityStarter
  //启动活动并确定活动是否应添加到现有任务的顶部或向现有活动传递新意图。还将活动任务操作到请求的或有效的堆栈显示上。
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            boolean restrictedBgActivity, NeededUriGrants intentGrants) {
          
  ......
    
  if (newTask) {
            final Task taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                    ? mSourceRecord.getTask() : null;
    //内部会创建一个新的Activity任务栈
            setNewTask(taskToAffiliate);
            if (mService.getLockTaskController().isLockTaskModeViolation(
                    mStartActivity.getTask())) {
                Slog.e(TAG, "Attempted Lock Task Mode violation mStartActivity=" + mStartActivity);
                return START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
        } else if (mAddingToTask) {
            addOrReparentStartingActivity(targetTask, "adding to task");
        }
  
  ......
    //主要关注这行代码
    mRootWindowContainer.resumeFocusedStacksTopActivities(
                        mTargetStack, mStartActivity, mOptions);  1
  ......
    return START_SUCCESS;
            }
```

`RootWindowContainer`是Android10新增的类，分担了之前`ActivityStackSupervisor`的部分功能。`resumeFocusedStacksTopActivities`内部又会调用`ActivityStack`的`resumeTopActivityUncheckedLocked`方法，接着我们进去看下这个方法。

```java
### ActivityStack
      boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mInResumeTopActivity) {
            // 如果顶部Activity正在执行则不处理
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mInResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);

            //恢复top Activity时，可能需要暂停top Activity
            final ActivityRecord next = topRunningActivity(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mInResumeTopActivity = false;
        }

        return result;
    }
```

接着跳到`resumeTopActivityInnerLocked`方法里来看：

```java
### ActivityStack
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
  ......
  if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
    //当上一个界面活动，则会暂停上个界面
            pausing |= startPausingLocked(userLeaving, false /* uiSleeping */, next);
        }
  
  .......
  final ClientTransaction transaction =
                        ClientTransaction.obtain(next.app.getThread(), next.appToken);
  .......
  //启动了的Activity就发送ResumeActivityItem事务给客户端了
  transaction.setLifecycleStateRequest(
                        ResumeActivityItem.obtain(next.app.getReportedProcState(),
                                dc.isNextTransitionForward()));
                mAtmService.getLifecycleManager().scheduleTransaction(transaction);
  
  ......
  if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                  //冷启动 显示白屏
                    next.showStartingWindow(null /* prev */, false /* newTask */,
                            false /* taskSwich */);
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
    						//继续当前Activity，普通activity的正常启动
   							 mStackSupervisor.startSpecificActivity(next, true, true);
            }
  ......
     return true;
}
```

（这个方法有三百多行真的多！！！）做了对上个Activity的暂停操作，以及对上个界面进行一系列判断操作后，到了启动Activity前调用了`next.showStartingWindow`来展示一个Window，然后就是调用`mStackSupervisor.startSpecificActivity`，发起正常启动。

```java
### ActivityStackSupervisor
   void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) { 
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
  //是否启动了应用进程
        if (wpc != null && wpc.hasThread()) {
            try {
              //终于看到真正的启动activity的方法了
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            knownToBeDead = true;
        }

        r.notifyUnknownVisibilityLaunchedForKeyguardTransition();

        final boolean isTop = andResume && r.isTopRunningActivity();
  			//todo 分析  没有进程，就会去创建进程
        mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
    }
  
```

我们马上来看下`realStartActivityLocked`方法，有点迫不及待了。

```java
### ActivityStackSupervisor.java
    boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
       ......

        final Task task = r.getTask();
        final ActivityStack stack = task.getStack();

        beginDeferResume();

        try {
            ......
               
                final MergedConfiguration mergedConfiguration = new MergedConfiguration(
                        proc.getConfiguration(), r.getMergedOverrideConfiguration());
                r.setLastReportedConfiguration(mergedConfiguration);
                logIfTransactionTooLarge(r.intent, r.getSavedState());

                // 创建活动启动事务
          //proc.getThread也就是IApplicationThread ，就是前面提到的ApplicationThread在系统进程的代理。
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.appToken);

                final DisplayContent dc = r.getDisplay().mDisplayContent;
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                     
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                        r.assistToken, r.createFixedRotationAdjustmentsIfNeeded()));

                // 设置所需的最终状态
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
              //客户端在执行事务后应该处于的生命周期状态。
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);

            ......

        return true;
    }
```

好了我们来分析下，`ClientTransaction`可以看出来是包含了一系列消息，这些消息可以发送到客户端。这包括回调列表和最终弄生命中周期状态。再通过`ClientTransaction.obtain( proc.getThread(), r.appToken)`进行获取，其中`proc.getThread`是`IApplicationThread` ，就是前面提到的ApplicationThread在系统进程的代理。

接下来` clientTransaction`的`addCallback`方法，传入`LaunchActivityItem`，那我们来具体看下`LaunchActivityItem.obtain`调了什么?

````java
### LaunchActivityItem
  //获取使用提供的参数初始化的实例。
public static LaunchActivityItem obtain(Intent intent, int ident, ActivityInfo info,
            Configuration curConfig, Configuration overrideConfig, CompatibilityInfo compatInfo,
            String referrer, IVoiceInteractor voiceInteractor, int procState, Bundle state,
            PersistableBundle persistentState, List<ResultInfo> pendingResults,
            List<ReferrerIntent> pendingNewIntents, boolean isForward, ProfilerInfo profilerInfo,
            IBinder assistToken, FixedRotationAdjustments fixedRotationAdjustments) {
        LaunchActivityItem instance = ObjectPool.obtain(LaunchActivityItem.class);
        if (instance == null) {
            instance = new LaunchActivityItem();
        }
        setValues(instance, intent, ident, info, curConfig, overrideConfig, compatInfo, referrer,
                voiceInteractor, procState, state, persistentState, pendingResults,
                pendingNewIntents, isForward, profilerInfo, assistToken, fixedRotationAdjustments);

        return instance;
    }

````

可以是在启动前做各种设置操作。那最后又是怎么成功启动Activity的呢？

那要看下下面这个方法了`mService.getLifecycleManager().scheduleTransaction(clientTransaction)`，这里`mService`也就是`ActivityTaskManagerService`，调用`getLifeCycleManager`方法，获得了`ClientLifecycleManager`，又调用了里面的`scheduleTransaction`。

```java
### ClientLifecycleManager
 void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            transaction.recycle();
        }
    }
```

再来看下`transaction.schedule()`又做了什么。

```java
### ClientTransaction
   public void schedule() throws RemoteException {
  //它将被发送到客户端
        mClient.scheduleTransaction(this);
    }
```

看到这里调用了`IApplicationThread`的`scheduleTransaction`方法，我们也清楚了启动操作又跨进程的到了客户端。因为`IApplicationThread`是`ApplicationThread`在系统进程的代理，也就是说真正执行的地方在客户端的`ApplicationThread`中。换句话说`ApplicationThread`是ATMS所在进程和应用程序进程的通信桥梁。

## 结语

到了这里，我们也了解了Launcher的请求ATMS过程以及ATMS处理启动到ApplicationThread的过程，还有一部分ActivityThread启动Activity就放在下篇进行分析了。

**参考**

《Android开发艺术探索》

[Android深入四大组件（六）Android8.0 根Activity启动过程（前篇）](http://liuwangshu.cn/framework/component/6-activity-start-1.html)

[Android深入四大组件（七）Android8.0 根Activity启动过程（后篇）](http://liuwangshu.cn/framework/component/7-activity-start-2.html)

[Activity的启动过程详解（基于Android10.0）](https://juejin.cn/post/6847902222294990862#heading-2)

[进程内Activity启动过程源码研究，基于 android 9.0](https://www.jianshu.com/p/3985a358a92d)

