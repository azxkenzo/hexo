---
title: Kotlin - 函数
date: 2022-11-23 17:00:55
tags: Kotlin
---

## 参数
重写方法总是使用基本方法的默认参数值。 重写具有默认参数值的方法时，必须从签名中省略默认参数值。

如果有默认值的参数在没有默认值的参数之前，则只能通过使用命名参数调用函数来使用默认值：
```
fun foo(
    bar: Int = 0,
    baz: Int,
) { /*...*/ }

foo(baz = 1) // The default value bar = 0 is used
```

如果默认参数之后的最后一个参数是 lambda，则可以将其作为命名参数或在括号外传递：
```
fun foo(
    bar: Int = 0,
    baz: Int = 1,
    qux: () -> Unit,
) { /*...*/ }

foo(1) { println("hello") }     // Uses the default value baz = 1
foo(qux = { println("hello") }) // Uses both default values bar = 0 and baz = 1
foo { println("hello") }        // Uses both default values bar = 0 and baz = 1
```

### 命名参数
可以在调用函数时命名一个或多个函数的参数。

当在函数调用中使用命名参数时，可以自由更改它们列出的顺序。如果想使用它们的默认值，可以将这些参数完全省略。

可以使用默认值跳过特定参数，而不是全部省略。 但是，在第一个跳过的参数之后，必须命名所有后续参数。

可以使用扩展运算符传递带有名称的可变数量的参数 (`vararg`)：
```
fun foo(vararg strings: String) { /*...*/ }

foo(strings = *arrayOf("a", "b", "c"))
```

在 JVM 上调用 Java 函数时，不能使用命名参数语法，因为 Java 字节码并不总是保留函数参数的名称。


## 返回 Unit 的函数
如果一个函数没有返回有用的值，它的返回类型是 `Unit`。 `Unit` 是一种只有一个值的类型——`Unit`。 不必显式返回此值。

## 单表达式函数
当函数返回单个表达式时，大括号可以省略，主体在 = 符号之后指定。

## 显式返回类型
具有块主体的函数必须始终明确指定返回类型，除非它们打算返回 Unit，在这种情况下指定返回类型是可选的。

Kotlin 不会为具有块主体的函数推断返回类型。

## 可变数量参数
使用 `vararg` 修饰符标记函数的参数（通常是最后一个）。

只有一个参数可以标记为可变参数。 如果可变参数不是列表中的最后一个参数，则可以使用命名参数语法传递后续参数的值，或者，如果参数具有函数类型，则通过在括号外传递 lambda。

调用可变参数函数时，可以单独传递参数，例如 `asList(1, 2, 3)`。 如果已经有一个数组并想将其内容传递给函数，请使用展开运算符（在数组前加上 `*`）：
```
val a = arrayOf(1, 2, 3)
val list = asList(-1, 0, *a, 4)
```

如果要将原始类型数组传递给可变参数，则需要使用 toTypedArray() 函数将其转换为常规（类型化）数组：
```
val a = intArrayOf(1, 2, 3) // IntArray is a primitive type array
val list = asList(-1, 0, *a.toTypedArray(), 4)
```

## 中缀符号
用 `infix` 关键字标记的函数也可以使用中缀表示法调用（省略调用的点和括号）。 中缀函数必须满足以下要求：
* 它们必须是成员函数或扩展函数。
* 它们必须有一个参数。
* 该参数不得接受可变数量的参数，并且必须没有默认值。

```
infix fun Int.shl(x: Int): Int { ... }

// calling the function using the infix notation
1 shl 2

// is the same as
1.shl(2)
```

中缀函数调用的优先级低于算术运算符、类型转换和 rangeTo 运算符。 以下表达式是等效的：
* `1 shl 2 + 3` is equivalent to `1 shl (2 + 3)`
* `0 until n * 2` is equivalent to `0 until (n * 2)`
* `xs union ys as Set<*>` is equivalent to `xs union (ys as Set<*>)`

另一方面，中缀函数调用的优先级高于布尔运算符 && 和 ||、is- 和 in-checks 以及其他一些运算符。 这些表达式也是等价的：
* `a && b xor c` is equivalent to `a && (b xor c)`
* `a xor b in c` is equivalent to `(a xor b) in c`


## 函数范围
Kotlin 函数可以在文件的顶层声明，这意味着不需要创建一个类来保存函数。除了顶级函数，Kotlin 函数还可以在本地声明为成员函数和扩展函数。

### 局部函数
局部函数可以访问外部函数（闭包）的局部变量。

### 尾递归函数
Kotlin 支持一种称为尾递归的函数式编程风格。 对于一些通常会使用循环的算法，可以使用递归函数来代替，而不会出现堆栈溢出的风险。 
当一个函数被标记为 `tailrec` 修饰符并满足所需的形式条件时，编译器会优化递归，留下一个快速高效的基于循环的版本：
```
val eps = 1E-10 // "good enough", could be 10^-15

tailrec fun findFixPoint(x: Double = 1.0): Double =
    if (Math.abs(x - Math.cos(x)) < eps) x else findFixPoint(Math.cos(x))
```

此代码计算余弦的定点，余弦是一个数学常数。 它只是从 1.0 开始重复调用 Math.cos，直到结果不再改变，针对指定的 eps 精度产生 0.7390851332151611 的结果。 生成的代码等同于这种更传统的风格：
```
val eps = 1E-10 // "good enough", could be 10^-15

private fun findFixPoint(): Double {
    var x = 1.0
    while (true) {
        val y = Math.cos(x)
        if (Math.abs(x - y) < eps) return x
        x = Math.cos(x)
    }
}
```

要符合 `tailrec` 修饰符的条件，函数必须在执行的最后一个操作时调用自身。 当在递归调用之后、在 try/catch/finally 块内或在打开的函数上有更多代码时，不能使用尾递归。


## 高阶函数和 lambda

### 函数类型
Kotlin 使用函数类型，例如 `(Int) -> String`，用于处理函数的声明：`val onClick: () -> Unit = ....` 。

这些类型有一个特殊的符号，对应于函数的签名——它们的参数和返回值：
* 所有函数类型都有一个带括号的参数类型列表和一个返回类型：`(A, B) -> C` 表示一种类型，该类型表示接受两个类型 A 和 B 的参数并返回类型 C 的值的函数。参数列表 types 可以为空，如 `() -> A`。Unit 返回类型不能省略。
* 函数类型可以选择有一个额外的接收者类型，它在符号中的点之前指定：类型 `A.(B) -> C` 表示可以在接收者对象 A 上调用带有参数 B 并返回值 C 的函数 . 带有接收者的函数字面量通常与这些类型一起使用。
* 挂起函数属于一种特殊的函数类型，在其符号中带有挂起修饰符，例如`suspend () -> Unit` 或`suspend A.(B) -> C`。

函数类型符号可以选择性地包含函数参数的名称：`(x: Int, y: Int) -> Point`。 这些名称可用于记录参数的含义。

要指定函数类型可为空，请使用括号，如下所示：`((Int, Int) -> Int)?` 。

也可以使用括号组合函数类型：`(Int) -> ((Int) -> Unit)`。

箭头符号是右结合的，`(Int) -> (Int) -> Unit` 等同于前面的示例，但不是 `((Int) -> (Int)) -> Unit`。

还可以使用类型别名为函数类型指定一个替代名称：`typealias ClickHandler = (Button, ClickEvent) -> Unit`

### 实例化函数类型
有几种方法可以获取函数类型的实例：
* 在函数字面量中使用代码块，采用以下形式之一：
   * lambda 表达式：`{ a, b -> a + b } `
   * 匿名函数：`fun(s: String): Int { return s.toIntOrNull() ?: 0 }`
* 使用对现有声明的 callable 引用：
   * 顶层、局部、成员或扩展函数: `::isOdd`, `String::toInt`,
   * 顶层、成员或扩展属性: `List<Int>::size`,
   * 构造器: `::Regex`
* 使用实现函数类型的自定义类的实例作为接口：
```
class IntTransformer: (Int) -> Int {
    override operator fun invoke(x: Int): Int = TODO()
}

val intFunction: (Int) -> Int = IntTransformer()
```

带接收者和不带接收者的函数类型的非字面量值是可以互换的，因此接收者可以代表第一个参数，反之亦然。 例如，类型为 `(A, B) -> C` 的值可以传递或分配给类型为 `A.(B) -> C` 的值，反之亦然.

### 调用函数类型实例
可以使用其 `invoke(...)` 运算符调用函数类型的值：`f.invoke(x)` 或仅 `f(x)`。

如果该值具有接收者类型，则接收者对象应作为第一个参数传递。 另一种调用带有接收者的函数类型的值的方法是在它前面加上接收者对象，就好像该值是一个扩展函数：1.foo(2)。

### lambda 表达式和匿名函数
Lambda 表达式和匿名函数是函数字面量。 函数字面量是未声明但作为表达式立即传递的函数。

### lambda 表达式语法
* lambda 表达式总是被大括号括起来。
* 完整句法形式的参数声明放在大括号内，并具有可选的类型注释。
* 主体在 `->` 之后。
* 如果 lambda 的推断返回类型不是 Unit，则 lambda 主体内的最后一个（或可能是单个）表达式将被视为返回值。

### 传递尾随 lambda
根据 Kotlin 约定，如果函数的最后一个参数是一个函数，那么作为相应参数传递的 lambda 表达式可以放在括号外

如果 lambda 是该调用中的唯一参数，则可以完全省略括号


## 内联函数
使用高阶函数会带来一定的运行时惩罚：每个函数都是一个对象，并且它捕获一个闭包。 闭包是可以在函数体中访问的变量范围。
内存分配（包括函数对象和类）和虚拟调用会引入运行时开销。

`inline` 修饰符***影响函数本身和传递给它的 lambda***：所有这些都将内联到调用站点。

### 非内联
如果不希望所有传递给内联函数的 lambda 都被内联，使用 `noinline` 修饰符标记那些函数参数。

可内联 lambda 只能在内联函数内部调用或作为可内联参数传递。 然而，noinline lambda 可以按照任何方式进行操作，包括存储在字段中或传递。

### 非局部返回
在 Kotlin 中，只能使用普通的、无限定的 `return` 来退出命名函数或匿名函数。 要退出 lambda，请使用***标签***。 在 lambda 中禁止直接 return，因为 lambda 不能使封闭函数返回。

但是如果 lambda 传递给的函数是内联的，那么返回值也可以是内联的。

此类return（位于 lambda 中，但退出封闭函数）称为非局部返回。 这种结构通常发生在循环中，内联函数通常包含在循环中

请注意，一些内联函数可能会调用作为参数传递给它们的 lambda 表达式，而不是直接从函数体中调用，而是从另一个执行上下文调用，例如局部对象或嵌套函数。
在这种情况下，lambda 中也不允许使用非本地控制流。 要指示内联函数的 lambda 参数不能使用非本地返回，请使用 `crossinline` 修饰符标记 lambda 参数.
```
inline fun f(crossinline body: () -> Unit) {
    val f = object: Runnable {
        override fun run() = body()
    }
    // ...
}
```

