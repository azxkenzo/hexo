---
title: Android - View绘制流程
date: 2023-03-01 16:50:31
tags: Android
---

Activity 与 Window 和 DecorView 等之间等关系：
* `Activity`：每个 Activity 都持有一个 Window 对象
* `Window`：顶层 Window 外观和行为策略的抽象基类。 此类的实例应该用作添加到 window manager 的顶层view。 它提供标准的 UI 策略，例如背景、标题区域、默认键处理等。
* `PhoneWindow`：Window 类的指定实现。PhoneWindow 持有 DecorView 对象。
* `DecorView`：继承自 FrameLayout，是 Window 的 顶层View。DecorView 初始化时会根据主题、配置等的不同来选择子布局
* `WindowManager`：用来与 window manager 对话的接口。每个 window manager 实例都绑定到一个 Display。存在两种类型的 WindowManager：一个是系统级别的 WindowManager；另一个是与 Window 实例相关联的 WindowManager
* `ViewRootImpl`：View层次结构的顶部，实现 view 和 WindowManager 之间所需的协议。 这大部分是 WindowManagerGlobal 的内部实现细节。



从 `AppCompatActivity.setContentView()` 开始，看看里面都做了些什么。主要是调用 AppCompatDelegateImpl.setContentView()。
`AppCompatDelegateImpl.setContentView()` 做了这几件事：
1. 调用 `ensureSubDecor()` 以确保 SubDecor 已创建。在此过程中会调用 `Window.getDecorView()` 来确保 Window 的 DecorView 已创建。然后把 SubDecor 添加到 DecorView 中
2. 从 SubDecor 获取 contentParent，清空 contentParent 的子View，然后在 contentParent 下填充指定的view。



`ActivityThread` 管理应用程序进程中主线程的执行，根据 activity manager 的请求调度和执行 activity、广播和其他操作。看下 `ActivityThread.handleResumeActivity()` 中主要做了什么：
1. 获取 Window 的 DecorView，并将其设置为 INVISIBLE。接着把 DecorView 保存到 Activity 中。然后调用 `WindowManager.addView()` 把 DecorView 添加到 WindowManager 中。
2. 在调起了输入法的情况下，调用 `WindowManager.updateViewLayout()` 来更新 DecorView 的布局
3. 最后调用 `Activity.makeVisible()` 来确保 DecorView 被添加到 WindowManager 中，并将 DecorView 设置为 VISIBLE

`WindowManager` 的具体实现是 `WindowManagerImpl`。WindowManagerImpl 把 `addView()` 委托给 `WindowManagerGlobal` 去执行。WindowManagerGlobal 是一个单例对象。看下 `WindowManagerGlobal.addView()` 里面是怎么做的：
1. 创建 ViewRootImpl 实例
2. 把 DecorView 保存到一个 ArrayList 中；把刚创建 ViewRootImpl 保存到一个 ArrayList 中
3. 调用 `ViewRootImpl.setView()` 把 DecorView 保存到 ViewRootImpl 中

看下 `ViewRootImpl.setView()` 做了什么：
1. 保存 DecorView
2. 调用 `requestLayout()`，在添加到 window manager 之前安排第一个 layout，以确保在从系统接收任何其他事件之前进行重新布局。

```
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}

void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }
        performTraversals();
        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```
`ViewRootImpl.requestLayout()` 最终会调用到 `ViewRootImpl.performTraversals()`，`performTraversals()` 主要做了三件事：
1. 调用 `performMeasure()`，执行 DecorView 的 Measure 流程
2. 调用 `performLayout()`，执行 DecorView 的 Layout 流程
3. 调用 `performDraw()`，执行 DecorView 的 Draw 流程

下面看看这三步分别是怎么执行的：

### DecorView 的 Measure 流程
ViewRootImpl.`performTraversals()` 中，
1. 定义期望的 Window 的宽、高变量 desiredWindowWidth、desiredWindowHeight
2. 根据规则计算出 desiredWindowWidth、desiredWindowHeight 的值
3. 调用 `measureHierarchy()`，使用 desiredWindowWidth、desiredWindowHeight 计算 DecorView 的想要的大小

在 `measureHierarchy()` 中，
1. 调用 `getRootMeasureSpec()`， 计算出 Window 中 root view 的 measure spec。这一步会将 Dialog 和 其他情况分开处理。看下 getRootMeasureSpec是怎么做的：
    ```
    private static int getRootMeasureSpec(int windowSize, int measurement, int privateFlags) {
        int measureSpec;
        final int rootDimension = (privateFlags & PRIVATE_FLAG_LAYOUT_SIZE_EXTENDED_BY_CUTOUT) != 0
                ? MATCH_PARENT : measurement;
        switch (rootDimension) {
            case ViewGroup.LayoutParams.MATCH_PARENT:
                // Window can't resize. Force root view to be windowSize.
                measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
                break;
            case ViewGroup.LayoutParams.WRAP_CONTENT:
                // Window can resize. Set max size for root view.
                measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
                break;
            default:
                // Window wants to be an exact size. Force root view to be that size.
                measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
                break;
        }
        return measureSpec;
    }
    ```
2. 调用 `performMeasure()`，使用刚计算好的宽高限定计算 DecorView 的宽高

`performMeasure()` 会调用 DecorView 的 `measure()`，看下 `measure()` 里面是怎么做的：
1. 判断是否是强制布局。当 mPrivateFlags 中包含 PFLAG_FORCE_LAYOUT 标志时则是强制布局。如调用 View.requestLayout() 时会在 mPrivateFlags 中加入此标记
2. 判断是否需要布局
3. 如果是强制布局或需要布局，就调用 `onMeasure()` 来测量子View 或直接使用缓存中保存的值
4. 检查是否调用 `setMeasuredDimension()` 来保存测量好的值，若没有则抛出异常
5. 在 mPrivateFlags 中添加 PFLAG_LAYOUT_REQUIRED 标志，表示可以进行测量

看下 DecorView.`onMeasure()` 做了什么：
1. 如果宽模式为 AT_MOST，就对 widthMeasureSpec 做一下调整
2. 如果高模式为 AT_MOST，就对 heightMeasureSpec 做一下调整
3. 调用父类即 FrameLayout 的 `onMeasure()` 对 子View 进行测量

看下 FrameLayout 的 `onMeasure()` 做了什么：
1. 遍历子View，调用 `measureChildWithMargins()` 测量子View的大小。然后根据子view测量完的大小和子view的margin值来确定自身的maxWidth和maxHeight。
此时如果 FrameLayout 的宽高模式中只要有一个不是 `EXACTLY` 的话，就把布局参数的宽或高为 `MATCH_PARENT` 的 子view 保存到 `mMatchParentChildren` 中
2. 使用自身的padding更新maxWidth和maxHeight；使用最小宽高更新 maxWidth 和 maxHeight；使用Foreground的最小宽高更新maxWidth和maxHeight
3. 调用 `setMeasuredDimension()` 保存使用 `resolveSizeAndState()` 计算出来的宽高
4. 遍历 `mMatchParentChildren` 中保存的子view，对其进行重新测量

`measureChildWithMargins()`是怎么做的：
```
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
调用 `getChildMeasureSpec()` 计算出 子View 的 MeasureSpec，然后调用 `measure()` 来测量子view。`getChildMeasureSpec()` 是如何计算子view的 MeasureSpec 的：
```
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    int size = Math.max(0, specSize - padding);
    int resultSize = 0;
    int resultMode = 0;
    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let them have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```
子view 的 mode 和 size 由 父view 的 mode 和 子view LayoutParams 的 width/height 共同决定：
* 子view 尺寸为 `具体数值` 时，mode 均为 MeasureSpec.EXACTLY，size 为 子view LayoutParams中设定的值
* 子view 尺寸为 `MATCH_PARENT` 时，mode 取 父view mode，size 为 父view size 减去相应 padding
* 子view 尺寸为 `WRAP_CONTENT` 时，在父view mode为EXACTLY/AT_MOST时取AT_MOST，父view mode为UNSPECIFIED时取UNSPECIFIED；size 为 父view size 减去相应 padding


在测量完所有子View并计算出最大宽高后，就会调用 `resolveSizeAndState()` 计算出自身的大小，看下是怎么做的：
```
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    final int specMode = MeasureSpec.getMode(measureSpec);
    final int specSize = MeasureSpec.getSize(measureSpec);
    final int result;
    switch (specMode) {
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
                result = specSize | MEASURED_STATE_TOO_SMALL;
            } else {
                result = size;
            }
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            result = size;
    }
    return result | (childMeasuredState & MEASURED_STATE_MASK);
}
```


### DecorView 的 Layout 流程
在 Measure 流程执行完后，会调用 `performLayout()` 开始 layout 流程。`performLayout()` 里面做了什么：
1. 调用 DecorView 的 `layout(0, 0, measuredWidth, measuredHeight)` 对 DecorView 进行布局
2. 如果在 DecorView 布局过程中有收到布局请求，则在DecorView 布局执行完后，筛选出有效的布局请求并执行，然后再对DecorView进行重新测量和布局

调用 DecorView 的 `layout()` 对其进行布局，看下里面是怎么做的：
1. 调用 `setFrame()` 保存 view 的大小和位置
2. 如果大小改变或需要布局，就调用 `onLayout()` 进行布局

`setFrame()` 做了什么：
1. 判断 view 的位置是否改变
2. 若 view 的位置改变则再判断 view 的大小是否改变，并调用 `invalidate()` 使旧位置无效，然后保存新的位置。
3. 若 view 的大小是否改变，就调用 `sizeChange()` 进行通知。`sizeChange()` 中会调用 `onSizeChanged()` 回调

DecorView 在确定好自身位置之后，就会调用 `onLayout()` 来确定子view的位置。FrameLayout 中 `onLayout()` 把工作委托给了 `layoutChildren()`，看下里面做了什么：
1. 计算 DecorView 的剩余空间范围
2. 遍历子view，根据 子view 的 layout_gravity 属性、布局方向 使用 父View的剩余空间范围、子view的测量宽度和子view的左右margin 计算 子view的左坐标；
   根据 子view 的 layout_gravity 属性 使用 父View的剩余空间范围、子view的测量高度和子view的上下margin 计算 子view的上坐标
3. 调用 子view 的 `layout()`，传入刚才计算好的子view的位置


### DecorView 的 Draw 流程
在 Layout 流程执行完后，会调用 `performDraw()` 开始 Draw 流程。而 `performDraw()` 主要是通过调用 `draw(fullRedrawNeeded, forceDraw)` 来处理绘制的，看下 `draw()` 里面做了什么：
1. 判断是否需要完全重绘，若需要则将重绘区域设置为整个屏幕
2. 如果重绘区域不为空 或 动画执行中，则判断是否启用HW，若启用则调用 `mAttachInfo.mThreadedRenderer.draw()`；否则调用 `drawSoftware()`

`drawSoftware()` 从 mSurface 处获取 canvas，然后调用 decorView 的 `draw()`。decorView`draw()` 调用了 `View.draw()` 看下里面做了什么：
绘制遍历执行几个绘制步骤，必须以适当的顺序执行：
1. 绘制 background，调用 `drawBackground()`
2. 如有必要，保存 canvas 的 layers 以备 fading
3. 绘制 view 的内容，调用 `onDraw()`
4. 绘制 children，调用 `dispatchDraw()`
5. 如有必要，绘制 fading 边缘并恢复 layers
6. 绘制 decorations（例如 scrollbars），调用 `onDrawForeground()`
7. 如有必要，绘制默认焦点高亮，调用 `drawDefaultFocusHighlight()`

DecorView 的`dispatchDraw()`会调用 `ViewGroup.dispatchDraw()`，看下里面做了什么：
1. 使用 padding 裁剪出canvas正确的绘制区域
2. 获取绘制顺序列表，然后遍历子view，调用 `drawChild()` 绘制子view。`drawChild()` 直接调用子view的 `draw(canvas, this, drawingTime)`

view的 `draw(canvas, this, drawingTime)` 做了什么：
1. 判断是否使用缓存进行绘制，若是则直接绘制缓存；若不是则调用 `dispatchDraw() / draw()`