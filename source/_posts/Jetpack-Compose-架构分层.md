---
title: Jetpack Compose - 架构分层
date: 2022-12-05 10:13:27
tags: Compose
---

# 层次

![major layers](https://developer.android.com/static/images/jetpack/compose/layering-major-layers.svg)

每一层都建立在较低的层次之上，结合功能来创建更高层次的组件。 每一层都建立在较低层的公共 API 之上，以验证模块边界并使您能够在需要时替换任何层。 让我们从下往上检查这些层。

### Runtime
该模块提供了 Compose 运行时的基础知识，例如 `remember`、`mutableStateOf`、`@Composable` 注解和 `SideEffect`。 
如果只需要 Compose 的树管理功能，而不需要它的 UI，可以考虑直接在这一层上构建。

### UI
UI层由多个模块（`ui-text`、`ui-graphics`、`ui-tooling`等）组成。 这些模块实现了 UI 工具包的基础，例如 `LayoutNode`、`Modifier`、输入处理程序、自定义布局和绘图。 如果只需要 UI 工具包的基本概念，可以考虑在此层上进行构建。

### Foundation
该模块为 Compose UI 提供与设计系统无关的构建块，例如 `Row` 和 `Column`、`LazyColumn`、特定手势的识别等。可以考虑在 Foundation 层上构建以创建自己的设计系统。

### Material
该模块为 Compose UI 提供了 Material Design 系统的实现，提供了主题系统、样式组件、波纹指示和图标。 在应用程序中使用 Material Design 时，构建在这一层之上。


# 设计原则
Jetpack Compose 的指导原则是提供可以组装（或 composed）在一起的小而集中的功能片段，而不是提供几个单一的组件。 这种方法有许多优点。

## Control
更高级别的组件往往会为您做更多的事情，但会限制您拥有的直接控制量。 如果您需要更多控制，您可以“下拉”以使用较低级别的组件。

## Customization
从较小的构建块组装更高级别的组件可以更轻松地根据需要自定义组件。 例如，考虑 Material 层提供的 Button 的实现：

一个 Button 由 4 个组件组装而成：
1. 提供背景、形状、点击处理等的 material `Surface`。
2. `CompositionLocalProvider`，它在启用或禁用按钮时更改内容的 alpha
3. `ProvideTextStyle` 设置要使用的默认文本样式
4. `Row` 为按钮内容提供默认布局策略

像 Button 这样的组件对它们公开的参数有自己的看法，在启用常见自定义与可能使组件更难使用的参数爆炸之间取得平衡。 例如，Material 组件提供 Material Design 系统中指定的自定义功能，使遵循 Material Design 原则变得容易。

但是，如果您希望在组件参数之外进行自定义，那么您可以“下拉”一个级别并分叉一个组件。 例如，Material Design 指定按钮应具有纯色背景。 如果您需要渐变背景，则 Button 参数不支持此选项。 在这种情况下，您可以使用 Material Button 实现作为参考并构建您自己的组件：

Jetpack Compose 为最高级别的组件保留了最简单的名称。 例如，androidx.compose.material.Text 是基于 androidx.compose.foundation.text.BasicText 构建的。 如果您希望替换更高级别，这可以为您自己的实现提供最容易发现的名称。


## Picking the right abstraction
Compose 构建分层、可重用组件的理念意味着您不应总是使用较低级别的构建块。 许多更高级别的组件不仅提供更多功能，而且经常实施最佳实践，例如支持可访问性。

例如，如果您想为您的自定义组件添加手势支持，您可以使用 Modifier.pointerInput 从头开始构建它，但还有其他构建在此之上的更高级别的组件可能提供更好的起点，例如 Modifier。 可拖动、Modifier.scrollable 或 Modifier.swipeable。

通常，更喜欢在提供所需功能的最高级别组件上构建，以便从它们包含的最佳实践中获益。




