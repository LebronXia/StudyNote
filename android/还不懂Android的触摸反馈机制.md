当我们的手指从触摸屏幕上的各种View，开始到这个点击事件的结束到底经历了什么，我们来详细分析下。

## 事件类型

触摸事件会有三种类型：

```java
 int action = MotionEventCompat.getActionMasked(event);
    switch(action) {
        case MotionEvent.ACTION_DOWN:
            break;
        case MotionEvent.ACTION_MOVE:
            break;
        case MotionEvent.ACTION_UP:
            break;
    }
```

除了这三个类型之外，还有个`ACTION_CANCEL，`它表示当前的手势被取消。可以这样理解，一个子View接收父View发给它的`ACTION_DOWN`事件，当父View对事件进行拦截，不再继续转发事件给子VIew的话，就会给子View一个`ACTION_CANCEL`事件。

## Activity的事件分发机制

当我们触摸屏幕时，会首先将点击事件传递到`Activity`,有`Activity`中的`PhoneWindow`完成，PhoneWindow再把事件传递到整个控件树的根`DecorView`，之后再由`DecorView`将事件处理工作交给`ViewGroup`。

```java
### Activity.dispatchTouchEvent
  public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
  // 由Window进行分发，当getWindow().superDispatchTouchEvent(ev)返回true
  //则 dispatchTouchEvent返回true,
  //方法结束，不会执行onTouchEvent方法
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev); 1
    }
```

```java
### PhoneWindow
//将事件传递到ViewGroup进行处理
   @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```

当事件未处理，又会回到`Activity`的`onTouchEvent`方法那里

```java
### Activity.onTouchEvent
public boolean onTouchEvent(MotionEvent event) {
  //只有在点击事件在Window边界外才返回true,一般都返回false
        if (mWindow.shouldCloseOnTouch(this, event)) {
            finish();
            return true;
        }
			
        return false;
    }

### Window.shouldCloseOnTouch
  public boolean shouldCloseOnTouch(Context context, MotionEvent event) {
  //是不是Down事件，event坐标是否在边界内等待
        final boolean isOutside =
                event.getAction() == MotionEvent.ACTION_DOWN && isOutOfBounds(context, event)
                || event.getAction() == MotionEvent.ACTION_OUTSIDE;
        if (mCloseOnTouchOutside && peekDecorView() != null && isOutside) {
          //返回true：说明事件在边界外，即 消费事件
            return true;
        }
        return false;
    }
```

## ViewGroip的事件分发机制

从上面的分发机制可以看到，ViewGroup的事件分发机制从`dispatchTouchEvent()`开始。

```java
	### ViewGroup.dispatchTouchEvent
    public boolean dispatchTouchEvent(MotionEvent ev) {
      if (mInputEventConsistencyVerifier != null) {
                mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
            }
       ......

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {             
                cancelAndClearTouchTargets(ev);
              //重置所有的触摸状态以这准备新的循环
                resetTouchState();
            }

            // 事件被处理， mFirstTouchTarget为第一个触摸目标
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
              //子View通过requestDisllowInterceptTouchEvet设置FLAG_DISALLOW_INTERCEPT标志，表示父View不再拦截事件
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                  //返回false默认
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // 恢复
                } else {
                    intercepted = false;
                }
            } else {            
                intercepted = true;
            }

           ......
             
            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
          //不取消 不拦截事件
            if (!canceled && !intercepted) {
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                  
                   ......
                     
                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        ......
                      
                      //遍历ViiewGroup的子元素，如果子元素能够接收到点击事件，则交给子元素处理
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);
                           ......
                             
                             //子 View的范围内或者子View是否在播放动画
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // 在触摸范围内
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                          
                          //条件判断的内部调用了该View的dispatchTouchEvent()
                          //实现了点击事件从ViewGroup到子View的传递
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                               .....
                    }
                    ......
                }
            }
            ......
        return handled;
    }
  
  
  //拦截
  //返回false：不拦截（默认）
  // 返回true：拦截，即事件停止往下传递（需手动复写onInterceptTouchEvent()其返回true）
     public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
                && ev.getAction() == MotionEvent.ACTION_DOWN
                && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
                && isOnScrollbarThumb(ev.getX(), ev.getY())) {
            return true;
        }
        return false;
    }


```

看到上面调用了`dispatchTransformedTouchEvent`方法，那ViewGroup如何将事件分发给子View

```java
### ViewGroup.dispatchTransformedTouchEvent
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
          //有子View，则调用子View的dispatchTouchEvent(event)方法，如果没有子View，
        // 则调用super.dispatchTouchEvent(event)方法。
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }
       ......
        // Done.
        transformedEvent.recycle();
        return handled;
    }  

```

可以得知，事件分发总是先传递到`ViewGroup`，之后遍历子View找到有点击的子View，再传递给子View。

> TouchTarget:记录每个子 View 是被哪些 pointer(手指)按下的   是个单向链表

其中关键的3个重要方法关系，可以用伪代码表示：

```java
public boolean dispatchTouchEvent(MotionEvent ev){
  
  boolean consume = false;
  
  if(onInterceptTouchEvent(ev)){
    	consume = onTouchEvent(ev);
  } else {
    	consume = childDispatchTouchEvent(ev);
  }
  return consume;
}
```

//图片案例

## View的事件分发机制

事件传递来到了View的`dispatchTouchEvent`方法中。

```java
### View.dispatchTouchEvent
public boolean dispatchTouchEvent(MotionEvent event) {
        ......
        boolean result = false;
        ......

        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //onTouchListener方法的优先级要高于onTouchEvent方法
            ListenerInfo li = mListenerInfo;
          //有三个条件符合才会返回true
           //mOnTouchListener != null      mOnTouchListener变量在View.setOnTouchListener()里赋值
          //(mViewFlags & ENABLED_MASK) == ENABLED   判断当前点击的控件是否是enabled
          // mOnTouchListener.onTouch(this, event)   回调控件注册Touch事件时的onTouch()
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        ......

        return result;
    }
```

三个条件如果有个条件不符合，就会把事件传递到View的`onTouchEvent()`,下面详细分析`onTouchEvent`方法

```java
### View.onTouchEvent 
public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();

   		//可点击 可长按
        final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
                || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

   		//如果被禁用了  把事件吃掉
        if ((viewFlags & ENABLED_MASK) == DISABLED) {
          //抬起，仍然设置setPressed(false)，表示点击过了
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
 
            return clickable;
        }
   
   	//触摸代理  增加点击区域
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

   		//TOOLTIP   tooptipText是个解释工具
        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    if ((viewFlags & TOOLTIP) == TOOLTIP) {
                      //松手消失ToolTip
                        handleTooltipUp();
                    }
                    if (!clickable) {
                      //取消监听器
                        removeTapCallback();
                        removeLongPressCallback();
                        mInContextButtonPress = false;
                        mHasPerformedLongPress = false;
                        mIgnoreNextUpEvent = false;
                        break;
                    }
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                //按下 或者是预按下状态
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                       
                        boolean focusTaken = false;
                      //可以获取焦点
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                      //如果是预按下状态
                        if (prepressed) {
                            setPressed(true, x, y);
                        }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // 取消长按回调
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                              
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                  //触发点击
                                    performClickInternal();
                                }
                            }
                        }

                       ......

                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_DOWN:
                //你是不是摸到屏幕了
                    if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                        mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                    }
                    mHasPerformedLongPress = false;

                //不可点击 检查长按， 设置一个长按的监听器
                    if (!clickable) {
                        checkForLongClick(0, x, y);
                        break;
                    }

                //判断鼠标右键点击
                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // 在滑动控件中
                    boolean isInScrollingContainer = isInScrollingContainer();

                //对于滚动容器内的视图，如果这是滚动，请将按下的反馈延迟一小段时间。
                //滑动控件中不知道你是操作父View还是子View
                //不是滑动的 可以重写shouleDelayChildPressedState为false，不用延迟子View按下时间
                    if (isInScrollingContainer) {
                      //设置状态设为 预按下
                        mPrivateFlags |= PFLAG_PREPRESSED;

                        if (mPendingCheckForTap == null) {
                          //设置按下等待器
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        //这是核心   如果不在滑动控件中  又是按下
                        setPressed(true, x, y);
                      
                        checkForLongClick(0, x, y);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    if (clickable) {
                        setPressed(false);
                    }
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    break;

                case MotionEvent.ACTION_MOVE:
                //可点击  波纹改变
                    if (clickable) {
                        drawableHotspotChanged(x, y);
                    }

                    // 如果手出界了
                    if (!pointInView(x, y, mTouchSlop)) {
                        // 全部取消掉
                        removeTapCallback();
                        removeLongPressCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            setPressed(false);
                        }
                        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    }
                    break;
            }

            return true;
        }

        return false;
    }
```

最终调用到了`performClick（）`方法

```java
### View.performClick 
public boolean performClick() {
     
        notifyAutofillManagerOnClick();

        final boolean result;
        final ListenerInfo li = mListenerInfo;
   // 如果View设置了点击事件，onClick方法就会执行。
   //onTouch（）的执行 先于 onClick（）
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);

        notifyEnterOrExitForAutoFillIfNeeded(true);

        return result;
    }
```



## 事件传递总结

1. 同一个事件序列是指从手指接触屏幕到手指离开，整个事件序列都是以down事件开始，中间含还有数量不等的move事件，最终以up事件结束。
2. 正常情况下，一个事件序列只能被一个View拦截且消耗。
3. 某个View一旦确定拦截，那么这一个事件序列都只能由它处理，并且它的`onInterceptToucheEvent`不会再被调用。
4. 某个View一旦开始处理事件，如果它不消耗`ACTION_DOWN`事件，那么同一事件序列中的其他事件都不会再交给它开处理，并且事件将重新交给它的父元素去处理，即父元素的`onTouchEvent`会被调用。
5. 如果View不消耗除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续收到后续的事件，最终这些消失的点击事件会传递给Activity处理。
6. ViewGroup默认不拦截任何事件。
7. View的onTouchEvent默认都会消耗事件（返回true），除非它是不可点击的（clickable和longClickable同时为false）。View的longClickable属性默认都为false，clickable属性要分情况，比如Button的clickable属性默认为true，而TextView的clickable默认为false。
8. View的enable属性不影响onTouchEvent的默认返回值。哪怕一个View是`disable`状态，只要它的`clickab`le或者`longClickable`有一个为true,那么它的`onTouchEvent`就返回true。
9. 事件传递过程都是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View,通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，但是ACTION_DOWN事件除外。

完整示意图：

![事件分发业务流程图](../../../../Library/Application Support/typora-user-images/image-20211116180227233.png)

​															摘自网络- 《Android事件分发机制详解：史上最全面、最易懂》

参考：

[Android事件分发机制详解：史上最全面、最易懂](https://www.jianshu.com/p/38015afcdb58)

[Android触摸事件传递机制](https://juejin.cn/post/6844904041487532045#heading-16)





