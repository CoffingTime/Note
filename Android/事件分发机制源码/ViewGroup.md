

### ViewGroup
这一篇是续上篇[Android 事件分发机制源码攻略（一） —— Activity篇](http://blog.csdn.net/u013927241/article/details/76945539) 的ViewGroup，想了解Activity篇的也可以点击查看。
这篇算是Android事件分发中最为关键的一篇，因为这里会分析哪些事件会被拦截，是以何种形式获取子View，以及对ACTION_DOWN后续事件传递等问题，都会在这里得到答案。好了，废话不多说，现在开始分析。
上一篇，我们走到的ViewGroup的dispatchTouchEvent（）这个方法。以下该方法的源码（有所省略），以及我个人的注释，篇幅有点长，我这边提供一个粗略的逻辑线路给大家，这个会比较容易理解。

> 如果是ACTION_DOWN事件，就会去寻找子View来处理，如果找不到子View来处理，就自己处理。
> 如果不是ACTION_DOWN事件，就会把这个事件传给处理了ACTION_DOWN事件的View来处理。

大致就这两个逻辑，虽说比较粗略，不过，这对于接下来看源码就足够了，并且源码有比较多的注释，基本上大致的方向是可以弄懂了。
```java
@Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        
        ... 
        //返回值的关键，注意留意handled的值发生改变的地方
        boolean handled = false;
        //判断当前window是否有被遮挡，true为分发这个事件，false为丢弃这个事件
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
         //在新的事件开始（即是新的ACTION_DOWN事件），需要清除掉之前的状态以及设置mFirstTouchTarget = null;
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
            //子View唯一一个可以用来控制父类事件传递
            //只有ACTION_DOWN事件跟mFirstTouchTarget不为空的情况，后面的讨论大多是围绕着mFirstTouchTarget来进行的
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                //是否拦截事件，disallowIntercept为true是不拦截，false是拦截
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
	                //一般重写onInterceptTouchEvent方法
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
            //split是否分发给多个子View，默认为false
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            //如果不被拦截即可进入或者不是ACTION_CANCEL事件
            if (!canceled && !intercepted) {

                // If the event is targeting accessiiblity focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;
				//只有ACTION_DOWN等事件能够进入
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
                        //获取按Z轴从大到小排序的子View列表
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        //是否有自定义顺序，一般为false
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
	                        //确认这个子View的下标
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            //确认这个子View
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

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
							//是否获得可见，并且落在child的布局范围内
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }
							//Child是否已经处理过事件了，有的话更改pointerIdBits值，并结束查找
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            //分发给View的dispatchTouchEvent
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
                                //给mFirstTouchTarget赋值，该事件已经被子View确认处理了
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
            // 没有子View处理，则自己处理
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
				//处理除了ACTION_DOWN以外的事件
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        //如果这个事件被拦截了，intercepted为true
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        //如果事件被拦截掉，
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
像这么长的代码，很多地方是可以跳过的，不过仔仔细细分析，特别是像Google出品的（个人拙见），因为这些东西考虑的方方面面比较多，而我们这个只是为了了解事件的分发，绘制那块我们不会过多涉及。（说跑题了）回到正题来，像这么长的代码，之前学习的时候，有个牛人是这么写的（个人总结）。

```
从结果出发，留意改变的结果的地方
```
上面的dispatchTouchEvent返回值是由handle决定，我们先来看第一处第8行代码

```java
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
        ...
        }
        return false;
```
这个onFilterTouchEventForSecurity方法如果返回false的话，基本上里面的代码都不用分析了，直接返回false。那我们进去看看这个方法做了什么。

```java
	/**
     * Filter the touch event to apply security policies.
     *
     * @param event The motion event to be filtered.
     * @return True if the event should be dispatched, false if the event should be dropped.
     *
     * @see #getFilterTouchesWhenObscured
     */
    public boolean onFilterTouchEventForSecurity(MotionEvent event) {
        //noinspection RedundantIfStatement
        if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
                && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
            // Window is obscured, drop this touch.
            return false;
        }
        return true;
    }
```
这是一个安全策略方面的过滤，我们来看下这两个变量FILTER_TOUCHES_WHEN_OBSCURED、MotionEvent.FLAG_WINDOW_IS_OBSCURED是什么意思

```java
	/**
     * Indicates that the view should filter touches when its window is obscured.
     * Refer to the class comments for more information about this security feature.
     * {@hide}
     */
    static final int FILTER_TOUCHES_WHEN_OBSCURED = 0x00000400;
```

```java
	/**
     * This flag indicates that the window that received this motion event is partly
     * or wholly obscured by another visible window above it.  This flag is set to true
     * even if the event did not directly pass through the obscured area.
     * A security sensitive application can check this flag to identify situations in which
     * a malicious application may have covered up part of its content for the purpose
     * of misleading the user or hijacking touches.  An appropriate response might be
     * to drop the suspect touches or to take additional precautions to confirm the user's
     * actual intent.
     */
    public static final int FLAG_WINDOW_IS_OBSCURED = 0x1;
```
从上面的代码注释可以看出来，这个View不能被其他的window遮挡住，这是谷歌的一个安全策略，避免被恶意程序误导用户或劫持触摸。
第二处handle的改变是在172行

```java
		if (mFirstTouchTarget == null) {
               // No touch targets so treat this as an ordinary view.
               handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
        ...
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
                ...
```
很明显handled的值又跟mFirstTouchTarget、alreadyDispatchedToNewTouchTarget这两个值有关，另外还跟dispatchTransformedTouchEvent（）这个方法有关，dispatchTransformedTouchEvent（）方法，我们留在后面分析，我们先来看看这两个值是在什么时候在哪里被改变的。

```java
		 mLastTouchDownX = ev.getX();
         mLastTouchDownY = ev.getY();
         //给mFirstTouchTarget赋值，该事件已经被子View确认处理了
         newTouchTarget = addTouchTarget(child, idBitsToAssign);
         alreadyDispatchedToNewTouchTarget = true;
   
```
这个是第145行的代码，这里是找到处理事件的子View后，做的赋值，addTouchTarget这个方法里面会对
mFirstTouchTarget赋值。好了，如果是这样，我们就得从上面的第13行开始分析。

```java
 boolean handled = false;
        //判断当前window是否有被遮挡，true为分发这个事件，false为丢弃这个事件
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;
```



