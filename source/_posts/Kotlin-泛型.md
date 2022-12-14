---
title: Kotlin - 泛型
date: 2022-05-28 07:09:42
tags: Kotlin
---

[原文](https://kotlinlang.org/docs/generics.html)

Kotlin 中的类可以有类型参数，就像在 Java 中一样：
```
class Box<T>(t: T) {
    var value = t
}
```
要创建此类的实例，只需提供类型参数：
```
val box: Box<Int> = Box<Int>(1)
```
但是如果参数可以推断出来，例如，从构造函数参数，可以省略类型参数：
```
val box = Box(1) // 1 has type Int, so the compiler figures out that it is Box<Int>
```








## 差异
Java 类型系统最棘手的方面之一是通配符类型。 Kotlin 没有这些。相反，Kotlin 具有声明站点差异和类型预测。

思考一下为什么 Java 需要这些神秘的通配符。首先，Java 中的泛型类型是不变的，这意味着 `List<String>` 不是 `List<Object>` 的子类型。如果 List 不是不变的，它不会比 Java 的数组好，因为下面的代码会编译但在运行时会导致异常：
``` java
List<String> strs = new ArrayList<String>();
List<Object> objs = strs; // !!! A compile-time error here saves us from a runtime exception later.
objs.add(1); // Put an Integer into a list of Strings
String s = strs.get(0); // !!! ClassCastException: Cannot cast Integer to String
```

Java 禁止此类事情以保证运行时安全。但这有影响。例如，考虑 Collection 接口中的 addAll() 方法。这个方法的签名是什么？直观地说，会这样写：
``` java
interface Collection<E> ... {
    void addAll(Collection<E> items);
}
```
但是，将无法执行以下操作（这是非常安全的）：
``` java
void copyAll(Collection<Object> to, Collection<String> from) {
    to.addAll(from);
    // !!! Would not compile with the naive declaration of addAll:
    // Collection<String> is not a subtype of Collection<Object>
}
```
这就是为什么 addAll() 的实际签名如下：
``` java
interface Collection<E> ... {
    void addAll(Collection<? extends E> items);
}
```
通配符类型参数 `? extends E` 表示此方法接受 `E` 的对象集合或 `E` 的子类型的对象集合，而不仅仅是 `E` 本身。这意味着您可以安全地从 items 中读取 `E`（此集合的元素是 `E` 的子类的实例），但不能写入它，因为不知道哪些对象符合 `E` 的未知子类型。作为此限制的回报，会得到所需的行为： `Collection<String>` 是 `Collection<? extends Object>` 的子类型。换句话说，带有扩展绑定（上限）的通配符使类型 **协变**。

理解其工作原理的关键相当简单：如果只能从集合中获取 item，那么使用字符串集合并从中读取 object 就可以了。相反，如果只能将 item 放入集合中，则可以将 object 的集合放入其中：在 Java 中有 `List<? super String>`，是 `List<Object>` 的超类型。

后者称为 **逆变**，只能在 `List<? super String>` 上调用将 String 作为参数的方法。（例如，可以调用 add(String) 或 set(int, String)）。如果你在 List<T> 中调用返回 `T` 的东西，你不会得到一个字符串，而是一个 object。


### 声明站点差异
假设有一个通用接口 `Source<T>` 没有任何将 `T` 作为参数的方法，只有返回 `T` 的方法：
``` java
interface Source<T> {
    T nextT();
}
```
然后，将 `Source<String>` 实例的引用存储在 `Source<Object>` 类型的变量中是完全安全的 - 无需调用消费者方法。但是 Java 不知道这一点，并且仍然禁止它：
```java
void demo(Source<String> strs) {
    Source<Object> objects = strs; // !!! Not allowed in Java
    // ...
}
```
要解决此问题，应该声明 `Source<? extends Object>`。这样做是没有意义的，因为可以像以前一样在这样的变量上调用所有相同的方法，因此更复杂的类型不会增加任何值。但是编译器不知道这一点。

在 Kotlin 中，有一种方法可以向编译器解释这类事情。这称为 **声明站点差异**：可以对 Source 的类型参数 `T` 进行注释，以确保它仅从 `Source<T>` 的成员返回（产生），而不会被消费。为此，请使用 `out` 修饰符：
```kotlin
interface Source<out T> {
    fun nextT(): T
}

fun demo(strs: Source<String>) {
    val objects: Source<Any> = strs // This is OK, since T is an out-parameter
    // ...
}
```
一般规则是：当类 C 的类型参数 `T` 被声明为 out 时，它可能只出现在 C 的成员中的 out 位置，但作为回报，`C<Base>` 可以安全地成为 `C<Derived>` 的超类型。

换句话说，可以说类 C 在参数 `T` 中是 **协变** 的，或者说 `T` 是**协变**类型参数。可以将 C 视为 `T` 的生产者，而不是 `T` 的消费者。

`out` 修饰符称为差异注释，由于它是在类型参数声明站点提供的，因此它提供了声明站点差异。这与 Java 的使用站点差异形成对比，其中类型使用中的通配符使类型**协变**。

除了 `out`，Kotlin 还提供了一个互补的差异注解：`in`。它使类型参数 **逆变**，这意味着它只能被消费而不能被生产。**逆变**类型的一个很好的例子是 `Comparable`：
```kotlin
interface Comparable<in T> {
    operator fun compareTo(other: T): Int
}

fun demo(x: Comparable<Number>) {
    x.compareTo(1.0) // 1.0 has type Double, which is a subtype of Number
    // Thus, you can assign x to a variable of type Comparable<Double>
    val y: Comparable<Double> = x // OK!
}
```
`in` 和 `out` 这两个词似乎是不言自明的（因为它们已经在 C# 中成功使用了一段时间），所以上面提到的助记符并不是真正需要的。事实上，它可以在更高的抽象层次上重新表述：



## 类型预测
### 使用站点差异：类型预测
很容易将类型参数 `T` 声明为 out 并避免在使用站点上进行子类型化的麻烦，但实际上某些类不能被限制为只返回 `T`！ Array 就是一个很好的例子：
```kotlin
class Array<T>(val size: Int) {
    operator fun get(index: Int): T { ... }
    operator fun set(index: Int, value: T) { ... }
}
```
这个类在 `T` 中既不能是协变的，也不能是逆变的。这会带来一定的不灵活性。考虑以下函数：
```kotlin
fun copy(from: Array<Any>, to: Array<Any>) {
    assert(from.size == to.size)
    for (i in from.indices)
        to[i] = from[i]
}
```
此函数应该将item从一个数组复制到另一个数组。让我们尝试在实践中应用它：
```kotlin
val ints: Array<Int> = arrayOf(1, 2, 3)
val any = Array<Any>(3) { "" }
copy(ints, any)
//   ^ type is Array<Int> but Array<Any> was expected
```
在这里会遇到同样熟悉的问题：`Array<T>` 在 `T` 中是不变的，因此 `Array<Int>` 和 `Array<Any>` 都不是另一个的子类型。为什么不？同样，这是因为 copy 可能有意外的行为，例如，它可能会尝试将 `String` 写入 `from`，如果实际上在那里传递了一个 `Int` 数组，稍后将抛出 `ClassCastException`。

To prohibit the copy function from writing to from, you can do the following:
```kotlin
fun copy(from: Array<out Any>, to: Array<Any>) { ... }
```
这是类型预测，这意味着 `from` 不是一个简单的数组，而是一个受限（预测）数组。只能调用返回类型参数 `T` 的方法，这意味着只能调用 `get()`。这是我们使用站点差异的方法，它对应于 Java 的 `Array<? extends Object>`，同时稍微简单一些。

您也可以使用 `in` 预测类型：
```kotlin
fun fill(dest: Array<in String>, value: String) { ... }
```
`Array<in String>` 对应 Java 的 `Array<? super String>`。这意味着可以将 CharSequence 数组或 Object 数组传递给 `fill()` 函数。


### Star-projections
有时你想说你对类型参数一无所知，但你仍然想以一种安全的方式使用它。这里安全的方法是定义泛型类型的投影，该泛型类型的每个具体实例化都将是该投影的子类型。

Kotlin provides so-called star-projection syntax for this:
* 对于 `Foo<out T : TUpper>`，其中 `T` 是具有上限 `TUpper` 的**协变**类型参数，`Foo<*>` 等效于 `Foo<out TUpper>`。这意味着当 `T` 未知时，可以安全地从 `Foo<*>` 读取 `TUpper` 的值。
* 对于 `Foo<in T>`，其中 `T` 是**逆变**类型参数，`Foo<*>` 等效于 `Foo<in Nothing>`。这意味着当 `T` 未知时，无法以安全的方式向 `Foo<*>` 写入任何内容。
* 对于 `Foo<T : TUpper>`，其中 `T` 是具有上限 `TUpper` 的不变类型参数，`Foo<*>` 等效于读取值的 `Foo<out TUpper>` 和写入值的 `Foo<in Nothing>`。

如果泛型类型有多个类型参数，则每个类型参数都可以独立投影。例如，如果类型声明为接口 `Function<in T, out U>` 可以使用以下星形投影：
* `Function<*, String>` means `Function<in Nothing, String>`.
* `Function<Int, *>` means `Function<Int, out Any?>`.
* `Function<*, *>` means `Function<in Nothing, out Any?>`.



## 泛型函数
```kotlin
fun <T> singletonList(item: T): List<T> {
    // ...
}

fun <T> T.basicToString(): String { // extension function
    // ...
}
```



## 泛型约束
可以替代给定类型参数的所有可能类型的集合可能受到泛型约束的限制。

### 上界
最常见的约束类型是上界，它对应于 Java 的 extends 关键字：
```kotlin
fun <T : Comparable<T>> sort(list: List<T>) {  ... }
```
冒号后指定的类型为上界，表示只能用 `Comparable<T>` 的子类型替换 `T`。例如：
```kotlin
sort(listOf(1, 2, 3)) // OK. Int is a subtype of Comparable<Int>
sort(listOf(HashMap<Int, String>())) // Error: HashMap<Int, String> is not a subtype of Comparable<HashMap<Int, String>>
```
默认上限（如果没有指定）是 `Any?`。在尖括号内只能指定一个上界。如果同一类型参数需要多个上界，则需要单独的 where 子句：
```kotlin
fun <T> copyWhenGreater(list: List<T>, threshold: T): List<String>
    where T : CharSequence,
          T : Comparable<T> {
    return list.filter { it > threshold }.map { it.toString() }
}
```
传递的类型必须同时满足 `where` 子句的所有条件。在上面的例子中，`T` 类型必须同时实现 `CharSequence` 和 `Comparable`。



## 类型擦除
Kotlin 对泛型声明使用执行的类型安全检查是在编译时完成的。在运行时，泛型类型的实例不包含有关其实际类型参数的任何信息。类型信息已被擦除。例如，`Foo<Bar>` 和 `Foo<Baz?>` 的实例被擦除为 `Foo<*>`。

因此，没有通用的方法来检查泛型类型的实例是否在运行时使用某些类型参数创建，并且编译器禁止此类 `is`-检查。

无法在运行时检查到具有具体类型参数的泛型类型的类型转换，例如，作为 `List<String>` 的 foo。当高级程序逻辑暗示类型安全但编译器无法直接推断时，可以使用这些未经检查的强制转换。编译器对未检查的强制转换发出警告，并且在运行时，仅检查非泛型部分（相当于 `foo as List<*>`）。

泛型函数调用的类型参数也只在编译时检查。在函数体内，类型参数不能用于类型检查，并且类型转换为类型参数（`foo as T`）是未检查的。但是，`inline`函数的[reified类型参数](https://kotlinlang.org/docs/inline-functions.html#reified-type-parameters)在调用站点被内联函数体中的实际类型参数替换，因此可用于类型检查和强制转换，对泛型类型的实例具有与上述相同的限制。

