`Window`应该都比较清楚，它是一个抽象类，具体的实现类为`PhoneWindow`， 它对View进行管理。`WindowManager`是一个接口类，继承自接口`ViewManager`，从名称上来看它是用来管理`Window`的，它的实现类为`WindowManagerImpl`。如果我们想要对`Window`进行添加、更新和删除操作就可以使用`WindowManger`，`WindowManger`会将具体的工作交由WMS来处理，`WindowManager`和`WMS`通过`Binder`来进行跨进程通信。

## Window和WindowManager

来实现一个通过`WIndowManager`添加`Window`的过程：

```java
        Button mFloatingButton = new Button(this);
        mFloatingButton.setText("button");
        WindowManager.LayoutParams layoutParams = new WindowManager.LayoutParams(
                WindowManager.LayoutParams.WRAP_CONTENT, WindowManager.LayoutParams.WRAP_CONTENT, 0, 0, PixelFormat.TRANSPARENT);

        layoutParams.type = WindowManager.LayoutParams.TYPE_APPLICATION;

        layoutParams.flags= WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
                | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;

        layoutParams.gravity= Gravity.LEFT|Gravity.TOP;
        layoutParams.x=100;
        layoutParams.y=300;

        WindowManager windowManager = getWindowManager();

        windowManager.addView(mFloatingButton,layoutParams);

```

将一个Button添加到屏幕坐标为（100，300）的位置上。关注两个属性`Type`和`Flag`，这两个参数比较重要。

**Type属性**

指的是Window的类型，它分为三种，分别是应用Window、子Window和系统Window。

- 应用类Window对应一个Activity。
- 子Window不能单独存在，它需要附属在特定父WIindow之中（Dialog就是一个子Window）
- 系统Window是需要声明权限在能创建的Window（Toast和系统状态栏都是系统Window）

| Window类型 | 层级范围  |
| :--------: | :-------: |
|   Window   |   1~99    |
|  子Window  | 1000~1999 |
| 系统WIndow | 2000~2999 |

**Flag属性**

Window的标志就是Flag, 用于控制Window的显示，同样定义在WindowManager的内部类LaoyoutParams中，来看下几个常用的：

|              Flag               |                             描述                             |
| :-----------------------------: | :----------------------------------------------------------: |
| FLAG_ALLOW_LOCK_WHITE_SCREEN_ON |          只要窗口可见，就允许在开启状态的屏幕上锁屏          |
|       FLAG_NOT_FOCUSABLE        | 窗口不能获得输入焦点，设置改标志的同时，FLAG_NOT_TOUCH_MODAL也会被设置 |
|       FLAG_NOT_TOUCHABLE        |                    窗口不接收任何触摸事件                    |
|      FLAG_NOT_TOUCH_MODAL       | 将该窗口区域外的触摸事件，传递给其他Window，而自己只会处理窗口区域内的触摸事件 |
|       FLAG_KEEP_SCREEN_ON       |                只要窗口可见，屏幕就会一直亮着                |
|      FLAG_LAYOUT_NO_LIMITS      |                      允许窗口超过屏幕外                      |
|         FLAG_FULLSCREEN         |     隐藏所有的屏幕装饰窗口，比如游戏、播放器中的全屏播放     |
|      FLAG_SHOW_WHEN_LOCKED      |                  窗口可以在锁屏窗口之上显示                  |
|    FLAG_IGNORE_CHEEK_PRESSES    |      当用户脸贴近屏幕时（比如打电话时），不会响应此事件      |
|       FLAG_TURN_SCREEN_ON       |                     窗口显示时将屏幕点亮                     |

## Window的内部操作

每一个Window都对应一个View和一个`ViewRootImpl`，WIndow和View通过`ViewRootImpl`来建立联系的，因此Window并不是时机存在的，它是以View的形式存在。`WindowManager`对Window进行管理，说到管理那就离不开对Window的添加、更新和删除操作。

```java
//都是针对View进行操作，说明View才是Window存在的实体
public interface ViewManager
{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

看下面代码可以发现，`WindowMangerImpl`并没有直接实现Window的三大操作，而是全部交给了`WindoowManagerGlobal`来处理，下面我们分析都从`WindowmanagerGloabal开始分析`。

```java
### WindowManagerImpl  
@Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
  
  @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }

 @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }

```

### Window的添加过程

在`WindowManagerGlobal`的`addView`中做了创建`ViewRootImpl`并将View添加到列表中的操作，以及通过`ViewRootImpl`来更新界面并完成Window的添加过程。

```java
### WindowManagerGlobal
  
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
  //检查参数是否合法，如果是子Window那么还需要调整一些布局参数
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        ......

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
          
           ......
             
             //创建ViewRootImpl并将View添加到列表中
            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

          //mViews存储的所有Window所对应的View
            mViews.add(view);
          //mRoots存储的是所有Window所对应的ViewRootImpl
            mRoots.add(root);
            mParams.add(wparams);

            try {
              //通过ViewRootImpl来更新界面并完成Window的添加过程
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }
```

调用了`requestLayouot`方法，以及通过`WindowSession`最终来完成Window的添加过程，`mWIndowSession`的类型是`IWindowSession，`是一个Binder对象，真正的实现类是`Sessiion`，也就是Window的添加过程是一次IPC调用。

```java
### ViewRootImpl
  public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
  
  ......
    //在添加到窗口管理器之前安排第一个布局，以确保我们在从系统接收任何其他事件之前进行重新布局。
    requestLayout();
  
  ......
    
    try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
      
      						//会通过WindowSession最终来完成Window的添加过程
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
                } catch (RemoteException e) {
                    mAdded = false;
                    mView = null;
                    mAttachInfo.mRootView = null;
                    mInputChannel = null;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }
}

### ViewRootImpl
public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
          //实际是View绘制的入口
            scheduleTraversals();
        }
    }
```

在Session内部会通过`WindowManagerService`来实现Windw的添加。

```java
### WindowManagerService
@Override
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
            Rect outOutsets, InputChannel outInputChannel) {

  //交给它来处理
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
                outContentInsets, outStableInsets, outOutsets, outInputChannel);
    }
```

`addToDisplay`方法内部调用了`WMS`的`addWindow`方法并将自身也就是`Session`传入了进去，每个应用程序都会有一个`Session`，`WMS`会用`ArrayList`来保存起来，这样所有的工作都交给了`WMS`来做。

//WMS补充

### Window的删除过程

下面再来看下`WindowManagerGlobal`的`removeView`的实现：

```java
### WindowManagerGlobal
   public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
          //查找待删除的View的索引
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
          //进一步删除
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }
```

在`WindowManger`中提供了两种删除接口`removeView`和`removeViewImmediate`，分别标识异步删除和同步删除。主要用到了`removeViewLocked`来做进一步删除,具体的删除操作由`ViewRoootImpl`的`die`方法。

```java
### WindowManagerGlobal
private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();
  ......
    
  //通过ViewRoootImpl来完成删除操作
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }
```

在`die`方法里，如果是同步删除会直接调用`doDie()`，如果是异步删除则会发送一个`MSG_DIE`的消息。在`doDie`方法中，最终调用`dispatchDetachedFromWindow()`来真正删除View。

```java
### ViewRootImpl 
boolean die(boolean immediate) {
        // 同步删除直接调用doDie
        if (immediate && !mIsInTraversal) {
            doDie();
            return false;
        }

        if (!mIsDrawing) {
            destroyHardwareRenderer();
        } else {
            Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
                    "  window=" + this + ", title=" + mWindowAttributes.getTitle());
        }
  	//异步删除户发送一个MSG_DIE的消息，还是会调用doDie()
        mHandler.sendEmptyMessage(MSG_DIE);
        return true;
    }

### ViewRootImpl 
  //两个最终都会调用doDie方法
     void doDie() {
        checkThread();
        if (LOCAL_LOGV) Log.v(mTag, "DIE in " + this + " of " + mSurface);
        synchronized (this) {
            if (mRemoved) {
                return;
            }
            mRemoved = true;
            if (mAdded) {
              //真正删除View的逻辑是在它的内部实现
                dispatchDetachedFromWindow();
            }

            .......

            mAdded = false;
        }
        WindowManagerGlobal.getInstance().doRemoveView(this);
    }
```

### Window的更新过程

再来看下`updateViewLayout`更新操作了什么

```java
### WindowManagerGlobal
   public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;

  //更新View的LayoutParams并替换老的LayooutParams
        view.setLayoutParams(wparams);

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            ViewRootImpl root = mRoots.get(index);
            mParams.remove(index);
            mParams.add(index, wparams);
          //再更新ViewRootImpl中的LayoutParams
            root.setLayoutParams(wparams, false);
        }
    }
```

`ViewRootImpl` 的 `setLayoutParams(wparams, false)`方法最后会调用`ViewRootImpl`的`schedultTraversals`方法来对View重新布局，包括测量、布局、重绘这三个操作。同时还会通过`WindowSession`来更新Window的视图，这个过程最终是由`WindowManagerService`的`relayoutWindow`来具体实现。

```java
### ViewRootImpl
 void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
          //用于接收显示系统的VSYN新号，再下一帧渲染时控制执行一些操作。
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```

## Activity的Window创建添加过程

无论哪种窗口，它的添加过程在WMS处理部分基本都是相似的，这里以最典型的应用程序窗口`Activity`为例。

`Activity`在启动过程中，如果`Activity`所在的进程不存在则会创建新的进程，创建新的进程就会运行代表主线程的实例`ActivityThread`，过程就不讲了，最终会由`ActivityThread`中的`preformLaunchActivity()`来完成整个启动过程。

在这个方法内部会通过类加载器创建`Activity`的实例对象，并调用`attach`方法进行一系列关联操作。

```java
### ActivityThread
   private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
  		.......
  	 try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
          	.......
            
            if (activity != null) {
               .......
                appContext.setOuterContext(activity);
              //调用attch方法，绑定window
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }

                .....
        }
}
```

在`Activity`的`attach`方法里，系统会创建`Activity`所属的`Window`并为其设置回调接口。由于`Activity`实现了`Window`的`Callback`接口，因此当`Window`接收到外界的状态改变就会回调Activity方法。

```java
###Activity.attach
  
  	//创建PhoneWindow,唯一实现类
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
		//传入window回调，绑定
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
```

由于`Actiivity`的视图调用了`setContentView`方法进行展示，我们现在来看下`setContentView`方法。

```java
### Activity
//将所有顶级视图添加到活动中
  public void setContentView(@LayoutRes int layoutResID) {
  		//具体实现还是交给了window处理
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```

具体实现还是交给了window处理，而window的具体实现是phoneWndow，接着再来看下`PhoneWindow`的`setConotentView`方法。

```java
### PhoneWindow
   public void setContentView(int layoutResID) {
        if (mContentParent == null) {
          	//DecorView的创建
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
          //添加Activity的布局文件
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
          //回调Activity的onContentChanged方法通知Activity视图已经发生改变
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

`DecorView`的创建过程由`installDecor`方法来完成，在方法内部会通过`generateDecor`方法来直接创建`DecorView`,然后还需要`generateLayout`来加载具体的布局文件到`DecorView`中。

```java
### PhoneWindow.installDecor
  if (mDecor == null) {
    			//创建空白的FrameLayout
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }

				if (mContentParent == null) {
          //通过这个方法来加载具体的布局文件到DecorView中
            mContentParent = generateLayout(mDecor);
          ......
        }

### PhoneWiindow.generateDecor
  protected DecorView generateDecor(int featureId) {
  	//这个时候还是个空白的FrameLayout
   return new DecorView(context, featureId, this, getAttributes());
}

### PhoneWiindow.generateLayout
  protected ViewGroup generateLayout(DecorView decor) {
  
  ......
  //这个ID_ANDROID_CONTENT对应的ViewGroup就是mContentParent
  ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }
  ......
    
    return contentParent;
}
```

完成了对`DecorView`和`PhoneWindow`的创建，接下来一步就是直接将`Activity`的视图添加到`DecorView`的`mContentParent`中。我们再回到`setContentView`方法中看一行代码：

```java
### PhoneWindow.setContentView
  //添加xml视图
mLayoutInflater.inflate(layoutResID, mContentParent);
```

到这里，`Activity`的布局文件就被添加到`DecorView`中了，但是`DecorView`仍然没有被`WindowManager`添加到`Window`中，界面最后展示是在`ActiivtyThread`的`handleResumeActivity`方法中，这个方法在`onResume`后被调用，最后就可以看到`Activity`的视图啦。

```java
###ActivityThread.handleResumeActivity

public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
            ......
       if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                  //Decorview真正地完成了添加和显示这两个过程
                  //分析了那么久，这个wm.addView方法是不是很熟悉了
                    wm.addView(decor, l);
                } else {
                   
                    a.onWindowAttributesChanged(l);
                }
            }
         
        }
  .......
 }
```

## View的刷新过程

Window的添加和更新过程最终都会走到`scheduleTraversals()`方法，这里面又做了什么？

```java
### ViewRoootImpl
void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
          //添加同步屏障
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
          // 这个添加的回调，将在下一帧渲染时执行
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```

会在提交绘制任务时，添加同步屏障，让异步任务先执行,之后等Vsyn回调回来开始下一帧的绘制。这个添加的回调指的是`TraversalRunnabl`类型的`mTraversalRunnable`，会在`mChoreographer.postCallback`内发送异步消息延迟执行。

```java
### ViewRootImpl
final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    
   void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
          //移除同步屏障
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

          //开始绘制
            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```

这个方法开始移除了同步屏障，又调用了`performTraversals()`,开启了`View`的绘制流程，依次执行`performMeasure`、`performLayout`、`performDraw`方法，这样就完成了view的更新。

**总结下刷新原理**：

View 的` requestLayout` 和 `ViewRootImpl##setView` 最终都会调用 `ViewRootImpl `的 `requestLayout `方法。然后通过 `scheduleTraversals` 方法提交绘制任务，然后再通过`DisplayEventReceiver`向底层请求`vsync`垂直同步信号，当`vsync`信号来的时候，通过JNI回调回来，再通过`Handler`往消息队列post一个异步任务，最终是`ViewRootImpl`去执行绘制任务，最后调用`performTraversals`方法，完成绘制。

最终`performTraversals()`方法触发了`View`的绘制。该方法内部，依次调用了`performMeasure()`，`performLayout()`，`performDraw()`，将`View`的`measure`，`layout`，`draw`过程，从顶层`View`分发了下去。

其实View的刷新机制还有很多内容，后面再出一篇讲解。

**参考**

《Android开发艺术探索》

