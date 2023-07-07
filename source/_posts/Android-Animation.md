---
title: Android - Animation
date: 2023-04-11 15:13:58
tags: Android
---

## Transitions
Material Components for Android 支持 Material 规范中定义的所有四种运动模式。
* Container transform
* Shared axis (or Forward and Backward)
* Fade through (or Top Level)
* Fade (or Enter and Exit)

该库为这些模式提供了转换类，构建在 AndroidX 转换库 (androidx.transition) 和 Android 转换框架 (android.transition) 之上：

AndroidX (preferred)
* Available in the com.google.android.material.transition package
* Supports API Level 14+
* Supports Fragments and Views, but not Activities or Windows
* Contains backported bug fixes and consistent behavior across API Levels

Platform
* Available in the com.google.android.material.transition.platform package
* Supports API Level 21+
* Supports Fragments, Views, Activities, and Windows
* Bug fixes not backported and may have different behavior across API Levels

### Container transform
Container transform 模式专为包含容器的 UI 元素之间的转换而设计。 此模式在两个 UI 元素之间创建可见连接。

`MaterialContainerTransform` 是 shared element transition。 与传统的 Android 共享元素不同，它不是围绕要在两个场景之间移动的单个共享内容（例如图像）设计的。 相反，这里的共享元素指的是起始 View 或 ViewGroup 的边界容器（例如列表中某项的整行布局）将其大小和形状转换为结束 View 或 ViewGroup（根 ViewGroup）的大小和形状 全屏片段）。 这些开始和结束容器视图是容器转换的“共享元素”。 当这些容器被转换时，它们的内容被交换以创建转换。

#### Transition between Fragments
在 Fragment A 和 Fragment B 的布局中，确定将共享的开始和结束 View。 将匹配的 `transitionName` 添加到这些 View 中的每一个。

注意：开始和结束布局之间的 transitionNames 映射不能超过 1:1。 如果您的开始布局中有多个视图可以映射到最终布局中的结束视图（例如，每个 RecyclerView 项都映射到详细信息屏幕），请阅读 Continuous Shared Element Transitions: RecyclerView to ViewPager 中的共享元素映射。



















