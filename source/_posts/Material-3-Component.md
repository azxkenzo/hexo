---
title: Material 3 - Component
date: 2022-12-05 10:43:42
tags: Material3
---

# Theming

|                  | API                         | 描述                            |
|------------------|-----------------------------|-------------------------------|
| Material Theming | **MaterialTheme**           | M3 theme                      |
| Color scheme     | **ColorScheme**             | M3 color scheme               |
|                  | **lightColorScheme**        | M3 light color scheme         |
|                  | **darkColorScheme**         | M3 dark color scheme          |
| Dynamic color    | **dynamicLightColorScheme** | M3 dynamic light color scheme |
|                  | **dynamicDarkColorScheme**  | M3 dynamic dark color scheme  |
| Typography       | **Typography** 	            | M3 typography                 |
| Shape            | **Shapes**                  | M3 shape                      |



# Surfaces and layout

|          | API          | 描述         |
|----------|--------------|------------|
| Surfaces | **Surface**  | M3 surface |
| Scaffold | **Scaffold** | M3 layout  |



# Icons and text

|      | API      | 描述      |
|------|----------|---------|
| Icon | **Icon** | M3 icon |
| Text | **Text** | M3 text |



# Components

|                | API                        | 描述                            |
|----------------|----------------------------|-------------------------------|
| Top app bar    | **TopAppBar**              | M3 small top app bar          |
|                | **CenterAlignedTopAppBar** | M3 center-aligned top app bar |
|                | **MediumTopAppBar**        | M3 medium top app bar         |
|                | **LargeTopAppBar**         | M3 large top app bar          |
| Tabs           | **Tab**                    | M3 tab                        |
|                | **LeadingIconTab**         |                               |
|                | **TabRow**                 |                               |
|                | **ScrollableTabRow**       |                               |
| Bottom app bar | **BottomAppBar**           |                               |
| Buttons        | **Button**                 | M3 filled button              |
|                | **ElevatedButton**         |                               |
|                | **FilledTonalButton**      |                               |
|                | **OutlinedButton**         |                               |
|                | **TextButton**             |                               |
| Cards          | **Card**                   | M3 filled card                |
|                | **ElevatedCard**           |                               |
|                | **OutlinedCard**           |                               |
| Text fields    | **TextField**              |                               |
|                | **OutlinedTextField**      |                               |
| Lists          | **ListItem**               |                               |
| Dialogs        | **AlertDialog**            |                               |
| Menus          | **DropdownMenu**           |                               |
|                | **DropdownMenuItem**       |                               |
|                | **ExposedDropdownMenuBox** |                               |



## Surface
每个 surface 都存在于给定的高度，这会影响该 surface 与其他 surface 的视觉关联方式以及该 surface 如何通过色调变化进行修改。

Surface 负责：
1. 裁剪：Surface 将其子项裁剪为由 `shape` 指定的形状
2. 边框：如果 `shape` 有边框，那么它也会被绘制。
3. 背景：Surface 用 `color` 填充 `shape` 指定的形状。 如果 `color` 是 `ColorScheme.surface`，将应用颜色叠加。 叠加层的颜色取决于此 Surface 的 `tonalElevation` 以及任何父 surface 设置的 `LocalAbsoluteTonalElevation`。 通过将所有先前 Surfaces 的高度相加，这确保了 Surfaces 永远不会出现比其祖先更低的高度覆盖。
4. 内容颜色：Surface 使用 `contentColor` 为该 surface 的内容指定首选颜色 - 这被 `Text` 和 `Icon` 组件用作默认颜色。

如果未设置 `contentColor`，此 surface 将尝试将其背景颜色与主题 `ColorScheme` 中定义的颜色相匹配，并返回相应的内容颜色。 例如，如果此 surface 的 `color` 为 `ColorScheme.surface`，则 `contentColor` 将设置为 `ColorScheme.onSurface`。 如果 `color` 不是主题调色板的一部分，则 `contentColor` 将保持在此 Surface 上方设置的相同值。


## Scaffold
Scaffold 实现了基本的材料设计视觉布局结构。

此组件提供 API 以将多个材料组件组合在一起以构建您的屏幕，通过确保它们的正确布局策略并收集必要的数据以使这些组件正确地协同工作。


