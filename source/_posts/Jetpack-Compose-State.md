---
title: Jetpack Compose - State
date: 2022-12-07 16:42:09
tags: Compose
---

应用程序中的 state 是可以随时间变化的任何值。 这是一个非常宽泛的定义，涵盖了从 Room 数据库到类变量的所有内容。

# State and composition
**_Compose 是声明式的，因此更新它的唯一方法是使用新参数调用相同的 composable_**。 这些参数是 UI state 的表示。 
_**任何时候更新 state 都会发生 recomposition**_。 因此，像 `TextField` 这样的东西不会像在基于 XML 的命令式视图中那样自动更新。 
必须明确告知 composable 新 state，以便它进行相应更新。

    关键术语：Composition：对 Jetpack Compose 在执行 composable 时构建的 UI 的描述。  
    Initial composition：通过第一次运行 composable 来创建 Composition。  
    Recomposition：重新运行 composable 以在数据更改时更新 Composition。


# State in composable
Composable 函数可以使用 `remember` API 将对象存储在内存中。 `remember` 计算出的值在 initial composition 期间存储在 Composition 中，并在 recomposition 期间返回存储的值。 `remember` 可用于存储可变和不可变对象。

**注意**：**remember** 将对象存储在 `Composition` 中，并在调用 `remember` 的 composable 从 Composition 中移除时忘记该对象。

`mutableStateOf` 创建一个可观察的 `MutableState<T>`，它是与 compose runtime 集成的可观察类型。
```
interface MutableState<T> : State<T> {
    override var value: T
}
```
_**对 `value` 的任何更改都会安排任何读取 `value` 的 composable 函数的 recomposition**_。 
在 `ExpandingCard` 的情况下，每当 `expanded` 发生变化时，都会导致 `ExpandingCard` 重新组合。

可通过三种方式在 composable 中声明 `MutableState` 对象：
* `val mutableState = remember { mutableStateOf(default) }`
* `var value by remember { mutableStateOf(default) }`
* `val (value, setValue) = remember { mutableStateOf(default) }`

可以将 remembered value 用作其他 composable 的参数，甚至可以用作语句中的逻辑来更改显示哪些 composable。 

虽然 `remember` 可以帮助在 recomposition 中保留状态，但在配置更改时不会保留状态。 为此，必须使用 `rememberSaveable`。 
`rememberSaveable` 自动保存任何可以保存在 Bundle 中的值。 对于其他值，可以传入自定义 saver 对象。


# Other supported types of state
Compose 不要求使用 `MutableState<T>` 来保存状态。Compose 支持其他可观察类型。 在读取 Compose 中的另一个可观察类型之前，必须将其转换为 `State<T>` 以便 Compose 可以在状态更改时自动 recompose。

Compose 附带了从 Android 应用程序中使用的常见可观察类型创建 `State<T>` 的函数：
* LiveData
* Flow
* RxJava2

如果应用使用自定义可观察类，可以为 Compose 构建扩展函数以读取其他可观察类型。 有关如何执行此操作的示例，请参阅内置函数的实现。 
任何允许 Compose 订阅每个更改的对象都可以转换为 `State<T>` 并由 composable 对象读取。

## Stateful versus stateless
使用 `remember` 存储对象的 composable 会创建内部状态，从而使可组合项成为 _stateful_ 的。

_stateless_ composable 是不持有任何状态的 composable。 实现 stateless 的一种简单方法是使用 state hoisting。

在开发可重用的 composable 时，通常希望公开同一 composable 的 stateful 和 stateless 版本。 有状态版本对于不关心状态的调用者来说很方便，而无状态版本对于需要控制或提升状态的调用者来说是必需的。


# State hoisting
Compose 中的 State hoisting 是一种将状态移动到 composable 的调用方以使 composable 无状态的模式。 
Compose 中 State hoisting 的一般模式是用两个参数替换状态变量：
* `value: T`: the current value to display
* `onValueChange: (T) -> Unit`: an event that requests the value to change, where T is the proposed new value

但是，不仅限于 `onValueChange`。 如果更具体的事件适用于 composable，应该使用 lambda 来定义它们，就像 `ExpandingCard` 对 `onExpand` 和 `onCollapse` 所做的那样。

以这种方式提升的状态有一些重要的属性：
* 单一信任源：通过移动状态而不是复制状态，确保只有一个真实来源。 这有助于避免错误。
* 封装：只有有状态的 composable 才能修改它们的状态。 它完全是内部的。
* 可共享：提升的状态可以与多个 composable 共享。
* 可拦截：无状态 composable 的调用者可以决定在更改状态之前忽略或修改事件。
* 解耦：无状态 ExpandingCard 的状态可以存储在任何地方。 例如，现在可以将 name 移动到 `ViewModel` 中。





