---
title: Android - 事件分发机制
date: 2022-09-05 11:07:53
tags: Android
---

# 主要方法
* `Activity.dispatchTouchEvent()`
* `Activity.onTouchEvent()`
* `Window.superDispatchTouchEvent()`
* `ViewGroup.dispatchTouchEvent()`
* `ViewGroup.onInterceptTouchEvent()`
* `View.dispatchTouchEvent()`
* `View.onTouchListener.onTouch()`
* `View.onTouchEvent()`
* `TouchDelegate.onTouchEvent()`
* `View.performClickInternal()`
* `View.performClick()`
* `View.OnClickListener.onClick()`
* `View.performLongClickInternal()`
* `View.onLongClickListener.onLongClick()`

### Activity.dispatchTouchEvent()
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {  
        return true;
    }
    // 如果 Window.superDispatchTouchEvent() 返回false，则调用 onTouchEvent
    return onTouchEvent(ev);
}
```

### ViewGroup.dispatchTouchEvent()
```java
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
        || mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);  // 自定义拦截规则
        ev.setAction(action); // restore action in case it was changed
    } else {
        intercepted = false;
    }
} else {
    // There are no touch targets and this action is not an initial down
    // so this view group continues to intercept touches.
    intercepted = true;
}

...

if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {  // 将event分发给子view
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

...

// Dispatch to touch targets.
if (mFirstTouchTarget == null) {
    // No touch targets so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null,  // 将event分发给子view
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
            if (dispatchTransformedTouchEvent(ev, cancelChild,  // 将event分发给子view
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
```

```java
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

    // If the number of pointers is the same and we don't need to perform any fancy
    // irreversible transformations, then we can reuse the motion event for this
    // dispatch as long as we are careful to revert any changes we make.
    // Otherwise we need to make a copy.
    final MotionEvent transformedEvent;
    if (newPointerIdBits == oldPointerIdBits) {
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                event.offsetLocation(offsetX, offsetY);

                handled = child.dispatchTouchEvent(event);

                event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        transformedEvent = MotionEvent.obtain(event);
    } else {
        transformedEvent = event.split(newPointerIdBits);
    }

    // Perform any necessary transformations and dispatch.
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }

        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // Done.
    transformedEvent.recycle();
    return handled;
}
```

### View.dispatchTouchEvent()
```java
final int actionMasked = event.getActionMasked();
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // Defensive cleanup for new gesture
    stopNestedScroll();
}

...

if (li != null && li.mOnTouchListener != null
        && (mViewFlags & ENABLED_MASK) == ENABLED
        && li.mOnTouchListener.onTouch(this, event)) {
    result = true;
}
// 先调用 onTouchListener.onTouch() ，如果返回false再调用 onTouchEvent
if (!result && onTouchEvent(event)) {
    result = true;
}

...

// Clean up after nested scrolls if this is the end of a gesture;
// also cancel it if we tried an ACTION_DOWN but we didn't want the rest
// of the gesture.
if (actionMasked == MotionEvent.ACTION_UP ||
        actionMasked == MotionEvent.ACTION_CANCEL ||
        (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
    stopNestedScroll();
}
```

### View.onTouchEvent()
```java
if (mTouchDelegate != null) {
    if (mTouchDelegate.onTouchEvent(event)) {
        return true;
    }
}

...

case MotionEvent.ACTION_UP:
    if (!clickable) {
        removeTapCallback();
        removeLongPressCallback();
        mInContextButtonPress = false;
        mHasPerformedLongPress = false;
        mIgnoreNextUpEvent = false;
        break;
    }

    // Use a Runnable and post this rather than calling
    // performClick directly. This lets other visual state
    // of the view update before click actions start.
    if (mPerformClick == null) {
        mPerformClick = new PerformClick();
    }
    if (!post(mPerformClick)) {
        performClickInternal();  // 处理click事件
    }


case MotionEvent.ACTION_DOWN:
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
        checkForLongClick(  // 处理 LongClick 事件
                ViewConfiguration.getLongPressTimeout(),
                x,
                y,
                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
    }


case MotionEvent.ACTION_MOVE:
    if (ambiguousGesture && hasPendingLongPressCallback()) {
        if (!pointInView(x, y, touchSlop)) {
            // The default action here is to cancel long press. But instead, we
            // just extend the timeout here, in case the classification
            // stays ambiguous.
            removeLongPressCallback();
            long delay = (long) (ViewConfiguration.getLongPressTimeout()
                    * mAmbiguousGestureMultiplier);
            // Subtract the time already spent
            delay -= event.getEventTime() - event.getDownTime();
            checkForLongClick(  // 处理 LongClick 事件
                    delay,
                    x,
                    y,
                    TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
        }
        touchSlop *= mAmbiguousGestureMultiplier;
    }
    
    final boolean deepPress =
        motionClassification == MotionEvent.CLASSIFICATION_DEEP_PRESS;
    if (deepPress && hasPendingLongPressCallback()) {
        // process the long click action immediately
        removeLongPressCallback();
        checkForLongClick(  // 处理 LongClick 事件
                0 /* send immediately */,
                x,
                y,
                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DEEP_PRESS);
    }

```