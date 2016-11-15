# View的事件传递机制

```

```

## dispatchTouchEvent方法

```
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
//     默认为false
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
           if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
               result = true;
           }
           //noinspection SimplifiableIfStatement
          //  内部类，内部类里有接口
           ListenerInfo li = mListenerInfo;
           // 且运算 一个为false就为false
           if (li != null && li.mOnTouchListener != null
                   && (mViewFlags & ENABLED_MASK) == ENABLED
// 在这里调用onTouch方法，onTouch方法是有返回值的。
                   && li.mOnTouchListener.onTouch(this, event)) {
                 result = true;
             }
 //           onTouch是一个接口 onTouchEvent是View里的方法
 //         result为 false且onTouchEvent返回true的时候
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

   boolean isAccessibilityFocusedViewOrHost() {
       return isAccessibilityFocused() || (getViewRootImpl() != null && getViewRootImpl()
               .getAccessibilityFocusedHost() == this);
   }

```

dispatchTouchEvent方法里首先执行onTouch方法，这是一个接口里的方法。
然后执行onTouchEvent方法，此方法也定义在View里。
如果控件（View）是enable的，且onTouch返回true时，
dispatchTouchEvent就直接返回true，onTouchEvent不执行。


##  onTouchEvent

```

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
//              ACTION_UP事件 在这里执行点击方法
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
                                // 使用Runnable和post而不是直接调用performClick方法。
                                // 这样就可以在点击动作开始之前，更新其他可见的状态。
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
                        checkForLongClick(0, x, y);
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
在这个方法里，执行了 performClick()方法




## ViewGroup的dispatchTouchEvent方法

```
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
                // 开始一个新的触摸手势之前，要清掉之前所有的状态。
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            // 检查拦截
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                // 当事件为 ACTION_DOWN，或者有touch targets，就执行如下代码。
                //  requestDisallowInterceptTouchEvent方法
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                  // 如果没有disallow就执行onInterceptTouchEvent方法，即拦截方法。
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                // 没有touch targets，且不是ACTION_DOWN事件，就返回true,表示已拦截。
                intercepted = true;
            }

            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            // 如果被拦截了，就弄一个normal的事件分发
            // 还有如果现在正在有一个view处理这个手势也弄一个normal的事件分发
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

            // Check for cancelation.
            // 检查取消
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            // 给pointer down更新touch targets的列表
            // 下面开始真正的事件分发

            // setMotionEventSplittingEnabled方法可以设置这个标志
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            // 如果没有被取消，且没有被拦截。
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
                    // child个数不为0 进if
                    if (newTouchTarget == null && childrenCount != 0)
                     {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        // 从前往后寻找哪个child可以接收这个事件
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;

                        // 循环
                        for (int i = childrenCount - 1; i >= 0; i--)
                        {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            // 拿到  child view
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null)
                            {
                                if (childWithAccessibilityFocus != child)
                                {
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
                            // TouchTarget是一个ViewGroup的内部类，存了 view和pointerIdBits
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null)
                            {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            // 开始分发
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                // child想要接收这个 touch
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
                                // 已经被分发到新的 Touch Target里去了
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        // 清空 preorderedList for循环之外了
                        if (preorderedList != null) preorderedList.clear();
                    }


//                  没找到
                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        // 没找到接收touch event的 child
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
                DOWN事件的括号
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

## requestDisallowInterceptTouchEvent设置是否禁止拦截

```
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
