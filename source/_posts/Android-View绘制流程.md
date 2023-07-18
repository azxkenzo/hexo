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



## AppCompatActivity

### setContentView()
```java
public void setContentView(View view) {
    initViewTreeOwners();
    getDelegate().setContentView(view);
}
```
调用 `AppCompatDelegateImpl`.`setContentView()`。

### AppCompatDelegateImpl
```java
public void setContentView(View v) {
    ensureSubDecor();
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    contentParent.addView(v);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}
```
1. 调用 `ensureSubDecor()` 确保 SubDecor 已创建。在此过程中会调用 `Window.getDecorView()` 来确保 Window 的 DecorView 已创建, 然后把 SubDecor 添加到 DecorView 中
2. 从 SubDecor 获取 contentParent，清空 contentParent 的子View，然后把指定的 view 添加到 contentParent 中。

```java
// class PhoneWindow
public final @NonNull View getDecorView() {
    if (mDecor == null || mForceDecorInstall) {
        installDecor();
    }
    return mDecor;
}
```



## ActivityThread

`ActivityThread` 管理应用程序进程中主线程的执行，根据 activity manager 的请求调度和执行 activity、广播和其他操作。

### main()
程序的入口点

```java
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

    // Install selective syscall interception
    AndroidOs.install();

    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);

    Environment.initForCurrentUser();

    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);

    // Call per-process mainline module initialization.
    initializeMainlineModules();

    Process.setArgV0("<pre-initialized>");

    Looper.prepareMainLooper();

    // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
    // It will be in the format "seq=114"
    long startSeq = 0;
    if (args != null) {
        for (int i = args.length - 1; i >= 0; --i) {
            if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                startSeq = Long.parseLong(
                        args[i].substring(PROC_START_SEQ_IDENT.length()));
            }
        }
    }
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
1. `Looper`.`prepareMainLooper()`
2. thread.`attach()`
3. sMainThreadHandler = thread.getHandler()
4. Looper.`loop()`

### handleResumeActivity()

```java
public void handleResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
        boolean isForward, String reason) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    // TODO Push resumeArgs into the activity for consideration
    // skip below steps for double-resume and r.mFinish = true case.
    if (!performResumeActivity(r, finalStateRequest, reason)) {
        return;
    }
    if (mActivitiesToBeDestroyed.containsKey(r.token)) {
        // Although the activity is resumed, it is going to be destroyed. So the following
        // UI operations are unnecessary and also prevents exception because its token may
        // be gone that window manager cannot recognize it. All necessary cleanup actions
        // performed below will be done while handling destruction.
        return;
    }

    final Activity a = r.activity;

    if (localLOGV) {
        Slog.v(TAG, "Resume " + r + " started activity: " + a.mStartedActivity
                + ", hideForNow: " + r.hideForNow + ", finished: " + a.mFinished);
    }

    final int forwardBit = isForward
            ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

    // If the window hasn't yet been added to the window manager,
    // and this guy didn't finish itself or start another activity,
    // then go ahead and add the window.
    boolean willBeVisible = !a.mStartedActivity;
    if (!willBeVisible) {
        willBeVisible = ActivityClient.getInstance().willActivityBeVisible(
                a.getActivityToken());
    }
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
            // Normally the ViewRoot sets up callbacks with the Activity
            // in addView->ViewRootImpl#setView. If we are instead reusing
            // the decor view we have to notify the view root that the
            // callbacks may have changed.
            ViewRootImpl impl = decor.getViewRootImpl();
            if (impl != null) {
                impl.notifyChildRebuilt();
            }
        }
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
            } else {
                // The activity will get a callback for this {@link LayoutParams} change
                // earlier. However, at that time the decor will not be set (this is set
                // in this method), so no action will be taken. This call ensures the
                // callback occurs with the decor set.
                a.onWindowAttributesChanged(l);
            }
        }

        // If the window has already been added, but during resume
        // we started another activity, then don't yet make the
        // window visible.
    } else if (!willBeVisible) {
        if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
        r.hideForNow = true;
    }

    // Get rid of anything left hanging around.
    cleanUpPendingRemoveWindows(r, false /* force */);

    // The window is now visible if it has been added, we are not
    // simply finishing, and we are not starting another activity.
    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
        if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
        ViewRootImpl impl = r.window.getDecorView().getViewRootImpl();
        WindowManager.LayoutParams l = impl != null
                ? impl.mWindowAttributes : r.window.getAttributes();
        if ((l.softInputMode
                & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                != forwardBit) {
            l.softInputMode = (l.softInputMode
                    & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                    | forwardBit;
            if (r.activity.mVisibleFromClient) {
                ViewManager wm = a.getWindowManager();
                View decor = r.window.getDecorView();
                wm.updateViewLayout(decor, l);
            }
        }

        r.activity.mVisibleFromServer = true;
        mNumVisibleActivities++;
        if (r.activity.mVisibleFromClient) {
            r.activity.makeVisible();
        }
    }

    r.nextIdle = mNewActivities;
    mNewActivities = r;
    if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
    Looper.myQueue().addIdleHandler(new Idler());
}
```
1. `performResumeActivity()`，进而调用 `Activity`.`performResume()`
2. `Activity`.`getWindow()`，Activity 的 Window 在其 `attach()` 方法中初始化，是一个 `PhoneWindow` 对象。
3. decor.`setVisibility(View.INVISIBLE)`； a.mDecor = decor
4. a.getWindowManager()； wm.`addView(decor, l)`。  Activity 的 WindowManager 从 Window.getWindowManager() 获取，Window 的 WindowManager 从 Context.getSystemService(Context.WINDOW_SERVICE) (实例是 `WindowManagerImpl`) 的 `createLocalWindowManager()` 获取
5. 在调起了输入法的情况下，调用 `WindowManager.updateViewLayout(decor, l)` 来更新 DecorView 的布局
6. 调用 `Activity.makeVisible()` 来确保 DecorView 被添加到 WindowManager 中，并将 DecorView 设置为 VISIBLE

## WindowManager

`WindowManager` 的实现类是 `WindowManagerImpl`。

### addView()

`WindowManagerImpl` 把 `addView()` 委托给 `WindowManagerGlobal` 去执行。
```java
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyTokens(params);
    mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
            mContext.getUserId());
}
```

## WindowManagerGlobal
`WindowManagerGlobal` 是一个单例对象。

### addView()
```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow, int userId) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (display == null) {
        throw new IllegalArgumentException("display must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        // If there's no parent, then hardware acceleration for this view is
        // set from the application's hardware acceleration setting.
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }

    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        // Start watching for system property changes.
        if (mSystemPropertyUpdater == null) {
            mSystemPropertyUpdater = new Runnable() {
                @Override public void run() {
                    synchronized (mLock) {
                        for (int i = mRoots.size() - 1; i >= 0; --i) {
                            mRoots.get(i).loadSystemProperties();
                        }
                    }
                }
            };
            SystemProperties.addChangeCallback(mSystemPropertyUpdater);
        }

        int index = findViewLocked(view, false);
        if (index >= 0) {
            if (mDyingViews.contains(view)) {
                // Don't wait for MSG_DIE to make it's way through root's queue.
                mRoots.get(index).doDie();
            } else {
                throw new IllegalStateException("View " + view
                        + " has already been added to the window manager.");
            }
            // The previous removeView() had not completed executing. Now it has.
        }

        // If this is a panel window, then find the window it is being
        // attached to for future reference.
        if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
            final int count = mViews.size();
            for (int i = 0; i < count; i++) {
                if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                    panelParentView = mViews.get(i);
                }
            }
        }

        IWindowSession windowlessSession = null;
        // If there is a parent set, but we can't find it, it may be coming
        // from a SurfaceControlViewHost hierarchy.
        if (wparams.token != null && panelParentView == null) {
            for (int i = 0; i < mWindowlessRoots.size(); i++) {
                ViewRootImpl maybeParent = mWindowlessRoots.get(i);
                if (maybeParent.getWindowToken() == wparams.token) {
                    windowlessSession = maybeParent.getWindowSession();
                    break;
                }
            }
        }

        if (windowlessSession == null) {
            root = new ViewRootImpl(view.getContext(), display);
        } else {
            root = new ViewRootImpl(view.getContext(), display,
                    windowlessSession);
        }

        view.setLayoutParams(wparams);

        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView, userId);
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
1. 创建 ViewRootImpl
2. view.setLayoutParams(wparams), DecorView 设置 WindowManager.LayoutParams
3. DecorView 添加到 mViews 中
4. ViewRootImpl 添加到 mRoots 中
5. ViewRootImpl.`setView(view, wparams, panelParentView, userId)`


## ViewRootImpl

### setView()

1. 保存 DecorView: mView = view
2. mAttachInfo.mRootView = view
3. 调用 `requestLayout()`，在添加到 window manager 之前安排第一个 layout，以确保在从系统接收任何其他事件之前进行重新布局。
4. view.`assignParent(this)`, 设置 DecorView 的 mParent 为 ViewRootImpl


### requestLayout()
```java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

```java
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
```
1. 向 Looper 的 Queue 中提交 SyncBarrier 
2. mChoreographer 提交执行 `mTraversalRunnable` 的 Message

#### SyncBarrier
```java
// class MessageQueue
private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```
将新的同步屏障令牌放入队列

```java
// class MessageQueue
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

        ......
    }
}
```
遇到同步屏障后，查找队列中的下一条异步消息(!msg.`isAsynchronous()`)。

#### Choreographer
```java
private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    if (DEBUG_FRAMES) {
        Log.d(TAG, "PostCallback: type=" + callbackType
                + ", action=" + action + ", token=" + token
                + ", delayMillis=" + delayMillis);
    }

    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        if (dueTime <= now) {
            scheduleFrameLocked(now);
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```
提交异步 Message (`setAsynchronous(true)`)


### performTraversals()
`mTraversalRunnable` 会调用 `doTraversal()`，`doTraversal()` 调用 `performTraversals()`

1. 调用 `performMeasure()`，执行 DecorView 的 Measure 流程
2. 调用 `performLayout()`，执行 DecorView 的 Layout 流程
3. 调用 `performDraw()`，执行 DecorView 的 Draw 流程

### DecorView 的 Measure 流程
ViewRootImpl.`performTraversals()` 中，
1. 定义期望的 Window 的宽、高变量 desiredWindowWidth、desiredWindowHeight
2. 根据规则计算出 desiredWindowWidth、desiredWindowHeight 的值
3. 调用 `measureHierarchy()`，使用 desiredWindowWidth、desiredWindowHeight 计算 DecorView 的想要的大小

在 `measureHierarchy()` 中，
1. 调用 `getRootMeasureSpec()`， 计算出 Window 中 root view 的 measure spec。这一步会将 Dialog 和 其他情况分开处理。看下 getRootMeasureSpec是怎么做的：
    ```java
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
```java
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
```java
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
```java
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
在 Layout 流程执行完后，会调用 `performDraw()` 开始 Draw 流程。`performDraw()` 通过调用 `draw(fullRedrawNeeded, forceDraw)` 来处理绘制

```java
private boolean draw(boolean fullRedrawNeeded, boolean forceDraw) {
    Surface surface = mSurface;
    if (!surface.isValid()) {
        return false;
    }

    if (DEBUG_FPS) {
        trackFPS();
    }

    if (!sFirstDrawComplete) {
        synchronized (sFirstDrawHandlers) {
            sFirstDrawComplete = true;
            final int count = sFirstDrawHandlers.size();
            for (int i = 0; i< count; i++) {
                mHandler.post(sFirstDrawHandlers.get(i));
            }
        }
    }

    scrollToRectOrFocus(null, false);

    if (mAttachInfo.mViewScrollChanged) {
        mAttachInfo.mViewScrollChanged = false;
        mAttachInfo.mTreeObserver.dispatchOnScrollChanged();
    }

    boolean animating = mScroller != null && mScroller.computeScrollOffset();
    final int curScrollY;
    if (animating) {
        curScrollY = mScroller.getCurrY();
    } else {
        curScrollY = mScrollY;
    }
    if (mCurScrollY != curScrollY) {
        mCurScrollY = curScrollY;
        fullRedrawNeeded = true;
        if (mView instanceof RootViewSurfaceTaker) {
            ((RootViewSurfaceTaker) mView).onRootViewScrollYChanged(mCurScrollY);
        }
    }

    final float appScale = mAttachInfo.mApplicationScale;
    final boolean scalingRequired = mAttachInfo.mScalingRequired;

    final Rect dirty = mDirty;
    if (mSurfaceHolder != null) {
        // The app owns the surface, we won't draw.
        dirty.setEmpty();
        if (animating && mScroller != null) {
            mScroller.abortAnimation();
        }
        return false;
    }

    if (fullRedrawNeeded) {
        dirty.set(0, 0, (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
    }

    mAttachInfo.mTreeObserver.dispatchOnDraw();

    int xOffset = -mCanvasOffsetX;
    int yOffset = -mCanvasOffsetY + curScrollY;
    final WindowManager.LayoutParams params = mWindowAttributes;
    final Rect surfaceInsets = params != null ? params.surfaceInsets : null;
    if (surfaceInsets != null) {
        xOffset -= surfaceInsets.left;
        yOffset -= surfaceInsets.top;

        // Offset dirty rect for surface insets.
        dirty.offset(surfaceInsets.left, surfaceInsets.top);
    }

    boolean accessibilityFocusDirty = false;
    final Drawable drawable = mAttachInfo.mAccessibilityFocusDrawable;
    if (drawable != null) {
        final Rect bounds = mAttachInfo.mTmpInvalRect;
        final boolean hasFocus = getAccessibilityFocusedRect(bounds);
        if (!hasFocus) {
            bounds.setEmpty();
        }
        if (!bounds.equals(drawable.getBounds())) {
            accessibilityFocusDirty = true;
        }
    }

    mAttachInfo.mDrawingTime =
            mChoreographer.getFrameTimeNanos() / TimeUtils.NANOS_PER_MS;

    boolean useAsyncReport = false;
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (isHardwareEnabled()) {
            // If accessibility focus moved, always invalidate the root.
            boolean invalidateRoot = accessibilityFocusDirty || mInvalidateRootRequested;
            mInvalidateRootRequested = false;

            // Draw with hardware renderer.
            mIsAnimating = false;

            if (mHardwareYOffset != yOffset || mHardwareXOffset != xOffset) {
                mHardwareYOffset = yOffset;
                mHardwareXOffset = xOffset;
                invalidateRoot = true;
            }

            if (invalidateRoot) {
                mAttachInfo.mThreadedRenderer.invalidateRoot();
            }

            dirty.setEmpty();

            // Stage the content drawn size now. It will be transferred to the renderer
            // shortly before the draw commands get send to the renderer.
            final boolean updated = updateContentDrawBounds();

            if (mReportNextDraw) {
                // report next draw overrides setStopped()
                // This value is re-sync'd to the value of mStopped
                // in the handling of mReportNextDraw post-draw.
                mAttachInfo.mThreadedRenderer.setStopped(false);
            }

            if (updated) {
                requestDrawWindow();
            }

            useAsyncReport = true;

            if (forceDraw) {
                mAttachInfo.mThreadedRenderer.forceDrawNextFrame();
            }
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        } else {
            // If we get here with a disabled & requested hardware renderer, something went
            // wrong (an invalidate posted right before we destroyed the hardware surface
            // for instance) so we should just bail out. Locking the surface with software
            // rendering at this point would lock it forever and prevent hardware renderer
            // from doing its job when it comes back.
            // Before we request a new frame we must however attempt to reinitiliaze the
            // hardware renderer if it's in requested state. This would happen after an
            // eglTerminate() for instance.
            if (mAttachInfo.mThreadedRenderer != null &&
                    !mAttachInfo.mThreadedRenderer.isEnabled() &&
                    mAttachInfo.mThreadedRenderer.isRequested() &&
                    mSurface.isValid()) {

                try {
                    mAttachInfo.mThreadedRenderer.initializeIfNeeded(
                            mWidth, mHeight, mAttachInfo, mSurface, surfaceInsets);
                } catch (OutOfResourcesException e) {
                    handleOutOfResourcesException(e);
                    return false;
                }

                mFullRedrawNeeded = true;
                scheduleTraversals();
                return false;
            }

            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                    scalingRequired, dirty, surfaceInsets)) {
                return false;
            }
        }
    }

    if (animating) {
        mFullRedrawNeeded = true;
        scheduleTraversals();
    }
    return useAsyncReport;
}
```
分两种情况进行处理：
* 启用了硬件渲染：mAttachInfo.mThreadedRenderer.draw() -> updateRootDisplayList() -> view.updateDisplayListIfDirty() -> draw(canvas)
* 未启用硬件渲染：drawSoftware() -> mView.draw(canvas)

### View

#### draw(Canvas canvas)

1. 调用 `drawBackground()` 绘制 background
2. 如有必要，保存 canvas 的 layers 以备 fading
3. 调用 `onDraw()` 绘制 view 的内容
4. 调用 `dispatchDraw()` 绘制 children
5. 如有必要，绘制 fading 边缘并恢复 layers
6. 调用 `onDrawForeground()` 绘制 decorations（例如 scrollbars）
7. 如有必要，调用 `drawDefaultFocusHighlight()` 绘制默认焦点高亮

#### dispatchDraw()
```java
    protected void dispatchDraw(Canvas canvas) {
        final int childrenCount = mChildrenCount;
        final View[] children = mChildren;
        int flags = mGroupFlags;

        if ((flags & FLAG_RUN_ANIMATION) != 0 && canAnimate()) {
            for (int i = 0; i < childrenCount; i++) {
                final View child = children[i];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {
                    final LayoutParams params = child.getLayoutParams();
                    attachLayoutAnimationParameters(child, params, i, childrenCount);
                    bindLayoutAnimation(child);
                }
            }

            final LayoutAnimationController controller = mLayoutAnimationController;
            if (controller.willOverlap()) {
                mGroupFlags |= FLAG_OPTIMIZE_INVALIDATE;
            }

            controller.start();

            mGroupFlags &= ~FLAG_RUN_ANIMATION;
            mGroupFlags &= ~FLAG_ANIMATION_DONE;

            if (mAnimationListener != null) {
                mAnimationListener.onAnimationStart(controller.getAnimation());
            }
        }

        int clipSaveCount = 0;
        final boolean clipToPadding = (flags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK;
        if (clipToPadding) {
            clipSaveCount = canvas.save(Canvas.CLIP_SAVE_FLAG);
            canvas.clipRect(mScrollX + mPaddingLeft, mScrollY + mPaddingTop,
                    mScrollX + mRight - mLeft - mPaddingRight,
                    mScrollY + mBottom - mTop - mPaddingBottom);
        }

        // We will draw our child's animation, let's reset the flag
        mPrivateFlags &= ~PFLAG_DRAW_ANIMATION;
        mGroupFlags &= ~FLAG_INVALIDATE_REQUIRED;

        boolean more = false;
        final long drawingTime = getDrawingTime();

        canvas.enableZ();
        final int transientCount = mTransientIndices == null ? 0 : mTransientIndices.size();
        int transientIndex = transientCount != 0 ? 0 : -1;
        // Only use the preordered list if not HW accelerated, since the HW pipeline will do the
        // draw reordering internally
        final ArrayList<View> preorderedList = drawsWithRenderNode(canvas)
                ? null : buildOrderedChildList();
        final boolean customOrder = preorderedList == null
                && isChildrenDrawingOrderEnabled();
        for (int i = 0; i < childrenCount; i++) {
            while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
                final View transientChild = mTransientViews.get(transientIndex);
                if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                        transientChild.getAnimation() != null) {
                    more |= drawChild(canvas, transientChild, drawingTime);
                }
                transientIndex++;
                if (transientIndex >= transientCount) {
                    transientIndex = -1;
                }
            }

            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
            final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        while (transientIndex >= 0) {
            // there may be additional transient views after the normal views
            final View transientChild = mTransientViews.get(transientIndex);
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                    transientChild.getAnimation() != null) {
                more |= drawChild(canvas, transientChild, drawingTime);
            }
            transientIndex++;
            if (transientIndex >= transientCount) {
                break;
            }
        }
        if (preorderedList != null) preorderedList.clear();

        // Draw any disappearing views that have animations
        if (mDisappearingChildren != null) {
            final ArrayList<View> disappearingChildren = mDisappearingChildren;
            final int disappearingCount = disappearingChildren.size() - 1;
            // Go backwards -- we may delete as animations finish
            for (int i = disappearingCount; i >= 0; i--) {
                final View child = disappearingChildren.get(i);
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        canvas.disableZ();

        if (isShowingLayoutBounds()) {
            onDebugDraw(canvas);
        }

        if (clipToPadding) {
            canvas.restoreToCount(clipSaveCount);
        }

        // mGroupFlags might have been updated by drawChild()
        flags = mGroupFlags;

        if ((flags & FLAG_INVALIDATE_REQUIRED) == FLAG_INVALIDATE_REQUIRED) {
            invalidate(true);
        }

        if ((flags & FLAG_ANIMATION_DONE) == 0 && (flags & FLAG_NOTIFY_ANIMATION_LISTENER) == 0 &&
                mLayoutAnimationController.isDone() && !more) {
            // We want to erase the drawing cache and notify the listener after the
            // next frame is drawn because one extra invalidate() is caused by
            // drawChild() after the animation is over
            mGroupFlags |= FLAG_NOTIFY_ANIMATION_LISTENER;
            final Runnable end = new Runnable() {
               @Override
               public void run() {
                   notifyAnimationListener();
               }
            };
            post(end);
        }
    }
```
调用 `drawChild()` 绘制子 view。`drawChild()` 调用子 view 的 `draw(Canvas canvas, ViewGroup parent, long drawingTime)`


#### draw(Canvas canvas, ViewGroup parent, long drawingTime)
```java
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {

    final boolean hardwareAcceleratedCanvas = canvas.isHardwareAccelerated();

    boolean drawingWithRenderNode = drawsWithRenderNode(canvas);

    boolean more = false;
    final boolean childHasIdentityMatrix = hasIdentityMatrix();
    final int parentFlags = parent.mGroupFlags;

    if ((parentFlags & ViewGroup.FLAG_CLEAR_TRANSFORMATION) != 0) {
        parent.getChildTransformation().clear();
        parent.mGroupFlags &= ~ViewGroup.FLAG_CLEAR_TRANSFORMATION;
    }

    Transformation transformToApply = null;
    boolean concatMatrix = false;
    final boolean scalingRequired = mAttachInfo != null && mAttachInfo.mScalingRequired;
    final Animation a = getAnimation();
    if (a != null) {
        more = applyLegacyAnimation(parent, drawingTime, a, scalingRequired);
        concatMatrix = a.willChangeTransformationMatrix();
        if (concatMatrix) {
            mPrivateFlags3 |= PFLAG3_VIEW_IS_ANIMATING_TRANSFORM;
        }
        transformToApply = parent.getChildTransformation();
    } else {
        if ((mPrivateFlags3 & PFLAG3_VIEW_IS_ANIMATING_TRANSFORM) != 0) {
            // No longer animating: clear out old animation matrix
            mRenderNode.setAnimationMatrix(null);
            mPrivateFlags3 &= ~PFLAG3_VIEW_IS_ANIMATING_TRANSFORM;
        }
        if (!drawingWithRenderNode
                && (parentFlags & ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS) != 0) {
            final Transformation t = parent.getChildTransformation();
            final boolean hasTransform = parent.getChildStaticTransformation(this, t);
            if (hasTransform) {
                final int transformType = t.getTransformationType();
                transformToApply = transformType != Transformation.TYPE_IDENTITY ? t : null;
                concatMatrix = (transformType & Transformation.TYPE_MATRIX) != 0;
            }
        }
    }

    concatMatrix |= !childHasIdentityMatrix;

    // Sets the flag as early as possible to allow draw() implementations
    // to call invalidate() successfully when doing animations
    mPrivateFlags |= PFLAG_DRAWN;

    if (!concatMatrix &&
            (parentFlags & (ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS |
                    ViewGroup.FLAG_CLIP_CHILDREN)) == ViewGroup.FLAG_CLIP_CHILDREN &&
            canvas.quickReject(mLeft, mTop, mRight, mBottom) &&
            (mPrivateFlags & PFLAG_DRAW_ANIMATION) == 0) {
        mPrivateFlags2 |= PFLAG2_VIEW_QUICK_REJECTED;
        return more;
    }
    mPrivateFlags2 &= ~PFLAG2_VIEW_QUICK_REJECTED;

    if (hardwareAcceleratedCanvas) {
        // Clear INVALIDATED flag to allow invalidation to occur during rendering, but
        // retain the flag's value temporarily in the mRecreateDisplayList flag
        mRecreateDisplayList = (mPrivateFlags & PFLAG_INVALIDATED) != 0;
        mPrivateFlags &= ~PFLAG_INVALIDATED;
    }

    RenderNode renderNode = null;
    Bitmap cache = null;
    int layerType = getLayerType(); // TODO: signify cache state with just 'cache' local
    if (layerType == LAYER_TYPE_SOFTWARE || !drawingWithRenderNode) {
         if (layerType != LAYER_TYPE_NONE) {
             // If not drawing with RenderNode, treat HW layers as SW
             layerType = LAYER_TYPE_SOFTWARE;
             buildDrawingCache(true);
        }
        cache = getDrawingCache(true);
    }

    if (drawingWithRenderNode) {
        // Delay getting the display list until animation-driven alpha values are
        // set up and possibly passed on to the view
        renderNode = updateDisplayListIfDirty();
        if (!renderNode.hasDisplayList()) {
            // Uncommon, but possible. If a view is removed from the hierarchy during the call
            // to getDisplayList(), the display list will be marked invalid and we should not
            // try to use it again.
            renderNode = null;
            drawingWithRenderNode = false;
        }
    }

    int sx = 0;
    int sy = 0;
    if (!drawingWithRenderNode) {
        computeScroll();
        sx = mScrollX;
        sy = mScrollY;
    }

    final boolean drawingWithDrawingCache = cache != null && !drawingWithRenderNode;
    final boolean offsetForScroll = cache == null && !drawingWithRenderNode;

    int restoreTo = -1;
    if (!drawingWithRenderNode || transformToApply != null) {
        restoreTo = canvas.save();
    }
    if (offsetForScroll) {
        canvas.translate(mLeft - sx, mTop - sy);
    } else {
        if (!drawingWithRenderNode) {
            canvas.translate(mLeft, mTop);
        }
        if (scalingRequired) {
            if (drawingWithRenderNode) {
                // TODO: Might not need this if we put everything inside the DL
                restoreTo = canvas.save();
            }
            // mAttachInfo cannot be null, otherwise scalingRequired == false
            final float scale = 1.0f / mAttachInfo.mApplicationScale;
            canvas.scale(scale, scale);
        }
    }

    float alpha = drawingWithRenderNode ? 1 : (getAlpha() * getTransitionAlpha());
    if (transformToApply != null
            || alpha < 1
            || !hasIdentityMatrix()
            || (mPrivateFlags3 & PFLAG3_VIEW_IS_ANIMATING_ALPHA) != 0) {
        if (transformToApply != null || !childHasIdentityMatrix) {
            int transX = 0;
            int transY = 0;

            if (offsetForScroll) {
                transX = -sx;
                transY = -sy;
            }

            if (transformToApply != null) {
                if (concatMatrix) {
                    if (drawingWithRenderNode) {
                        renderNode.setAnimationMatrix(transformToApply.getMatrix());
                    } else {
                        // Undo the scroll translation, apply the transformation matrix,
                        // then redo the scroll translate to get the correct result.
                        canvas.translate(-transX, -transY);
                        canvas.concat(transformToApply.getMatrix());
                        canvas.translate(transX, transY);
                    }
                    parent.mGroupFlags |= ViewGroup.FLAG_CLEAR_TRANSFORMATION;
                }

                float transformAlpha = transformToApply.getAlpha();
                if (transformAlpha < 1) {
                    alpha *= transformAlpha;
                    parent.mGroupFlags |= ViewGroup.FLAG_CLEAR_TRANSFORMATION;
                }
            }

            if (!childHasIdentityMatrix && !drawingWithRenderNode) {
                canvas.translate(-transX, -transY);
                canvas.concat(getMatrix());
                canvas.translate(transX, transY);
            }
        }

        // Deal with alpha if it is or used to be <1
        if (alpha < 1 || (mPrivateFlags3 & PFLAG3_VIEW_IS_ANIMATING_ALPHA) != 0) {
            if (alpha < 1) {
                mPrivateFlags3 |= PFLAG3_VIEW_IS_ANIMATING_ALPHA;
            } else {
                mPrivateFlags3 &= ~PFLAG3_VIEW_IS_ANIMATING_ALPHA;
            }
            parent.mGroupFlags |= ViewGroup.FLAG_CLEAR_TRANSFORMATION;
            if (!drawingWithDrawingCache) {
                final int multipliedAlpha = (int) (255 * alpha);
                if (!onSetAlpha(multipliedAlpha)) {
                    if (drawingWithRenderNode) {
                        renderNode.setAlpha(alpha * getAlpha() * getTransitionAlpha());
                    } else if (layerType == LAYER_TYPE_NONE) {
                        canvas.saveLayerAlpha(sx, sy, sx + getWidth(), sy + getHeight(),
                                multipliedAlpha);
                    }
                } else {
                    // Alpha is handled by the child directly, clobber the layer's alpha
                    mPrivateFlags |= PFLAG_ALPHA_SET;
                }
            }
        }
    } else if ((mPrivateFlags & PFLAG_ALPHA_SET) == PFLAG_ALPHA_SET) {
        onSetAlpha(255);
        mPrivateFlags &= ~PFLAG_ALPHA_SET;
    }

    if (!drawingWithRenderNode) {
        // apply clips directly, since RenderNode won't do it for this draw
        if ((parentFlags & ViewGroup.FLAG_CLIP_CHILDREN) != 0 && cache == null) {
            if (offsetForScroll) {
                canvas.clipRect(sx, sy, sx + getWidth(), sy + getHeight());
            } else {
                if (!scalingRequired || cache == null) {
                    canvas.clipRect(0, 0, getWidth(), getHeight());
                } else {
                    canvas.clipRect(0, 0, cache.getWidth(), cache.getHeight());
                }
            }
        }

        if (mClipBounds != null) {
            // clip bounds ignore scroll
            canvas.clipRect(mClipBounds);
        }
    }

    if (!drawingWithDrawingCache) {
        if (drawingWithRenderNode) {
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            ((RecordingCanvas) canvas).drawRenderNode(renderNode);
        } else {
            // Fast path for layouts with no backgrounds
            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                dispatchDraw(canvas);
            } else {
                draw(canvas);
            }
        }
    } else if (cache != null) {
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        if (layerType == LAYER_TYPE_NONE || mLayerPaint == null) {
            // no layer paint, use temporary paint to draw bitmap
            Paint cachePaint = parent.mCachePaint;
            if (cachePaint == null) {
                cachePaint = new Paint();
                cachePaint.setDither(false);
                parent.mCachePaint = cachePaint;
            }
            cachePaint.setAlpha((int) (alpha * 255));
            canvas.drawBitmap(cache, 0.0f, 0.0f, cachePaint);
        } else {
            // use layer paint to draw the bitmap, merging the two alphas, but also restore
            int layerPaintAlpha = mLayerPaint.getAlpha();
            if (alpha < 1) {
                mLayerPaint.setAlpha((int) (alpha * layerPaintAlpha));
            }
            canvas.drawBitmap(cache, 0.0f, 0.0f, mLayerPaint);
            if (alpha < 1) {
                mLayerPaint.setAlpha(layerPaintAlpha);
            }
        }
    }

    if (restoreTo >= 0) {
        canvas.restoreToCount(restoreTo);
    }

    if (a != null && !more) {
        if (!hardwareAcceleratedCanvas && !a.getFillAfter()) {
            onSetAlpha(255);
        }
        parent.finishAnimatingView(this, a);
    }

    if (more && hardwareAcceleratedCanvas) {
        if (a.hasAlpha() && (mPrivateFlags & PFLAG_ALPHA_SET) == PFLAG_ALPHA_SET) {
            // alpha animations should cause the child to recreate its display list
            invalidate(true);
        }
    }

    mRecreateDisplayList = false;

    return more;
}
```
1. 判断是否支持硬件加速
2. 若支持硬件加速，则调用 `updateDisplayListIfDirty()` 更新 renderNode
3. 判断是否使用 cache 进行绘制：
   * 使用 cache：canvas.`drawBitmap(cache, 0.0f, 0.0f, mLayerPaint)`
   * 不使用 cache：若支持硬件加速，((RecordingCanvas) canvas).`drawRenderNode(renderNode)`；否则，`draw(canvas)` 或者 `dispatchDraw(canvas)`


#### updateDisplayListIfDirty()
```java
public RenderNode updateDisplayListIfDirty() {
    final RenderNode renderNode = mRenderNode;
    if (!canHaveDisplayList()) {
        // can't populate RenderNode, don't try
        return renderNode;
    }

    if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
            || !renderNode.hasDisplayList()
            || (mRecreateDisplayList)) {
        // Don't need to recreate the display list, just need to tell our
        // children to restore/recreate theirs
        if (renderNode.hasDisplayList()
                && !mRecreateDisplayList) {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            dispatchGetDisplayList();

            return renderNode; // no work needed
        }

        // If we got here, we're recreating it. Mark it as such to ensure that
        // we copy in child display lists into ours in drawChild()
        mRecreateDisplayList = true;

        int width = mRight - mLeft;
        int height = mBottom - mTop;
        int layerType = getLayerType();

        // Hacky hack: Reset any stretch effects as those are applied during the draw pass
        // instead of being "stateful" like other RenderNode properties
        renderNode.clearStretch();

        final RecordingCanvas canvas = renderNode.beginRecording(width, height);

        try {
            if (layerType == LAYER_TYPE_SOFTWARE) {
                buildDrawingCache(true);
                Bitmap cache = getDrawingCache(true);
                if (cache != null) {
                    canvas.drawBitmap(cache, 0, 0, mLayerPaint);
                }
            } else {
                computeScroll();

                canvas.translate(-mScrollX, -mScrollY);
                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;

                // Fast path for layouts with no backgrounds
                if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                    dispatchDraw(canvas);
                    drawAutofilledHighlight(canvas);
                    if (mOverlay != null && !mOverlay.isEmpty()) {
                        mOverlay.getOverlayView().draw(canvas);
                    }
                    if (isShowingLayoutBounds()) {
                        debugDrawFocus(canvas);
                    }
                } else {
                    draw(canvas);
                }
            }
        } finally {
            renderNode.endRecording();
            setDisplayListProperties(renderNode);
        }
    } else {
        mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    }
    return renderNode;
}
```
1. 获取 canvas： RecordingCanvas canvas = renderNode.beginRecording(width, height)
2. 绘制：
   * layerType == LAYER_TYPE_SOFTWARE：调用 `buildDrawingCache()` 构建 cache，然后 canvas.drawBitmap(cache, 0, 0, mLayerPaint)
   * else： `draw(canvas)` 或者 `dispatchDraw(canvas)`





