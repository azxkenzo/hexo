---
title: Kotlin - 控制流
date: 2022-11-17 12:26:18
tags: Kotlin
---

## 条件和循环

### if 表达式
在 Kotlin 中，if 是一个表达式：它返回一个值。 因此，没有三元运算符（条件？那么：else），因为普通的 if 在这个角色中可以正常工作。

if 表达式的分支可以是块。 在这种情况下，最后一个表达式是块的值。

如果使用 if 作为表达式，例如，返回其值或将其分配给变量，则 else 分支是强制性的。

### when 表达式
when 定义具有多个分支的条件表达式。 它类似于类 C 语言中的 switch 语句。

when 按顺序将其参数与所有分支匹配，直到满足某个分支条件。

when 既可以用作表达式，也可以用作语句。 如果用作表达式，则第一个匹配分支的值成为整个表达式的值。 如果将其用作语句，则忽略各个分支的值。 就像 if 一样，每个分支都可以是一个块，它的值是块中最后一个表达式的值。

如果 when 用作表达式，则 else 分支是强制性的，除非编译器可以证明所有可能的情况都被分支条件覆盖，例如，枚举类条目和密封类子类型）。

在 when 语句中，else 分支在以下情况下是强制性的：
* when 有一个布尔型、枚举型或密封类型的主题，或其可为空的对应物。
* when 的分支并未涵盖该主题的所有可能情况。

要为多种情况定义共同行为，请将它们的条件用逗号组合在一行中。

可以使用任意表达式（不仅是常量）作为分支条件。

还可以检查一个值是否在一个范围或一个集合中或！。

另一种选择是检查一个值是或！是特定类型。 请注意，由于智能转换，可以访问类型的方法和属性而无需任何额外检查。

when 也可以用作 if-else if 链的替代品。 如果没有提供参数，则分支条件只是布尔表达式，当条件为真时执行分支。

可以使用以下语法在变量中捕获主题：
```kotlin
fun Request.getBody() =
    when (val response = executeRequest()) {
        is Success -> response.body
        is HttpError -> throw HttpException(response.status)
    }
```

### for 循环
for 循环遍历任何提供迭代器的东西。
```kotlin
for (item in collection) print(item)
```

for 遍历任何提供迭代器的东西。 这意味着它：
* 有一个返回 Iterator<> 的成员或扩展函数 iterator():
   * 有一个成员或扩展函数 next()
   * 有一个返回布尔值的成员或扩展函数 hasNext()

所有这三个函数都需要标记为 operator。

range 或数组上的 for 循环被编译为不创建迭代器对象的基于索引的循环。

如果想遍历一个带有索引的数组或列表，可以这样做：
```kotlin
for (i in array.indices) {
    println(array[i])
}
```
或者，可以使用 withIndex 库函数：
```kotlin
for ((index, value) in array.withIndex()) {
    println("the element at $index is $value")
}
```

### while 循环
while 和 do-while 循环在满足条件时连续执行它们的主体。


## return 和 跳转
Kotlin 有三种结构跳转表达式：
* return 默认情况下从最近的封闭函数或匿名函数返回。
* break 终止最近的封闭循环。
* continue 进入最近的封闭循环的下一步。

所有这些表达式都可以用作更大表达式的一部分：
```kotlin
val s = person.name ?: return
```
这些表达式的类型是 Nothing 类型。

### Break 和 continue 到 labels
Kotlin 中的任何表达式都可以用标签进行标记。 标签的形式为标识符后跟 @ 符号，例如 abc@ 或 fooBar@。 要标记表达式，只需在其前面添加一个标签。
```kotlin
loop@ for (i in 1..100) {
    for (j in 1..100) {
        if (...) break@loop
    }
}
```

### Return to label
在 Kotlin 中，可以使用函数字面量、局部函数和对象表达式来嵌套函数。 合格的 return 允许从外部函数返回。 最重要的用例是从 lambda 表达式返回。
```kotlin
fun foo() {
    listOf(1, 2, 3, 4, 5).forEach {
        if (it == 3) return // non-local return directly to the caller of foo()
        print(it)
    }
    println("this point is unreachable")
}
```
注意，只有传递给内联函数的 lambda 表达式才支持此类非本地返回。 要从 lambda 表达式返回，请将其标记并限定返回：
```kotlin
fun foo() {
    listOf(1, 2, 3, 4, 5).forEach lit@{
        if (it == 3) return@lit // local return to the caller of the lambda - the forEach loop
        print(it)
    }
    print(" done with explicit label")
}
```
通常使用隐式标签更方便，因为这样的标签与传递 lambda 的函数具有相同的名称。
```kotlin
fun foo() {
    listOf(1, 2, 3, 4, 5).forEach {
        if (it == 3) return@forEach // local return to the caller of the lambda - the forEach loop
        print(it)
    }
    print(" done with implicit label")
}
```
或者，可以将 lambda 表达式替换为匿名函数。 匿名函数中的 return 语句将从匿名函数本身返回。

请注意，在前面三个示例中使用本地返回类似于在常规循环中使用 `continue`。

break 没有直接的等价物，但可以通过添加另一个嵌套 lambda 并从其非本地返回来模拟它：
```kotlin
fun foo() {
    run loop@{
        listOf(1, 2, 3, 4, 5).forEach {
            if (it == 3) return@loop // non-local return from the lambda passed to run
            print(it)
        }
    }
    print(" done with nested loop")
}
```

当返回值时，解析器优先考虑合格的返回：
```kotlin
return@a 1
```
这意味着“在标签 @a 处返回 1”，而不是“返回标签表达式 (@a 1)”。


## 异常
Kotlin 中的所有异常类都继承了 Throwable 类。 每个异常都有一条消息、一个堆栈跟踪和一个可选的原因。

使用 `throw` 表达式抛出一个异常对象。使用 `try catch` 捕获异常。

#### try 是一个表达式
try 是一个表达式，这意味着它可以有一个返回值
try 表达式的返回值要么是 try 块中的最后一个表达式，要么是 catch 块中的最后一个表达式。 finally 块的内容不影响表达式的结果。

### Checked exception
Kotlin does not have checked exceptions.

如果想在从 Java、Swift 或 Objective-C 调用 Kotlin 代码时提醒调用者可能出现的异常，可以使用 `@Throws` 注释。

### Nothing 类型
`throw` 是 Kotlin 中的一个表达式，因此可以使用它，例如，作为 Elvis 表达式的一部分：
```kotlin
val s = person.name ?: throw IllegalArgumentException("Name required")
```
`throw` 表达式的类型为 `Nothing`。 此类型没有值，用于标记永远无法到达的代码位置。 在自己的代码中，可以使用 Nothing 来标记永远不会返回的函数。

当调用这个函数时，编译器会知道在调用之后执行不会继续。
```kotlin
val s = person.name ?: fail("Name required")
println(s)     // 's' is known to be initialized at this point
```

在处理类型推断时，也可能会遇到这种类型。 这种类型的可空变体 `Nothing?` 恰好有一个可能的值，即 `null`。 
如果使用 `null` 来初始化推断类型的值，并且没有其他信息可用于确定更具体的类型，编译器将推断 `Nothing?` 类型：
```kotlin
val x = null           // 'x' has type `Nothing?`
val l = listOf(null)   // 'l' has type `List<Nothing?>
```





