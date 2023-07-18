---
title: Android - RecyclerView
date: 2023-02-24 17:28:34
tags: Android
---

## RecyclerView
* **Adapter**：提供表示数据集中 item 的 view
* **Position**：Adapter 中数据项的位置
* **Index**：调用 ViewGroup.getChildAt 时使用的 attached 子View 的索引
* **Binding**：准备子View以显示与 Adapter 内某个 position 对应数据的过程
* **Recycle**(view)：之前用于显示特定 adapter position 的数据的 view 可能会被放置在缓存中以供以后重用以稍后再次显示相同类型的数据
* **Scrap**(view)：在 layout 时进入暂时分离状态的子View。可以在不完全脱离父RecyclerView的情况下重用 Scrap view，如果不需要重新绑定则不修改，或者 view 被认为是 dirty 则由 adapter 修改
* **Dirty**(view)：一个子View，在显示之前必须被 adapter 重新绑定

### Position
RecyclerView 中有两类 position 相关的方法：
* layout position：item 在最新的 layout计算 中的 position
* adapter position：item 在 adapter 中的位置

除了调度 adapter.notify* 事件和计算 updated layout 的时间之外，这两个 position 是相同的

返回或接收 LayoutPosition 的方法使用最新布局计算的 position。这些 position 包括最后一次布局计算之前的所有更改。可以依赖这些 position 来与当前屏幕上看到的内容保持一致

另一组方法采用 AdapterPosition 形式。当需要使用最新的 adapter position 时应该使用这些方法，即使它们可能还没有反映到 layout

### 呈现动态数据
在 Recycler 中显示可更新数据，adapter 要发出 insert、move 和 delete 信号。可以通过在内容更改时手动调用 adapter.notify* 方法，或者使用 RecyclerView 提供的解决方案：
* 使用 DiffUtil 列出差异：如果 RecyclerView 显示的列表是为每次更新从头开始重新获取的（例如从网络或数据库），DiffUtil 可以计算列表版本之间的差异。 
DiffUtil 将两个列表作为输入并计算差异，可以将差异传递给 RecyclerView 以触发最少的动画和更新以保持 UI 性能和动画有意义。
这种方法要求每个列表在内存中以不可变的内容表示，并依赖于接收更新作为列表的新实例。 如果 UI 层没有实现排序，这种方法也是理想的，它只是按照给定的顺序显示数据。  
这种方法最好的部分是它扩展到任何任意更改——项目更新、移动、添加和删除都可以用相同的方式计算和处理。 虽然在比较时必须在内存中保留列表的两个副本，并且必须避免改变它们，但可以在列表版本之间共享未修改的元素。  
对于 RecyclerView，主要有三种方法可以做到这一点。 建议从 ListAdapter 开始，它是在后台线程上用最少的代码构建在 List diffing 中的更高级别的 API。 AsyncListDiffer 也提供了这种行为，但没有为子类定义适配器。 如果想要更多的控制，DiffUtil 是可以用来自己计算差异的低级 API。 每种方法都允许指定应如何根据项目数据计算差异。
* 使用 SortedList 列出变异：如果 RecyclerView 以增量方式接收更新，例如 插入item X，或删除item Y，可以使用 SortedList 来管理列表。
定义如何 order item，它会自动触发 RecyclerView 可以使用的更新信号。 如果只需要处理插入和删除事件，SortedList 就可以工作，并且好处是只需要在内存中拥有一个列表副本。 它还可以计算与 SortedList.replaceAll(Object[]) 的差异，但此方法比上面的列表差异行为更受限制。

### 布局过程
onLayout() -> dispatchLayout() -> dispatchLayoutStep1() -> dispatchLayoutStep2() -> LayoutManager.onLayoutChildren() -> dispatchLayoutStep3()

* `dispatchLayout()`：layoutChildren() 的包装器，用于处理由布局引起的动画变化。
* `dispatchLayoutStep1()`：布局的第一步； - 处理 adapter 更新 - 决定应该运行哪个动画 - 保存有关当前 views 的信息 - 如有必要，运行预测布局并保存其信息
* `dispatchLayoutStep2()`：第二个布局步骤，对最终状态的 view 进行实际布局。 如有必要（例如测量），此步骤可能会运行多次。
* `LayoutManager.onLayoutChildren()`：布置给定 adapter 的所有相关子视图。
* `dispatchLayoutStep3()`：布局的最后一步，保存有关用于动画的 view 的信息，触发动画并进行任何必要的清理。


### 测量流程
RecyclerView.`onMeasure()` 做了什么：
```java
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {
        defaultOnMeasure(widthSpec, heightSpec);
        return;
    }
    
    if (mLayout.isAutoMeasureEnabled()) {
        final int widthMode = MeasureSpec.getMode(widthSpec);
        final int heightMode = MeasureSpec.getMode(heightSpec);

        /**
         * This specific call should be considered deprecated and replaced with
         * {@link #defaultOnMeasure(int, int)}. It can't actually be replaced as it could
         * break existing third party code but all documentation directs developers to not
         * override {@link LayoutManager#onMeasure(int, int)} when
         * {@link LayoutManager#isAutoMeasureEnabled()} returns true.
         */
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

        // 计算并跟踪是否应该在此处跳过测量，因为两个维度中的 MeasureSpec 模式都是 EXACTLY。
        mLastAutoMeasureSkippedDueToExact =
                widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
        if (mLastAutoMeasureSkippedDueToExact || mAdapter == null) {
            return;
        }

        // 第一步
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
        }
        // 在第二步中设置尺寸。 为了保持一致性，应该使用旧尺寸进行预布局
        mLayout.setMeasureSpecs(widthSpec, heightSpec);
        mState.mIsMeasuring = true;
        // 第二步
        dispatchLayoutStep2();

        // 现在可以从 children 那里得到宽度和高度。
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

        // 如果 RecyclerView 的宽度和高度不准确，并且至少有一个 child 的宽度和高度也不准确，必须重新测量。
        if (mLayout.shouldMeasureTwice()) {
            mLayout.setMeasureSpecs(
                    MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                    MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
            mState.mIsMeasuring = true;
            dispatchLayoutStep2();
            
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
        }

        mLastAutoMeasureNonExactMeasuredWidth = getMeasuredWidth();
        mLastAutoMeasureNonExactMeasuredHeight = getMeasuredHeight();
    } else {
    
    }
}
```
State.`mLayoutStep` 有 3 种取值情况：
* `STEP_START`：默认值，这种情况下，表示 RecyclerView 还未经历 `dispatchLayoutStep1`，因为 `dispatchLayoutStep1` 调用之后 mState.mLayoutStep 会变为 State.STEP_LAYOUT。
* `STEP_LAYOUT`：当 mState.mLayoutStep 为 State.STEP_LAYOUT 时，表示此时处于 layout 阶段，这个阶段会调用 dispatchLayoutStep2 方法 layout RecyclerView 的children。调用 dispatchLayoutStep2 方法之后，此时 mState.mLayoutStep 变为了 State.STEP_ANIMATIONS。
* `STEP_ANIMATIONS`：当 mState.mLayoutStep 为 State.STEP_ANIMATIONS 时，表示 RecyclerView 处于第三个阶段，也就是执行动画的阶段，也就是调用 dispatchLayoutStep3方法。当 dispatchLayoutStep3 方法执行完毕之后，mState.mLayoutStep 又变为了 State.STEP_START。

在 onMeasure 中，RV 根据 LayoutManager 的状态做了不同处理：
* 未设置 LayoutManager 时，调用 `defaultOnMeasure()` 进行测量
* LayoutManager 启用了 AutoMeasure 时，先调用 `LayoutManager.onMeasure()` 进行测量，然后调用 `dispatchLayoutStep1()` 和 `dispatchLayoutStep2()` 完成布局流程的前两步
* LayoutManager 未启用 AutoMeasure 时，


#### dispatchLayoutStep1
```java
The first step of a layout where we; 
- process adapter updates 
- decide which animation should run 
- save information about current views 
- If necessary, run predictive layout and save its information

private void dispatchLayoutStep1() {
    mState.assertLayoutStep(State.STEP_START);
    fillRemainingScrollValues(mState);
    mState.mIsMeasuring = false;
    startInterceptRequestLayout();
    mViewInfoStore.clear();
    onEnterLayoutOrScroll();
    // 处理 adapter 更新
    processAdapterUpdatesAndSetAnimationFlags();
    saveFocusInfo();
    mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
    mItemsAddedOrRemoved = mItemsChanged = false;
    mState.mInPreLayout = mState.mRunPredictiveAnimations;
    mState.mItemCount = mAdapter.getItemCount();
    findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);

    if (mState.mRunSimpleAnimations) {
        // Step 0: 找出所有未删除的 item 在哪里，预布局
        int count = mChildHelper.getChildCount();
        for (int i = 0; i < count; ++i) {
            final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
            if (holder.shouldIgnore() || (holder.isInvalid() && !mAdapter.hasStableIds())) {
                continue;
            }
            final ItemHolderInfo animationInfo = mItemAnimator
                    .recordPreLayoutInformation(mState, holder,
                            ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                            holder.getUnmodifiedPayloads());
            mViewInfoStore.addToPreLayout(holder, animationInfo);
            if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.isRemoved()
                    && !holder.shouldIgnore() && !holder.isInvalid()) {
                long key = getChangedHolderKey(holder);
                // This is NOT the only place where a ViewHolder is added to old change holders
                // list. There is another case where:
                //    * A VH is currently hidden but not deleted
                //    * The hidden item is changed in the adapter
                //    * Layout manager decides to layout the item in the pre-Layout pass (step1)
                // When this case is detected, RV will un-hide that view and add to the old
                // change holders list.
                mViewInfoStore.addToOldChangeHolders(key, holder);
            }
        }
    }
    if (mState.mRunPredictiveAnimations) {
        // Step 1: 进行预布局: This will use the old positions of items. The layout manager
        // is expected to layout everything, even removed items (though not to add removed
        // items back to the container). This gives the pre-layout position of APPEARING views
        // which come into existence as part of the real layout.

        // Save old positions so that LayoutManager can run its mapping logic.
        saveOldPositions();
        final boolean didStructureChange = mState.mStructureChanged;
        mState.mStructureChanged = false;
        // temporarily disable flag because we are asking for previous layout
        mLayout.onLayoutChildren(mRecycler, mState);
        mState.mStructureChanged = didStructureChange;

        for (int i = 0; i < mChildHelper.getChildCount(); ++i) {
            final View child = mChildHelper.getChildAt(i);
            final ViewHolder viewHolder = getChildViewHolderInt(child);
            if (viewHolder.shouldIgnore()) {
                continue;
            }
            if (!mViewInfoStore.isInPreLayout(viewHolder)) {
                int flags = ItemAnimator.buildAdapterChangeFlagsForAnimations(viewHolder);
                boolean wasHidden = viewHolder
                        .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
                if (!wasHidden) {
                    flags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
                }
                final ItemHolderInfo animationInfo = mItemAnimator.recordPreLayoutInformation(
                        mState, viewHolder, flags, viewHolder.getUnmodifiedPayloads());
                if (wasHidden) {
                    recordAnimationInfoIfBouncedHiddenView(viewHolder, animationInfo);
                } else {
                    mViewInfoStore.addToAppearedInPreLayoutHolders(viewHolder, animationInfo);
                }
            }
        }
        // we don't process disappearing list because they may re-appear in post layout pass.
        clearOldPositions();
    } else {
        clearOldPositions();
    }
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
    mState.mLayoutStep = State.STEP_LAYOUT;
}


private void processAdapterUpdatesAndSetAnimationFlags() {
    if (mDataSetHasChangedAfterLayout) {
        // Processing these items have no value since data set changed unexpectedly.
        // Instead, we just reset it.
        mAdapterHelper.reset();
        if (mDispatchItemsChangedEvent) {
            mLayout.onItemsChanged(this);
        }
    }
    // simple animations are a subset of advanced animations (which will cause a
    // pre-layout step)
    // If layout supports predictive animations, pre-process to decide if we want to run them
    if (predictiveItemAnimationsEnabled()) {
        mAdapterHelper.preProcess();
    } else {
        mAdapterHelper.consumeUpdatesInOnePass();
    }
    boolean animationTypeSupported = mItemsAddedOrRemoved || mItemsChanged;
    mState.mRunSimpleAnimations = mFirstLayoutComplete
            && mItemAnimator != null
            && (mDataSetHasChangedAfterLayout
            || animationTypeSupported
            || mLayout.mRequestedSimpleAnimations)
            && (!mDataSetHasChangedAfterLayout
            || mAdapter.hasStableIds());
    mState.mRunPredictiveAnimations = mState.mRunSimpleAnimations
            && animationTypeSupported
            && !mDataSetHasChangedAfterLayout
            && predictiveItemAnimationsEnabled();
}
```
`dispatchLayoutStep1` 主要做以下几件事：
1. 处理 adapter 更新
2. 决定执行哪个动画
3. 保存每个 view 的信息
4. 有必要的话，进行预布局并保存相关信息

#### dispatchLayoutStep2
```java
The second layout step where we do the actual layout of the views for the final state. This step might be run multiple times if necessary (e.g. measure).

private void dispatchLayoutStep2() {
    startInterceptRequestLayout();
    onEnterLayoutOrScroll();
    mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
    mAdapterHelper.consumeUpdatesInOnePass();
    mState.mItemCount = mAdapter.getItemCount();  // 
    mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;
    if (mPendingSavedState != null && mAdapter.canRestoreState()) {
        if (mPendingSavedState.mLayoutState != null) {
            mLayout.onRestoreInstanceState(mPendingSavedState.mLayoutState);
        }
        mPendingSavedState = null;
    }
    // Step 2: 进行布局
    mState.mInPreLayout = false;
    mLayout.onLayoutChildren(mRecycler, mState);

    mState.mStructureChanged = false;

    // onLayoutChildren may have caused client code to disable item animations; re-check
    mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
    mState.mLayoutStep = State.STEP_ANIMATIONS;
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
}
```
`dispatchLayoutStep2` 通过调用 mLayout.`onLayoutChildren()` 来完成布局。

mLayout.`onLayoutChildren()` 是如何执行的：
```java
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    // layout algorithm:
    // 1) 通过检查 children 和其他变量，找到锚点坐标和锚点项目位置
    // 2) fill towards start, stacking from bottom
    // 3) fill towards end, stacking from top
    // 4) scroll to fulfill requirements like stack from bottom.
    
    // create layout state
    if (mPendingSavedState != null || mPendingScrollPosition != RecyclerView.NO_POSITION) {
        if (state.getItemCount() == 0) {
            removeAndRecycleAllViews(recycler);
            return;
        }
    }
    if (mPendingSavedState != null && mPendingSavedState.hasValidAnchor()) {
        mPendingScrollPosition = mPendingSavedState.mAnchorPosition;
    }
    ensureLayoutState();
    mLayoutState.mRecycle = false;
    // resolve layout direction
    resolveShouldLayoutReverse();
    // 寻找 子view 的锚点坐标及位置
    final View focused = getFocusedChild();
    if (!mAnchorInfo.mValid || mPendingScrollPosition != RecyclerView.NO_POSITION
            || mPendingSavedState != null) {
        mAnchorInfo.reset();
        mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
        // calculate anchor position and coordinate
        updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
        mAnchorInfo.mValid = true;
    } else if (focused != null && (mOrientationHelper.getDecoratedStart(focused)
            >= mOrientationHelper.getEndAfterPadding()
            || mOrientationHelper.getDecoratedEnd(focused)
            <= mOrientationHelper.getStartAfterPadding())) {
        // This case relates to when the anchor child is the focused view and due to layout
        // shrinking the focused view fell outside the viewport, e.g. when soft keyboard shows
        // up after tapping an EditText which shrinks RV causing the focused view (The tapped
        // EditText which is the anchor child) to get kicked out of the screen. Will update the
        // anchor coordinate in order to make sure that the focused view is laid out. Otherwise,
        // the available space in layoutState will be calculated as negative preventing the
        // focused view from being laid out in fill.
        // Note that we won't update the anchor position between layout passes (refer to
        // TestResizingRelayoutWithAutoMeasure), which happens if we were to call
        // updateAnchorInfoForLayout for an anchor that's not the focused view (e.g. a reference
        // child which can change between layout passes).
        mAnchorInfo.assignFromViewAndKeepVisibleRect(focused, getPosition(focused));
    }
    
    // LLM may decide to layout items for "extra" pixels to account for scrolling target,
    // caching or predictive animations.
    mLayoutState.mLayoutDirection = mLayoutState.mLastScrollDelta >= 0
            ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
    mReusableIntPair[0] = 0;
    mReusableIntPair[1] = 0;
    calculateExtraLayoutSpace(state, mReusableIntPair);
    int extraForStart = Math.max(0, mReusableIntPair[0])
            + mOrientationHelper.getStartAfterPadding();
    int extraForEnd = Math.max(0, mReusableIntPair[1])
            + mOrientationHelper.getEndPadding();
    if (state.isPreLayout() && mPendingScrollPosition != RecyclerView.NO_POSITION
            && mPendingScrollPositionOffset != INVALID_OFFSET) {
        // if the child is visible and we are going to move it around, we should layout
        // extra items in the opposite direction to make sure new items animate nicely
        // instead of just fading in
        final View existing = findViewByPosition(mPendingScrollPosition);
        if (existing != null) {
            final int current;
            final int upcomingOffset;
            if (mShouldReverseLayout) {
                current = mOrientationHelper.getEndAfterPadding()
                        - mOrientationHelper.getDecoratedEnd(existing);
                upcomingOffset = current - mPendingScrollPositionOffset;
            } else {
                current = mOrientationHelper.getDecoratedStart(existing)
                        - mOrientationHelper.getStartAfterPadding();
                upcomingOffset = mPendingScrollPositionOffset - current;
            }
            if (upcomingOffset > 0) {
                extraForStart += upcomingOffset;
            } else {
                extraForEnd -= upcomingOffset;
            }
        }
    }
    int startOffset;
    int endOffset;
    final int firstLayoutDirection;
    if (mAnchorInfo.mLayoutFromEnd) {
        firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_TAIL
                : LayoutState.ITEM_DIRECTION_HEAD;
    } else {
        firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD
                : LayoutState.ITEM_DIRECTION_TAIL;
    }
    onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
    // detach 子view 并缓存到 Recycler 中
    detachAndScrapAttachedViews(recycler);
    mLayoutState.mInfinite = resolveIsInfinite();
    mLayoutState.mIsPreLayout = state.isPreLayout();
    // noRecycleSpace not needed: recycling doesn't happen in below's fill
    // invocations because mScrollingOffset is set to SCROLLING_OFFSET_NaN
    mLayoutState.mNoRecycleSpace = 0;
    if (mAnchorInfo.mLayoutFromEnd) {
        // fill towards start
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtraFillSpace = extraForStart;
        // fill
        fill(recycler, mLayoutState, state, false);
        startOffset = mLayoutState.mOffset;
        final int firstElement = mLayoutState.mCurrentPosition;
        if (mLayoutState.mAvailable > 0) {
            extraForEnd += mLayoutState.mAvailable;
        }
        // fill towards end
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtraFillSpace = extraForEnd;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        fill(recycler, mLayoutState, state, false);
        endOffset = mLayoutState.mOffset;
        if (mLayoutState.mAvailable > 0) {
            // end could not consume all. add more items towards start
            extraForStart = mLayoutState.mAvailable;
            updateLayoutStateToFillStart(firstElement, startOffset);
            mLayoutState.mExtraFillSpace = extraForStart;
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;
        }
    } else {
        // fill towards end
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtraFillSpace = extraForEnd;
        fill(recycler, mLayoutState, state, false);
        endOffset = mLayoutState.mOffset;
        final int lastElement = mLayoutState.mCurrentPosition;
        if (mLayoutState.mAvailable > 0) {
            extraForStart += mLayoutState.mAvailable;
        }
        // fill towards start
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtraFillSpace = extraForStart;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        fill(recycler, mLayoutState, state, false);
        startOffset = mLayoutState.mOffset;
        if (mLayoutState.mAvailable > 0) {
            extraForEnd = mLayoutState.mAvailable;
            // start could not consume all it should. add more items towards end
            updateLayoutStateToFillEnd(lastElement, endOffset);
            mLayoutState.mExtraFillSpace = extraForEnd;
            fill(recycler, mLayoutState, state, false);
            endOffset = mLayoutState.mOffset;
        }
    }
    // changes may cause gaps on the UI, try to fix them.
    // TODO we can probably avoid this if neither stackFromEnd/reverseLayout/RTL values have
    // changed
    if (getChildCount() > 0) {
        // because layout from end may be changed by scroll to position
        // we re-calculate it.
        // find which side we should check for gaps.
        if (mShouldReverseLayout ^ mStackFromEnd) {
            int fixOffset = fixLayoutEndGap(endOffset, recycler, state, true);
            startOffset += fixOffset;
            endOffset += fixOffset;
            fixOffset = fixLayoutStartGap(startOffset, recycler, state, false);
            startOffset += fixOffset;
            endOffset += fixOffset;
        } else {
            int fixOffset = fixLayoutStartGap(startOffset, recycler, state, true);
            startOffset += fixOffset;
            endOffset += fixOffset;
            fixOffset = fixLayoutEndGap(endOffset, recycler, state, false);
            startOffset += fixOffset;
            endOffset += fixOffset;
        }
    }
    layoutForPredictiveAnimations(recycler, state, startOffset, endOffset);
    if (!state.isPreLayout()) {
        mOrientationHelper.onLayoutComplete();
    } else {
        mAnchorInfo.reset();
    }
    mLastStackFromEnd = mStackFromEnd;
}
```


#### detachAndScrapAttachedViews
`detachAndScrapAttachedViews()` 的作用是：暂时 detach 并 scrap 所有当前附加的 子view。view 将被 scrap 到给定的 Recycler 中。 Recycler 可能更愿意在之前回收的其他 view 之前重用 scrap view。
```java
public void detachAndScrapAttachedViews(@NonNull Recycler recycler) {
    final int childCount = getChildCount();
    for (int i = childCount - 1; i >= 0; i--) {
        final View v = getChildAt(i);
        scrapOrRecycleView(recycler, i, v);
    }
}

private void scrapOrRecycleView(Recycler recycler, int index, View view) {
    final ViewHolder viewHolder = getChildViewHolderInt(view);
    if (viewHolder.shouldIgnore()) {
        return;
    }
    if (viewHolder.isInvalid() && !viewHolder.isRemoved()
            && !mRecyclerView.mAdapter.hasStableIds()) {
        // remove 子view 并缓存
        removeViewAt(index);
        recycler.recycleViewHolderInternal(viewHolder);
    } else {
        // detach 子view 并缓存
        detachViewAt(index);
        recycler.scrapView(view);
        mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
    }
}

void recycleViewHolderInternal(ViewHolder holder) {
    if (holder.isScrap() || holder.itemView.getParent() != null) {
        throw new IllegalArgumentException();
    }
    if (holder.isTmpDetached()) {
        throw new IllegalArgumentException();
    }
    if (holder.shouldIgnore()) {
        throw new IllegalArgumentException();
    }
    final boolean transientStatePreventsRecycling = holder
            .doesTransientStatePreventRecycling();
    @SuppressWarnings("unchecked") final boolean forceRecycle = mAdapter != null
            && transientStatePreventsRecycling
            && mAdapter.onFailedToRecycleView(holder);
    boolean cached = false;
    boolean recycled = false;
    if (DEBUG && mCachedViews.contains(holder)) {
        throw new IllegalArgumentException();
    }
    if (forceRecycle || holder.isRecyclable()) {
        if (mViewCacheMax > 0
                && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                | ViewHolder.FLAG_REMOVED
                | ViewHolder.FLAG_UPDATE
                | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
            // Retire oldest cached view
            int cachedViewSize = mCachedViews.size();
            // 超过缓存容量，把最旧的 view 放到 RecycledViewPool 里
            if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                recycleCachedViewAt(0);
                cachedViewSize--;
            }

            int targetCacheIndex = cachedViewSize;
            if (ALLOW_THREAD_GAP_WORK
                    && cachedViewSize > 0
                    && !mPrefetchRegistry.lastPrefetchIncludedPosition(holder.mPosition)) {
                // when adding the view, skip past most recently prefetched views
                int cacheIndex = cachedViewSize - 1;
                while (cacheIndex >= 0) {
                    int cachedPos = mCachedViews.get(cacheIndex).mPosition;
                    if (!mPrefetchRegistry.lastPrefetchIncludedPosition(cachedPos)) {
                        break;
                    }
                    cacheIndex--;
                }
                targetCacheIndex = cacheIndex + 1;
            }
            // 缓存 子view
            mCachedViews.add(targetCacheIndex, holder);
            cached = true;
        }
        if (!cached) {
            addViewHolderToRecycledViewPool(holder, true);
            recycled = true;
        }
    } else {
        // NOTE: A view can fail to be recycled when it is scrolled off while an animation
        // runs. In this case, the item is eventually recycled by
        // ItemAnimatorRestoreListener#onAnimationFinished.

        // TODO: consider cancelling an animation when an item is removed scrollBy,
        // to return it to the pool faster
    }
    // even if the holder is not removed, we still call this method so that it is removed
    // from view holder lists.
    mViewInfoStore.removeViewHolder(holder);
    if (!cached && !recycled && transientStatePreventsRecycling) {
        holder.mBindingAdapter = null;
        holder.mOwnerRecyclerView = null;
    }
}

void recycleCachedViewAt(int cachedViewIndex) {
    ViewHolder viewHolder = mCachedViews.get(cachedViewIndex);
    addViewHolderToRecycledViewPool(viewHolder, true);
    mCachedViews.remove(cachedViewIndex);
}

void scrapView(View view) {
    final ViewHolder holder = getChildViewHolderInt(view);
    if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
            || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
        if (holder.isInvalid() && !holder.isRemoved() && !mAdapter.hasStableIds()) {
            throw new IllegalArgumentException();
        }
        holder.setScrapContainer(this, false);
        mAttachedScrap.add(holder);
    } else {
        if (mChangedScrap == null) {
            mChangedScrap = new ArrayList<ViewHolder>();
        }
        holder.setScrapContainer(this, true);
        mChangedScrap.add(holder);
    }
}
```
关于 detach 和 remove ：
* detach：将 子view 从 父View 的 childView 数组中，子view 的 mParent 设为 null
* remove：不仅从 childView 数组中被移除，其他与 view树 有关的信息也会被清除，如焦点等

缓存策略：
* VH无效 且 VH没有被remove 且 adapter没有稳定ID 时：remove 子View 并缓存，缓存策略为：
  * 优先缓存到 `mCachedViews` 中。若 mCachedViews 已满，则将 mCachedViews 中最先放入到view移除并缓存到RecycledViewPool中，然后再将view缓存到mCachedViews中
  * 若上一步执行完后，view没有被缓存到mCachedViews中，则将其缓存到`RecycledViewPool`中
* 否则，detach 子View 并缓存，缓存策略为：
  * 若 VH被remove或无效 或 VH未更新 或 可以重用已更新的VH，则缓存到 `mAttachedScrap` 中
  * 否则，缓存到 `mChangedScrap` 中

#### fill
```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
        RecyclerView.State state, boolean stopOnFocusable) {
    // max offset we should set is mFastScroll + available
    final int start = layoutState.mAvailable;
    if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
        // TODO ugly bug fix. should not happen
        if (layoutState.mAvailable < 0) {
            layoutState.mScrollingOffset += layoutState.mAvailable;
        }
        recycleByLayoutState(recycler, layoutState);
    }
    int remainingSpace = layoutState.mAvailable + layoutState.mExtraFillSpace;
    LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
    // 只加载一个屏幕能显示的 子view
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        layoutChunkResult.resetInternal();
        // 加载 子view
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
        if (layoutChunkResult.mFinished) {
            break;
        }
        layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
        /**
         * Consume the available space if:
         * * layoutChunk did not request to be ignored
         * * OR we are laying out scrap children
         * * OR we are not doing pre-layout
         */
        if (!layoutChunkResult.mIgnoreConsumed || layoutState.mScrapList != null
                || !state.isPreLayout()) {
            layoutState.mAvailable -= layoutChunkResult.mConsumed;
            // we keep a separate remaining space because mAvailable is important for recycling
            remainingSpace -= layoutChunkResult.mConsumed;
        }

        if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
            layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
            if (layoutState.mAvailable < 0) {
                layoutState.mScrollingOffset += layoutState.mAvailable;
            }
            // 把移出屏幕的 view 缓存到 Recycler 中
            recycleByLayoutState(recycler, layoutState);
        }
        if (stopOnFocusable && layoutChunkResult.mFocusable) {
            break;
        }
    }
    return start - layoutState.mAvailable;
}

void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
        LayoutState layoutState, LayoutChunkResult result) {
    // 从 Recycler 或 adapter 获取一个 view
    View view = layoutState.next(recycler);
    if (view == null) {
        if (DEBUG && layoutState.mScrapList == null) {
            throw new RuntimeException("received null view when unexpected");
        }
        // if we are laying out views in scrap, this may return null which means there is
        // no more items to layout.
        result.mFinished = true;
        return;
    }
    RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) view.getLayoutParams();
    // 添加 子view 到 RecyclerView 上
    if (layoutState.mScrapList == null) {
        if (mShouldReverseLayout == (layoutState.mLayoutDirection
                == LayoutState.LAYOUT_START)) {
            addView(view);
        } else {
            addView(view, 0);
        }
    } else {
        if (mShouldReverseLayout == (layoutState.mLayoutDirection
                == LayoutState.LAYOUT_START)) {
            addDisappearingView(view);
        } else {
            addDisappearingView(view, 0);
        }
    }
    // 执行 子view 的 measure 流程
    measureChildWithMargins(view, 0, 0);
    result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
    int left, top, right, bottom;
    if (mOrientation == VERTICAL) {
        if (isLayoutRTL()) {
            right = getWidth() - getPaddingRight();
            left = right - mOrientationHelper.getDecoratedMeasurementInOther(view);
        } else {
            left = getPaddingLeft();
            right = left + mOrientationHelper.getDecoratedMeasurementInOther(view);
        }
        if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
            bottom = layoutState.mOffset;
            top = layoutState.mOffset - result.mConsumed;
        } else {
            top = layoutState.mOffset;
            bottom = layoutState.mOffset + result.mConsumed;
        }
    } else {
        top = getPaddingTop();
        bottom = top + mOrientationHelper.getDecoratedMeasurementInOther(view);

        if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
            right = layoutState.mOffset;
            left = layoutState.mOffset - result.mConsumed;
        } else {
            left = layoutState.mOffset;
            right = layoutState.mOffset + result.mConsumed;
        }
    }
    // We calculate everything with View's bounding box (which includes decor and margins)
    // To calculate correct layout position, we subtract margins.
    layoutDecoratedWithMargins(view, left, top, right, bottom);
    if (DEBUG) {
        Log.d(TAG, "laid out child at position " + getPosition(view) + ", with l:"
                + (left + params.leftMargin) + ", t:" + (top + params.topMargin) + ", r:"
                + (right - params.rightMargin) + ", b:" + (bottom - params.bottomMargin));
    }
    // Consume the available space if the view is not removed OR changed
    if (params.isItemRemoved() || params.isItemChanged()) {
        result.mIgnoreConsumed = true;
    }
    result.mFocusable = view.hasFocusable();
}
```
`fill` 所做的是通过循环获取各个子view，然后把它们add到RecyclerView上，直到屏幕空间不够或子view不够。具体做法是，先通过 Recycler.getViewForPosition()
获取子view，接着调用 addView() 来把子view添加到RecyclerView 上，然后在执行子view 的 measure 流程。

`getViewForPosition` 怎么执行的：
```java
View getViewForPosition(int position, boolean dryRun) {
    return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
}

ViewHolder tryGetViewHolderForPositionByDeadline(int position,
        boolean dryRun, long deadlineNs) {
    if (position < 0 || position >= mState.getItemCount()) {
        throw new IndexOutOfBoundsException();
    }
    boolean fromScrapOrHiddenOrCache = false;
    ViewHolder holder = null;
    // 0) If there is a changed scrap, try to find from there
    // 如果是预布局，则从 mChangedScrap 中获取
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }
    // 1) Find by position from scrap/hidden list/cache
    // 根据 position 依次从 mAttachedScrap、mHiddenViews、mCachedViews 中获取
    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
        if (holder != null) {
            if (!validateViewHolderForOffsetPosition(holder)) {
                // recycle holder (and unscrap if relevant) since it can't be used
                if (!dryRun) {
                    // we would like to recycle this but need to make sure it is not used by
                    // animation logic etc.
                    holder.addFlags(ViewHolder.FLAG_INVALID);
                    if (holder.isScrap()) {
                        removeDetachedView(holder.itemView, false);
                        holder.unScrap();
                    } else if (holder.wasReturnedFromScrap()) {
                        holder.clearReturnedFromScrapFlag();
                    }
                    recycleViewHolderInternal(holder);
                }
                holder = null;
            } else {
                fromScrapOrHiddenOrCache = true;
            }
        }
    }
    if (holder == null) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        if (offsetPosition < 0 || offsetPosition >= mAdapter.getItemCount()) {
            throw new IndexOutOfBoundsException();
        }

        final int type = mAdapter.getItemViewType(offsetPosition);
        // 2) Find from scrap/cache via stable ids, if exists
        // 如果上一步没获取到，则根据id依次从 mAttachedScrap、mCachedViews 中获取
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                    type, dryRun);
            if (holder != null) {
                // update position
                holder.mPosition = offsetPosition;
                fromScrapOrHiddenOrCache = true;
            }
        }
        if (holder == null && mViewCacheExtension != null) {
            // We are NOT sending the offsetPosition because LayoutManager does not
            // know it.
            // 如果上一步没获取到，则从 mViewCacheExtension 中获取
            final View view = mViewCacheExtension
                    .getViewForPositionAndType(this, position, type);
            if (view != null) {
                holder = getChildViewHolder(view);
                if (holder == null) {
                    throw new IllegalArgumentException("getViewForPositionAndType returned"
                            + " a view which does not have a ViewHolder"
                            + exceptionLabel());
                } else if (holder.shouldIgnore()) {
                    throw new IllegalArgumentException("getViewForPositionAndType returned"
                            + " a view that is ignored. You must call stopIgnoring before"
                            + " returning this view." + exceptionLabel());
                }
            }
        }
        if (holder == null) { // fallback to pool
        // 如果上一步没获取到，则从 RecycledViewPool 中获取
            holder = getRecycledViewPool().getRecycledView(type);
            if (holder != null) {
                holder.resetInternal();
                if (FORCE_INVALIDATE_DISPLAY_LIST) {
                    invalidateDisplayListInt(holder);
                }
            }
        }
        if (holder == null) {
            long start = getNanoTime();
            if (deadlineNs != FOREVER_NS
                    && !mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
                // abort - we have a deadline we can't meet
                return null;
            }
            // 如果还没获取到，就通过 adapter 创建一个新的
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
            if (ALLOW_THREAD_GAP_WORK) {
                // only bother finding nested RV if prefetching
                RecyclerView innerView = findNestedRecyclerView(holder.itemView);
                if (innerView != null) {
                    holder.mNestedRecyclerView = new WeakReference<>(innerView);
                }
            }

            long end = getNanoTime();
            mRecyclerPool.factorInCreateTime(type, end - start);
            if (DEBUG) {
                Log.d(TAG, "tryGetViewHolderForPositionByDeadline created new ViewHolder");
            }
        }
    }

    // This is very ugly but the only place we can grab this information
    // before the View is rebound and returned to the LayoutManager for post layout ops.
    // We don't need this in pre-layout since the VH is not updated by the LM.
    if (fromScrapOrHiddenOrCache && !mState.isPreLayout() && holder
            .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST)) {
        holder.setFlags(0, ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
        if (mState.mRunSimpleAnimations) {
            int changeFlags = ItemAnimator
                    .buildAdapterChangeFlagsForAnimations(holder);
            changeFlags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
            final ItemHolderInfo info = mItemAnimator.recordPreLayoutInformation(mState,
                    holder, changeFlags, holder.getUnmodifiedPayloads());
            recordAnimationInfoIfBouncedHiddenView(holder, info);
        }
    }

    boolean bound = false;
    if (mState.isPreLayout() && holder.isBound()) {
        // do not update unless we absolutely have to.
        // 如果是预布局且holder已绑定，则设置 position 信息即可
        holder.mPreLayoutPosition = position;
    } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
        if (DEBUG && holder.isRemoved()) {
            throw new IllegalStateException();
        }
        // 否则调用 adapter 的 bindViewHolder() 来绑定数据
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
    }

    final ViewGroup.LayoutParams lp = holder.itemView.getLayoutParams();
    final LayoutParams rvLayoutParams;
    // 为 子view 设置 LP，把将 VH 绑定到 LP 上
    if (lp == null) {
        rvLayoutParams = (LayoutParams) generateDefaultLayoutParams();
        holder.itemView.setLayoutParams(rvLayoutParams);
    } else if (!checkLayoutParams(lp)) {
        rvLayoutParams = (LayoutParams) generateLayoutParams(lp);
        holder.itemView.setLayoutParams(rvLayoutParams);
    } else {
        rvLayoutParams = (LayoutParams) lp;
    }
    rvLayoutParams.mViewHolder = holder;
    rvLayoutParams.mPendingInvalidate = fromScrapOrHiddenOrCache && bound;
    return holder;
}
```
getViewForPosition 是按照一个固定的顺序来获取子view的，即：`mChangedScrap` -> `mAttachedScrap` -> `mCachedViews` -> `mViewCacheExtension` -> `RecycledViewPool`，
如果没有可用的缓存，则通过 adapter 加载一个新的view。

获取到子view之后，如果当前处于预布局阶段，则只设置子view 的position 信息；否则就通过adapter绑定子view到数据。然后就调用addView把子view添加到RecyclerView上。
```java
public void addView(View child, int index) {
    addViewInt(child, index, false);
}

private void addViewInt(View child, int index, boolean disappearing) {
    final ViewHolder holder = getChildViewHolderInt(child);
    if (disappearing || holder.isRemoved()) {
        // these views will be hidden at the end of the layout pass.
        mRecyclerView.mViewInfoStore.addToDisappearedInLayout(holder);
    } else {
        // This may look like unnecessary but may happen if layout manager supports
        // predictive layouts and adapter removed then re-added the same item.
        // In this case, added version will be visible in the post layout (because add is
        // deferred) but RV will still bind it to the same View.
        // So if a View re-appears in post layout pass, remove it from disappearing list.
        mRecyclerView.mViewInfoStore.removeFromDisappearedInLayout(holder);
    }
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    if (holder.wasReturnedFromScrap() || holder.isScrap()) {
        if (holder.isScrap()) {
            holder.unScrap();
        } else {
            holder.clearReturnedFromScrapFlag();
        }
        mChildHelper.attachViewToParent(child, index, child.getLayoutParams(), false);
        if (DISPATCH_TEMP_DETACH) {
            ViewCompat.dispatchFinishTemporaryDetach(child);
        }
    } else if (child.getParent() == mRecyclerView) { // it was not a scrap but a valid child
        // ensure in correct position
        int currentIndex = mChildHelper.indexOfChild(child);
        if (index == -1) {
            index = mChildHelper.getChildCount();
        }
        if (currentIndex == -1) {
            throw new IllegalStateException();
        }
        if (currentIndex != index) {
            mRecyclerView.mLayout.moveView(currentIndex, index);
        }
    } else {
        mChildHelper.addView(child, index, false);
        lp.mInsetsDirty = true;
        if (mSmoothScroller != null && mSmoothScroller.isRunning()) {
            mSmoothScroller.onChildAttachedToWindow(child);
        }
    }
    if (lp.mPendingInvalidate) {
        holder.itemView.invalidate();
        lp.mPendingInvalidate = false;
    }
}
```




## LayoutManager
LayoutManager 负责测量和定位 RecyclerView 中的 item view，以及确定何时回收用户不再可见的 item view的策略。

如果 LayoutManager 指定了默认构造函数或带有签名 (Context, AttributeSet, int, int) 的构造函数，RecyclerView 将在 inflate 时实例化并设置 LayoutManager。
然后可以从 getProperties(Context, AttributeSet, int, int) 获得最常用的属性。 如果 LayoutManager 指定了两个构造函数，则非默认构造函数将优先。



## Adapter
提供从数据集到显示在 RecyclerView 中的 View 的绑定

### 关键方法
* `onCreateViewHolder()`：当 RecyclerView 需要给定类型的新 RecyclerView.ViewHolder 来表示 item 时调用。
* `createViewHolder()`：调用 onCreateViewHolder() 创建一个新的 ViewHolder 并初始化一些私有字段以供 RecyclerView 使用。在 Recycler.tryGetViewHolderForPositionByDeadline() 中被调用。
* `onBindViewHolder()`：被 RecyclerView 调用，显示指定 position 的数据。 此方法应更新 ViewHolder.itemView 的内容以反映给定 position 的 item。
* `bindViewHolder()`：此方法在内部调用 onBindViewHolder() 以使用给定 position 的 item 更新 ViewHolder 内容，并设置一些供 RecyclerView 使用的私有字段。在 Recycler.tryBindViewHolderByDeadline() 中被调用。



## ViewHolder
ViewHolder 描述 item view 和关于它在 RecyclerView 中位置的元数据。

### 重要方法
* addFlags()
* isBound()
* isInvalid()
* isRecyclable()
* isRemoved()
* isScrap()
* isUpdated()
* getLayoutPosition()：根据最新的布局传递返回 ViewHolder 的 position
* getBindingAdapterPosition()：返回此 ViewHolder 表示的 item 相对于绑定它的 Adapter 的 Adapter position。
* getAbsoluteAdapterPosition()：

相关常量：
* FLAG_BOUND：ViewHolder 已经绑定到一个 position；mPosition、mItemId 和 mItemViewType 均有效。
* FLAG_UPDATE：ViewHolder 的 view 反映的数据是陈旧的，需要由 adapter 重新绑定。 mPosition 和 mItemId 是一致的。
* FLAG_INVALID：ViewHolder 的数据无效。 mPosition 和 mItemId 隐含的标识不可信，可能不再匹配 item view 类型。 这个 ViewHolder 必须完全重新绑定到不同的数据。
* FLAG_REMOVED：ViewHolder 指向代表先前从数据集中删除的item的数据。 它的view可能仍用于诸如传出动画之类的事情。
* FLAG_NOT_RECYCLABLE：ViewHolder 不应该被回收。 这个标志是通过 `setIsRecyclable()` 设置的，目的是在动画期间保持view。





## Recycler
Recycler 管理 scrapped 或 detached 的 item view 以供重用。

“scrapped” view 是仍附加到其父 RecyclerView 但已标记为 removal 或 reuse 的 view。

RecyclerView.LayoutManager 对 Recycler 的典型使用是为 adapter 的数据集获取表示给定 position 或 item ID 处数据的 view。 如果要重用的 view 被认为是“dirty”，adapter 将被要求重新绑定它。 如果不是，view 可以被 LayoutManager 快速重用，无需进一步的工作。 没有请求布局的干净 view 可以由 LayoutManager 重新定位而无需重新测量。

### 关键方法
* `tryBindViewHolderByDeadline()`：尝试绑定 view，并说明相关的时间信息。 如果deadlineNs != FOREVER_NS，该方法可能绑定失败，返回false。
* `tryGetViewHolderForPositionByDeadline()`：尝试从 Recycler scrap、cache、RecycledViewPool 或直接创建它来获取给定 position 的 ViewHolder。
* `bindViewToPosition()`
* `getViewForPosition()`：获取为给定 position 初始化的 view。 LayoutManager 实现应该使用此方法来获取 view 以表示来自 Adapter 的数据。如果一个 view 可用于正确的 view 类型，则 Recycler 可以重用共享池中的 scrap or detached view。 如果 adapter 未指示给定 position 的数据已更改，则 Recycler 将尝试交回先前为该数据初始化的 scrap view 而不重新绑定。在 LayoutState.next() 中被调用。



## State
包含有关当前 RecyclerView 状态的有用信息，例如目标滚动 position 或 view 焦点。 State 对象还可以保存任意数据，由资源 ID 标识。

很多时候，RecyclerView 组件需要在彼此之间传递信息。 为了在组件之间提供定义良好的数据总线，RecyclerView 将相同的 State 对象传递给组件回调，这些组件可以使用它来交换数据。

如果实现自定义组件，则可以使用 State 的 put/get/remove 方法在组件之间传递数据，而无需管理它们的生命周期。







