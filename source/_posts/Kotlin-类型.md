---
title: Kotlin - 类型
date: 2022-11-17 11:01:29
tags: Kotlin
---

## 数字
Kotlin 提供了一组内建类型来表示数字。

### 整型类型
对于整型数字，有四种类型，它们有不同的大小和取值范围：
* Byte  8bits  -128～127
* Short  16bits  -32768～32767
* Int  32bits
* Long  64bits

当初始化一个没有明确制定类型的变量时，编译器会自动推断具有足以表示该值的最小范围的类型。如果不超过 Int 的范围，则类型为 Int。如果超过，则类型为 Long。
要明确指定 Long 值，需要将后缀 L 附加到该值。显式指定类型会触发编译器检查值不超过指定类型的范围。

### 浮点型类型
对于实数，Kotlin 提供符合 IEEE 754 标准的浮点类型 Float 和 Double。

| 类型     | 大小  | 有效位 | 指数位 |
|--------|-----|-----|-----|
| Float  | 32  | 24  | 8   |
| Double | 64  | 53  | 11  |

使用具有小数部分的数字初始化 Double 和 Float 变量。对于用小数初始化的变量，编译器会推断出 Double 类型。

要显式指定值的 Float 类型，需要添加后缀 f 或 F。如果此类值包含超过 6-7 个十进制数字，则将对其进行四舍五入。

与其他一些语言不同，Kotlin 中没有隐式的数字扩展转换。 例如，具有 Double 参数的函数只能在 Double 值上调用，而不能在 Float、Int 或其他数值上调用。

### 数字的字面量常量
整数值有以下几种字面量常量：
* 十进制：123 / 123L
* 十六进制：0x0F
* 二进制：0b00001011

Kotlin 不支持八进制字面量。

Kotlin 还支持浮点数的传统表示法：
* Double：123.5, 123.5e10
* Float：123.5f / 123.5F

可以使用下划线使数字常量更具可读性：
```
val oneMillion = 1_000_000
val creditCardNumber = 1234_5678_9012_3456L
val socialSecurityNumber = 999_99_9999L
val hexBytes = 0xFF_EC_DE_5E
val bytes = 0b11010010_01101001_10010100_10010010
```

### JVM 上的数字表示
在 JVM 平台上，数字存储为原始类型：int、double 等。 例外情况是创建可为空的数字引用（例如 Int? 或使用泛型。 在这些情况下，数字被装在 Java 类 Integer、Double 等中。

```
val a: Int = 100
val boxedA: Int? = a
val anotherBoxedA: Int? = a

val b: Int = 10000
val boxedB: Int? = b
val anotherBoxedB: Int? = b

println(boxedA === anotherBoxedA) // true
println(boxedB === anotherBoxedB) // false
```
由于 JVM 对 -128 到 127 之间的整数进行了内存优化，所有对 a 的可空引用实际上都是同一个对象。它不适用于 b 引用，因此它们是不同的对象。

### 显式数字转换
由于表示方式不同，较小的类型不是较大类型的子类型。如果是这样，将遇到以下问题：
```
// Hypothetical code, does not actually compile:
val a: Int? = 1 // A boxed Int (java.lang.Integer)
val b: Long? = a // Implicit conversion yields a boxed Long (java.lang.Long)
print(b == a) // Surprise! This prints "false" as Long's equals() checks whether the other is Long as well
```

因此，较小的类型不会隐式转换为较大的类型。 这意味着将 Byte 类型的值分配给 Int 变量需要显式转换。

在许多情况下，不需要显式转换，因为类型是从上下文中推断出来的，并且算术运算会为适当的转换而重载，例如:
```
val l = 1L + 3 // Long + Int => Long
```

### 数字运算

#### 整数除法
整数之间的除法总是返回一个整数。 任何小数部分都将被丢弃。
```
val x = 5 / 2
//println(x == 2.5) // ERROR: Operator '==' cannot be applied to 'Int' and 'Double'
println(x == 2)
```

要返回浮点类型，需要将其中一个参数显式转换为浮点类型。

#### 位运算
Kotlin 提供了一组对整数的按位运算。 它们直接使用数字表示的位在二进制级别上进行操作。 按位运算由可以以中缀形式调用的函数表示。 它们只能应用于 Int 和 Long：
```
val x = (1 shl 2) and 0x000FF000
```

位运算的完整列表：
shl(bits) – signed shift left
* shr(bits) – signed shift right
* ushr(bits) – unsigned shift right
* and(bits) – bitwise AND
* or(bits) – bitwise OR
* xor(bits) – bitwise XOR
* inv() – 位反转

#### 浮点型数字比较
这里讨论的浮点数运算是：
* Equality checks: a == b and a != b
* Comparison operators: a < b, a > b, a <= b, a >= b
* Range instantiation and range checks: a..b, x in a..b, x !in a..b

当操作数 a 和 b 静态已知为 Float 或 Double 或其可为空的对应物（类型已声明或推断或是智能转换的结果）时，对数字的操作和它们形成的范围遵循 IEEE 754 浮点算术标准。

但是，为了支持通用用例并提供总排序，当操作数不是静态类型为浮点数（例如，Any、Comparable<...>、类型参数）时，操作使用 equals 和 compareTo 实现 Float 和 Double 不符合标准，因此：
* NaN 被认为等于它自己
* NaN 被认为大于任何其他元素，包括 POSITIVE_INFINITY
* -0.0 被认为小于 0.0


## 布尔型
Boolean 类型表示可以有两个值的布尔对象：true 和 false。

布尔值的内置操作包括：|| / && / !

|| 和 && 是懒惰执行的。

在 JVM 上：对布尔对象的可为空引用与数字类似地被装箱。

## 字符
字符由 Char 类型表示。 字符字面量放在单引号中：'1'。

特殊字符从转义的反斜杠 \ 开始。 支持以下转义序列：
* \t – tab
* \b – backspace
* \n – new line (LF)
* \r – carriage return (CR)
* \' – single quotation mark
* \" – double quotation mark
* \\ – backslash
* \$ – dollar sign

要编码任何其他字符，请使用 Unicode 转义序列语法：'\uFF00'。

如果字符变量的值是数字，则可以使用 digitToInt() 函数将其显式转换为 Int 数字。

在 JVM 上：与数字一样，当需要可空引用时，字符会被装箱。 装箱操作不保留身份。

## 字符串
Kotlin 中的字符串由 String 类型表示。 通常，字符串值是双引号 (") 中的字符序列。

字符串的元素是可以通过索引操作访问的字符：`s[i]`。 可以使用 for 循环遍历这些字符。

字符串是不可变的。 初始化字符串后，将无法更改其值或为其分配新值。 所有转换字符串的操作都在一个新的 String 对象中返回它们的结果，而原始字符串保持不变。

要连接字符串，使用 + 运算符。 这也适用于将字符串与其他类型的值连接，只要表达式中的第一个元素是字符串。

### 字符串字面量
Kotlin 有两种类型的字符串字面量：
* 转义字符串
* 原始字符串

#### 转义字符串
转义字符串可以包含转义字符。
```
val s = "Hello, world!\n"
```

#### 原始字符串
原始字符串可以包含换行符和任意文本。 它由三引号 (""") 分隔，不包含转义，并且可以包含换行符和任何其他字符：
```
val text = """
    for (c in "foo")
        print(c)
"""
```
要从原始字符串中删除前导空格，使用 `trimMargin()` 函数。

默认情况下，管道符号 | 用作边距前缀，但可以选择另一个字符并将其作为参数传递，例如 trimMargin(">")。

### 字符串模版
字符串文字可能包含模板表达式——被评估的代码片段，其结果被连接到字符串中。 模板表达式以美元符号 ($) 开头，由以下任一名称组成。

可以在原始字符串和转义字符串中使用模板。 要在允许作为标识符开头的任何符号之前在原始字符串（不支持反斜杠转义）中插入美元符号 $，请使用以下语法：
```
val price = """
${'$'}_9.99
"""
```


## 数组
Kotlin 中的数组由 Array 类表示。 它具有 get() 和 set() 函数，这些函数通过运算符重载约定变成 [] 和 size 属性，以及其他有用的成员函数：
```
class Array<T> private constructor() {
    val size: Int
    operator fun get(index: Int): T
    operator fun set(index: Int, value: T): Unit

    operator fun iterator(): Iterator<T>
    // ...
}
```
要创建一个数组，请使用函数 `arrayOf()` 并将项目值传递给它，以便 `arrayOf(1, 2, 3)` 创建一个数组 `[1, 2, 3]`。 或者，`arrayOfNulls()` 函数可用于创建一个给定大小的数组，其中填充了空元素。

另一种选择是使用 Array 构造函数，该构造函数采用数组大小和返回给定索引的数组元素值的函数。

Kotlin 中的数组是不变的。 这意味着 Kotlin 不允许将 Array<String> 分配给 Array<Any>，这可以防止可能的运行时故障（但可以使用 Array<out Any>）。

### 原始类型数组
Kotlin 还具有表示基本类型数组而无需装箱开销的类：`ByteArray`、`ShortArray`、`IntArray` 等。 这些类与 Array 类没有继承关系，但它们具有相同的方法和属性集。 它们中的每一个也都有对应的工厂函数。


## 无符号整型类型
除了整数类型，Kotlin 还为无符号整数提供了以下类型：
* UByte: an unsigned 8-bit integer, ranges from 0 to 255
* UShort: an unsigned 16-bit integer, ranges from 0 to 65535
* UInt: an unsigned 32-bit integer, ranges from 0 to 2^32 - 1
* ULong: an unsigned 64-bit integer, ranges from 0 to 2^64 - 1

无符号类型支持其有符号对应对象的大部分操作。

无符号数字被实现为内联类，具有相同宽度的相应有符号对应类型的单个存储属性。 然而，将类型从无符号类型更改为有符号对应（反之亦然）是二进制不兼容的更改。

### 无符号数组和区间
无符号数组和对它们的操作处于测试阶段。 它们可以随时进行不兼容的更改。 需要选择加入。

与原始类型相同，每个无符号类型都有对应的类型，表示该类型的数组：
* UByteArray: an array of unsigned bytes
* UShortArray: an array of unsigned shorts
* UIntArray: an array of unsigned ints
* ULongArray: an array of unsigned longs

UIntRange、UIntProgression、ULongRange 和 ULongProgression 类支持 UInt 和 ULong 的 range 和级数。 与无符号整数类型一起，这些类是稳定的。

### 无符号整型字面量
为了使无符号整数更易于使用，Kotlin 提供了使用表示特定无符号类型的后缀标记整数文字的功能（类似于 Float 或 Long）：
* u 和 U 标记用于无符号文字。 确切的类型是根据预期的类型确定的。 如果没有提供预期的类型，编译器将根据文字的大小使用 UInt 或 ULong：
* uL 和 UL 将文字显式标记为 unsigned long：

### 使用案例
无符号数的主要用例是利用整数的整个位范围来表示正值。
例如，要表示不适合有符号类型的十六进制常量，例如 32 位 AARRGGBB 格式的颜色：
```
data class Color(val representation: UInt)

val yellow = Color(0xFFCC00CCu)
```

可以使用无符号数来初始化字节数组，而无需显式 `toByte()` 字面量转换：
```
val byteOrderMarkUtf8 = ubyteArrayOf(0xEFu, 0xBBu, 0xBFu)
```

另一个用例是与native API 的互操作性。 Kotlin 允许表示在签名中包含无符号类型的 native 声明。 映射不会用有符号整数替换无符号整数，保持语义不变。

## 类型检查和转换

### is 和 !is 操作符
使用 `is` 运算符或其否定形式 `!is` 执行运行时检查，以识别对象是否符合给定类型。

### 智能转换
在大多数情况下，不需要在 Kotlin 中使用显式转换运算符，因为编译器会跟踪**不可变值**的 `is` 检查和显式转换，并在必要时自动插入（安全）转换。

编译器足够聪明，知道如果否定检查导致返回，则强制转换是安全的：
```
if (x !is String) return

print(x.length) // x is automatically cast to String
```

或者如果它在 && 或 || 的右侧 正确的检查（常规或否定）在左侧。

智能转换也适用于 when 表达式和 while 循环。

需要注意的是，只有当编译器可以保证变量在检查和使用之间不会改变时，智能转换才起作用。 更具体地说，智能转换可以在以下条件下使用：
* `val` local variables - always, with the exception of local delegated properties.
* `val` properties - 如果属性是 private 或 internal，或者检查是在声明属性的同一模块中执行的。 智能转换不能用于 open 属性或具有自定义 getter 的属性。
* `var` local variables - if the variable is not modified between the check and the usage, is not captured in a lambda that modifies it, and is not a local delegated property.
* `var` properties - never, because the variable can be modified at any time by other code.

### 不安全的转换操作符
通常，如果无法进行强制转换，则强制转换运算符会引发异常。 因此，它被称为不安全。 Kotlin 中的不安全强制转换由中缀运算符 `as` 完成。

注意，不能将 null 强制转换为 String，因为此类型不可为 null。 如果 y 为 null，则上面的代码将引发异常。 要使这样的代码对空值正确，请使用强制转换右侧的可为空类型。

### 安全(可空)的转换操作符
为避免异常，请使用安全转换运算符 `as?`，它在失败时返回 null。