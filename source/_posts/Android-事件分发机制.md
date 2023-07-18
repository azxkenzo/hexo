---
title: Android - 事件分发机制
date: 2022-09-05 11:07:53
tags: Android
---

## 基础知识

### MotionEvent
`MotionEvent` 是用于报告移动（鼠标、笔、手指、轨迹球）事件的对象。Motion Event 可能包含绝对或相对运动和其他数据，具体取决于设备的类型。

Motion Event 根据 action code 和一组 axis values 描述运动。action code 指定发生的状态变化，例如指针向下或向上。 axis values 描述位置和其他运动属性。

例如，当用户第一次触摸屏幕时，系统会使用 action code **ACTION_DOWN** 和一组 axis values（包括触摸的 X 和 Y 坐标以及有关接触区域的压力、大小和方向的信息）向适当的 View 传递 touch event。

一些设备可以同时报告多个移动轨迹。 多点触摸屏为每个手指发出一个移动轨迹。 生成运动轨迹的单个手指或其他对象称为 pointer。 Motion event 包含有关当前活动的所有 pointer 的信息，即使自上次事件传递以来其中一些 pointer 没有移动。

pointer 的数量只会随着单个 pointer 的上下移动而改变一个，除非手势被取消。

每个 pointer 都有一个唯一的 id，在它第一次下降时分配（由 **ACTION_DOWN** 或 **ACTION_POINTER_DOWN** 指示）。 pointer id 保持有效，直到 pointer 最终上升（由 **ACTION_UP** 或 **ACTION_POINTER_UP** 指示）或手势被取消（由 **ACTION_CANCEL** 指示）。

MotionEvent 类提供了许多方法来查询 pointer 的位置和其他属性，例如 `getX(int)`、`getY(int)`、`getAxisValue`、`getPointerId(int)`、`getToolType(int)` 等。 大多数这些方法接受 pointer 索引作为参数而不是 pointer id。 事件中每个 pointer 的 pointer 索引范围从 0 到比 `getPointerCount()` 返回的值小一。

单个 pointer 在运动事件中出现的顺序未定义。 因此，pointer 的 pointer 索引可以从一个事件更改为下一个事件，但只要 pointer 保持活动状态，pointer 的 pointer id 就保证保持不变。 使用 `getPointerId(int)` 方法获取 pointer 的 pointer ID，以在手势中的所有后续运动事件中跟踪它。 然后对于连续的运动事件，使用 `findPointerIndex(int)` 方法获取该运动事件中给定pointer id 的 pointer索引。

### 主要方法
* `Activity.dispatchTouchEvent()`：调用以处理触摸屏事件。 可以重写它以在将所有触摸屏事件发送到 Window 之前拦截它们。 **请务必为应该正常处理的触摸屏事件调用此实现**。如果此事件被消费，则返回 true。
* `Activity.onTouchEvent()`：当触摸屏事件未被其下的任何 View 处理时调用。 这对于处理发生在 window 边界之外的触摸事件最有用，因为那里没有 view 可以接收它。如果消费了该事件，则返回 true，否则返回 false。 默认实现始终返回 false。
* `View.dispatchTouchEvent()`：将触摸屏运动事件向下传递给目标View，如果它是目标View，则传递给该View。如果事件被view处理了则为 true，否则为 false。
* `OnTouchListener.onTouch()`：在触摸事件被分发给 view 时调用。 这使 listener 有机会在目标view之前做出响应。
* `View.onTouchEvent()`：实现此方法来处理触摸屏运动事件。如果使用此方法检测 click 动作，建议通过实现并调用 `performClick()` 来执行动作。 这将确保一致的系统行为
* `ViewGroup.dispatchTouchEvent()`：ViewGroup 将触摸事件分发给合适的子view来处理
* `ViewGroup.onInterceptTouchEvent()`：实现这个方法来拦截所有的触摸屏运动事件。 这使可以在事件发送给children时查看事件，并随时掌握当前手势。使用此函数需要小心，因为它与 `View.onTouchEvent(MotionEvent)` 的交互相当复杂，并且使用它需要以正确的方式实现该方法和这个方法。
   事件将按以下顺序接收：
   1. 将在这里收到 down 事件
   2. down 事件将由该ViewGroup的子view处理，或者交给自己的 `onTouchEvent()` 方法来处理； 这意味着应该实现 `onTouchEvent()` 以返回 true，这样将继续看到手势的其余部分（而不是寻找父view来处理它）。 此外，通过从 onTouchEvent() 返回 true，将不会在 onInterceptTouchEvent() 中收到任何后续事件，并且所有触摸处理必须像往常一样在 onTouchEvent() 中进行。
   3. 只要从此函数返回 false，每个后续事件（直到并包括最后一个 up）将首先传递到此处，然后传递到目标的 onTouchEvent()。
   4. 如果从这里返回 true，将不会收到任何以下事件：目标view将收到相同的事件，但带有动作 MotionEvent.ACTION_CANCEL，并且所有其他事件将被传递到 onTouchEvent() 方法并且不再出现在这里。

### 事件传递顺序
对于一个 MotionEvent，首先会分发给 Activity，接着再将其分发给 DecorView，通过 DecorView 再将事件向下逐级分发。如果向下分发到最后一个view后事件还没被消费，则再交给Activity的`onTouchEvent`处理

## 分发过程分析
事件分发首先从 `Activity.dispatchTouchEvent()` 开始
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
这里，Activity 先调用 Window 的 `superDispatchTouchEvent()` 来处理事件
```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```
Window 把事件交给 DecorView 去处理
```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```
DecorView 调用父类也就是ViewGroup的 `dispatchTouchEvent()` 来处理事件。

### ViewGroup 的事件分发
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) { // 过滤触摸事件以应用安全策略。如果应分派事件，则为 True；如果应删除事件，则为 false。
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // 处理初始 down 事件.
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // 开始新的触摸手势时丢弃所有先前的状态。 由于应用程序切换、ANR 或其他一些状态更改，框架可能已丢弃先前手势的up or cancel事件。
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        // Check for interception.
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0; 
            // 子View可以通过调用父view的 requestDisallowInterceptTouchEvent 来请求禁止拦截事件
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // 没有触摸目标，并且此操作不是初始 down，因此此ViewGroup会继续拦截触摸。
            intercepted = true;
        }

        // 如果被拦截，则开始正常的事件分发。 此外，如果已经有一个正在处理手势的view，请执行正常的事件调度。
        if (intercepted || mFirstTouchTarget != null) {
            ev.setTargetAccessibilityFocus(false);
        }

        // Check for cancelation.
        final boolean canceled = resetCancelNextUpFlag(this) || actionMasked == MotionEvent.ACTION_CANCEL;

        // 如果需要，更新 pointer down 的触摸目标列表。
        final boolean isMouseEvent = ev.getSource() == InputDevice.SOURCE_MOUSE;
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0 && !isMouseEvent;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        
        if (!canceled && !intercepted) {
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus() ? findChildWithAccessibilityFocus() : null;

            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // down 事件的 actionIndex 总为 0
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex) : TouchTarget.ALL_POINTER_IDS;

                // 清除此 pointer ID 的早期触摸目标，以防它们变得不同步。
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                if (childrenCount != 0) {
                    final float x = isMouseEvent ? ev.getXCursorPosition() : ev.getX(actionIndex);
                    final float y = isMouseEvent ? ev.getYCursorPosition() : ev.getY(actionIndex);
                    
                    // 找到一个可以接收事件的child。 从前到后扫描children。
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    final boolean customOrder = preorderedList == null && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);

                        if (!child.canReceivePointerEvents() || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }
                        // 如果子view (可见 或 当前相关联的Animation不为null) 且 事件坐标在子view区域内，则继续执行
                        // 获取指定子view的触摸目标。 如果找不到则返回 null。
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // Child 已经正在其范围内接收touch。 除了它正在处理的pointer之外，还给它一个新的pointer。
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
                        
                        // dispatchTransformedTouchEvent 将 motion event 转换为特定子view的坐标空间，过滤掉不相关的pointer ID，并在必要时覆盖其action。
                        // 如果 child 为 null，则假定 MotionEvent 将改为发送到此 ViewGroup。
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
                            newTouchTarget = addTouchTarget(child, idBitsToAssign); // addTouchTarget 中对 mFirstTouchTarget 进行了赋值
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

### View 的事件分发
```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean result = false;
    
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
如果设置了 OnTouchListener，就先调用 `OnTouchListener.onTouch()`，再判断是否需要调用 `onTouchEvent()`
```java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

    if ((viewFlags & ENABLED_MASK) == DISABLED
            && (mPrivateFlags4 & PFLAG4_ALLOW_CLICK_WHEN_DISABLED) == 0) {
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        // clickable 的 disabled view 仍然会消费 touch event, 但是它不会做任何处理
        return clickable;
    }
    
    // 如果设置了 mTouchDelegate 就把事件交给它来处理，然后直接返回 true
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            // 事件为手指抬起时
            case MotionEvent.ACTION_UP:
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                if ((viewFlags & TOOLTIP) == TOOLTIP) {
                    handleTooltipUp();
                }
                if (!clickable) {
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;
                }
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
                                performClickInternal();
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
                if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                    mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                }
                mHasPerformedLongPress = false;

                if (!clickable) {
                    checkForLongClick(
                            ViewConfiguration.getLongPressTimeout(),
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                    break;
                }

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
                    checkForLongClick(
                            ViewConfiguration.getLongPressTimeout(),
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
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
                if (clickable) {
                    drawableHotspotChanged(x, y);
                }

                final int motionClassification = event.getClassification();
                final boolean ambiguousGesture =
                        motionClassification == MotionEvent.CLASSIFICATION_AMBIGUOUS_GESTURE;
                int touchSlop = mTouchSlop;
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
                        checkForLongClick(
                                delay,
                                x,
                                y,
                                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                    }
                    touchSlop *= mAmbiguousGestureMultiplier;
                }

                // Be lenient about moving outside of buttons
                if (!pointInView(x, y, touchSlop)) {
                    // Outside button
                    // Remove any future long press/tap checks
                    removeTapCallback();
                    removeLongPressCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        setPressed(false);
                    }
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                }

                final boolean deepPress =
                        motionClassification == MotionEvent.CLASSIFICATION_DEEP_PRESS;
                if (deepPress && hasPendingLongPressCallback()) {
                    // process the long click action immediately
                    removeLongPressCallback();
                    checkForLongClick(
                            0 /* send immediately */,
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DEEP_PRESS);
                }

                break;
        }

        return true;
    }

    return false;
}
```
处理逻辑：
* 事件为 `ACTION_UP` 时：如果带有 `PFLAG_PRESSED` 或 `PFLAG_PREPRESSED` 标志，则调用 `performClickInternal` 处理点击事件
* 事件为 `ACTION_DOWN` 时：处理长按事件
* 事件为 `ACTION_MOVE`







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
