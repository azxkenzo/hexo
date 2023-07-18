---
title: Jetpack Compose - list 和 grid
date: 2022-12-07 14:51:57
tags: Compose
---

## Lazy list
如果需要显示大量 item（或未知长度的列表），使用 `Column` 等布局可能会导致性能问题，因为所有 item 都将被组合和布局，无论它们是否可见。

Compose 提供了一组组件，这些组件仅组合和布局组件视口中可见的 item。 这些组件包括 `LazyColumn` 和 `LazyRow`。

Lazy 组件与 Compose 中的大多数布局不同。 Lazy 组件提供了一个 `LazyListScope.()` 块，而不是接受 `@Composable` 内容块参数，允许应用程序直接发出 composable。 
这个 `LazyListScope` 块提供了一个 DSL，它允许应用程序描述 item 内容。 Lazy 组件然后负责根据布局和滚动位置的需要添加每个 item 的内容。

## LazyListScope DSL
`LazyListScope` 的 DSL 提供了许多用于描述布局中 item 的函数。 在最基本的情况下，`item()` 添加单个 item，而 `items(Int)` 添加多个 item：

还有许多扩展函数允许您添加 item 集合，例如 `List`。

还有一个名为 `itemsIndexed()` 的 `items()` 扩展函数的变体，它提供索引。 


## Lazy grids
`LazyVerticalGrid` 和 `LazyHorizontalGrid` composable 支持在网格中显示 item。 Lazy vertical grid 将在垂直滚动的容器中显示其项目，跨越多列，而 Lazy horizontal grid 将在水平轴上具有相同的行为。

Grid 具有与 list 相同的强大 API 功能，并且它们还使用非常相似的 DSL - `LazyGridScope.()` 来描述内容。

`LazyVerticalGrid` 中的 `columns` 参数和 `LazyHorizontalGrid` 中的 `rows` 参数控制单元格如何形成列或行。 以下示例显示网格中的项目，使用 GridCells.Adaptive 将每列设置为至少 128.dp 宽：

`LazyVerticalGrid` 允许为 item 指定宽度，然后网格将适合尽可能多的列。 计算出列数后，剩余的宽度将平均分配给各列。

如果知道要使用的确切列数，则可以改为提供包含所需列数的 `GridCells.Fixed` 实例。

如果设计只需要某些 item 具有非标准尺寸，可以使用 grid 支持为 item 提供自定义列跨度。 使用 `LazyGridScope DSL` `item` 和 `items` 方法的 `span` 参数指定列跨度。 `maxLineSpan` 是跨度范围的值之一，在使用自适应调整大小时特别有用，因为列数不固定。


## Content padding
有时需要在内容的边缘周围添加 padding。 lazy 组件允许将一些 `PaddingValues` 传递给 `contentPadding` 参数以支持此操作：


## Content spacing
要在 item 之间添加间距，可以使用 `Arrangement.spacedBy()`。 下面的示例在每个 item 之间添加了 4.dp 的空间：

网格接受垂直和水平排列：


## Item keys
默认情况下，每个 item 的状态都根据 item 在 list 或 grid 中的位置设置键控。 但是，如果数据集发生变化，这可能会导致问题，因为改变位置的 item 实际上会丢失任何 remembered state。 
如果想象 LazyRow 在 LazyColumn 中的场景，如果行更改 item 位置，则用户将失去他们在 row 中的滚动位置。

为了解决这个问题，可以为每个 item 提供一个稳定且唯一的 key，为 `key` 参数提供一个块。 提供稳定的 key 可以使项状态在数据集更改之间保持一致：

通过提供 key，可以帮助 Compose 正确处理重新排序。 例如，如果 item 包含 remembered state，则设置 key 将允许 Compose 在其位置发生变化时将此状态与 item 一起移动。

但是，对于可以用作 item key 的类型有一个限制。 key 的类型必须得到 `Bundle` 的支持。 `Bundle` 支持原始类型、枚举或 Parceables 等类型。


## Item animations
如果使用过 RecyclerView 组件，就会知道它会自动为 item 更改设置动画。 lazy 布局为 item 重新排序提供了相同的功能。 API 很简单 —— 只需将 `animateItemPlacement` modifier 设置为 item 内容：
```kotlin
LazyColumn {
    items(books, key = { it.id }) {
        Row(Modifier.animateItemPlacement()) {
            // ...
        }
    }
}
```

如果需要，也可以提供自定义动画规范：
```kotlin
LazyColumn {
    items(books, key = { it.id }) {
        Row(Modifier.animateItemPlacement(
            tween(durationMillis = 250)
        )) {
            // ...
        }
    }
}
```

确保为 item 提供 key，以便可以找到移动元素的新位置。

除了重新排序之外，用于添加和删除的 item 动画目前正在开发中。


## Sticky headers (experimental)
“Sticky header”模式在显示分组数据列表时很有用。

要使用 `LazyColumn` 实现 Sticky header，可以使用实验性的 `stickyHeader()` 函数，提供标头内容：
```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun ListWithHeader(items: List<Item>) {
    LazyColumn {
        stickyHeader {
            Header()
        }

        items(items) { item ->
            ItemRow(item)
        }
    }
}
```

要实现具有多个标题的列表，可以这样做：
```kotlin
// TODO: This ideally would be done in the ViewModel
val grouped = contacts.groupBy { it.firstName[0] }

@OptIn(ExperimentalFoundationApi::class)
@Composable
fun ContactsList(grouped: Map<Char, List<Contact>>) {
    LazyColumn {
        grouped.forEach { (initial, contactsForInitial) ->
            stickyHeader {
                CharacterHeader(initial)
            }

            items(contactsForInitial) { contact ->
                ContactListItem(contact)
            }
        }
    }
}
```

## Reacting to scroll position
许多应用程序需要做出反应并监听滚动位置和 item 布局的变化。 Lazy 组件通过提升 `LazyListState` 来支持此用例：
```kotlin
@Composable
fun MessageList(messages: List<Message>) {
    // Remember our own LazyListState
    val listState = rememberLazyListState()

    // Provide it to LazyColumn
    LazyColumn(state = listState) {
        // ...
    }
}
```

对于简单的用例，应用程序通常只需要知道第一个可见 item 的信息。 为此 `LazyListState` 提供了 `firstVisibleItemIndex` 和 `firstVisibleItemScrollOffset` 属性。

如果使用基于用户是否滚动过第一项来显示和隐藏按钮的示例：
```kotlin
@OptIn(ExperimentalAnimationApi::class) // AnimatedVisibility
@Composable
fun MessageList(messages: List<Message>) {
    Box {
        val listState = rememberLazyListState()

        LazyColumn(state = listState) {
            // ...
        }

        // Show the button if the first visible item is past
        // the first item. We use a remembered derived state to
        // minimize unnecessary compositions
        val showButton by remember {
            derivedStateOf {
                listState.firstVisibleItemIndex > 0
            }
        }

        AnimatedVisibility(visible = showButton) {
            ScrollToTopButton()
        }
    }
}
```

当需要更新其他 UI composable 时，直接在 composition 中读取状态很有用，但也存在不需要在同一组合中处理事件的情况。 
一个常见的例子是在用户滚动到某个点后发送一个分析事件。 为了有效地处理这个问题，可以使用 `snapshotFlow()`：
```kotlin
val listState = rememberLazyListState()

LazyColumn(state = listState) {
    // ...
}

LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .map { index -> index > 0 }
        .distinctUntilChanged()
        .filter { it == true }
        .collect {
            MyAnalyticsService.sendScrolledPastFirstItemEvent()
        }
}
```

`LazyListState` 还通过 `layoutInfo` 属性提供有关当前显示的所有 item 及其在屏幕上的边界的信息。


## Controlling the scroll position
除了对滚动位置做出反应外，应用程序还可以控制滚动位置。 `LazyListState` 通过 `scrollToItem()` 函数和 `animateScrollToItem()` 来支持这一点，
`scrollToItem()` “立即”捕捉滚动位置，`animateScrollToItem()` 使用动画滚动（也称为平滑滚动）

注意： `scrollToItem()` 和 `animateScrollToItem()` 都是挂起函数，这意味着需要在协程中调用它们。
```kotlin
@Composable
fun MessageList(messages: List<Message>) {
    val listState = rememberLazyListState()
    // Remember a CoroutineScope to be able to launch
    val coroutineScope = rememberCoroutineScope()

    LazyColumn(state = listState) {
        // ...
    }

    ScrollToTopButton(
        onClick = {
            coroutineScope.launch {
                // Animate scroll to the first item
                listState.animateScrollToItem(index = 0)
            }
        }
    )
}
```


## Tips on using Lazy layouts

### Avoid using 0-pixel sized items
这可能发生在以下场景中，例如，希望异步检索某些数据（如图像）以在稍后阶段填充列表的 item。 这将导致 Lazy 布局在第一次测量中组合其所有 item，因为它们的高度为 0 像素并且它可以将它们全部放入视口中。 
一旦 item 加载完毕并且它们的高度扩大，lazy 布局就会丢弃所有其他第一次不必要地组合的 item，因为它们实际上无法适应视口。 
为避免这种情况，应该为 item 设置默认大小，以便 lazy 布局可以正确计算实际上有多少 item 可以适合视口：

当在异步加载数据后知道 item 的大致大小时，一个好的做法是确保 item 的大小在加载前后保持不变，例如，通过添加一些占位符。 这将有助于保持正确的滚动位置。

### Avoid nesting components scrollable in the same direction
这仅适用于将没有预定义大小的可滚动子项嵌套在另一个相同方向的可滚动父项中的情况。 
例如，尝试将一个没有固定高度的子 `LazyColumn` 嵌套在一个垂直滚动的 `Column` 父级中：

相反，可以通过将所有 composable 包装在一个父 `LazyColumn` 中并使用其 DSL 传递不同类型的内容来实现相同的结果。 这可以在一个地方发出单个 item 以及多个列表 item：

### Beware of putting multiple elements in one item
在此示例中，第二项 lambda 在一个块中发出 2 个项目：
```kotlin
LazyVerticalGrid(
    // ...
) {
    item { Item(0) }
    item {
        Item(1)
        Item(2)
    }
    item { Item(3) }
    // ...
}
```
lazy 布局将按预期处理这一问题 —— 它们将一个接一个地布置元素，就好像它们是不同的 item 一样。 但是，这样做有几个问题。

当多个元素作为一个 item 的一部分发出时，它们将作为一个实体处理，这意味着它们不能再单独组合。 如果一个元素在屏幕上可见，则必须组合和测量与该 item 对应的所有元素。 
如果过度使用，这会损害性能。 在将所有元素放在一个 item 中的极端情况下，它完全违背了使用 Lazy 布局的目的。 除了潜在的性能问题外，在一项中放置更多元素也会干扰 `scrollToItem()` 和 `animateScrollToItem()`。

但是，将多个元素放在一个项目中有一些有效的用例，例如在列表中使用分隔线。 不希望分隔符更改滚动索引，因为它们不应被视为独立元素。 此外，性能不会因为分频器很小而受到影响。 当前面的项目可见时，分隔线可能需要可见，因此它们可以是前一个项目的一部分：

### Consider using custom arrangements
通常 lazy 列表有很多 item，它们占用的空间超过滚动容器的大小。 但是，当列表中填充的 item 很少时，该设计可能会对这些 item 在视口中的定位方式有更具体的要求。

为此，可以使用自定义垂直 `Arrangement` 并将其传递给 `LazyColumn`。 在下面的例子中，`TopWithFooter` 对象只需要实现 `arrange` 方法即可。 首先，它将一个接一个地定位 item。 其次，如果总使用高度低于视口高度，它会将页脚定位在底部：
```kotlin
object TopWithFooter : Arrangement.Vertical {
    override fun Density.arrange(
        totalSize: Int,
        sizes: IntArray,
        outPositions: IntArray
    ) {
        var y = 0
        sizes.forEachIndexed { index, size ->
            outPositions[index] = y
            y += size
        }
        if (y < totalSize) {
            val lastIndex =
                outPositions.lastIndex
            outPositions[lastIndex] =
                totalSize - sizes.last()
        }
    }
}
```

### Consider adding contentType
从 Compose 1.2 开始，为了最大限度地提高 Lazy 布局的性能，请考虑将 `contentType` 添加到列表或网格中。 这允许为布局的每个 item 指定内容类型，在正在编写由多个不同类型的项目组成的列表或网格的情况下：

当提供 `contentType` 时，Compose 只能在相同类型的 item 之间重用 composition。 当组合结构相似的 item 时，重用效率更高，提供内容类型可确保 Compose 不会尝试在类型 B 的完全不同的 item 之上组合类型 A 的 item。这有助于最大限度地发挥 composition 重用的好处和 lazy 布局性能。












