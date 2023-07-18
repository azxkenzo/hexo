---
title: Android - Activity
date: 2022-08-28 15:43:51
tags: Android
---

## Activity 是什么
Activity 是与用户交互的入口点。它代表具有用户界面的单个屏幕。

## Activity 的生命周期
当用户浏览、离开和返回 App 时，App 中的 Activity 实例会在其生命周期中通过不同的状态进行转换。Activity 类提供了许多回调，允许 Activity 知道状态已更改。

Activity 类提供了一组核心的6个回调：
* `onCreate()`
* `onStart()`
* `onResume()`
* `onPause()`
* `onStop()`
* `onDestroy()`

![callback](https://developer.android.google.cn/guide/components/images/activity_lifecycle.png)


### 生命周期回调

#### onCreate()
该回调在系统首次创建 Activity 时触发。在该方法中，应该执行基本的 App 启动逻辑，该逻辑在 Activity 的整个生命周期内只发生一次。
该方法接收参数 savedInstanceState，它是一个包含 Activity 先前保存状态的 Bundle 对象。如果 Activity 以前从未存在过，则 Bundle 对象的值为 null。

#### onStart()
`onStart()` 调用使 Activity对用户可见。

#### onResume()
当 activity 进入 Resumed 状态时，它来到前台，然后系统调用 `onResume()` 回调。这是App与用户交互的状态，App会一直保持这种状态。

#### onPause()
系统调用此方法作为用户离开 Activity 的第一个指示；它表示 Activity 不再在前台。使用 `onPause()` 方法暂停或调整在 Activity 处于暂停状态时不应继续并且希望很快恢复的操作。

Activity 进入该状态的原因有多种：
* 某些事件会中断App执行，如 `onResume()` 部分所述。
* 多个App以多窗口模式运行。由于任何时候只有一个App（窗口）具有焦点，因此系统会暂停其他所有App。
* 一个新的、半透明的 Activity（如 dialog）打开。只要 Activity 仍然部分可见但不在焦点上，它就会保持暂停状态。

#### onStop()
当 activity 不再对用户可见，它会进入 Stopped 状态，然后系统调用 `onStop()` 回调。在该方法中，app应该释放或调整在app对用户不可见时不需要的资源。

还应该使用 `onStop()` 来执行相对CPU密集型的关闭操作。例如，可以在 `onStop()` 期间将数据保存到数据库。

当 Activity 进入 Stopped 状态时，Activity 对象一直驻留在内存中：它维护所有状态和成员信息，但不附加到窗口管理器。

一旦 activity 停止，如果系统需要恢复内存，系统可能会销毁包含 activity 的进程。 即使系统在 Activity 停止时销毁进程，系统仍将 View 对象的状态（例如 EditText 小部件中的文本）保留在 Bundle（键值对的 blob）中，如果用户 导航回活动。

从 Stopped 状态，activity 要么返回与用户交互，要么 activity 完成运行并离开。 如果 activity 返回，系统调用 onRestart()。 如果 Activity 运行完毕，系统会调用 onDestroy()。

#### onDestroy()
onDestroy() 在 activity 被销毁之前被调用。系统调用此回调是因为：
1. activity 正在结束（由于用户完全关闭 activity 或由于在 activity 上调用了 finish()），或者
2. 由于配置更改（例如设备旋转或多窗口模式），系统正在临时销毁 activity

onDestroy() 回调应该释放所有尚未被早期回调（例如 onStop()）释放的资源。


### Activity 状态和从内存中弹出
系统在需要释放RAM时终止进程；系统杀死给定进程的可能性取决于当时进程的状态。反过来，进程状态取决于进程中运行的 Activity 的状态。 下表显示了进程状态、activity状态和系统杀死进程的可能性之间的相关性。

| 被杀掉的可能性 | 进程状态          | 最终 Activity 状态   |
|---------|---------------|------------------|
| 最小      | 前台（已经或即将获得焦点） | Resumed          |
| 更小      | 可见 (没有焦点)     | Started / paused |
| 更多      | 后台 (不可见)      | Stopped          |
| 最大      | 空的            | Destroyed        |

系统从不直接杀死 activity 以释放内存。 相反，它会杀死 activity 运行的进程，不仅会销毁 activity，还会销毁进程中运行的所有其他内容。



### 保存和恢复瞬时UI状态
当发生配置更改时，系统会默认销毁 activity，从而清除 activity 实例中存储的任何 UI 状态。
当 Activity 由于系统限制而被销毁时，应该使用 ViewModel、onSaveInstanceState() 和/或本地存储的组合来保留用户的瞬态 UI 状态。

#### 实例状态
如果系统由于系统约束（例如配置更改或内存压力）而销毁了 Activity，那么尽管实际的 Activity 实例已经消失，但系统会记住它存在。 如果用户试图导航回 activity，系统会使用一组保存的数据创建该 activity 的新实例，这些数据描述 activity 被销毁时的状态。

系统用来恢复之前状态的保存数据称为 instance state，是存储在 Bundle 对象中的键值对的集合。 默认情况下，系统使用 Bundle 实例状态来保存有关 activity 布局中每个 View 对象的信息（例如输入到 EditText 小部件中的文本值）。 因此，如果 activity 实例被销毁并重新创建，则布局的状态将恢复到之前的状态，而不需要任何代码。 但是，activity 可能有更多想要恢复的状态信息，例如跟踪用户在 activity 中的进度的成员变量。

注意：为了让系统恢复 Activity 中 view 的状态，每个view必须有一个唯一的 ID，由 android:id 属性提供。

Bundle 对象不适合保存大量数据，因为它需要在主线程上进行序列化并消耗系统进程内存。

#### 使用 onSaveInstanceState() 保存简单的、轻量级的UI状态
当 activity 开始停止时，系统会调用 onSaveInstanceState() 方法，以便 activity 可以将状态信息保存到 instance state bundle。 此方法的默认实现保存有关 activity view 层次结构状态的临时信息，例如 EditText 小部件中的文本或 ListView 小部件的滚动位置。

要为 Activity 保存其他实例状态信息，必须重写 onSaveInstanceState() 并将键值对添加到 Bundle 对象中。

#### 使用保存的 instance state 恢复 activity UI 状态
当 Activity 在之前被销毁后重新创建时，可以从系统传递给 Activity 的 Bundle 中恢复保存的实例状态。 onCreate() 和 onRestoreInstanceState() 回调方法都接收包含实例状态信息的同一个 Bundle

可以选择实现 onRestoreInstanceState()，而不是在 onCreate() 期间恢复状态，系统在 onStart() 方法之后调用它。 系统只有在有保存状态需要恢复时才会调用onRestoreInstanceState()，所以不需要检查Bundle是否为null。
注意：始终调用 onRestoreInstanceState() 的超类实现，以便默认实现可以恢复view层次结构的状态。


### 协调 activity
当两个 activity 在同一个进程（app）中并且一个正在启动另一个时。 以下是 Activity A 启动 Activity B 时发生的操作顺序：
1. Activity A 的 onPause() 方法执行
2. Activity B 的 onCreate()、onStart() 和 onResume() 方法依次执行。 （activity B 现在具有用户焦点。）
3. 然后，如果 Activity A 在屏幕上不再可见，则执行其 onStop() 方法。



## 处理 Activity 状态更改

### 发生配置更改
触发配置更改的事件有：设备旋转、更改语言或输入设备等等。配置更改发生时，activity 会被销毁并重新创建

#### 处理多窗口
当App进入多窗口模式时，系统会通知当前运行的 activity 有配置更改。如果调整了已经处于多窗口模式的app的大小，也会发生此行为。

在多窗口模式下，虽然有两个app对用户可见，但只有用户正在与之交互的app处于前台并具有焦点。该 activity 处于 Resumed 状态，另一个窗口中的 activity 处于 Paused 状态。

### Activity 或 dialog 出现在前台
如果前台出现新的 activity 或 dialog，获得焦点并部分覆盖正在进行的 activity，则被覆盖的 activity 失去焦点并进入 Paused 状态。

如果前台出现新的 activity 或 dialog，获得焦点并完全覆盖正在进行的 activity，则被覆盖的 activity 失去焦点并进入 Stopped 状态

当被覆盖 activity 的同一个实例回到前台时，系统会在 activity 上调用 onRestart()、onStart() 和 onResume()。 如果是被覆盖activity的新实例来到后台，系统不会调用onRestart()，只调用onStart()和onResume()。

### 用户按下或手势返回
如果一个 activity 在前台，并且用户按下或手势返回，activity 将通过 onPause()、onStop() 和 onDestroy() 回调进行转换。 除了被销毁之外，该 activity 还从后台堆栈中移除。

### 系统杀掉App进程
如果app在后台并且系统需要为前台app释放额外的内存，则系统可以杀死后台app以释放更多内存。


## Task 和 back stack
Task 是用户在app中尝试执行某些操作时与之交互的 activity 的集合。 这些activity按每个activity的打开顺序排列在一个堆栈（back stack）中。

### Task 和其 返回栈 的生命周期
设备主屏幕是大多数Task的起始位置。 当用户在应用启动器（或主屏幕）中触摸应用或快捷方式的图标时，该应用的Task就会出现在前台。 
如果app不存在任何Task（该应用程序最近未使用过），则会创建一个新Task，并且该app的主 Activity 作为堆栈中的根 Activity 打开。

当当前activity开始另一个activity时，新activity被推到堆栈顶部并获得焦点。前一个activity保留在堆栈中，但已停止。
当activity停止时，系统会保留其用户界面的当前状态。 当用户执行返回操作时，当前activity从栈顶弹出（activity被销毁）并恢复前一个activity（其 UI 的先前状态被恢复）。
堆栈中的 Activity 永远不会重新排列，只会从堆栈中压入和弹出——由当前 Activity 启动时压入堆栈，并在用户使用后退按钮或手势离开时弹出。 因此，返回栈作为后进先出对象结构运行。

如果用户继续按下或手势返回，则堆栈中的每个activity都会弹出以显示前一个activity，直到用户返回主屏幕（或返回到Task开始时正在运行的任何activity）。 当所有activity都从堆栈中删除时，该Task不再存在。

#### 根启动器activity的返回行为
根启动器activity是使用 ACTION_MAIN 和 CATEGORY_LAUNCHER 声明 Intent 过滤器的activity。 这些activity充当从应用启动器进入应用的入口点，并用于启动Task。

当用户从根启动器activity按下或手势返回时，系统会根据设备运行的 Android 版本以不同方式处理事件。在Android 11及更低版本上，系统会finish activity；
在Android 12及更高版本上，系统将activity及其Task移至后台，而不是finish activity。

如果重写 onBackPressed() 以处理返回导航并finish activity，请调用 super.onBackPressed() 而不是finish。 调用 super.onBackPressed() 在适当的时候将activity及其Task移至后台，并为跨app的用户提供更一致的导航体验。

#### 后台和前台Task

#### 多Activity实例

#### 多窗口环境
当app在多窗口环境中同时运行时，系统为每个窗口单独管理Task； 每个窗口可能有多个Task。


### 管理Task
可以使用 `<activity>` 清单元素中的属性和传递给 startActivity() 的intent中的flag来执行这些以及更多操作。

可以使用的主要 `<activity>` 属性：
* taskAffinity
* launchMode
* allowTaskReparenting
* clearTaskOnLaunch
* alwaysRetainTaskState
* finishOnTaskLaunch

可以使用的主要intent flag：
* FLAG_ACTIVITY_NEW_TASK
* FLAG_ACTIVITY_CLEAR_TOP
* FLAG_ACTIVITY_SINGLE_TOP

#### 定义启动模式
启动模式允许定义activity的新实例如何与当前Task相关联。可以用清单文件和intent flag两个方式定义不同的启动模式。

如果两个activity都定义了activity B 应如何与Task关联，则activity A 的请求（如intent中所定义）优先于activity B 的请求（如其清单中所定义）。

使用`<activity>`元素的`launchMode`属性指定activity应如何与Task关联。

`standard` 默认模式。
系统在启动它的Task中创建一个新的activity实例，并将intent路由到它。 Activity可以被实例化多次，每个实例可以属于不同的Task，一个Task可以有多个实例。

`singleTop` 
如果 activity 的实例已经存在于当前Task的顶部，则系统通过调用其 `onNewIntent()` 方法将intent路由到该实例，而不是创建activity的新实例。

`singleTask`
系统在新Task的根部创建activity，或将activity定位在具有相同affinity的现有Task上。 
如果 Activity 的实例已经存在并且位于Task的根目录，则系统通过调用其 `onNewIntent()` 方法将intent路由到现有实例，而不是创建新实例。 同时，它上面的所有其他activity都被销毁了。

`singleInstance`
与`singleTask`相同，只是系统不会在持有该实例的Task中启动任何其他activity。 activity始终是其Task中唯一且唯一的成员； 由这个启动的任何activity都在单独的Task中打开。

`singleInstancePerTask`
Activity 只能作为Task的根 Activity 运行，即创建Task的第一个 Activity，因此任务中只会有该 Activity 的一个实例。 
与 singleTask 启动模式相比，如果设置了 FLAG_ACTIVITY_MULTIPLE_TASK 或 FLAG_ACTIVITY_NEW_DOCUMENT 标志，则可以在不同Task的多个实例中启动此activity。

`onNewIntent()` 在 `onStart()` 之后、`onResume()` 之前调用

使用intent flag定义启动模式：

`FLAG_ACTIVITY_NEW_TASK`
在新Task中启动activity。 如果正在启动的 Activity 的Task已经在运行，则该Task将被带到前台并恢复其最后状态，并且该 Activity 在 onNewIntent() 中接收新intent。
这会产生与`singleTask`的launchMode值相同的行为。

`FLAG_ACTIVITY_SINGLE_TOP`
如果正在启动的activity是当前activity，则现有实例会收到对 onNewIntent() 的调用，而不是创建activity的新实例。

`FLAG_ACTIVITY_CLEAR_TOP`
如果正在启动的 Activity 已经在当前Task中运行，那么不会启动该 Activity 的新实例，而是将其之上的所有其他 Activity 销毁，并将此 Intent 传递给该 Activity 的恢复实例，通过 onNewIntent())。
`FLAG_ACTIVITY_CLEAR_TOP` 最常与 `FLAG_ACTIVITY_NEW_TASK` 结合使用。 当一起使用时，这些标志是一种在另一个Task中定位现有Activity并将其放置在可以响应intent的位置的方法。

**注意**：如果指定Activity的启动模式是`standard`，它也会从堆栈中删除，并在其位置启动一个新实例以处理传入的intent。 
这是因为当启动模式为`standard`时，总是会为新intent创建一个新实例。


#### 处理 affinity
affinity 指示 activity 更喜欢属于哪个Task。使用 `<activity>` 元素的 `taskAffinity` 属性修改任何给定activity的affinity.

taskAffinity 属性采用字符串值，该值必须与 `<manifest>` 元素中声明的默认包名称不同，因为系统使用该名称来标识app的默认 task affinity。

affinity 在两种情况下发挥作用：
* 当启动 activity 的intent包含 FLAG_ACTIVITY_NEW_TASK 标志时。
默认情况下，新activity会启动到调用 startActivity() 的activity的Task中。它被推入与调用者相同的返回栈。
但是，如果intent包含 FLAG_ACTIVITY_NEW_TASK 标志，系统会寻找不同的Task来容纳新activity。通常，这是一项新Task。
但是，它不一定是。如果已经存在与新activity具有相同affinity的现有task，则该activity将启动到该task中。如果没有，它会开始一个新Task。

* 当 Activity 的 allowTaskReparenting 属性设置为“true”时。
在这种情况下，当该Task来到前台时，活动可以从它启动的task移动到它具有affinity的task。


#### 清除返回栈
如果用户长时间离开一个task，系统会清除该task除根Activity之外的所有Activity。 当用户再次返回task时，仅恢复根activity。

可以使用一些 activity 属性来修改此行为：

`alwaysRetainTaskState`
如果在Task的根 activity 中将此属性设置为“true”，则不会发生上述的默认行为。 即使经过很长时间，该task也会将所有activity保留在其堆栈中。

`clearTaskOnLaunch`
如果在task的根activity中将此属性设置为“true”，则每当用户离开task并返回到它时，该task就会被清除到根activity。 
换句话说，它与 alwaysRetainTaskState 正好相反。 用户总是以初始状态返回task，即使离开task仅片刻。
**注意**：如果未设置 FLAG_ACTIVITY_RESET_TASK_IF_NEEDED，则忽略此属性。

`finishOnTaskLaunch`
此属性类似于 clearTaskOnLaunch，但它作用于单个activity，而不是整个task。 它还可以导致任何activity finish，除了根activity。 
当它设置为“true”时，activity仅在当前会话中保留为task的一部分。 如果用户离开然后返回到task，它就不再存在。
**注意**：如果未设置 FLAG_ACTIVITY_RESET_TASK_IF_NEEDED，则忽略此属性。


## 进程和应用的生命周期
进程生命周期错误的一个常见示例是 BroadcastReceiver，它在其 BroadcastReceiver.onReceive() 方法中接收到 Intent 时启动线程，然后从函数返回。
一旦它返回，系统就认为 BroadcastReceiver 不再处于活动状态，因此不再需要它的托管进程（除非其他应用程序组件在其中处于活动状态）。 
因此，系统可能随时终止进程以回收内存，并在这样做时终止进程中运行的衍生线程。 这个问题的解决方案通常是从 BroadcastReceiver 调度一个 JobService，这样系统就知道在这个过程中仍然有活动的工作正在完成。

为了确定在内存不足时应该杀死哪些进程，Android 会根据其中运行的组件和这些组件的状态将每个进程放入“重要性层次结构”中。 这些进程类型是（按重要性排序）：
1. **前台进程**是用户当前正在执行的操作所必需的进程。 各种应用程序组件可能会导致其包含的进程以不同的方式被视为前台。 如果满足以下任何条件，则认为进程处于前台：
    * 它正在与用户交互的屏幕顶部运行一个 Activity（它的 onResume() 方法已被调用）。
    * 它有一个当前正在运行的 BroadcastReceiver（它的 BroadcastReceiver.onReceive() 方法正在执行）。
    * 它有一个 service，当前正在其回调之一（Service.onCreate()、Service.onStart() 或 Service.onDestroy()）中执行代码。

    系统中只会有几个这样的进程，如果内存太低以至于这些进程都无法继续运行，这些进程只会作为最后的手段被杀死。 通常，此时设备已达到内存分页状态，因此需要此操作以保持用户界面响应。

2. **可见进程**正在做用户当前知道的工作，因此杀死它会对用户体验产生明显的负面影响。 在以下情况下，进程被视为可见：
    * 它正在运行一个用户在屏幕上可见但在前台不可见的 Activity（它的 onPause() 方法已被调用）。 例如，如果前台 Activity 显示为允许在其后面看到前一个 Activity 的对话框，则可能会发生这种情况。
    * 它有一个作为前台服务运行的 Service，通过 Service.startForeground() （它要求系统将服务视为用户知道的东西，或者对他们基本上可见）。
    * 它托管系统用于用户知道的特定功能的服务，例如动态壁纸、输入法服务等。

    系统中运行的这些进程的数量没有前台进程那么有限，但仍然相对受控。 这些进程被认为非常重要并且不会被杀死，除非需要这样做以保持所有前台进程运行。

3. **服务进程**是一个持有已使用 startService() 方法启动的 Service 的进程。 虽然这些进程对用户来说是不直接可见的，但它们一般都在做用户关心的事情（比如后台网络数据上传或下载），所以系统会一直保持这些进程运行，除非没有足够的内存来保留所有前台和可见进程。

    已经运行了很长时间（例如 30 分钟或更长时间）的服务的重要性可能会被降级，以允许它们的进程下降到下面描述的缓存列表中。 这有助于避免长时间运行的服务使用过多资源（例如，通过泄漏内存）阻止系统提供良好的用户体验的情况。

4. **缓存进程**是当前不需要的进程，因此当其他地方需要内存等资源时，系统可以根据需要自由地终止它。 在正常运行的系统中，这些是唯一涉及资源管理的进程：运行良好的系统将有多个缓存进程始终可用（以便更有效地在应用程序之间切换）并根据需要定期杀死最旧的进程。 
    只有在非常危急（且不受欢迎）的情况下，系统才会到达所有缓存进程都被杀死并且必须开始杀死服务进程的地步。

   这些进程通常持有一个或多个当前对用户不可见的 Activity 实例（已调用并返回 onStop() 方法）。 如果他们正确实现了 Activity 生命周期，当系统终止此类进程时，它不会影响用户返回该应用程序时的体验：它可以在新的进程重新创建关联的 Activity 时恢复先前保存的状态 。

   这些进程保存在一个列表中。 在此列表中排序的确切策略是平台的实现细节，但通常它会尝试在其他类型的进程之前保留更多有用的进程（一个托管用户的主应用程序，他们看到的最后一个活动等）。 也可以应用其他杀死进程的策略：对允许的进程数量的硬限制，对进程可以持续缓存的时间量的限制等。

进程的优先级也可以基于进程对其的其他依赖关系而增加。 例如，如果进程 A 已绑定到带有 Context.BIND_AUTO_CREATE 标志的服务，或者正在进程 B 中使用 ContentProvider，那么进程 B 的分类将始终至少与进程 A 的分类一样重要。


## Parcelable 和 Bundle
`Parcelable` 和 `Bundle` 对象旨在用于跨进程边界（例如 IPC/Binder 事务）、具有intent的activity之间以及跨配置更改存储瞬态状态。

**注意**：Parcel 不是一种通用的序列化机制，不应该将任何 Parcel 数据存储在磁盘上或通过网络发送。

### 在 activity 之间发送数据
OS将intent的底层 Bundle 打包(parcel)。 然后，OS创建新的activity，解包(un-parcel)数据，并将intent传递给新的活动。

建议使用 Bundle 类在 Intent 对象上设置OS已知的原语。 Bundle 类针对使用parcel的编组和解组进行了高度优化。

在某些情况下，可能需要一种机制来跨activity发送复合或复杂对象。 在这种情况下，自定义类应该实现 Parcelable，并提供适当的 writeToParcel(android.os.Parcel, int) 方法。 它还必须提供一个名为 CREATOR 的非空字段，该字段实现 Parcelable.Creator 接口，其 createFromParcel() 方法用于将 Parcel 转换回当前对象。

通过 Intent 发送数据时，应小心将数据大小限制为几KB。 发送过多数据会导致系统抛出 TransactionTooLargeException 异常。

### 在进程间发送数据
在进程之间发送数据类似于在activity之间发送数据。 但是，在进程之间发送时，建议不要使用自定义 Parcelables。 如果将自定义 Parcelable 对象从一个应用程序发送到另一个应用程序，需要确保发送和接收应用程序上都存在完全相同版本的自定义类。 通常，这可能是跨两个应用程序使用的通用库。 如果您的应用程序尝试将自定义 Parcelable 发送到系统，则可能会发生错误，因为系统无法解组它不知道的类。

Binder 事务缓冲区有一个有限的固定大小，目前为 1MB，由进程正在进行的所有事务共享。 由于此限制是在进程级别而不是在每个活动级别，因此这些事务包括应用程序中的所有绑定事务，例如 onSaveInstanceState、startActivity 以及与系统的任何交互。 当超出大小限制时，会引发 TransactionTooLargeException。