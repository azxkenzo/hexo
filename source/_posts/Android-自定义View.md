---
title: Android - 自定义View
date: 2023-02-09 14:00:32
tags: Android
---

## 自定义 attribute
定义自定义 attribute：
* 在 `<declare-styleable>` 资源元素中为 View 定义自定义 attribute。
* 为 XML 布局中的属性指定值。
* 在运行时检索属性值。
* 将检索到的属性值应用于 View。

要定义自定义属性，将 `<declare-styleable>` 资源添加到项目中。 通常将这些资源放入 `res/values/attrs.xml` 文件中。 以下是 attrs.xml 文件的示例：
```xml
<resources>
   <declare-styleable name="PieChart">
       <attr name="showText" format="boolean" />
       <attr name="labelPosition" format="enum">
           <enum name="left" value="0"/>
           <enum name="right" value="1"/>
       </attr>
   </declare-styleable>
</resources>
```
此代码声明了两个自定义 attribute，showText 和 labelPosition，它们属于名为 PieChart 的 styleable 的实体。 
按照惯例，styleable 的实体的名称与定义自定义视图的类的名称相同。 尽管不必遵循此约定，但许多流行的代码编辑器都依赖此命名约定来提供语句完成功能。

一旦定义了自定义的 attribute，就可以像使用内置 attribute 一样在布局 XML 文件中使用它们。 
唯一的区别是自定义 attribute 属于不同的命名空间。 它们不属于 `http://schemas.android.com/apk/res/android` 命名空间，
而是属于 `http://schemas.android.com/apk/res/[你的包名]` 例如，下面是如何使用为 PieChart 定义的属性：
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
   xmlns:custom="http://schemas.android.com/apk/res-auto">
 <com.example.customviews.charting.PieChart
     custom:showText="true"
     custom:labelPosition="left" />
</LinearLayout>
```
为避免必须重复长名称空间 URI，示例使用了 `xmlns` 指令。 该指令将别名 custom 分配给命名空间 `http://schemas.android.com/apk/res/com.example.customviews`  可以为命名空间选择任何您想要的别名。


### 应用自定义 attribute
当从 XML 布局创建 View 时，XML 标记中的所有属性都从资源包中读取并作为 `AttributeSet` 传递到 View 的构造函数中。 
虽然可以直接从 AttributeSet 中读取值，但这样做有一些缺点：
* 属性值中的资源引用未解析
* Style 未应用

相反，将 `AttributeSet` 传递给 `obtainStyledAttributes()`。 此方法传回一个 `TypedArray` 数组，其中包含已取消引用和样式化的值。

Android 资源编译器做了很多工作，使调用 `obtainStyledAttributes()` 更容易。 对于 `res/` 目录中的每个 `<declare-styleable>` 资源，生成的 R.java 定义了一个属性 ID 数组和一组常量，这些常量定义了数组中每个属性的索引。 
使用预定义常量从 `TypedArray` 中读取属性。 以下是 PieChart 类读取其属性的方式

请注意，TypedArray 对象是共享资源，使用后必须回收。


## 常用方法
* `onDraw()`
* `onMeasure()`
* `invalidate()`
* `requestLayout()`
* `onLayout()`
* `onSizeChanged()`


## 自定义绘制
`onDraw()` 的参数是一个 `Canvas` 对象，View 可以使用它来绘制自己。 `Canvas` 类定义了绘制文本、线条、位图和许多其他图形基元的方法。 可以在 `onDraw()` 中使用这些方法来创建自定义用户界面 (UI)。

### 创建绘图对象
android.graphics 框架将绘图分为两个区域：
* 画什么，由 `Canvas` 处理
* 如何绘制，由 `Paint` 处理。

例如，`Canvas` 提供了一种绘制线条的方法，而 `Paint` 提供了定义线条颜色的方法。 Canvas 有一个绘制矩形的方法，而 Paint 定义了是用颜色填充该矩形还是将其留空。 简单来说，Canvas 定义了你可以在屏幕上绘制的形状，而 Paint 定义了你绘制的每个形状的颜色、样式、字体等。

提前创建对象是一个重要的优化。 View 重绘非常频繁，许多绘图对象需要昂贵的初始化。 在 onDraw() 方法中创建绘图对象会显着降低性能并使 UI 显得迟缓。

### 处理布局事件
为了正确绘制自定义 View，需要知道它的大小。 复杂的自定义 View 通常需要根据屏幕上区域的大小和形状执行多个布局计算。 永远不应该对屏幕上视图的大小做出假设。 即使只有一个应用程序使用您的 View，该应用程序也需要在纵向和横向模式下处理不同的屏幕尺寸、多种屏幕密度和各种纵横比。

尽管 View 有许多处理测量的方法，但其中大部分不需要重写。 如果 View 不需要对其大小进行特殊控制，则只需重写一个方法：`onSizeChanged()`。

`onSizeChanged()` 会在 View 首次分配大小时调用，如果 View 大小因任何原因发生更改，则会再次调用。 在 `onSizeChanged()` 中计算与 View 大小相关的位置、尺寸和任何其他值，而不是每次绘制时都重新计算它们。 在 PieChart 示例中，onSizeChanged() 是 PieChart 视图计算饼图边界矩形以及文本标签和其他可视元素的相对位置的地方。

当 View 分配了一个大小时，布局管理器假定该大小包括 View 的所有填充。 计算 View 大小时必须处理填充值。 下面是 PieChart.onSizeChanged() 的一个片段，展示了如何做到这一点：
```kotlin
// Account for padding
var xpad = (paddingLeft + paddingRight).toFloat()
val ypad = (paddingTop + paddingBottom).toFloat()

// Account for the label
if (showText) xpad += textWidth

val ww = w.toFloat() - xpad
val hh = h.toFloat() - ypad

// Figure out how big we can make the pie.
val diameter = Math.min(ww, hh)
```

如果需要更好地控制 View 的布局参数，实现 `onMeasure()`。 此方法的参数是 `View.MeasureSpec` 值，它告诉你 View 的父级希望 View 有多大，以及该大小是硬性最大值还是只是一个建议。 作为一种优化，这些值存储为压缩整数，并且使用 `View.MeasureSpec` 的静态方法来解压缩存储在每个整数中的信息。

需要注意三点：
* 计算考虑了 View 的填充。
* 辅助方法 `resolveSizeAndState()` 用于创建最终的宽度和高度值。 此帮助器通过将 View 的所需大小与传递到 `onMeasure()` 的规范进行比较来返回适当的 View.MeasureSpec 值。
* `onMeasure()` 没有返回值。 相反，该方法通过调用 `setMeasuredDimension()` 来传达其结果。 调用此方法是强制性的。 如果省略此调用，View 类将抛出运行时异常。

### 绘制
一旦定义了对象创建和测量代码，就可以实现 onDraw()。 每个 View 都以不同的方式实现 `onDraw()`，但是大多数 View 都共享一些通用操作：
* 使用 `drawText()` 绘制文本。 通过调用 `setTypeface()` 指定字体，通过调用 `setColor()` 指定文本颜色。
* 使用 `drawRect()`、`drawOval()` 和 `drawArc()` 绘制原始形状。 通过调用 `setStyle()` 更改形状是填充的、轮廓的还是两者。
* 使用 `Path` 类绘制更复杂的形状。 通过向 `Path` 对象添加直线和曲线来定义形状，然后使用 `drawPath()` 绘制形状。 与原始形状一样，路径可以是轮廓线、填充线或两者，具体取决于 setStyle()。
* 通过创建 `LinearGradient` 对象来定义渐变填充。 调用 `setShader()` 以在填充形状上使用 LinearGradient。
* 使用 `drawBitmap()` 绘制位图。

### 应用 graphics effects
Android 12（API 级别 31）添加了 RenderEffect 类，该类将常见的图形效果（例如模糊、滤色器、Android 着色器效果等）应用于 View 对象和渲染层次结构。 效果可以组合为连锁效果（由内部和外部效果组成）或混合效果。 由于处理能力有限，不同的 Android 设备可能支持也可能不支持该功能。



## Canvas
canvas 可以看作是是一张无限大的画布，其绘制的中心点是默认为 View 的左上角。Canvas 上绘制的内容，只有在 View 限制范围内的才会显示在屏幕上。

裁切：一系列 `clip...()` 方法

绘制：一系列 `draw...()` 方法

变换：
* `translate()`
* `scale()`
* `rotate()`

其他：
* getWidth() : 返回当前绘制层的宽度
* getHeight() : 返回当前绘制层的高度

