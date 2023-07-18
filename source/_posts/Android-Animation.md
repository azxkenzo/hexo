---
title: Android - Animation
date: 2023-04-11 15:13:58
tags: Android
---

# 关于 Android 中的 animation

## 为 bitmap 设置动画
要对 bitmap 图形（例如 icon 或插图）进行动画处理，请使用 drawable animation API。 通常，这些动画是使用 drawable 资源静态定义的，但您也可以在运行时定义动画行为。

例如，向用户传达两个操作相关的一个好方法是为播放按钮设置动画，当点击该按钮时该按钮会转换为暂停按钮。

## 为 UI 可见性和运动设置动画
当您需要更改布局中 view 的可见性或位置时，最好包含微妙的动画来帮助用户了解 UI 的变化方式。

要在当前布局中移动、显示或隐藏 view，您可以使用 `android.animation` 包提供的 property animation 系统。 
这些 API 在一段时间内更新 `View` 对象的 properties，随着属性的变化不断重绘 view。 例如，当您更改 position 属性时，view 会在屏幕上移动。 
当您更改 alpha 属性时，view 会淡入或淡出。

要创建这些动画的最简单方法，请在布局上启用动画，以便当您更改 view 的可见性时，动画会自动应用。

### 基于物理的运动
只要有可能，将现实世界的物理原理应用到你的动画中，使它们看起来很自然。 例如，当目标发生变化时，他们应该保持动力，并在任何变化期间实现平稳过渡。

为了提供这些行为，Android 支持库包含基于物理的动画 API，它们依靠物理定律来控制动画的发生方式。

两种常见的基于物理的动画如下：
* Spring animation.
* Fling animation.

不基于物理的动画（例如使用 `ObjectAnimator` API 构建的动画）是相当静态的，并且具有固定的持续时间。 
如果目标值发生变化，则必须取消目标值变化时的动画，使用新值作为新的起始值重新配置动画，并添加新的目标值。 
从视觉上看，这个过程会导致动画突然停止，然后出现脱节的运动。

使用基于物理的动画 API（例如 `DynamicAnimation`）构建的动画是由力驱动的。 目标值的变化导致力的变化。 
新的力作用于现有的速度，从而不断过渡到新的目标。 此过程会产生更加自然的动画。

## 为布局更改设置动画
当在当前 Activity 或 Fragment 内交换布局时，可以使用 transition framework 来创建动画。 
所需要做的就是指定开始和结束布局以及要使用的动画类型。 然后系统计算出并在两个布局之间执行动画。 可以使用它来交换整个 UI 或仅移动或替换某些 view。

例如，当用户点击某个项目以查看更多信息时，您可以将布局替换为项目详细信息，并应用 transition。

开始和结束布局均存储在 `Scene` 中，但 starting scene 通常是根据当前布局自动确定的。 
创建一个 `Transition` 来告诉系统您想要什么类型的动画，然后调用 `TransitionManager.go()` ，系统运行动画来交换布局。

## 在 activity 之间设置动画
还可以创建在 Activity 之间过渡的动画。 这基于上一节中描述的相同 transition framework，但它允许您在单独 activity 的布局之间创建动画。

您可以应用简单的动画，例如从侧面滑入新 activity 或使其淡入，但您也可以创建在每个 activity 的共享 view 之间转换的动画。 
例如，当用户点击某个项目以查看更多信息时，您可以 transition 到带有动画的新 Activity，该动画会无缝地扩展该项目以填满屏幕。

像往常一样，您调用 `startActivity()`，但向其传递由 `ActivityOptions.makeSceneTransitionAnimation()` 提供的一组选项。 
这组选项可能包括哪些 view 在 activity 之间共享，以便 transition framework 可以在动画期间连接它们。


# 关于 Property Animation
属性动画系统是一个强大的框架，允许对几乎所有内容进行动画处理。 可以定义动画来随着时间的推移更改任何对象属性，无论它是否绘制到屏幕上。 
属性动画会在指定的时间长度内更改属性（对象中的字段）值。 要为某些内容设置动画，可以指定要设置动画的对象属性，例如对象在屏幕上的位置、要设置动画的时间以及要在哪些值之间设置动画。

属性动画系统允许您定义动画的以下特征：
* Duration: 可以指定动画的持续时间。 默认长度为 300 毫秒。
* Time interpolation: 可以指定如何根据动画的当前运行时间来计算属性值。
* Repeat count and behavior: 可以指定当动画到达持续时间结束时是否重复动画以及重复动画的次数。 您还可以指定是否希望动画反向播放。 将其设置为反向播放动画向前然后向后重复播放，直到达到重复次数。
* Animator sets: 可以将动画分组为逻辑集，这些逻辑集一起播放、按顺序播放或在指定的延迟后播放。
* Frame refresh delay: 可以指定刷新动画帧的频率。 默认设置为每 10 毫秒刷新一次，但应用程序刷新帧的速度最终取决于系统整体的繁忙程度以及系统为底层计时器提供服务的速度。

## 属性动画的工作原理
首先，让我们通过一个简单的示例来了解动画的工作原理。 图 1 描绘了一个使用其 x 属性进行动画处理的假设对象，该属性表示其在屏幕上的水平位置。 动画的持续时间设置为 40 毫秒，行进距离为 40 像素。 每 10 毫秒（默认帧刷新率），对象水平移动 10 个像素。 在 40 毫秒结束时，动画停止，对象在水平位置 40 处结束。这是线性插值动画的示例，意味着对象以恒定速度移动。

![](https://developer.android.com/static/images/animation/animation-linear.png)

您还可以指定动画进行非线性插值。 图 2 说明了一个假设的对象，该对象在动画开始时加速，在动画结束时减速。 对象仍会在 40 毫秒内移动 40 个像素，但是是非线性的。 一开始，该动画加速到中间点，然后从中间点减速，直到动画结束。 如图 2 所示，动画开始和结束时的行进距离小于中间的行进距离。

![](https://developer.android.com/static/images/animation/animation-nonlinear.png)

让我们详细看看属性动画系统的重要组件如何计算如上所示的动画。 图 3 描述了主要类如何相互协作。

![](https://developer.android.com/static/images/animation/valueanimator.png)

`ValueAnimator` 对象跟踪动画的计时，例如动画运行了多长时间，以及它所动画的属性的当前值。

`ValueAnimator` 封装了 `TimeInterpolator`（定义动画插值）和 `TypeEvaluator`（定义如何计算动画属性的值）。 例如，在图 2 中，使用的 `TimeInterpolator` 为 `AccelerateDecelerateInterpolator`，`TypeEvaluator` 为 `IntEvaluator`。

要启动动画，请创建一个 `ValueAnimator` 并为其指定要设置动画的属性的起始值和结束值以及动画的持续时间。 当您调用 `start()` 时，动画开始。 
在整个动画过程中，`ValueAnimator` 根据动画的持续时间和已用时间来计算 0 到 1 之间的已用分数。 经过的分数表示动画完成的时间百分比，0 表示 0%，1 表示 100%。 
例如，在图 1 中，t = 10 ms 时的消耗分数将为 0.25，因为总持续时间为 t = 40 ms。

当 `ValueAnimator` 完成计算经过的分数时，它会调用当前设置的 `TimeInterpolator` 来计算插值分数。 插值分数将经过的分数映射到考虑了所设置的时间插值的新分数。 
例如，在图 2 中，由于动画缓慢加速，因此在 t = 10 ms 时，插值分数（约 0.15）小于经过分数（0.25）。 在图 1 中，插值分数始终与经过分数相同。

计算插值分数时，`ValueAnimator` 会调用相应的 `TypeEvaluator`，根据插值分数、动画的起始值和结束值来计算要设置动画的属性的值。 
例如，在图 2 中，t = 10 ms 时的插值分数为 0.15，因此此时的属性值为 0.15 × (40 - 0)，即 6。

## 属性动画与 view 动画有何不同
view 动画系统仅提供对 View 对象进行动画处理的功能，因此如果您想要对非 View 对象进行动画处理，则必须实现自己的代码才能执行此操作。
view 动画系统还受到以下事实的限制：它仅公开 View 对象的几个方面进行动画处理，例如 View 的缩放和旋转，但不公开背景颜色。

view 动画系统的另一个缺点是它只修改了绘制 view 的位置，而不是实际的 view 本身。 
例如，如果您对按钮进行动画处理以使其在屏幕上移动，则该按钮会正确绘制，但您可以单击该按钮的实际位置不会改变，因此您必须实现自己的逻辑来处理此问题。

使用属性动画系统，这些约束被完全消除，您可以为任何对象（view 和非 view）的任何属性设置动画，并且对象本身实际上被修改。 
属性动画系统在执行动画的方式上也更加稳健。 在较高级别上，您可以将 animator 分配给要设置动画的属性，例如颜色、位置或大小，并且可以定义动画的各个方面，例如多个 animator 的插值和同步。

然而，view 动画系统的设置时间较短，并且需要编写的代码也较少。 如果 view 动画完成了您需要做的一切，或者您现有的代码已经按照您想要的方式工作，则无需使用属性动画系统。 如果出现用例，针对不同情况使用两种动画系统也可能是有意义的。

## API 概览
可以在 `android.animation` 中找到大多数属性动画系统的 API。 由于 view 动画系统已经在 `android.view.animation` 中定义了许多插值器，
因此您也可以在属性动画系统中使用这些插值器。 下表描述了属性动画系统的主要组件。

`Animator` 类提供了创建动画的基本结构。 您通常不直接使用此类，因为它只提供必须扩展才能完全支持动画值的最少功能。 以下子类扩展了 `Animator`：
* ValueAnimator: 属性动画的主要计时引擎，还计算要进行动画处理的属性的值。 它具有计算动画值的所有核心功能，并包含每个动画的计时详细信息、有关动画是否重复的信息、接收更新事件的侦听器以及设置要评估的自定义类型的能力。 动画属性有两个部分：计算动画值并在正在动画的对象和属性上设置这些值。 `ValueAnimator` 不执行第二部分，因此您必须侦听 `ValueAnimator` 计算的值的更新，并使用您自己的逻辑修改要设置动画的对象。
* ObjectAnimator: `ValueAnimator` 的子类，允许您设置目标对象和对象属性以进行动画处理。 当该类计算动画的新值时，该类会相应地更新属性。 大多数时候您希望使用 `ObjectAnimator`，因为它使目标对象上的值的动画处理过程变得更加容易。 但是，有时您想直接使用 `ValueAnimator`，因为 `ObjectAnimator` 还有一些限制，例如要求目标对象上存在特定的访问器方法。
* AnimatorSet: 提供一种将动画分组在一起的机制，以便它们彼此相关地运行。 您可以将动画设置为一起播放、顺序播放或在指定的延迟后播放。

Evaluator 告诉属性动画系统如何计算给定属性的值。 它们获取 `Animator` 类提供的计时数据、动画的开始值和结束值，并根据该数据计算属性的动画值。 
属性动画系统提供以下 Evaluator：
* IntEvaluator: 用于计算 int 属性值的默认 evaluator。
* FloatEvaluator: 用于计算浮点属性值的默认 evaluator。
* ArgbEvaluator: 用于计算以十六进制值表示的颜色属性值的默认 evaluator。
* TypeEvaluator: 允许您创建自己的 evaluator 的 interface。 如果要对不是 int、float 或 color 的对象属性进行动画处理，则必须实现 `TypeEvaluator` 接口以指定如何计算对象属性的动画值。 
如果您希望以不同于默认行为的方式处理这些类型，您还可以为 int、float 和 color 值指定自定义 `TypeEvaluator`。 

时间 interpolator 定义如何将动画中的特定值计算为时间的函数。 例如，您可以指定动画在整个动画中线性发生，这意味着动画在整个时间中均匀移动，
或者您可以指定动画使用非线性时间，例如在开始时加速并在动画结束时减速。表 3 描述了 `android.view.animation` 中包含的 interpolator。 
如果提供的 interpolator 均不能满足您的需求，请实现 `TimeInterpolator` 接口并创建您自己的 interpolator。
* AccelerateDecelerateInterpolator: 其变化率开始和结束缓慢，但在中间加速。
* AccelerateInterpolator: 变化率开始缓慢然后加速的插值器。
* AnticipateInterpolator: 其变化开始向后然后向前猛冲。
* AnticipateOvershootInterpolator: 其变化开始向后，向前猛冲并超过目标值，然后最终返回到最终值。
* BounceInterpolator: 其变化在最后反弹的插值器。
* CycleInterpolator: 其动画重复指定周期数的插值器。
* DecelerateInterpolator: 其变化率开始时很快，然后减速。
* LinearInterpolator: 变化率恒定的插值器
* OvershootInterpolator: 插值器的变化向前猛冲并超过最后一个值，然后返回。
* TimeInterpolator: 允许您实现自己的插值器的接口。


# 为 drawable 图形设置动画
在某些情况下，图像需要动画化。 如果您想要显示由多个图像组成的自定义加载动画，或者想要图标在用户操作后变形，这非常有用。 Android 提供了两种用于绘制 drawable 动画的选项。

第一个选项是使用 `AnimationDrawable`。 这使您可以指定多个静态 drawable 文件，一次显示一个文件以创建动画。 第二个选项是使用 `AnimatedVectorDrawable`，它允许您为 vector drawable 的属性设置动画。

## 使用 AnimationDrawable
创建动画的一种方法是加载一系列 drawable 资源，例如一卷胶片。 `AnimationDrawable` 类是此类 drawable 动画的基础。

您可以使用 `AnimationDrawable` 类 API 在代码中定义动画的帧，但使用列出构成动画的帧的单个 XML 文件来定义它们会更容易。 
这种动画的 XML 文件位于 Android 项目的 `res/drawable/` 目录中。 在本例中，指令给出了动画中每一帧的顺序和持续时间。

XML 文件由一个作为根节点的 `<animation-list>` 元素和一系列子 `<item>` 节点组成，每个子节点定义一个帧 - drawable 资源及其持续时间。 下面是一个 `Drawable` 动画的 XML 文件示例：
```xml
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="true">
    <item android:drawable="@drawable/rocket_thrust1" android:duration="200" />
    <item android:drawable="@drawable/rocket_thrust2" android:duration="200" />
    <item android:drawable="@drawable/rocket_thrust3" android:duration="200" />
</animation-list>
```
该动画运行三帧。 将列表的 `android:oneshot` 属性设置为 `true` 使其循环一次，然后停止并保持在最后一帧。 如果将 `android:oneshot` 设置为 `false`，动画将循环播放。

如果将此 XML 保存为项目的 `res/drawable/` 目录中的 `rocket_thrust.xml`，则可以将其作为 background 图像添加到 `View`，然后调用 `start()` 使其播放。 
下面是一个示例，其中将动画添加到 `ImageView`，然后在触摸屏幕时进行动画处理：
```
val rocketImage = findViewById<ImageView>(R.id.rocket_image).apply {
        setBackgroundResource(R.drawable.rocket_thrust)
        rocketAnimation = background as AnimationDrawable
    }

rocketImage.setOnClickListener({ rocketAnimation.start() })
```
需要注意的是，在 Activity 的 onCreate() 方法期间无法调用在 AnimationDrawable 上调用的 start() 方法，因为 AnimationDrawable 尚未完全附加到 window。 要立即播放动画而不需要交互，您可以从 Activity 中的 onStart() 方法调用它，当 Android 使 view 在屏幕上可见时调用该方法。

## 使用 AnimatedVectorDrawable
vector drawable 是一种可缩放且不会像素化或模糊的可绘制对象类型。 `AnimatedVectorDrawable` 类以及用于向后兼容的 AnimatedVectorDrawableCompat 允许您对 vector drawable 的属性进行动画处理，例如旋转它或更改路径数据以将其变形为不同的图像。

您通常在三个 XML 文件中定义 animated vector drawable：
* A vector drawable with the `<vector>` element in `res/drawable/`.
* An animated vector drawable with the `<animated-vector>` element in `res/drawable/`.
* One or more object animators with the `<objectAnimator>` element in `res/animator/`.

Animated vector drawables 可以为 `<group>` 和 `<path>` 元素的属性设置动画。 `<group>` 元素定义一组 path 或 subgroup，`<path>` 元素定义要绘制的路径。

当您定义要设置动画的 vector drawable 时，请使用 `android:name` 属性为 group 和 path 分配唯一的名称，以便您可以从 animator 定义中引用它们。

animated vector drawable 定义通过 name 引用 vector drawable 中的 group 和 path：
```
<animated-vector xmlns:android="http://schemas.android.com/apk/res/android"
  android:drawable="@drawable/vectordrawable" >
    <target
        android:name="rotationGroup"
        android:animation="@animator/rotation" />
    <target
        android:name="v"
        android:animation="@animator/path_morph" />
</animated-vector>
```
动画定义表示 `ObjectAnimator` 或 `AnimatorSet` 对象。 此示例中的第一个 animator 将目标组旋转 360 度：
```
<objectAnimator
    android:duration="6000"
    android:propertyName="rotation"
    android:valueFrom="0"
    android:valueTo="360" />
```
此示例中的第二个 animator 将 vector drawable 的 path 从一种形状变形为另一种形状。 这些 path 必须兼容变形：它们必须具有相同数量的命令以及每个命令相同数量的参数。


# 为 view 设置动画

## View Animation
可以使用 view 动画系统在 view 上执行 Tween 动画。 Tween(补间)动画使用诸如起点、终点、大小、旋转和动画的其他常见方面等信息来计算动画。

Tween 动画可以对 View 对象的内容执行一系列简单的转换（位置、大小、旋转和透明度）。 因此，如果您有 `TextView` 对象，则可以移动、旋转、放大或缩小文本。 
如果它有背景图像，则背景图像将随文本一起变换。 `animation package` 提供了 tween 动画中使用的所有类。

动画指令序列定义 tween 动画，由 XML 或 Android 代码定义。 与定义布局一样，建议使用 XML 文件，因为它比硬编码动画更具可读性、可重用性和可交换性。 
在下面的示例中，我们使用 XML。

动画指令定义您想要发生的转换、何时发生以及应用它们需要多长时间。 转换可以是顺序的或同时的 - 例如，您可以让 TextView 的内容从左向右移动，然后旋转 180 度，
或者您可以让文本同时移动和旋转。 每个变换都采用一组特定于该变换的参数（尺寸更改的起始尺寸和结束尺寸、旋转的起始角度和结束角度等），以及一组通用参数（例如，开始时间和持续时间） 。 
要使多个转换同时发生，请给它们相同的开始时间； 要使它们连续，请计算开始时间加上前面转换的持续时间。

动画 XML 文件位于 Android 项目的 res/anim/ 目录中。 文件必须有一个根元素：这将是单个 <alpha>、<scale>、<translate>、<rotate>、插值器元素或保存这些元素组的 <set> 元素（可能包括另一个元素） <设置>）。 
默认情况下，所有动画指令同时应用。 要使它们按顺序发生，您必须指定 startOffset 属性，如下例所示。


## 使用动画显示或隐藏 view
显示或隐藏 view 时可以使用三种常见的动画。 您可以使用圆形显示动画、交叉淡入淡出动画或卡片翻转动画。

### 创建 crossfade animation
Crossfade(交叉淡入淡出)动画（也称为溶解）逐渐淡出一个 `View` 或 `ViewGroup`，同时淡入另一个 `View` 或 `ViewGroup`。 此动画对于您想要在应用程序中切换内容或 view 的情况非常有用。 
此处显示的 crossfade 动画使用 `ViewPropertyAnimator`。

#### 创建 view
首先，您需要创建要 crossfade 的两个 view。 以下示例创建一个 progress indicator 和一个可滚动 text view：

#### 设置 crossfade 动画
要设置 crossfade 动画：
1. 为要 crossfade 的 view 创建成员变量。 稍后在动画期间修改 view 时需要这些引用。
2. 对于正在淡入的视图，将其可见性设置为 `GONE`。 这可以防止 view 占用布局空间并在布局计算中忽略它，从而加快处理速度。
3. 将 `config_shortAnimTime` 系统属性缓存在成员变量中。 此属性定义动画的标准 “短” 持续时间。 此持续时间对于微妙的动画或频繁出现的动画来说是理想的选择。 
如果您想使用 `config_longAnimTime` 和 `config_mediumAnimTime` 也可以使用。

#### Crossfade the views
现在 view 已正确设置，通过执行以下操作来 crossfade 它们：
1. 对于淡入的 view，将 alpha 值设置为 `0`，将可见性设置为 `VISIBLE`。 （请记住，它最初设置为 `GONE`。）这使得 view 可见但完全透明。
2. 对于淡入的 view，将其 alpha 值从 `0` 动画化到 `1`。对于淡出的 view，将 alpha 值从 `1` 动画化到 `0`。
3. 在 `Animator.AnimatorListener` 中使用 `onAnimationEnd()`，将淡出的 view 的可见性设置为 `GONE`。 尽管 alpha 值为 `0`，但将 view 的可见性设置为 `GONE` 可以防止 view 占用布局空间并在布局计算中忽略它，从而加快处理速度。

```
private fun crossfade() {
    contentView.apply {
        // Set the content view to 0% opacity but visible, so that it is visible
        // (but fully transparent) during the animation.
        alpha = 0f
        visibility = View.VISIBLE

        // Animate the content view to 100% opacity, and clear any animation
        // listener set on the view.
        animate()
                .alpha(1f)
                .setDuration(shortAnimationDuration.toLong())
                .setListener(null)
    }
    // Animate the loading view to 0% opacity. After the animation ends,
    // set its visibility to GONE as an optimization step (it won't
    // participate in layout passes, etc.)
    loadingView.animate()
            .alpha(0f)
            .setDuration(shortAnimationDuration.toLong())
            .setListener(object : AnimatorListenerAdapter() {
                override fun onAnimationEnd(animation: Animator) {
                    loadingView.visibility = View.GONE
                }
            })
}
```

### 创建卡片翻转动画
卡片通过显示模拟卡片翻转的动画，在内容 view 之间翻转动画。 此处显示的卡片翻转动画使用 `FragmentTransaction`。

#### 创建 Animator object
为了创建卡片翻转动画，总共需要四个 animator。 两个 animator，用于卡正面向左向外和从左侧向内动画。 还需要两个 animator来处理卡片背面从右侧进出以及从右侧进出的动画。

card_flip_left_in.xml

```
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- Before rotating, immediately set the alpha to 0. -->
    <objectAnimator
        android:valueFrom="1.0"
        android:valueTo="0.0"
        android:propertyName="alpha"
        android:duration="0" />

    <!-- Rotate. -->
    <objectAnimator
        android:valueFrom="-180"
        android:valueTo="0"
        android:propertyName="rotationY"
        android:interpolator="@android:interpolator/accelerate_decelerate"
        android:duration="@integer/card_flip_time_full" />

    <!-- Half-way through the rotation (see startOffset), set the alpha to 1. -->
    <objectAnimator
        android:valueFrom="0.0"
        android:valueTo="1.0"
        android:propertyName="alpha"
        android:startOffset="@integer/card_flip_time_half"
        android:duration="1" />
</set>
```

card_flip_left_out.xml

```
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- Rotate. -->
    <objectAnimator
        android:valueFrom="0"
        android:valueTo="180"
        android:propertyName="rotationY"
        android:interpolator="@android:interpolator/accelerate_decelerate"
        android:duration="@integer/card_flip_time_full" />

    <!-- Half-way through the rotation (see startOffset), set the alpha to 0. -->
    <objectAnimator
        android:valueFrom="1.0"
        android:valueTo="0.0"
        android:propertyName="alpha"
        android:startOffset="@integer/card_flip_time_half"
        android:duration="1" />
</set>
```

card_flip_right_in.xml

```
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- Before rotating, immediately set the alpha to 0. -->
    <objectAnimator
        android:valueFrom="1.0"
        android:valueTo="0.0"
        android:propertyName="alpha"
        android:duration="0" />

    <!-- Rotate. -->
    <objectAnimator
        android:valueFrom="180"
        android:valueTo="0"
        android:propertyName="rotationY"
        android:interpolator="@android:interpolator/accelerate_decelerate"
        android:duration="@integer/card_flip_time_full" />

    <!-- Half-way through the rotation (see startOffset), set the alpha to 1. -->
    <objectAnimator
        android:valueFrom="0.0"
        android:valueTo="1.0"
        android:propertyName="alpha"
        android:startOffset="@integer/card_flip_time_half"
        android:duration="1" />
</set>
```

card_flip_right_out.xml

```
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- Rotate. -->
    <objectAnimator
        android:valueFrom="0"
        android:valueTo="-180"
        android:propertyName="rotationY"
        android:interpolator="@android:interpolator/accelerate_decelerate"
        android:duration="@integer/card_flip_time_full" />

    <!-- Half-way through the rotation (see startOffset), set the alpha to 0. -->
    <objectAnimator
        android:valueFrom="1.0"
        android:valueTo="0.0"
        android:propertyName="alpha"
        android:startOffset="@integer/card_flip_time_half"
        android:duration="1" />
</set>
```

#### 创建 view
“卡片”的每一面都是一个单独的布局，可以包含您想要的任何内容，例如两个 text view、两个图像或要在之间翻转的 view 的任意组合。 
然后，将在稍后设置动画的 fragment 中使用这两种布局。 以下布局创建显示文本的卡片的一侧：

#### 创建 fragment
为卡片的正面和背面创建 fragment 类。 这些类返回您之前在每个 fragment 的 `onCreateView()` 方法中创建的布局。 
然后，您可以在要显示卡片的父 activity 中创建此 fragment 的实例。 以下示例显示了使用它们的父 activity 内部的嵌套 fragment 类：

#### 设置卡片翻转动画
现在，需要在父 activity 内显示 fragment。 为此，首先为 activity 创建布局。 以下示例创建一个 FrameLayout，可以在运行时向其中添加 fragment：

在 activity 代码中，将 content view 设置为刚刚创建的布局。 创建 activity 时显示默认 fragment 也是一个好主意

现在已经显示了卡片的正面，可以在适当的时候通过翻转动画显示卡片的背面。 创建一个方法来显示卡片的另一面，该方法执行以下操作：
* 设置之前为 fragment 过渡创建的自定义动画。
* 将当前显示的 fragment 替换为新 fragment，并使用创建的自定义动画为该事件设置动画。
* 将先前显示的 fragment添加到 fragment 后堆栈中，以便当用户按下“后退”按钮时，卡片会翻转回来。

```
private fun flipCard() {
    if (showingBack) {
        supportFragmentManager.popBackStack()
        return
    }

    // Flip to the back.

    showingBack = true

    // Create and commit a new fragment transaction that adds the fragment for
    // the back of the card, uses custom animations, and is part of the fragment
    // manager's back stack.

    supportFragmentManager.beginTransaction()

            // Replace the default fragment animations with animator resources
            // representing rotations when switching to the back of the card, as
            // well as animator resources representing rotations when flipping
            // back to the front (e.g. when the system Back button is pressed).
            .setCustomAnimations(
                    R.animator.card_flip_right_in,
                    R.animator.card_flip_right_out,
                    R.animator.card_flip_left_in,
                    R.animator.card_flip_left_out
            )

            // Replace any fragments currently in the container view with a
            // fragment representing the next page (indicated by the
            // just-incremented currentPage variable).
            .replace(R.id.container, CardBackFragment())

            // Add this transaction to the back stack, allowing users to press
            // Back to get to the front of the card.
            .addToBackStack(null)

            // Commit the transaction.
            .commit()
}
```


## 使用动画移动 view

## 使用 fling 动画移动 view
基于 Fling 的动画使用与对象速度成正比的摩擦力。 使用它来为对象的属性设置动画并逐渐结束动画。 它有一个初始动量，主要来自手势速度，然后逐渐减慢。 当动画的速度足够低以至于在设备屏幕上没有发生明显变化时，动画就会结束。

![](https://developer.android.com/static/images/guide/topics/graphics/fling-animation.gif)

### Add the AndroidX library
```
dependencies {
    implementation 'androidx.dynamicanimation:dynamicanimation:1.0.0'
}
```

### Create a fling animation
`FlingAnimation` 类允许为对象创建 fling 动画。 要构建 fling 动画，创建 `FlingAnimation` 类的实例并提供要设置动画的对象和对象的属性。
```
val fling = FlingAnimation(view, DynamicAnimation.SCROLL_X)
```

### Set velocity(速度)
起始速度定义动画属性在动画开始时发生变化的速度。 默认起始速度设置为每秒零像素。 因此，必须定义一个开始速度以确保动画不会立即结束。

可以使用固定值作为起始速度，也可以将其基于触摸手势的速度。 如果您选择提供固定值，则应以每秒 dp 为单位定义该值，然后将其转换为每秒像素数。 定义每秒 dp 的值可以使速度独立于设备的密度和外形尺寸。

要设置速度，请调用 `setStartVelocity()` 方法并传递以每秒像素为单位的速度。 该方法返回设置了速度的 fling 对象。

```
注意：使用 GestureDetector.OnGestureListener 和 VelocityTracker 类分别检索和计算触摸手势的速度。
```

#### Set an animation value range
当想将属性值限制在一定范围内时，可以设置最小和最大动画值。 当您为具有固有范围（例如 alpha（从 0 到 1））的属性设置动画时，此范围控件特别有用。
```
注意：当fling动画的值达到最小值或最大值时，动画结束。
```
要设置最小值和最大值，分别调用 `setMinValue()` 和 `setMaxValue()` 方法。 两种方法都会返回已设置值的动画对象。

#### Set friction(摩擦力)
`setFriction()` 方法可让更改动画的摩擦力。 它定义了动画中速度降低的速度。
```
注意：如果在动画开始时未设置摩擦力，则动画将使用默认摩擦力值 1。
```

#### Set the minimum visible change
当对未以像素为单位定义的自定义属性进行动画处理时，应该设置用户可见的动画值的最小变化。 它确定结束动画的合理阈值。

对 `DynamicAnimation.ViewProperty` 进行动画处理时无需调用此方法，因为最小的可见更改是从属性派生的。 例如：
* 对于 TRANSLATION_X、TRANSLATION_Y、TRANSLATION_Z、SCROLL_X 和 SCROLL_Y 等视图属性，默认的最小可见更改值为 1 像素。
* 对于使用旋转的动画（例如 ROTATION、ROTATION_X 和 ROTATION_Y），最小可见变化为 MIN_VISIBLE_CHANGE_ROTATION_DEGREES 或 1/10 像素。
* 对于使用不透明度的动画，最小可见变化为 MIN_VISIBLE_CHANGE_ALPHA 或 1/256。

要设置动画的最小可见更改，调用 `setMinimumVisibleChange()` 方法并传递最小可见常量之一或需要为自定义属性计算的值。

#### 计算 minimum visible change value
要计算自定义属性的最小可见变化值，请使用以下公式：

最小可见变化 = 自定义属性值范围 / 动画范围（以像素为单位）

例如，您想要设置动画的属性从 0 进展到 100。这对应于 200 像素的变化。 根据公式，最小可见变化值为 100 / 200 等于 0.5 像素。


## 使用缩放动画放大 view
本指南演示了如何实现点击缩放动画。 点击缩放功能可让照片库等应用程序以缩略图的形式呈现 view 动画以填充屏幕。   
