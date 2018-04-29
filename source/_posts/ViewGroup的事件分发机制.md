---
title: ViewGroup的事件分发机制
date: 2016-04-04 21:08:19
categories: Android
tags: [事件分发,回调,ViewGroup]
toc: true
---
### 引言
上一次我在**[View的事件分发机制](http://www.limuyang.cc/2016/07/24/View%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%E6%9C%BA%E5%88%B6%E8%A7%A3%E6%9E%90/)**里完整的分析了`View`对于触屏点击事件的分发过程，接下来继续探索之旅，紧接着分析`ViewGroup`的事件分发机制，`ViewGroup`其实就是一组`View`的集合，它也是继承于View的，它本身也可以包含`View`和`ViewGroup`，方便起见我们还是延用上一次的布局，不过这一次我们给根布局也设置了点击事件和触摸事件：

<!--more-->

```
public class MainActivity extends Activity implements OnClickListener,OnTouchListener{

	private RelativeLayout re_Layout;
	private Button btn;
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		btn=(Button) findViewById(R.id.btn);
		re_Layout=(RelativeLayout) findViewById(R.id.re_layout);
		btn.setOnClickListener(this);
		btn.setOnTouchListener(this);
		re_Layout.setOnClickListener(this);
		re_Layout.setOnTouchListener(this);
	}
	@Override
	public void onClick(View v) {
		Log.d("TAG", "OnClick--"+v);
	}
	@Override
	public boolean onTouch(View v, MotionEvent event) {
		Log.d("TAG", "onTouch--"+event.getAction()+"--"+v);
		return false;
	}

}
```
![效果图](http://oasusatoz.bkt.clouddn.com/%E7%82%B9%E5%87%BB%E6%95%88%E6%9E%9C%E5%9B%BE.jpg)

效果图也依旧没有变，现在我们先点击一下Button,查看Log输出：
![Log输出](http://oasusatoz.bkt.clouddn.com/Log%E8%BE%93%E5%87%BA.jpg)

可以很清楚的看到这里和上一节分析得情况一样：当点击事件发生时`onTouch()`方法是优先于`onClick()`方法执行的，并且如果`onTouch()`返回False既不消耗点击事件那么如果控件设置了`setOnClickListener`最终是会执行到`onClick()`方法的，可是我很好奇ViewGroup的点击事件和View的到底有什么区别，点击事件事件的分发到底是从`ViewGroup`开始还是从`View`开始的呢，于是我点击了Button以外的空白区域，捕捉到如下信息(注:我的根布局就是`RelativeLayout`)：
![Log输出](http://oasusatoz.bkt.clouddn.com/%E7%A9%BA%E7%99%BD%E5%8C%BA%E5%9F%9FLog%E8%BE%93%E5%87%BA.jpg)

说明根布局也就是`Viewroup`也是可以响应点击事件的，但是我们点击`View`的时候为什么没有`ViewGroup`的Log输出，这是不是说明android事件分发是先传到`View`的，当`View`消耗的这个事件它的`ViewGroup`就无法接收这个事件了呢，为了彻底的谈清楚原因，我们先重写一个`ViewGroup`，然后重写这个`ViewGroup`里面的`onInterceptTouchEvent(MotionEvent ev)`、`dispatchTouchEvent(MotionEvent event)`还有onTouchEvent(MotionEvent event)这三个方法通过Log输出信息来判断：

```
public class MyLayout extends RelativeLayout{

	public MyLayout(Context context,AttributeSet attrs) {
		super(context,attrs);

	}
	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev) {
		Log.d("TAG", ev.getAction()+" action"+"MyLayout onInterceptTouchEvent");
		return super.onInterceptTouchEvent(ev);
	}
	@Override
	public boolean onTouchEvent(MotionEvent event) {
		Log.d("TAG", event.getAction()+" action"+"MyLayout onTouchEvent");
		return super.onTouchEvent(event);
	}
	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		Log.d("TAG", ev.getAction()+" action"+"MyLayout dispatchTouchEvent");
		return super.dispatchTouchEvent(ev);
	}
```
同理我们还需要重写一个Button：

```
public class MyButton extends Button{
	public MyButton(Context context,AttributeSet attrs){
		super(context,attrs);
	}
	@Override
	public boolean onTouchEvent(MotionEvent event) {
		Log.d("TAG", event.getAction()+" action"+"MyButton onTouchEvent");
		return super.onTouchEvent(event);
	}
	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		Log.d("TAG", ev.getAction()+" action"+"MyButton dispatchTouchEvent");
		return super.dispatchTouchEvent(ev);
	}
	//这里注意View是没有onInterceptTouchEvent方法的
```
效果图是一样的这里就不再贴了，为了验证刚才的想法我们直接点击一下界面上的Button，Log输出如下：
![Log输出](http://oasusatoz.bkt.clouddn.com/%E5%86%8D%E6%AC%A1%E7%82%B9%E5%87%BB%E6%97%B6Log%E8%BE%93%E5%87%BA.jpg)

发现了什么，我们点击的是Button,然而这个事件最开始是传到了我们的根布局MyLayout，并且还按照：
`dispatchTouchEvent`、`onInterceptTouchEvent`、`dispatchTouchEvent`的顺序执行，紧接着执行`View`的`onTouch()`和`onTouchEvent()`方法，还有一点很奇怪的事只有最开始ACTION_DOWN的时候调用了ViewGroup的`onInterceptTouchEvent`方法，在后面的ACTION_UP事件派发过程中却没有调用，这里给出一个合理的猜想：一旦一个View开始处理这个触摸事件，那么接下来的ACTION_MOVE和ACTION_UP事件都会交给它去处理，就好比你在公司里面做事，分到你做的事你已经做了一些，那么接下来的事你的完完整整的做好，那么如果做到一半不做了会怎么样(即View不消耗ACTION_DOWN事件)？我们可以大胆的假设如果上级交给你做的事没有做好，那么上级_在短期内肯定不敢交代事情给你做了(后续的ACTION_MOVE、ACTION_DOWN事件这个View都接收不到了)，那么究竟如何我们还是从源码看起。

### 对源码的分析
我们已经知道当一个点击操作发生时事件是先传给`ViewGroup`处理的并且首先执行的是`ViewGroup`的`dispatchTouchEvent`,那么我们就先来看看它的源码：

```
 /**
     * {@inheritDoc}
     */
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }

        // If the event targets the accessibility focused view and this is it, start
        // normal event dispatch. Maybe a descendant is what will handle the click.
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {

                // If the event is targeting accessiiblity focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildOrderedChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }

```

相当长，还是一点点来看，源码这种东西看不懂肯定会觉得很枯燥，所以能弄懂的尽量弄懂，最开始只是一些对View是否可以获得焦点的判断、设置标志位以及初始化一些布尔值，并且在ACTION_DOWN事件产生的时候清楚以外的状态并且准备开始新一轮的手势操作，不重要：

```
     if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }

        // If the event targets the accessibility focused view and this is it, start
        // normal event dispatch. Maybe a descendant is what will handle the click.
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }
```
先看这一段：

```
  // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }
```
这里首先设立了一个布尔值`interception`去判断当前ViewGroup是否要拦截View的点击事件，`if`条件语句中的内容是当产生ACTION_DOWN按下事件或者`mFirstTouchTarget != null`的时候去判断是否要拦截当前事件，这里主要关注`mFirstTouchTarget != null`这个点，我们找一找哪个方法跟这个`mFirstTouchTarget`变量有关，还真给我找到了，看下面：
```
 private void clearTouchTargets() {
        TouchTarget target = mFirstTouchTarget;
        if (target != null) {
            do {
                TouchTarget next = target.next;
                target.recycle();
                target = next;
            } while (target != null);
            mFirstTouchTarget = null;
        }
    }
```

```
/**
     * Cancels and clears all touch targets.
     */
    private void cancelAndClearTouchTargets(MotionEvent event) {
        if (mFirstTouchTarget != null) {
            boolean syntheticEvent = false;
            if (event == null) {
                final long now = SystemClock.uptimeMillis();
                event = MotionEvent.obtain(now, now,
                        MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
                syntheticEvent = true;
            }

            for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
                resetCancelNextUpFlag(target.child);
                dispatchTransformedTouchEvent(event, true, target.child, target.pointerIdBits);
            }
            clearTouchTargets();

            if (syntheticEvent) {
                event.recycle();
            }
        }
    }
```
这两段代码结合起来，在加上在ACTION_DOWN初始时候是调用了`cancelAndClearTouchTargets(MotionEvent event)`这个方法的，所以我们可以推荐起初这个`mFirstTouchTarget` 的值是`null`的，那么`mFirstTouchTarget`是在哪里赋值的呢，我们在`dispatchTouchEvent(MotionEvent ev)`接着往下看：

```
    newTouchTarget = addTouchTarget(child, idBitsToAssign);
```

我们看到`newTouchTarget`是在这里赋值的，看一下`addTouchTarget`方法：

```
 /**
     * Adds a touch target for specified child to the beginning of the list.
     * Assumes the target child is not already present.
     */
    private TouchTarget addTouchTarget(View child, int pointerIdBits) {
        TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
        target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
        return target;
    }
```
从该方法的内部结构可以看出，mFirstTouchTarget其实是一中单链表结构，如果找到了处理该点击事件的子`View`那么`mFirstTouchTarget`就会被赋值并且会指向子元素。
这一下弄清楚了回到刚才的那段代码：

```
  final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }
```

```
 final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
```
这个布尔值是判断子元素是否调用了`requestDisallowInterceptTouchEvent`这个方法，如果调用了这个布尔值就为True，这里看一眼这个方法的代码：

```
 /**
     * {@inheritDoc}
     */
    public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {

        if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
            // We're already in this state, assume our ancestors are too
            return;
        }

        if (disallowIntercept) {
            mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
        } else {
            mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
        }

        // Pass it up to our parent
        if (mParent != null) {
            mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
        }
    }
```
你可以在子View中调用这个方法来让`ViewGroup`不拦截除了ACTION_DOWN以外的点击事件，这里为什么说是ACTION_DOWN以外呢，因为`ViewGroup`在分发事件的时候最开始是会重置`FLAG_DISALLOW_INTERCEPT`这个标志位的，所以无论你有没有在子View中设置`requestDisallowInterceptTouchEvent`方法都不会影响到`ViewGroup`去拦截ACTION_DOWN事件的，接着往下看：

```
  if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }
```

如果子`View`没有设置`requestDisallowInterceptTouchEvent`方法那么就调用`ViewGroup`的`onInterceptTouchEvent(ev)`方法，我们找到这个方法：

```
  /**
     * Implement this method to intercept all touch screen motion events.  This
     * allows you to watch events as they are dispatched to your children, and
     * take ownership of the current gesture at any point.
     *
     * <p>Using this function takes some care, as it has a fairly complicated
     * interaction with {@link View#onTouchEvent(MotionEvent)
     * View.onTouchEvent(MotionEvent)}, and using it requires implementing
     * that method as well as this one in the correct way.  Events will be
     * received in the following order:
     *
     * <ol>
     * <li> You will receive the down event here.
     * <li> The down event will be handled either by a child of this view
     * group, or given to your own onTouchEvent() method to handle; this means
     * you should implement onTouchEvent() to return true, so you will
     * continue to see the rest of the gesture (instead of looking for
     * a parent view to handle it).  Also, by returning true from
     * onTouchEvent(), you will not receive any following
     * events in onInterceptTouchEvent() and all touch processing must
     * happen in onTouchEvent() like normal.
     * <li> For as long as you return false from this function, each following
     * event (up to and including the final up) will be delivered first here
     * and then to the target's onTouchEvent().
     * <li> If you return true from here, you will not receive any
     * following events: the target view will receive the same event but
     * with the action {@link MotionEvent#ACTION_CANCEL}, and all further
     * events will be delivered to your onTouchEvent() method and no longer
     * appear here.
     * </ol>
     *
     * @param ev The motion event being dispatched down the hierarchy.
     * @return Return true to steal motion events from the children and have
     * them dispatched to this ViewGroup through onTouchEvent().
     * The current target will receive an ACTION_CANCEL event, and no further
     * messages will be delivered here.
     */
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return false;
    }
```
唔，注释相当长，但是有用的就一个返回值，这里返回False，说明`ViewGrup`不拦截点击事件，事件可以继续往下传递，这个方法的最后，如果当前界面除了一个`ViewGroup`没有任何子`View`，那么此时`ViewGroup`也会拦截点击事件，就好比一个公司人手不够，公司领导需要亲力亲为一样。
接着：

```
// If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

```
如果确定拦截或者已经有子View着手处理这个点击事件，那么就开始正常的事件分发流程。

```
 final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;
```
这里是通过标志位和ACTION_CANCLE来检查是否cancle，再下去：

```
 final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
```

> 首先可以看见获取一个boolean变量标记split来标记，默认是true，作用是是否把事件分发给多个子View，这个同样在ViewGroup中提供了public的方法设置，如下：
> 

```
public void setMotionEventSplittingEnabled(boolean split) {
        // TODO Applications really shouldn't change this setting mid-touch event,
        // but perhaps this should handle that case and send ACTION_CANCELs to any child views
        // with gestures in progress when this is changed.
        if (split) {
            mGroupFlags |= FLAG_SPLIT_MOTION_EVENTS;
        } else {
            mGroupFlags &= ~FLAG_SPLIT_MOTION_EVENTS;
        }
    }
```


这一段摘自：[Android触摸屏事件派发机制详解与源码分析二(ViewGroup篇)](http://blog.csdn.net/yanbober/article/details/45912661)	　

```
    if (!canceled && !intercepted) {
```
如果没有取消当前动作并且`ViewGroup`未拦截事件那么事件就传递到接收了该点击事件的`View`，接下来是一大段代码预警：

```
 // If the event is targeting accessiiblity focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildOrderedChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
```
嗯，这段代码的逻辑比较清晰，大体上就是遍历`ViewGroup`的所有子元素，然后判断子元素是否能够接收到点击事件，接收的依据有两种：第一种是判断子元素是否在播放动画，第二种是判断点击事件的坐标是否落在子元素的区域内，从这里可以看出来：

```
if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }
```
如果子元素满足了这两个条件，点击事件就会交给它处理，接下来这段代码里面有一个很重要的方法：

```
       if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
```
这里`if`判断里面这个`dispatchTransformedTouchEvent`是将Touch事件传递给特定的子`View`，它实际上在内部是调用了子元素的`disPatchTouchEvent`方法，找一下它的源码：

```
    /**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        // Calculate the number of pointers to deliver.
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        // If for some reason we ended up in an inconsistent state where it looks like we
        // might produce a motion event with no pointers in it, then drop the event.
        if (newPointerIdBits == 0) {
            return false;
        }
```
看这一段内容：

```
  if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
```
如果传递的child为null就调用父类的`dispatchTouchEvent`否则就调用子类的`dispatchTouchEvent`，而上面的代码中child不为null，所以执行子元素的`dispatchTouchEvent`，如果子元素的`dispatchTouchEvent`返回的是True，那么含有`dispatchTransformedTouchEvent`这个方法内部的for循环就不会继续下去，直接跳到这里：

```
 mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
```
这个地方前面说过是给`mFirstTouchTarget`赋值的地方，如果情况改变，前面那个`dispatchTransformedTouchEvent`方法中child返回的是False，那么如果当前`ViewGroup`会把点击事件传递给下一个子元素进行处理，执行`for`循环查找下一个子元素，此时`mFirstTouchTarget`依然未被赋值为null，那么这时候继续查看接下来的代码：

```
     if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }

```
这一段代码表示当前没有找到可以接收点击事件的`View`并且我们的`mFirstTouchTarget!=null`那么就把最开始的TouchTarget赋值给`newTouchTarget`，最后：

```
 // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
```
执行到这里的话有两种情况，一种是ViewGroup里面没有找到子View，另一种就是找到了处理这次点击事件的子View但是这个子View的`disPatchTouchEvent`返回了False，我们通过前面的分析知道`disPatchTouchEvent`中是先执行`onTouch()`方法的，而一般`onTouch()`方法返回的是False，此时`disPatchTouchEvent`方法的返回值由`onTouchEvent`方法决定，出现这种情况说明`onTouchEvent`返回了False，在以上两种情况下，`ViewGroup`会自己处理这个点击事件，注意这里这个方法里的child传入的是null，我们前面就知道了传入null会执行`handled = super.dispatchTouchEvent(event);`也就是说此时交由ViewGroup处理这个事件。而`ViewGroup`也是`View`的子类，它里面是没有重写`View`的`onTouchEvent`方法的，所以它自身处理点击事件的流程和我们在[**View的事件分发机制解析**](http://www.limuyang.cc/2016/07/24/View%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%E6%9C%BA%E5%88%B6%E8%A7%A3%E6%9E%90/)里面分析得是一样的，至此`ViewGroup`的分发事件分析完毕。
### 总结
这次我们只分析了点击Button时的Log输出，下面给出点击空白处的Log输出，可以自己检验一下分析成果：
![这里写图片描述](http://oasusatoz.bkt.clouddn.com/%E6%9C%80%E5%90%8E%E7%9A%84Log%E8%BE%93%E5%87%BA.jpg)

### 一些结论

-  `ViewGroup`默认不拦截任何事件，`Android`源码中`ViewGroup`的`onInterceptTouchEvent`方法默认返回`false`
- `View`的`onTouchEvent`默认都会消耗事件(返回`true`)，除非是不可点击的(`clickable`和`longClicjable`同时为`false`)，`View`的`longClickable`属性默认是`false`的
- `View`的`enable`属性不会影响`onTouchEvent`的返回值，哪怕该`View`为`disable`的，只要它的`clickable`和`longClickable`其中一个为`true`，那么它的`onTouchEvent`就返回`true`
- 事件传递过程是由外向内的，即事件总是先传给父元素，然后再由父元素分发给子`VIew`,子`View`可以通过`requestDisallowInterceptTouchEvent`来干预父元素的分发过程，但是影响不到`ACTION_DOWN`事件

参考：
[Android触摸屏事件派发机制详解与源码分析二(ViewGroup篇)](http://blog.csdn.net/yanbober/article/details/45912661)
[ Android事件分发机制完全解析，带你从源码的角度彻底理解(下)](http://blog.csdn.net/guolin_blog/article/details/9153747)

