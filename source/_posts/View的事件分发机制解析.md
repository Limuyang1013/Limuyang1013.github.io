---
title: View的事件分发机制解析
date: 2016-04-03T22:41:55.000Z
categories: Android
toc: true
tags:
  - 回调
  - 事件分发
---

**引言**

#### Android事件构成

在Android中，事件主要包括点按、长按、拖拽、滑动等，点按又包括单击和双击，另外还包括单指操作和多指操作。所有这些都构成了Android中的事件响应。总的来说，所有的事件都由如下三个部分作为基础：

- 按下(ACTION_DOWN)
- 移动(ACTION_MOVE)
- 抬起(ACTION_UP) <!-- more --> 所有的操作事件首先必须执行的是按下操作（ACTION_DOWN），之后所有的操作都是以按下操作作为前提，当按下操作完成后，接下来可能是一段移动（ACTION_MOVE）然后抬起（ACTION_UP），或者是按下操作执行完成后没有移动就直接抬起。这一系列的动作在Android中都可以进行控制。

这些操作事件都发生在我们手机的触摸屏上面，而我们手机上响应我们各种操作事件的就是各种各样的视图组件也就是`View`，在Android中，所有的视图都继承于`View`，另外通过各种布局组件（`ViewGroup`）来对`View`进行布局，`ViewGroup`也继承于`View`。所有的UI控件例如Button、TextView都是继承于`View`，而所有的布局控件例如RelativeLayout、容器控件例如ListView都是继承于ViewGroup。所以，我们的事件操作主要就是发生在`View`和`ViewGroup`之间。

#### 事件分发的概念

所谓点击事件的事件分发，就是当一个`MotionEvent`产生了以后，系统需要把这个事件传递给一个具体的`View`(`ViewGroup`也继承于`View`),这个传递的过程就叫做分发过程，这个点击事件的分发过程需要三个很重要的方法来共同完成：`disPatchTouchEvent、onInterceptTouchEvent、`和`onTouchEvent`。

> - **public boolean disPatchTouchEvent(MotionEvent ev)** 用来进行事件的分发，如果事件能够传递给当前的`View`,那么此方法一定会被调用，Android中所有的点击事件都必须经过这个方法的分发，然后决定是自身消费当前事件还是继续往下分发给子控件处理。返回`true`表示不继续分发，事件已被消费,返回`false`则继续往下分发，如果是`ViewGroup`则分发给`onInterceptTouchEvent`进行判断是否拦截该事件，这个方法的返回结果受到当前`View`的`onTouchEvent`和下级`View`的`disPatchTouchEvent`方法的影响，返回结果表示是否消耗当前事件。

--------------------------------------------------------------------------------

> - **public boolean onTouchEvent(MotionEvent ev)** 在**diaPatchTouchEvent方法中**调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列当中，当前`View`无法再次接收到事件。

--------------------------------------------------------------------------------

> - **public boolean onInterceptTouchEvent(MotionEvent ev)** 是ViewGroup中才有的方法，View中没有，它的作用是负责事件的拦截，返回true的时候表示拦截当前事件，不继续往下分发，交给自身的`onTouchEvent`进行处理。返回false则不拦截，继续往下传。这是`ViewGroup`特有的方法，因为ViewGroup中可能还有子View，而在Android中View中是不能再包含子`View`的(IOS可以)，在上述方法内部被调用，如果当前`View`拦截了某个事件，那么同一个事件序列中(指从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束，中间含有不定的`ACTION_MOVE`事件，最终以`ACTION_UP`事件结束)此方法不会被再次调用，返回结果表示是否拦截当前事件。

**这三个方法可以用如下伪代码表示：**

```java
public boolean disPathchTouchEvent(MotionEvent ev){
//consume指代点击事件是否被消耗
     boolean consume=false;
     //表示当前父布局要拦截该事件
     if(onInterceptTouchEvent(MotionEvent ev)){
              consume=onTouchEvent(ev);
     }else{
     //传递给子元素去处理
   child.disPatchTouchEvent(ev);
   }
   return consume;
}
```

**分析View的事件分发机制** 为了简单起见我们先从`View`的事件分发机制开始分析，然后在分析`ViewGroup`的，首先我们建一个简单的项目，这个项目里只有一个Button，并且我们给这个Button设置点击事件：

```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
 >

    <Button
        android:id="@+id/btn"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_alignParentTop="true"
        android:layout_centerHorizontal="true"
        android:layout_marginTop="100dp"
        android:text="Click me" />

</RelativeLayout>
```

```java
package com.example.testbtn;

import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import android.view.Menu;
import android.view.MenuItem;
import android.view.MotionEvent;
import android.view.View;
import android.view.View.OnClickListener;
import android.view.View.OnTouchListener;
import android.widget.Button;

public class MainActivity extends Activity implements OnClickListener,OnTouchListener{

    private Button btn;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        btn=(Button) findViewById(R.id.btn);
        btn.setOnClickListener(this);
        btn.setOnTouchListener(this);
    }
    @Override
    public void onClick(View v) {
        Log.d("TAG", "OnClick");
    }
    @Override
    public boolean onTouch(View v, MotionEvent event) {
        Log.d("TAG", "onTouch"+event.getAction());
        return false;
    }


}
```

界面是这样的：


运行这个程序，点击Button，查看Log打印输出的信息：


(这里onTouch0代表的是ACTION_DOWN，onTouch1代表的是ACTION_UP，onTouch2表示ACTION_MOVE，因为我们只是稳稳的点击了一下Button所以不会有ACTION_MOVE的Log信息出现) 这样我们可以得到一个初步的结论：`onTouch()`方法是优先于`onClick()`执行的，然后我们会发现`onTouch()`方法有一个很明显的和`onClick()`方法不同的地方的，那就是它有一个`Boolean`类型的返回值，如果我们把这个默认为False的返回值改为True会怎么样呢：


发现了什么：`onClick()`方法没有被执行，这里我们把这种现象叫做点击事件被`onTouch()`消费掉了，事件不会在继续向`onClick()`方法传递了，那么事件分发机制最基本的几条我们已经了解了，下面我们来分析产生这种机制的根本原因。

#### View对点击事件的处理过程

首先我们给出一个结论：Android中所有的事件都必须经过`disPatchTouchEvent(MotionEvent ev`)这个方法的分发，然后决定是自身消费当前事件还是继续往下分发给子控件处理，那么我们就来看看这个`disPatchTouchEvent(MotionEvent ev)`到底干了什么。

```java
public boolean dispatchTouchEvent(MotionEvent event) {
        // If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }
            // We have focus and got the event, then use normal event dispatch.
            event.setTargetAccessibilityFocus(false);
        }

        boolean result = false;

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        final int actionMasked = event.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }

        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        if (!result && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }

        // Clean up after nested scrolls if this is the end of a gesture;
        // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
        // of the gesture.
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }

        return result;
    }
```

代码有点多，我们一步步来看：

```java
// If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }
            // We have focus and got the event, then use normal event dispatch.
            event.setTargetAccessibilityFocus(false);
        }
```

最前面这一段就是判断当前事件是否能获得焦点，如果不能获得焦点或者不存在一个`View`那我们就直接返回False跳出循环，接下来：

```java
if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        final int actionMasked = event.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }
```

设置一些标记和处理input与手势等传递，不用管，到这里：

```java
if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
```

这里`if (onFilterTouchEventForSecurity(event))`是用来判断`View`是否被遮住等，`ListenerInfo`是`View`的静态内部类，专门用来定义一些XXXListener等方法的，到了重点：

```java
if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
```

很长的一个判断，一个个来解释：第一个`li`肯定不为空，因为在这个If判断语句之前就new了一个`li`，第二个条件`li.mOnTouchListener != null`，怎么确定这个`mOnTouchListener`不为空呢？我们在View类里面发现了如下方法：

```java
/**
     * Register a callback to be invoked when a touch event is sent to this view.
     * @param l the touch listener to attach to this view
     */
    public void setOnTouchListener(OnTouchListener l) {
        getListenerInfo().mOnTouchListener = l;
    }
```

意味着只要给控件注册了`onTouch`事件这个`mOnTouchListener`就一定会被赋值,接下来`(mViewFlags & ENABLED_MASK) == ENABLED`是通过位与运算来判断这个`View`是否是`ENABLED`的，我们默认控件都是`ENABLED`的所以这一条也成立，最后一条`li.mOnTouchListener.onTouch(this, event)`是判断`onTouch()`的返回值是否为True，我们后面把默认为False的返回值改成了True，所以这一整系列的判断都是True，那么这个`disPatchTouchEvent(MotionEvent ev)`方法直接就返回了True,那么接下来的代码都不会被执行，我们下面有这么一段代码：

```java
if (!result && onTouchEvent(event)) {
                result = true;
            }
```

最开始我们`onTouch()`方法的返回值是False的，那么

```java
if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
```

这里面的判断就不成立，result最开始的默认值也是false，那么此时如果

```java
onTouchEvent(event)
```

返回值也是True，那么`if (!result && onTouchEvent(event))`这个方法判断条件成立，`disPatchTouchEvent(MotionEvent ev)`返回True，否则返回False。

#### 这里我们得到两个结论：

- `OnTouchListener`的优先级比`onTouchEvent`要高，联想到刚才的小Demo也可以得出`onTouch`方法优先于`onClick()`方法执行(`onClick()`是在`onTouchEvent(event)`方法中被执行的这个待会会说到)
- 如果控件（View）的`onTouch`返回False或者`mOnTouchListener`为null（控件没有设置`setOnTouchListener`方法）或者控件不是`ENABLE`的情况下会调用`onTouchEvent`方法，此时`dispatchTouchEvent`方法的返回值与`onTouchEvent`的返回值一样。

那么接下来我们就分析`dispatchTouchEvent`方法里面`onTouchEvent`的实现,给出`onTouchEvent`的源码：

```java
/**
     * Implement this method to handle touch screen motion events.
     * <p>
     * If this method is used to detect click actions, it is recommended that
     * the actions be performed by implementing and calling
     * {@link #performClick()}. This will ensure consistent system behavior,
     * including:
     * <ul>
     * <li>obeying click sound preferences
     * <li>dispatching OnClickListener calls
     * <li>handling {@link AccessibilityNodeInfo#ACTION_CLICK ACTION_CLICK} when
     * accessibility features are enabled
     * </ul>
     *
     * @param event The motion event.
     * @return True if the event was handled, false otherwise.
     */
    public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();

        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
        }

        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                       }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_DOWN:
                    mHasPerformedLongPress = false;

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);
                        checkForLongClick(0);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    setPressed(false);
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_MOVE:
                    drawableHotspotChanged(x, y);

                    // Be lenient about moving outside of buttons
                    if (!pointInView(x, y, mTouchSlop)) {
                        // Outside button
                        removeTapCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            // Remove any future long press/tap checks
                            removeLongPressCallback();

                            setPressed(false);
                        }
                    }
                    break;
            }

            return true;
        }

        return false;
     }
```

代码还是很多，我们依然一段一段来分析，最前面的一段代码：

```java
if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
        }
```

根据前面的分析我们知道这一段代码是对当前`View`处于不可用状态的情况下的分析，通过注释我们知道即使是一个不可用状态下的`View`依然会消耗点击事件，只是不会对这个点击事件作出响应罢了，另外通过观察这个return返回值，只要这个`View`的`CLICKABLE`和`LONG_CLICKABLE`或者`CONTEXT_CLICKABLE`有一个为True，那么返回值就是True，`onTouchEvent`方法会消耗当前事件。

看下一段代码：

```java
if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }
```

> 这段代码的意思是如果`View`设置有代理，那么还会执行`TouchDelegate`的`onTouchEvent(event)`方法，这个`onTouchEvent(event)`的工作机制看起来和`OnTouchListener`类似，这里不深入研究。 --《Android开发艺术探索》

下面看一下`onTouchEvent`中对点击事件的具体处理流程：

```java
if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                       }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_DOWN:
                    mHasPerformedLongPress = false;

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);
                        checkForLongClick(0);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    setPressed(false);
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_MOVE:
                    drawableHotspotChanged(x, y);

                    // Be lenient about moving outside of buttons
                    if (!pointInView(x, y, mTouchSlop)) {
                        // Outside button
                        removeTapCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            // Remove any future long press/tap checks
                            removeLongPressCallback();

                            setPressed(false);
                        }
                    }
                    break;
            }

            return true;
        }

        return false;
    }
```

我们还是一行行来分解：

```java
if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch (action) {
            //省略
            }
             return false;
    }
```

这个判断之前描述过不再赘述，如果这个判断不成立直接跳到方法尾部返回False，如果判断成立则继续进入方法内部进行一个switch(event)的判断，这里ACTION_DOWN和ACTION_MOVE都只是进行一些必要的设置与置位，我们主要看ACTION_UP：

```java
case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                       }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
```

首先判断了是否被按下 `boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;`接下来判断是不是可以获得焦点，同时尝试去获取焦点：

```java
boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }
```

经过种种判断后我们看到这一行：

```java
if (!post(mPerformClick)) {
                                    performClick();
                                }
```

这是判断如果不是longPressed则通过post在UI Thread中执行一个PerformClick的Runnable，也就是performClick方法，这个方法的源码如下：

```java
/**
     * Call this view's OnClickListener, if it is defined.  Performs all normal
     * actions associated with clicking: reporting accessibility event, playing
     * a sound, etc.
     *
     * @return True there was an assigned OnClickListener that was called, false
     *         otherwise is returned.
     */
    public boolean performClick() {
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        return result;
    }
```

我们发现了什么，那就是当ACTION_UP事件发生时，会触发`performClick()`方法，如果这个`View`设置了`OnClickListener`那么最终会执行到`OnClickListener`的回调方法`onClick()`,这也就验证了刚才所说的：**onClick()方法是在`onTouchEvent`内部被调用的。** 同我们前面找到`mOnTouchListener`在哪里赋值的一样，我们也可以找到`mOnClickListener`在哪里赋值的：

```java
/**
     * Register a callback to be invoked when this view is clicked. If this view is not
     * clickable, it becomes clickable.
     *
     * @param l The callback that will run
     *
     * @see #setClickable(boolean)
     */
    public void setOnClickListener(@Nullable OnClickListener l) {
        if (!isClickable()) {
            setClickable(true);
        }
        getListenerInfo().mOnClickListener = l;
    }
```

我们知道`View`的`LONG_CLICKABLE`属性默认是False的，需要的话我们可以自己在xml或者java文件中去设置，但是`CLICKABLE`的False与True是和具体的`View`有关的，比如我们知道Button是可点击的，但是TextView默认是不可点击的，但是如果给TextView设置了点击事件，那么根据

```java
if (!isClickable()) {
            setClickable(true);
        }
```

这几行代码TextView也会被设置为可点击的，同理还有`setOnLongClickListener`也有这种作用：

```java
/**
     * Register a callback to be invoked when this view is clicked and held. If this view is not
     * long clickable, it becomes long clickable.
     *
     * @param l The callback that will run
     *
     * @see #setLongClickable(boolean)
     */
    public void setOnLongClickListener(@Nullable OnLongClickListener l) {
        if (!isLongClickable()) {
            setLongClickable(true);
        }
        getListenerInfo().mOnLongClickListener = l;
    }
```

#### 总结

到此，View的事件分发机制已经分析完了，整个过程查了许多资料，最主要的是跟着任玉刚老师的《Android开发艺术探索》学习，最后把这个学习的过程记录下来就是这篇博客了，等有时间的时候把ViewGroup的事件分发机制也分析一遍。

主要参考 [Android触摸屏事件派发机制详解与源码分析一(View篇)](http://blog.csdn.net/yanbober/article/details/45887547) [Android事件传递机制](http://www.infoq.com/cn/articles/android-event-delivery-mechanism)

#### 请看后续 [ViewGroup的事件分发机制](http://www.limuyang.cc/2016/04/04/ViewGroup%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%E6%9C%BA%E5%88%B6/)
