---
title: Android - Fragment
date: 2022-09-01 10:24:19
tags: Android
---

# Fragment 是什么
Fragment 代表应用程序 UI 的可重用部分。 Fragment 定义和管理自己的布局，有自己的生命周期，并且可以处理自己的输入事件。 Fragment 不能独立存在——它们必须由一个 activity 或另一个 Fragment 托管。
Fragment 的view层次成为宿主view层次的一部分或附加到宿主的view层次。

# 创建 Fragment
可以通过在 activity 的布局文件中定义 fragment 或在activity的布局文件中定义fragment容器，然后以编程方式从activity中添加fragment来将fragment添加到activity的view层次结构中。 
无论哪种情况，都需要添加一个 FragmentContainerView 来定义fragment应放置在 Activity 的view层次结构中的位置。 强烈建议始终使用 FragmentContainerView 作为fragment的容器，因为 FragmentContainerView 包含其他ViewGroup（例如 FrameLayout）不提供的特定于fragment的修复。

fragment transaction仅在 savedInstanceState 为 null 时创建。 这是为了确保在首次创建activity时只添加一次fragment。 
当配置发生变化并重新创建activity时，savedInstanceState 不再为空，并且不需要第二次添加fragment，因为fragment会自动从 savedInstanceState 恢复。

# Fragment manager
FragmentManager 是负责对应用程序的 Fragment 执行操作的类，例如添加、删除或替换它们，并将它们添加到返回栈。

## 访问 FragmentManager
**在 activity 中访问**。每个 FragmentActivity 及其子类，例如 AppCompatActivity，都可以通过 `getSupportFragmentManager()` 方法访问 FragmentManager。

**在 Fragment 中访问**。fragment 也能够托管一个或多个子fragment。 在 Fragment 内部，可以通过 `getChildFragmentManager()` 获得对管理 Fragment 子项的 FragmentManager 的引用。 如果需要访问它的宿主FragmentManager，可以使用`getParentFragmentManager()`。

## 使用 FragmentManager
FragmentManager 管理fragment 返回栈。在运行时，FragmentManager 可以执行返回栈操作，例如添加或删除fragment以响应用户交互。
每组更改作为一个单独的单元一起提交，称为 FragmentTransaction。

当用户按下返回按钮，或者当调用 FragmentManager.popBackStack() 时，最顶层的fragment事务会从堆栈中弹出。
如果堆栈上没有更多的fragment事务，并且您没有使用子fragment，则返回事件会冒泡到activity。

在transaction上调用 addToBackStack() 时，请注意transaction可以包含任意数量的操作，例如添加多个fragment、替换多个容器中的fragment等。
当返回栈被弹出时，所有这些操作都被反转为单个原子操作。如果在调用 popBackStack() 之前提交了其他transaction，并且没有对transaction使用 addToBackStack()，则这些操作不会被撤消。

## 执行 transaction
```kotlin
supportFragmentManager.commit {
   replace<ExampleFragment>(R.id.fragment_container)
   setReorderingAllowed(true)
   addToBackStack("name") // name can be null
}
```
在此示例中，ExampleFragment 替换当前位于由 R.id.fragment_container ID 标识的布局容器中的fragment（如果有）。 
将fragment的类提供给 replace() 方法允许 FragmentManager 使用其 FragmentFactory 处理实例化。

`setReorderingAllowed(true)` 优化transaction中涉及的fragment的状态变化，以便动画和过渡正常工作。

调用 `addToBackStack()` 将transaction提交到返回栈。 可以通过按“返回”按钮来撤销transaction并带回前一个fragment。 
如果在单个transaction中添加或删除了多个fragment，则所有这些操作都会在返回栈弹出时撤消。 
addToBackStack() 调用中提供的可选名称使能够使用 popBackStack() 弹回该特定transaction。

如果在执行删除fragment的transaction时不调用 addToBackStack()，则在提交transaction时删除的fragment将被销毁，并且用户无法导航回它。 
如果在删除fragment时确实调用了 addToBackStack()，那么fragment只会停止，然后在用户导航返回时恢复。 请注意，在这种情况下，它的view被销毁了。

### 查找现有fragment
可以使用 `findFragmentById()` 获取对布局容器中当前fragment的引用。 使用 findFragmentById() 从 XML inflate时通过给定 ID 查找fragment，或者在添加到 FragmentTransaction 时通过容器 ID 查找fragment。

或者，可以为fragment分配唯一tag并使用 `findFragmentByTag()` 获取引用。 
可以使用 `android:tag` XML 属性在布局中定义的fragment上分配标签，或者在 FragmentTransaction 中的 add() 或 replace() 操作期间分配tag。

### 子fragment和兄弟fragment的特殊注意事项
在任何给定时间，只允许一个 FragmentManager 控制fragment返回栈。如果应用同时在屏幕上显示多个同级 Fragment，或者应用使用子 Fragment，则必须指定一个 FragmentManager 来处理应用的主导航。

要在fragment事务中定义主导航fragment，请在transaction上调用 `setPrimaryNavigationFragment()` 方法，传入其 childFragmentManager 应具有主控制权的fragment实例。

将导航结构视为一系列层，activity为最外层，将每一层子fragment包裹在其下。每个层都必须有一个主导航fragment。当 Back 事件发生时，最内层控制导航行为。一旦最内层不再有要从中弹回的fragment事务，控制权就会返回到下一层，并重复此过程，直到到达activity。

请注意，当同时显示两个或多个 Fragment 时，只能将其中一个作为主导航 Fragment。将fragment设置为主导航fragment会删除前一个fragment的指定。

## 支持多返回栈
FragmentManager 允许使用 `saveBackStack()` 和 `restoreBackStack()` 方法支持多个返回栈。 这些方法允许通过保存一个返回栈并恢复另一个堆栈来在返回栈之间进行交换。

`saveBackStack()` 的工作方式与使用可选名称参数调用 popBackStack() 类似：弹出指定的事务及其在堆栈上之后的所有事务。 不同之处在于 saveBackStack() 保存了弹出事务中所有fragment的状态。

使用相同的名称参数调用 `restoreBackStack()` 来恢复所有弹出的事务和所有已保存的fragment状态

注意：只能将 saveBackStack() 与调用 setReorderingAllowed(true) 的事务一起使用，以确保可以将事务恢复为单个原子操作。

注意：除非使用 addToBackStack() 为fragment事务传递可选名称，否则不能使用 saveBackStack() 和 restoreBackStack()。

## 为 fragment 提供依赖
默认情况下，FragmentManager 使用framework提供的 FragmentFactory 来实例化fragment的新实例。 此默认工厂使用反射来查找和调用fragment的无参数构造函数。 
这意味着不能使用此默认工厂来为fragment提供依赖项。 这也意味着默认情况下，在第一次创建fragment时使用的任何自定义构造函数都不会在重新创建期间使用。

要为fragment提供依赖项，或使用任何自定义构造函数，必须改为创建自定义 FragmentFactory 子类，然后重写 FragmentFactory.instantiate。 然后，可以使用自定义工厂覆盖 FragmentManager 的默认工厂，然后使用该工厂来实例化fragment。

假设有一个 DessertsFragment 负责展示家乡流行的甜点。 假设 DessertsFragment 依赖于 DessertsRepository 类，该类为它提供了向用户显示正确 UI 所需的信息。

FragmentFactory 的简单实现可能类似于以下内容：
```kotlin
class MyFragmentFactory(val repository: DessertsRepository) : FragmentFactory() {
    override fun instantiate(classLoader: ClassLoader, className: String): Fragment =
        when (loadFragmentClass(classLoader, className)) {
            DessertsFragment::class.java -> DessertsFragment(repository)
            else -> super.instantiate(classLoader, className)
        }
}
```

然后，可以通过在 FragmentManager 上设置属性，将 MyFragmentFactory 指定为构造应用程序fragment时使用的工厂。 
必须在activity的 super.onCreate() 之前设置此属性，以确保在重新创建fragment时使用 MyFragmentFactory。
```kotlin
class MealActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        supportFragmentManager.fragmentFactory = MyFragmentFactory(DessertsRepository.getInstance())
        super.onCreate(savedInstanceState)
    }
}
```

请注意，在activity中设置 FragmentFactory 会覆盖整个activity的fragment层次结构中的fragment创建。 
换句话说，添加的任何子fragment的 childFragmentManager 都使用此处设置的自定义fragment工厂，除非在较低级别覆盖。


# Fragment transaction
通过调用 `beginTransaction()` 从 FragmentManager 获取 FragmentTransaction 的实例

## 允许对fragment状态更改重新排序
每个 FragmentTransaction 都应该使用 `setReorderingAllowed(true)`。

需要允许 FragmentManager 正确执行 FragmentTransaction，特别是当它在返回栈上运行并运行动画和转换时。 
启用该标志可确保如果多个事务一起执行，任何中间fragment（即添加然后立即替换的fragment）不会经历生命周期更改或执行其动画或转换。 
请注意，此标志会影响事务的初始执行和使用 popBackStack() 撤消事务。

## 添加和移除fragment
要将Fragment添加到 FragmentManager，请在事务上调用 `add()`。 此方法接收Fragment容器的 ID，以及希望添加的Fragment的类名。 
添加的Fragment被移动到 `RESUMED` 状态。 强烈建议容器是作为view层次结构一部分的 FragmentContainerView。

要从host中删除Fragment，请调用 `remove()`，传入通过 `findFragmentById()` 或 `findFragmentByTag()` 从FragmentManager检索到的Fragment实例。 
如果Fragment的view先前已添加到容器中，则此时view将从容器中删除。 移除的Fragment被移动到 `DESTROYED` 状态。

使用 `replace()` 将容器中的现有Fragment替换为提供的新Fragment类的实例。 调用 replace() 等效于在容器中调用 remove() 并将新Fragment添加到同一个容器。

### commit 是异步的
调用 `commit()` 不会立即执行事务。 相反，事务被安排在主 UI 线程上运行。 但是，如有必要，可以调用 `commitNow()` 立即在 UI 线程上运行片段事务。

请注意，commitNow 与 addToBackStack 不兼容。 或者，可以通过调用 `executePendingTransactions()` 来执行所有由 commit() 调用提交但尚未运行的挂起的 FragmentTransaction。 这种方法与 addToBackStack 兼容。

### 操作顺序很重要
在 FragmentTransaction 中执行操作的顺序很重要，尤其是在使用 setCustomAnimations() 时。 此方法将给定的动画应用于其后的所有fragment操作。

## 限制fragment的生命周期
FragmentTransactions 可以影响在事务范围内添加的各个fragment的生命周期状态。 
创建 FragmentTransaction 时，`setMaxLifecycle()` 设置给定fragment的最大状态。 
例如，ViewPager2 使用 setMaxLifecycle() 将屏幕外片段限制为 `STARTED` 状态。

## 显示和隐藏fragment的view
使用 FragmentTransaction 方法 `show()` 和 `hide()` 显示和隐藏已添加到容器的fragment的view。 
这些方法设置fragment view的可见性，而不影响fragment的生命周期。

虽然不需要使用fragment事务来切换fragment中view的可见性，但这些方法对于希望更改可见性状态与后台堆栈上的事务相关联的情况很有用。

## 附加和分离fragment
FragmentTransaction 方法 `detach()` 将fragment与 UI 分离，销毁其view层次结构。 fragment保持在与放入返回栈时相同的状态 (STOPPED)。 
这意味着fragment已从 UI 中删除，但仍由FragmentManager管理。

`attach()` 方法重新附加之前分离的fragment。 这会导致其view层次结构被重新创建、附加到 UI 并显示。

由于 FragmentTransaction 被视为单个原子操作集，因此在同一事务中对同一fragment实例的分离和附加调用有效地相互抵消，从而避免了fragment UI 的破坏和立即重建。 
使用单独的事务，如果使用 commit()，则由 executePendingOperations() 分隔，如果要分离然后立即重新附加fragment。

注意：attach() 和 detach() 方法与 onAttach() 和 onDetach() 的 Fragment 方法无关。


# Fragment生命周期
为了管理生命周期，Fragment 实现了 LifecycleOwner，公开了一个 Lifecycle 对象，可以通过 `getLifecycle()` 方法访问该对象。

每个可能的生命周期状态都在 Lifecycle.State 枚举中表示。
* INITIALIZED
* CREATED
* STARTED
* RESUMED
* DESTROYED

Fragment 类包括对应于fragment生命周期中的每个更改的回调方法。 这些包括 onCreate()、onStart()、onResume()、onPause()、onStop() 和 onDestroy()。

fragment的view有一个单独的生命周期，该生命周期独立于fragment的生命周期进行管理。 fragment为其view维护一个 LifecycleOwner，可以使用 `getViewLifecycleOwner()` 或 `getViewLifecycleOwnerLiveData()` 访问。 
访问view的生命周期对于感知生命周期的组件只应在片段视图存在时执行工作的情况很有用，例如观察仅应显示在屏幕上的 LiveData。

## Fragment 和 FragmentManager
当一个fragment被实例化时，它开始于 `INITIALIZED` 状态。要让fragment在其生命周期的其余部分过渡，必须将其添加到 FragmentManager。 
FragmentManager 负责确定其fragment应该处于什么状态，然后将它们移动到该状态。

除了fragment生命周期之外，FragmentManager 还负责将fragment附加到它们的宿主activity，并在fragment不再使用时将它们分离。 
Fragment 类有两个回调方法，onAttach() 和 onDetach()，可以在其中任何一个事件发生时重写它们以执行工作。

当fragment被添加到 FragmentManager 并附加到其宿主activity时，将调用 `onAttach()` 回调。此时，fragment 处于活动状态，FragmentManager 正在管理其生命周期状态。
此时，`findFragmentById()` 等FragmentManager 方法返回这个fragment。

`onAttach()` 总是在任何生命周期状态更改之前调用。

当fragment已从 FragmentManager 中移除并与其宿主activity分离时，将调用 `onDetach()` 回调。该fragment不再处于活动状态，无法再使用 `findFragmentById()` 检索。

`onDetach()` 总是在任何生命周期状态更改后调用。

## Fragment生命周期状态和回调
在确定fragment的生命周期状态时，FragmentManager 会考虑以下内容：
* fragment的最大状态由其 FragmentManager 确定。 fragment不能超出其 FragmentManager 的状态。
* 作为 FragmentTransaction 的一部分，可以使用 `setMaxLifecycle()` 在fragment上设置最大生命周期状态。
* fragment的生命周期状态永远不能大于其父级。 例如，父fragment或activity必须在其子fragment之前启动。 同样，子fragment必须在其父fragment或activity之前停止。

注意：避免使用 <fragment> 标记来使用 XML 添加片段，因为 <fragment> 标记允许片段超出其 FragmentManager 的状态。 相反，始终使用 FragmentContainerView 来使用 XML 添加片段。

![alt 属性文本](https://developer.android.google.cn/static/images/guide/fragments/fragment-view-lifecycle.png)

### 向上状态转换

#### Fragment CREATED
当Fragment达到 CREATED 状态时，它已被添加到 FragmentManager 并且已调用 onAttach() 方法。

这将是通过Fragment的 SavedStateRegistry 恢复与Fragment本身关联的任何已保存状态的适当位置。 请注意，此时 Fragment 的view尚未创建，并且与 Fragment 的view关联的任何状态都应仅在view创建后才能恢复。

此转换调用 onCreate() 回调。 回调还接收一个 savedInstanceState Bundle 参数，其中包含以前由 onSaveInstanceState() 保存的任何状态。 请注意，第一次创建Fragment时，savedInstanceState 的值为 null，但对于后续重新创建它始终为非 null，即使没有覆盖 onSaveInstanceState()。 

#### Fragment CREATED 和 View INITIALIZED
只有当 Fragment 提供有效的 View 实例时，才会创建 Fragment 的view生命周期。 在大多数情况下，可以使用带有@LayoutId 的fragment构造函数，它会在适当的时间自动填充view。 还可以覆盖 onCreateView() 以编程方式填充或创建fragment的view。

当且仅当fragment的view使用非空view实例化时，该view被设置在fragment上并且可以使用 getView() 检索。 getViewLifecycleOwnerLiveData() 然后使用与fragment的view对应的新初始化的 LifecycleOwner 进行更新。 此时也会调用 onViewCreated() 生命周期回调。

这是设置view初始状态、开始观察其回调更新fragment的view的 LiveData 实例以及在fragment中view的任何 RecyclerView 或 ViewPager2 实例上设置适配器的合适位置。

#### Fragment 和 View CREATED
在fragment的view被创建之后，之前的view状态（如果有的话）会被恢复，然后view的生命周期会被移动到 CREATED 状态。 
view生命周期所有者也会向其观察者发出 ON_CREATE 事件。 在这里，应该恢复与fragment的view关联的任何其他状态。

此转换还调用 onViewStateRestored() 回调。

#### Fragment 和 View STARTED
强烈建议将 Lifecycle-aware 组件绑定到 Fragment 的 STARTED 状态，因为该状态保证 Fragment 的View是可用的（如果已创建），并且在 Fragment 的子 FragmentManager 上执行 FragmentTransaction 是安全的 . 
如果fragment的view不为空，则fragment的view生命周期会在fragment的生命周期移至 STARTED 后立即移至 STARTED。

当fragment变为 STARTED 时，将调用 onStart() 回调。

#### Fragment 和 View RESUMED
当 Fragment 可见时，所有 Animator 和 Transition 效果都已完成，并且该 Fragment 已准备好进行用户交互。 
Fragment的生命周期移动到 RESUMED 状态，并调用 onResume() 回调。

过渡到 RESUMED 是表明用户现在可以与fragment交互的适当信号。 未 RESUMED 的fragment不应手动将焦点设置在其view上或尝试处理输入法可见性。

### 向下状态转换

#### Fragment 和 View STARTED
当用户开始离开 Fragment 并且 Fragment 仍然可见时，Fragment 及其view的 Lifecycles 将移回 STARTED 状态并向其观察者发出 ON_PAUSE 事件。 然后fragment调用其 onPause() 回调。

#### Fragment 和 View CREATED
一旦fragment不再可见，fragment及其view的生命周期将移动到 CREATED 状态并向其观察者发出 ON_STOP 事件。 
这种状态转换不仅由停止的父activity或fragment触发，而且由父activity或fragment保存状态触发。 
此行为保证在保存fragment状态之前调用 ON_STOP 事件。 这使得 ON_STOP 事件成为可以安全地对子 FragmentManager 执行 FragmentTransaction 的最后一点。

#### Fragment CREATED 和 View DESTROYED
在所有的退出动画和转换完成后，fragment的view已经从窗口中分离出来，fragment的view生命周期进入 DESTROYED 状态并向其观察者发出 ON_DESTROY 事件。 
该fragment然后调用其 onDestroyView() 回调。 此时，fragment 的view已到达其生命周期的末尾，getViewLifecycleOwnerLiveData() 返回一个空值。

此时，所有对fragment view的引用都应该被删除，从而允许对fragment view进行垃圾回收。

#### Fragment DESTROYED
如果 Fragment 被删除，或者 FragmentManager 被销毁，则 Fragment 的 Lifecycle 将进入 DESTROYED 状态并将 ON_DESTROY 事件发送给它的观察者。 
然后fragment调用其 onDestroy() 回调。 此时，fragment已达到其生命周期的终点。


# 使用动画在fragment之间导航
Fragment API 提供了两种在导航过程中使用运动效果和变换在视觉上连接fragment的方法。 
其中之一是 Animation Framework，它同时使用 Animation 和 Animator。 另一个是Transition Framework，其中包括共享元素过渡。

## 设置 animation
首先，需要为进入和退出效果创建animation，这些动画在导航到新fragment时运行。 可以将animation定义为补间动画资源。 这些资源允许定义fragment在动画期间应如何旋转、拉伸、淡化和移动。

这些动画可以在 res/anim 目录中定义：
```
<!-- res/anim/fade_out.xml -->
<?xml version="1.0" encoding="utf-8"?>
<alpha xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="@android:integer/config_shortAnimTime"
    android:interpolator="@android:anim/decelerate_interpolator"
    android:fromAlpha="1"
    android:toAlpha="0" />
```

```
<!-- res/anim/slide_in.xml -->
<?xml version="1.0" encoding="utf-8"?>
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="@android:integer/config_shortAnimTime"
    android:interpolator="@android:anim/decelerate_interpolator"
    android:fromXDelta="100%"
    android:toXDelta="0%" />
```

注意：强烈建议对涉及一种以上animation类型的效果使用transition，因为使用嵌套 AnimationSet 实例存在已知问题。

还可以为弹出返回堆栈时运行的进入和退出效果指定动画，这可能在用户点击向上或返回按钮时发生。 这些被称为 popEnter 和 popExit 动画。

定义动画后，通过调用 `FragmentTransaction.setCustomAnimations()` 来使用它们，并通过资源 ID 传入动画资源，如下例所示: 

```kotlin
val fragment = FragmentB()
supportFragmentManager.commit {
    setCustomAnimations(
        enter = R.anim.slide_in,
        exit = R.anim.fade_out,
        popEnter = R.anim.fade_in,
        popExit = R.anim.slide_out
    )
    replace(R.id.fragment_container, fragment)
    addToBackStack(null)
}
```

注意：FragmentTransaction.setCustomAnimations() 将自定义动画应用于 FragmentTransaction 中的所有未来fragment操作。 事务中的先前操作不受影响。

## 设置 transition
还可以使用transition来定义进入和退出效果。 这些转换可以在 XML 资源文件中定义。

```
<!-- res/transition/fade.xml -->
<fade xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="@android:integer/config_shortAnimTime"/>
```

```
<!-- res/transition/slide_right.xml -->
<slide xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="@android:integer/config_shortAnimTime"
    android:slideEdge="right" />
```

一旦定义了transition，通过在进入fragment上调用 `setEnterTransition()` 和在退出Fragment上调用 `setExitTransition()` 来应用它们，通过它们的资源 ID 传递inflated transition资源，如以下示例所示：

```kotlin
class FragmentA : Fragment() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val inflater = TransitionInflater.from(requireContext())
        exitTransition = inflater.inflateTransition(R.transition.fade)
    }
}

class FragmentB : Fragment() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val inflater = TransitionInflater.from(requireContext())
        enterTransition = inflater.inflateTransition(R.transition.slide_right)
    }
}
```

fragment支持 AndroidX transition。 虽然 Fragment 也支持framework transition，但强烈建议使用 AndroidX transition，
因为它们在 API 级别 14 及更高级别中受支持，并且包含旧版本的framework transition中不存在的错误修复。


## 使用共享元素 transition
作为Transition Framework的一部分，共享元素transition确定了相应view在fragment过渡期间如何在两个fragment之间移动。

在高层次上，以下是如何使用共享元素进行fragment transition：
1. 为每个共享元素view 分配一个唯一的transition名称。
2. 将共享元素view和transition名称添加到 FragmentTransaction。
3. 设置共享元素transition动画。

首先，必须为每个共享元素view分配一个唯一的transition名称，以允许view从一个fragment映射到下一个fragment。 
使用 `ViewCompat.setTransitionName()` 为每个fragment布局中的共享元素设置transition名称.

```kotlin
class FragmentA : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        ...
        val itemImageView = view.findViewById<ImageView>(R.id.item_image)
        ViewCompat.setTransitionName(itemImageView, “item_image”)
    }
}

class FragmentB : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        ...
        val heroImageView = view.findViewById<ImageView>(R.id.hero_image)
        ViewCompat.setTransitionName(heroImageView, “hero_image”)
    }
}
```

要将共享元素包含在fragment transition中，FragmentTransaction 必须知道每个共享元素的view如何从一个fragment映射到下一个fragment。 
通过调用 `FragmentTransaction.addSharedElement()` 将每个共享元素添加到 FragmentTransaction，并在下一个fragment中传入view和相应view的transition名称，如下例所示：

```kotlin
val fragment = FragmentB()
supportFragmentManager.commit {
    setCustomAnimations(...)
    addSharedElement(itemImageView, “hero_image”)
    replace(R.id.fragment_container, fragment)
    addToBackStack(null)
}
```

要指定共享元素如何从一个fragment过渡到下一个fragment，必须在被导航到的fragment上设置 enter transition。 
在 Fragment 的 onCreate() 方法中调用 `Fragment.setSharedElementEnterTransition()`，如下例所示：

```kotlin
class FragmentB : Fragment() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        sharedElementEnterTransition = TransitionInflater.from(requireContext())
             .inflateTransition(R.transition.shared_image)
    }
}
```

`shared_image` 转换定义如下：

```
<!-- res/transition/shared_image.xml -->
<transitionSet>
    <changeImageTransform />
</transitionSet>
```

默认情况下，共享元素的 enter transition 也用作共享元素的return transition。 Return transition确定当fragment事务从回栈弹出时共享元素如何转换回前一个fragment。 
如果想指定不同的return transition，可以在fragment的 onCreate() 方法中使用 `Fragment.setSharedElementReturnTransition()` 来实现。

## 推迟 transition
在某些情况下，可能需要将fragment transition推迟一小段时间。 例如，可能需要等到进入fragment中的所有view都被测量和布局，以便 Android 可以准确地捕获它们的开始和结束状态以进行转换。

此外，您的转换可能需要推迟到加载了一些必要的数据。 例如，可能需要等到为共享元素加载图像。 否则，如果图像在过渡期间或之后完成加载，则过渡可能会不和谐。

要推迟转换，必须首先确保fragment transaction允许对fragment状态更改进行重新排序。 要允许重新排序fragment状态更改，请调用 `FragmentTransaction.setReorderingAllowed()`。

要推迟 enter transition，在进入fragment的 `onViewCreated()` 方法中调用 `Fragment.postponeEnterTransition()`：

加载数据并准备好开始transition后，调用 `Fragment.startPostponedEnterTransition()`。 以下示例使用 Glide 库将图像加载到共享的 ImageView 中，将相应的transition推迟到图像加载完成。

```kotlin
class FragmentB : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        ...
        Glide.with(this)
            .load(url)
            .listener(object : RequestListener<Drawable> {
                override fun onLoadFailed(...): Boolean {
                    startPostponedEnterTransition()
                    return false
                }

                override fun onResourceReady(...): Boolean {
                    startPostponedEnterTransition()
                    return false
                }
            })
            .into(headerImage)
    }
}
```

在处理用户互联网连接缓慢等情况时，可能需要在一定时间后开始延迟transition，而不是等待所有数据加载完毕。 
对于这些情况，可以改为在进入fragment的 onViewCreated() 方法中调用 `Fragment.postponeEnterTransition(long, TimeUnit)`，并传入持续时间和时间单位。 一旦指定的时间过去，延迟就会自动开始。

## 将共享元素transition与 RecyclerView 一起使用


# 使用fragment保存状态
为了保证用户的状态被保存，Android框架会自动保存和恢复fragment和返回栈。 因此，需要确保fragment中的任何数据也被保存和恢复。

下表概述了导致fragment丢失状态的操作，以及各种类型的状态是否在这些更改中持续存在。 表中提到的状态类型如下：
* 变量：fragment中的局部变量。
* view状态：fragment中一个或多个view所拥有的任何数据。
* SavedState：此fragment实例固有的数据，应保存在 onSaveInstanceState() 中。
* NonConfig：从外部源（例如服务器或本地存储库）提取的数据，或一旦提交就发送到服务器的用户创建的数据。

| 操作                              | 变量  | view状态 | SavedState | NonConfig |
|---------------------------------|-----|--------|------------|-----------|
| 添加到返回栈                          | ✓   | ✓      | x          | ✓         |
| 配置更改                            | x   | ✓      | ✓          | ✓         |
| 进程死亡/重新创建                       | x   | ✓      | ✓          | ✓         |
| Removed not added to back stack | x   | x      | x          | x         |
| Host finished                   | x   | x      | x          | x         |

### View State
View负责管理自己的状态。所有 Android 框架提供的View都有自己的 onSaveInstanceState() 和 onRestoreInstanceState() 实现，因此不必在fragment中管理view state。

注意：为确保在配置更改期间正确处理，应该为创建的任何自定义View实现 onSaveInstanceState() 和 onRestoreInstanceState()。

View需要一个 ID 来保持其状态。 此 ID 在fragment及其View层次结构中必须是唯一的。 **没有 ID 的View无法保留其状态**。


### SavedState
Fragment负责管理fragment功能不可或缺的少量动态状态。 可以使用 Fragment.onSaveInstanceState(Bundle) 保留易于序列化的数据。 与 Activity.onSaveInstanceState(Bundle) 类似，放置在 bundle 中的数据通过配置更改和进程死亡和重新创建来保留，并且在 Fragment 的 onCreate(Bundle)、onCreateView(LayoutInflater、ViewGroup、Bundle) 和 onViewCreated(View, Bundle) 方法中可用。

注意：onSaveInstanceState(Bundle) 仅在fragment的宿主activity调用它自己的 onSaveInstanceState(Bundle) 时调用。


### NonConfig
NonConfig 数据应放置在fragment之外，例如在 ViewModel 中。

ViewModel 类固有地允许数据在配置更改（例如屏幕旋转）后保留下来，并在fragment被放置在返回栈时保留在内存中。 
在进程死亡和重新创建之后，ViewModel 被重新创建。 将 `SavedState` 模块添加到 ViewModel 允许 ViewModel 通过进程死亡和重新创建来保留简单状态。



# 与 Fragment 通信
为了正确响应用户事件或共享状态信息，通常需要在activity及其fragment之间或两个或多个fragment之间建立通信通道。
为了保持 Fragment 自包含，不应该让 Fragment 直接与其他 Fragment 或其宿主activity通信。

Fragment 库提供了两个通信选项：共享 `ViewModel` 和 Fragment Result API。
推荐的选项取决于用例。要与任何自定义 API 共享持久数据，应该使用 ViewModel。
对于可以放置在 Bundle 中的数据的一次性结果，应该使用 Fragment Result API。

## 使用 ViewModel 共享数据
当需要在多个fragment之间或fragment与其宿主activity之间共享数据时，ViewModel 是一个理想的选择。 ViewModel 对象存储和管理 UI 数据。

### 与宿主activity共享数据
Fragment及其宿主activity都可以通过将activity传递给 `ViewModelProvider` 构造函数来检索具有activity范围的 ViewModel 的共享实例。 
`ViewModelProvider` 处理实例化 ViewModel 或检索它（如果它已经存在）。

```kotlin
class MainActivity : AppCompatActivity() {
    // Using the viewModels() Kotlin property delegate from the activity-ktx
    // artifact to retrieve the ViewModel in the activity scope
    private val viewModel: ItemViewModel by viewModels()
}

class ListFragment : Fragment() {
    // Using the activityViewModels() Kotlin property delegate from the
    // fragment-ktx artifact to retrieve the ViewModel in the activity scope
    private val viewModel: ItemViewModel by activityViewModels()
}

```

**注意**：请务必使用 ViewModelProvider 的适当范围。 在上面的示例中，MainActivity 被用作 MainActivity 和 ListFragment 的范围，
因此它们都提供了相同的 ViewModel。 如果 ListFragment 将其自身用作范围，则会提供与 MainActivity 不同的 ViewModel。

### 在fragment之间共享数据
同一activity中的两个或多个fragment通常需要相互通信。 例如，假设一个fragment显示一个列表，另一个fragment允许用户对列表应用各种过滤器。 
如果没有fragment直接通信，这种情况可能并不容易实现，这意味着它们不再是独立的。 此外，两个fragment都必须处理另一个fragment尚未创建或不可见的情况。

这些fragment可以使用它们的activity范围共享一个 ViewModel 来处理这种通信。 通过这种方式共享 ViewModel，fragment 之间不需要相互了解，activity 也不需要做任何事情来方便通信。

### 在父fragment和子fragment之间共享数据
要在这些 Fragment 之间共享数据，请将父 Fragment 用作 ViewModel 范围。

```kotlin
class ListFragment: Fragment() {
    // Using the viewModels() Kotlin property delegate from the fragment-ktx
    // artifact to retrieve the ViewModel
    private val viewModel: ListViewModel by viewModels()
}

class ChildFragment: Fragment() {
    // Using the viewModels() Kotlin property delegate from the fragment-ktx
    // artifact to retrieve the ViewModel using the parent fragment's scope
    private val viewModel: ListViewModel by viewModels({requireParentFragment()})
    ...
}
```


## 使用 Fragment Result API 获取结果
在某些情况下，可能希望在两个fragment之间或fragment与其宿主activity之间传递一次性值。 例如，可能有一个读取 QR 码的fragment，将数据传递回之前的fragment。 
在 Fragment 1.3.0 及更高版本中，每个 FragmentManager 都实现了 FragmentResultOwner。 这意味着 FragmentManager 可以充当fragment结果的中央存储。 
此更改允许组件通过设置fragment结果并侦听这些结果来相互通信，而无需这些组件相互直接引用。

### 在fragment之间传递结果
要将数据从fragment B 传递回fragment A，首先在fragment A（接收结果的fragment）上设置结果侦听器。 
在 Fragment A 的 FragmentManager 上调用 setFragmentResultListener()，如下例所示：
```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // Use the Kotlin extension in the fragment-ktx artifact
    setFragmentResultListener("requestKey") { requestKey, bundle ->
        // We use a String here, but any type that can be put in a Bundle is supported
        val result = bundle.getString("bundleKey")
        // Do something with the result
    }
}
```
![](https://developer.android.google.cn/static/images/guide/fragments/fragment-a-to-b.png)

在产生结果的fragment B 中，必须使用相同的 `requestKey` 在相同的 `FragmentManager` 上设置结果。 通过使用 `setFragmentResult()` API 来做到这一点：

```kotlin
button.setOnClickListener {
    val result = "result"
    // Use the Kotlin extension in the fragment-ktx artifact
    setFragmentResult("requestKey", bundleOf("bundleKey" to result))
}
```

然后fragment A 接收结果并在fragment `STARTED`后执行侦听器回调。

对于给定的key ，只能有一个侦听器和结果。 如果对同一个key多次调用 `setFragmentResult()`，并且如果侦听器未 `STARTED`，则系统会用更新的结果替换任何待处理的结果。 
如果设置了一个没有对应监听器接收的result，result会存储在 FragmentManager 中，直到你设置了具有相同 key 的监听器。 一旦侦听器收到result并触发 `onFragmentResult()` 回调，result就会被清除。 这种行为有两个主要含义：

* 返回栈上的fragment在弹出并 `STARTED` 之前不会收到result。
* 如果在设置结果时侦听结果的fragment 处于 `STARTED`，则立即触发侦听器的回调。


### 在父和子fragment之间传递数据
要将结果从子fragment传递给父fragment，父fragment在调用 setFragmentResultListener() 时应使用 getChildFragmentManager() 而不是 getParentFragmentManager()。

### 在宿主activity中接收结果
要在宿主activity中接收fragment结果，使用 getSupportFragmentManager() 在fragment manager上设置结果侦听器。