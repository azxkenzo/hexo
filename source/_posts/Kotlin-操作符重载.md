---
title: Kotlin - 操作符重载
date: 2022-05-27 07:52:02
tags: Kotlin
---

Kotlin 允许为类型上预定义的一组运算符提供自定义实现。这些运算符具有预定义的符号表示（如 + 或 *）和优先级。要实现运算符，请为相应类型提供具有特定名称的成员函数或扩展函数。这种类型成为二元运算的左侧类型和一元运算的参数类型。

要重载运算符，请使用运算符修饰符标记相应的函数：
``` kotlin
interface IndexedContainer {
    operator fun get(index: Int)
}
```

重写运算符重载时，可以省略运算符：
``` kotlin
class OrdersList: IndexedContainer {
    override fun get(index: Int) { /*...*/ }
}
```


### 一元运算

#### 一元前缀运算符
| 表达式        | 转换为               |
|------------|-------------------|
| +a         | a.unaryPlus()     |
| -a         | a.unaryMinus()    |
| !a         | a.not()           |

该表表明，当编译器处理例如表达式 +a 时，它会执行以下步骤：
* 确定a的类型，设为T
* 查找带有操作符修饰符且没有接收器 T 参数的函数 unaryPlus()，这意味着成员函数或扩展函数。
* 如果函数不存在或不明确，则为编译错误。
* 如果函数存在且其返回类型为 R，则表达式 +a 的类型为 R。

> 这些操作以及所有其他操作都针对基本类型进行了优化，并且不会为它们引入函数调用的开销。



### 二元运算

#### 算术运算符
| 表达式   | 转换为          |
|-------|--------------|
| a + b | a.plus(b)    |
| a - b | a.minus(b)   |
| a * b | a.times(b)   |
| a / b | a.div(b)     |
| a % b | a.rem(b)     |
| a..b  | a.rangeTo(b) |

#### in 操作符
| 表达式     | 转换为            |
|---------|----------------|
| a in b  | b.contains(a)  |
| a !in b | !b.contains(a) |

#### 索引访问运算符
| 表达式                    | 转换为                     |
|------------------------|-------------------------|
| a\[i\]                 | a.get(i)                |
| a\[i, j\]              | a.get(i, j)             |
| a\[i_1, ..., i_n\]     | a.get(i_1, ..., i_n)    |
| a\[i\] = b             | a.set(i, b)             |
| a\[i, j\] = b          | a.set(i, j, b)          |
| a\[i_1, ..., i_n\] = b | a.set(i_1, ..., i_n, b) |

#### invoke 操作符
| 表达式              | 转换为                     |
|------------------|-------------------------|
| a()              | a.invoke()              |
| a(i)             | a.invoke(i)             |
| a(i, j)          | a.invoke(i, j)          |
| a(i_1, ..., i_n) | a.invoke(i_1, ..., i_n) |

#### 增强赋值
| 表达式    | 转换为              |
|--------|------------------|
| a += b | a.plusAssign(b)  |
| a -= b | a.minusAssign(b) |
| a *= b | a.timesAssign(b) |
| a /= b | a.divAssign(b)   |
| a %= b | a.remAssign(b)   |

> 在 Kotlin 中，赋值不是表达式

#### 等式和不等式运算符
| 表达式    | 转换为                             |
|--------|---------------------------------|
| a == b | a?.equals(b) ?: (b === null)    |
| a != b | !(a?.equals(b) ?: (b === null)) |

这些运算符仅适用于函数 equals(other: Any?): Boolean，可以重写该函数以提供自定义相等检查实现。任何其他具有相同名称的函数（如 equals(other: Foo)）都不会被调用。

> === 和 !== （身份检查）不可重载，因此不存在它们的约定。

== 操作很特殊：它被转换为一个复杂的表达式，用于筛选空值。 null == null 始终为真，非 null x 的 x == null 始终为假，不会调用 x.equals()


#### 比较操作符
| 表达式    | 转换为                 |
|--------|---------------------|
| a > b  | a.compareTo(b) > 0  |
| a < b  | a.compareTo(b) < 0  |
| a >= b | a.compareTo(b) >= 0 |
| a <= b | a.compareTo(b) <= 0 |



## 命名函数的中缀调用
可以使用[中缀函数调用]()来模拟自定义中缀操作。

