---
title: Kotlin - 面向对象
date: 2022-11-02 11:24:18
tags: Kotlin
---

在 Java 中，子类的构造方法实现中必须调用父类的构造方法。有以下规则：
* 如果父类有无参构造方法，则子类构造方法会隐式调用父类的无参构造方法
* 如果父类没有无参构造方法，则子类构造方法必须显式调用父类构造方法或子类其他构造方法（来间接调用父类构造方法）
* 对其他构造方法的调用必须放在方法体的第一行


在 Kotlin 中，如果一个类存在主构造器，则每个次构造器都要直接或间接委托给主构造器。
* 如果子类有主构造器，则必须调用父类的主构造器或次构造器进行初始化
* 如果子类没有主构造器，则子类的次构造器必须直接或间接委托给父类的主或次构造器进行初始化


## 类
类声明由类名、类头（指定其泛型、主构造器和其他一些内容）和用花括号括起来的类主体组成。 类头和类主体都是可选的； 如果类没有主体，则可以省略花括号。

### 构造器
Kotlin 中的类可以有一个主构造器和一个或多个次构造器。 主构造器是类头的一部分，位于类名和可选泛型之后。

如果主构造器没有任何注解或可见性修饰符，则 `constructor` 关键字可以省略。

主构造器不能包含任何代码。 初始化代码可以放在以 `init` 关键字为前缀的 initializer 块中。

在实例初始化期间，initializer 块按照它们在类主体中出现的顺序执行，并与属性 initializer 交错。

主构造器参数可以在 initializer 块中使用。 它们也可以用在类主体中声明的属性 initializer 中。

Kotlin 有一个简洁的语法来声明属性并从主构造器初始化它们。此类声明还可以包括类属性的默认值。

如果主构造器有注释或可见性修饰符，则 `constructor` 关键字是必需的，并且修饰符位于它之前。

主构造器的参数允许使用val或var声明，而在次构造器中，这是不允许的。

#### 次构造器
一个类也可以声明次构造器，以 `constructor` 为前缀。

如果类具有主构造器，则每个次构造器都需要直接或间接通过另一个次构造器委托给主构造器。 使用 `this` 关键字完成对同一类的另一个构造器的委托。

initializer 块中的代码有效地成为主构造器的一部分。 对主构造器的委托作为次构造器的第一条语句发生，因此所有 initializer 块和属性 initializer 中的代码都在次构造器的主体之前执行。

即使类没有主构造器，委托仍然隐式发生，initializer 块仍然执行。

**_如果一个非抽象类没有声明任何构造器（主要的或次要的），它将有一个生成的不带参数的主构造器。 构造器的可见性将是 public。_**

在 JVM 上，如果所有主构造器参数都有默认值，编译器将生成一个额外的无参数构造器，它将使用默认值。

### 创建类实例
要创建类的实例，像调用常规函数一样调用构造函数。

### 类成员
类可以包含：
* Constructors and initializer blocks
* Functions
* Properties
* Nested and inner classes
* Object declarations

### 抽象类
一个类可以连同它的部分或全部成员一起被声明为 `abstract`。 抽象成员在其类中没有实现。 不需要使用 `open` 注释抽象类或函数。

抽象类的主构造器参数不能被声明为 `abstract`。

可以用 abstract 的开放成员重写非抽象的 open 成员。
```
open class Polygon {
    open fun draw() {
        // some default polygon drawing method
    }
}

abstract class WildShape : Polygon() {
    // Classes that inherit WildShape need to provide their own
    // draw method instead of using the default on Polygon
    abstract override fun draw()
}
```

## 继承
Kotlin 中的所有类都有一个共同的超类 `Any`，它是没有声明超类型的类的默认超类。

`Any` 具有三个方法：`equals()`、`hashCode()` 和 `toString()`。 因此，这些方法是为所有 Kotlin 类定义的。

默认情况下，Kotlin 类是 final 的——它们不能被继承。 要使类可继承，请使用 `open` 关键字对其进行标记。

要声明显式超类型，请将类型放在类头中的冒号之后。

如果派生类具有主构造器，则可以（并且必须）根据其参数在该主构造器中初始化基类。

如果派生类没有主构造器，则每个次构造器都必须使用 `super` 关键字初始化基类，或者它必须委托给另一个构造器。 
请注意，在这种情况下，不同的次构造器可以调用基类型的不同构造器。

### 重写方法
Kotlin 中，`override` 修饰符不能省略。

标记为 `override` 的成员本身是 open 的，因此它可以在子类中被重写。 如果要禁止重新重写，使用 `final` 修饰该成员。

### 重写属性
重写机制对属性的工作方式与对方法的工作方式相同。 在超类上声明的属性，然后在派生类上重新声明，必须以 `override` 开头，并且它们必须具有兼容的类型。 
每个声明的属性都可以被带有初始化器的属性或带有 get 方法的属性覆盖。

可以使用 var 属性覆盖 val 属性，但反之则不行。 这是允许的，因为 val 属性本质上声明了一个 get 方法，并且将其覆盖为 var 还会在派生类中另外声明一个 set 方法。

可以在主构造器中使用 `override` 关键字作为属性声明的一部分。

### 派生类初始化顺序
在派生类的新实例的构造过程中，基类初始化作为第一步完成（之前仅对基类构造函数的参数求值），这意味着它发生在派生类的初始化逻辑之前。

这意味着在执行基类构造函数时，派生类中声明或覆盖的属性尚未初始化。 在基类初始化逻辑中使用任何这些属性（直接或间接通过另一个重写的开放成员实现）可能会导致不正确的行为或运行时故障。 
因此，在设计基类时，应避免在构造函数、属性初始值设定项或 init 块中使用 open 成员。

### 调用超类实现
在内部类中，访问外部类的超类是使用外部类名限定的 `super` 关键字完成的：super@Outer_Name。

### 重写规则
在 Kotlin 中，实现继承受以下规则的约束：如果一个类从其直接超类继承同一成员的**多个实现**，它必须覆盖该成员并提供自己的实现（可能使用继承的一个）。

要表示继承实现的超类型，请使用尖括号中超类型名称限定的 `super`，例如 `super<Base>`。


## 属性

### getter 和 setter
声明属性的完整语法如下所示：
```
var <propertyName>[: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]
```
initializer、getter 和 setter 是可选的。

只读属性自定义 getter 时不允许有 initializer，原因是只读属性没有后端字段。

如果需要注释访问器或更改其可见性，但不需要更改默认实现，则可以定义访问器而不定义其主体。

### 后端字段
在 Kotlin 中，field 仅用作属性的一部分，以将其值保存在内存中。 field 不能直接声明。 然而，当一个属性需要一个后端字段时，Kotlin 会自动提供它。 
可以使用 `field` 标识符在访问器中引用此后端字段：
```
var counter = 0 // the initializer assigns the backing field directly
    set(value) {
        if (value >= 0)
            field = value
            // counter = value // ERROR StackOverflow: Using actual name 'counter' would make setter recursive
    }
```
`field` 标识符只能在属性访问器中使用。

如果属性使用至少一个访问器的默认实现，或者自定义访问器通过 `field` 标识符引用它，则将为属性生成后端字段。

### 后端属性
如果想做一些不适合这个隐式后端字段方案的事情，总是可以回退到拥有一个后端属性：
```
private var _table: Map<String, Int>? = null
public val table: Map<String, Int>
    get() {
        if (_table == null) {
            _table = HashMap() // Type parameters are inferred
        }
        return _table ?: throw AssertionError("Set to null by another thread")
    }
```

在 JVM 上：对使用默认 getter 和 setter 的私有属性的访问进行了优化，以避免函数调用开销。

### 编译期常量
如果只读属性的值在编译期已知，则可以使用 const 修饰符将其标记为编译期常量。这样的属性需要满足以下要求：
* 它必须是顶层属性或者 object 声明或伴生对象的成员
* 必须使用 String 类型或原始类型的值对其进行初始化
* 不能有自定义 getter

编译器将内联常量的用法，用它的实际值替换对常量的引用。 但是，该字段不会被移除，因此可以使用反射进行交互。

### 延迟初始化属性和变量
通常，声明为非空类型的属性必须在构造器中被初始化。在一些情况中，无法提供非空的 initializer，但是又仍然想避免在类主体中引用该属性时进行判空，这时可以使用 lateinit 修饰符标记该属性。

此修饰符可用于在类主体内声明的 var 属性（不在主构造器中，并且仅当属性没有自定义 getter 或 setter 时），以及顶层属性和局部变量。 属性或变量的类型必须是非空的，并且不能是原始类型。

在初始化之前访问 lateinit 属性会引发一个特殊异常。

要检查是否已初始化 `lateinit var`，请在对该属性的引用上使用 `.isInitialized`。此检查仅适用于在同一类型、外部类型之一或同一文件的顶层声明时可通过词法访问的属性。

## 接口
在 Kotlin 中，接口既可以包含抽象方法的声明，又可以包含方法实现。它们与抽象类的不同之处在于接口不能存储状态。 它们可以具有属性，但这些属性需要是抽象的或提供访问器实现。

类或者 object 可以实现一个或多个接口。

### 接口中的属性
可以在接口中声明属性。 在接口中声明的属性可以是抽象的，也可以为访问器提供实现。 **_在接口中声明的属性不能有后端字段_**，因此在接口中声明的访问器不能引用它们。

### 解决重写冲突
当在超类型列表中声明许多类型时，可能会继承多个相同方法的实现，此时派生类需要重写继承的相同方法。

## 函数式接口
只有一个抽象方法的接口称为函数式接口或单一抽象方法 (SAM) 接口。 函数式接口可以有多个非抽象成员，但只有一个抽象成员。

在 kotlin 中，使用 `fun` 修饰符声明函数式接口。

函数式接口不能有抽象属性。

从 1.6.20 开始，Kotlin 支持对函数式接口构造函数的可调用引用，这增加了一种源兼容的方式来从具有构造函数的接口迁移到函数式接口。

### SAM 转换
对于函数式接口，可以使用 SAM 转换，通过使用 lambda 表达式使代码更加简洁和易读。

可以使用 lambda 表达式，而不是手动创建实现函数式接口的类。 通过 SAM 转换，Kotlin 可以将任何签名与接口的单个方法的签名匹配的 lambda 表达式转换为代码，从而动态实例化接口实现。
```
// Creating an instance of a class
val isEven = object : IntPredicate {
   override fun accept(i: Int): Boolean {
       return i % 2 == 0
   }
}

// Creating an instance using lambda
val isEven = IntPredicate { it % 2 == 0 }
```



### 函数式接口 VS typealias
```
typealias IntPredicate = (i: Int) -> Boolean

val isEven: IntPredicate = { it % 2 == 0 }

fun main() {
   println("Is 7 even? - ${isEven(7)}")
}
```
函数式接口和类型别名有不同的用途。 类型别名只是现有类型的名称——它们不会创建新类型，而函数式接口会。 
可以提供特定于特定函数式接口的扩展，使其不适用于普通函数或其类型别名。

类型别名只能有一个成员，而函数式接口可以有多个非抽象成员和一个抽象成员。 函数式接口也可以实现和扩展其他接口。

函数式接口比类型别名更灵活并提供更多功能，但它们在语法和运行时的成本更高，因为它们可能需要转换为特定接口。 当选择在代码中使用哪一个时，请考虑需求：
* 如果 API 需要接受具有某些特定参数和返回类型的函数（任何函数） - 使用简单的函数类型或定义类型别名来为相应的函数类型提供更短的名称。
* 如果 API 接受一个比函数更复杂的实体——例如，它有非平凡的常量和/或不能在函数类型的签名中表达的操作——为它声明一个单独的函数接口。


## 可见性修饰符
类、object、接口、构造器、函数，还有属性和其 setter 可以有可见性修饰符。getter 总是和其属性有相同的可见性。

kotlin 中有四种可见性修饰符：`private`, `protected`, `internal`, 和 `public`。默认可见性是 `public`。

Java 可见性修饰符：public、protected(包内可见、子类可见)、default(包内可见、不同包的子类中不可见)、private。

### 包
函数, 属性, 类, object, 和 接口 在包内可以直接声明为顶层的。

* `public` 是默认的。
* 如果标记为 `private`, 则只能在包含该声明的文件内可见。
* 如果标记为 `internal`, 则在同一模块内可见。
* `protected` 修饰符对顶层声明不可用。

### 类成员
* `private` 表明该成员只能在这个类里可见。
* `protected` 表明该成员与 private 有相同的可见性，但是在子类中也可见。
* `internal` 表明在这个模块中看到声明类的任何客户端都会看到它的 `internal` 成员。
* `public` 表明看到声明类的任何客户端都会看到它的 `public` 成员。

在 Kotlin 中，外部类看不到内部类的 private 成员，但是在 Java 中能看到。

如果重写 protected 或 internal 成员并且没有明确指明可见性，则重写的成员的将有与原始相同的可见性。

### 局部声明
局部的变量、函数和类不能有可见性修饰符。

### 模块
internal 可见性修饰符意味着该成员在同一模块中是可见的。 更具体地说，模块是一组编译在一起的 Kotlin 文件，例如：
* An IntelliJ IDEA module.
* A Maven project.
* A Gradle source set (with the exception that the test source set can access the internal declarations of main).
* A set of files compiled with one invocation of the <kotlinc> Ant task.



## 扩展
Kotlin 提供了使用新功能扩展类或接口的能力，而无需从类继承或使用 Decorator 等设计模式。 这是通过称为扩展的特殊声明完成的。

### 扩展函数
要声明扩展函数，需要在其名称前加上 receiver 类型，它指的是被扩展的类型。
```
fun MutableList<Int>.swap(index1: Int, index2: Int) {
    val tmp = this[index1] // 'this' corresponds to the list
    this[index1] = this[index2]
    this[index2] = tmp
}
```
扩展函数中的 `this` 关键字对应于 receiver 对象（在(.)之前传递的对象）。 

### 扩展是静态解析的
扩展实际上并不修改它们扩展的类。 通过定义扩展，不会将新成员插入到类中，而只是使新函数可以使用这种类型的变量的点符号调用。

扩展函数是静态分派的，这意味着它们不是按 receiver 类型虚拟的。 被调用的扩展函数由调用函数的表达式的类型决定，而不是由在运行时计算该表达式的结果类型决定。 例如：
```
open class Shape
class Rectangle: Shape()

fun Shape.getName() = "Shape"
fun Rectangle.getName() = "Rectangle"

fun printClassName(s: Shape) {
    println(s.getName())
}

printClassName(Rectangle())
```
这个例子打印了 Shape，因为调用的扩展函数只依赖于参数 `s` 的声明类型，也就是 `Shape` 类。

如果一个类有一个成员函数，并且定义了一个具有相同 receiver 类型、相同名称并且适用于给定参数的扩展函数，则该成员总是优先。

### 可空 receiver
注意，可以使用可为空的 receiver 类型定义扩展。 即使对象变量的值为 null，也可以在对象变量上调用这些扩展，并且它们可以在主体内检查 `this == null`。
```
fun Any?.toString(): String {
    if (this == null) return "null"
    // after the null check, 'this' is autocast to a non-null type, so the toString() below
    // resolves to the member function of the Any class
    return toString()
}
```

### 扩展属性
```
val <T> List<T>.lastIndex: Int
    get() = size - 1
```
由于扩展实际上并不将成员插入到类中，因此扩展属性没有有效的方法来拥有后端字段。 这就是扩展属性不允许使用 initializer 的原因。 
它们的行为只能通过显式提供 getter/setter 来定义。

### 伴生对象扩展
如果一个类定义了伴随对象，还可以为伴随对象定义扩展函数和属性。 就像伴生对象的常规成员一样，它们可以仅使用类名作为限定符来调用：
```
class MyClass {
    companion object { }  // will be called "Companion"
}

fun MyClass.Companion.printCompanion() { println("companion") }

fun main() {
    MyClass.printCompanion()
}
```

### 把扩展声明为成员
可以在另一个类中声明一个类的扩展。 在这样的扩展中，有多个隐式 receiver - 其成员可以在没有限定符的情况下访问的对象。 
声明扩展的类的实例称为 dispatch receiver，扩展方法的接收器类型的实例称为 extension receiver。

如果 dispatch receiver 和 extension receiver 的成员之间发生名称冲突，则 extension receiver 优先。 要引用 dispatch receiver 的成员，可以使用限定的 this 语法。

声明为成员的扩展可以在子类中声明为 open 和重写。 这意味着***此类函数的调度对于 dispatch receiver 类型是虚拟的，但对于 extension receiver 类型是静态的***。


扩展使用与在同一范围内声明的常规函数相同的可见性修饰符。 例如：
* 在文件顶层声明的扩展可以访问同一文件中的其他私有顶层声明。
* 如果扩展在其接收者类型之外声明，则它不能访问接收者的 private 或 protected 成员。



## Data class
编译器自动从主构造器中声明的所有属性派生以下成员：
* `equals()`/`hashCode()` pair
* `toString()` of the form "User(name=John, age=42)"
* `componentN()` functions corresponding to the properties in their order of declaration.
* `copy()` function (see below).

为了确保生成代码的一致性和有意义的行为，数据类必须满足以下要求：
* 主构造器至少需要一个参数。
* 所有主构造器参数都需要标记为 val 或 var。
* 数据类不能是 abstract, open, sealed, or inner。

此外，数据类成员的生成遵循以下关于成员继承的规则：
* 如果数据类主体中有`equals()`、`hashCode()` 或`toString()` 的显式实现或超类中的最终实现，则不会生成这些函数，而是使用现有的实现。
* 如果超类型具有 `open` 并返回兼容类型的 `componentN()` 函数，则会为数据类生成相应的函数并覆盖超类型的函数。 如果超类型的函数由于不兼容的签名或由于它们是最终的而无法覆盖，则会报告错误。
* 不允许为 `componentN()` 和 `copy()` 函数提供显式实现。

数据类可以扩展其他类。

### 声明在类主体中的参数
编译器仅将主构造器中定义的属性用于自动生成的函数。 要从生成的实现中排除属性，请在类主体中声明它。

### copy
使用 `copy()` 函数复制对象，允许更改其某些属性，同时保持其余属性不变。

### 数据类和解构声明
为数据类生成的 Component 函数可以在解构声明中使用它们：
```
val jane = User("Jane", 35)
val (name, age) = jane
println("$name, $age years of age") // prints "Jane, 35 years of age"
```


## Sealed classes
密封的类和接口表示受限制的类层次结构，可提供对继承的更多控制。 密封类的所有**直接子类**在编译时都是已知的。 任何其他子类都不能出现在定义了密封类的模块之外。 
例如，第三方客户端无法在其代码中扩展密封类。 因此，密封类的每个实例都有一个来自有限集合的类型，该类型在编译此类时是已知的。

密封接口及其实现也是如此：一旦编译了具有密封接口的模块，就不会出现新的实现。

在某种意义上，密封类类似于枚举类：枚举类型的值集也受到限制，但每个枚举常量仅作为单个实例存在，而密封类的子类可以有多个实例，每个实例都有自己的状态。

例如，考虑一个库的 API。 它可能包含错误类，让库用户处理它可能抛出的错误。 如果此类错误类的层次结构包括在公共 API 中可见的接口或抽象类，则没有什么能阻止在客户端代码中实现或扩展它们。 但是，该库不知道在其外部声明的错误，因此它不能与它自己的类一致地对待它们。 使用密封的错误类层次结构，库作者可以确保他们知道所有可能的错误类型，并且以后不会出现其他错误类型。

要声明一个密封的类或接口，将 sealed 修饰符放在其名称之前。

密封类本身是抽象的，不能直接实例化，可以有抽象成员。

密封类的构造器可以具有以下两种可见性之一：protected（默认）或 private。

### 直接子类的位置
密封类和接口的直接子类必须在同一个包中声明。 它们可以是顶层的，也可以嵌套在任意数量的其他命名类、命名接口或命名对象中。 
子类可以具有任何可见性，只要它们与 Kotlin 中的正常继承规则兼容。

密封类的子类必须具有适当的限定名称。 它们不能是本地对象，也不能是匿名对象。

枚举类不能扩展密封类（以及任何其他类），但它们可以实现密封接口。

这些限制不适用于间接子类。 如果密封类的直接子类未标记为密封，则可以通过其修饰符允许的任何方式对其进行扩展：

### 密封类和 when 表达式
当在 when 表达式中使用密封类时，使用密封类的主要好处就会发挥作用。 如果可以验证语句涵盖所有情况，则无需在语句中添加 else 子句。 
但是，这仅在使用 when 作为表达式（使用结果）而不是作为语句时才有效。


## 嵌套类和内部类
类可以嵌套在其他类中。还可以使用带有嵌套的接口。 类和接口的所有组合都是可能的：可以将接口嵌套在类中，将类嵌套在接口中，将接口嵌套在接口中。

### 内部类
标记为 inner 的嵌套类可以访问其外部类的成员。 内部类携带对外部类对象的引用。


## 枚举类
每个枚举常量都是个对象。枚举常量通过逗号分隔。

由于每个枚举都是枚举类的一个实例，因此可以将其初始化为：
```
enum class Color(val rgb: Int) {
    RED(0xFF0000),
    GREEN(0x00FF00),
    BLUE(0x0000FF)
}
```


枚举常量可以使用其相应的方法以及重写基方法来声明它们自己的匿名类。
```
enum class ProtocolState {
    WAITING {
        override fun signal() = TALKING
    },

    TALKING {
        override fun signal() = WAITING
    };

    abstract fun signal(): ProtocolState
}
```
如果枚举类定义了任何成员，则用分号将常量定义与成员定义分开。

### 在枚举类中实现接口
枚举类可以实现一个接口（但它不能从一个类派生），为所有条目提供接口成员的公共实现，或者为其匿名类中的每个条目提供单独的实现。 这是通过将要实现的接口添加到枚举类声明中来完成的，如下所示：
```
enum class IntArithmetics : BinaryOperator<Int>, IntBinaryOperator {
    PLUS {
        override fun apply(t: Int, u: Int): Int = t + u
    },
    TIMES {
        override fun apply(t: Int, u: Int): Int = t * u
    };

    override fun applyAsInt(t: Int, u: Int) = apply(t, u)
}
```

### 使用枚举常量
Kotlin 中的枚举类具有用于列出定义的枚举常量并通过其名称获取枚举常量的综合方法。 这些方法的签名如下（假设枚举类的名称是EnumClass）：
```
EnumClass.valueOf(value: String): EnumClass
EnumClass.values(): Array<EnumClass>
```
如果指定的名称与类中定义的任何枚举常量不匹配，则 `valueOf()` 方法将引发 `IllegalArgumentException`。

可以使用 enumValues<T>() 和 enumValueOf<T>() 函数以泛型方式访问枚举类中的常量：
```
enum class RGB { RED, GREEN, BLUE }

inline fun <reified T : Enum<T>> printAllValues() {
    print(enumValues<T>().joinToString { it.name })
}

printAllValues<RGB>() // prints RED, GREEN, BLUE
```
每个枚举常量都具有用于在枚举类声明中获取其名称和位置（从 0 开始）的属性：
```
val name: String
val ordinal: Int
```




## 内联类
有时，业务逻辑需要围绕某种类型创建包装器。 但是，由于额外的堆分配，它会引入运行时开销。 
此外，如果包装类型是原始类型，则性能损失很糟糕，因为原始类型通常由运行时进行大量优化，而它们的包装器没有得到任何特殊处理。

为了解决这些问题，Kotlin 引入了一种特殊的类，称为内联类。 内联类是基于值的类的子集。 他们没有身份，只能持有值。

要声明内联类，在类名之前使用 `value` 修饰符：
```
value class Password(private val s: String)
```

内联类必须具有在主构造器中初始化的**单个**属性。 在运行时，内联类的实例将使用这个单一属性表示 ：
```
// No actual instantiation of class 'Password' happens
// At runtime 'securePassword' contains just 'String'
val securePassword = Password("Don't try this in production")
```
这是内联类的主要特征，它启发了内联这个名称：类的数据被内联到它的用法中（类似于内联函数的内容如何内联到调用站点）。

### 成员
内联类支持常规类的一些功能。 特别是，它们可以声明属性和函数，并具有 `init` 块：
```
value class Name(val s: String) {
    init {
        require(s.length > 0) { }
    }

    val length: Int
        get() = s.length

    fun greet() {
        println("Hello, $s")
    }
}

fun main() {
    val name = Name("Kotlin")
    name.greet() // method `greet` is called as a static method
    println(name.length) // property getter is called as a static method
}
```
内联类属性不能有后端字段。 它们只能具有简单的可计算属性（没有 `lateinit`/delegated 属性）。

### 继承
允许内联类从接口继承。

禁止内联类参与类层次结构。 这意味着内联类不能扩展其他类并且始终是 `final` 类。

### 表示
在生成的代码中，Kotlin 编译器为每个内联类保留一个包装器。 内联类实例可以在运行时表示为包装器或底层类型。 
这类似于如何将 Int 表示为原始 int 或包装器 Integer。

Kotlin 编译器将更喜欢使用底层类型而不是包装器来生成最高性能和优化的代码。 但是，有时需要保留包装器。 根据经验，内联类在用作另一种类型时都会被装箱。
```
interface I

@JvmInline
value class Foo(val i: Int) : I

fun asInline(f: Foo) {}
fun <T> asGeneric(x: T) {}
fun asInterface(i: I) {}
fun asNullable(i: Foo?) {}

fun <T> id(x: T): T = x

fun main() {
    val f = Foo(42)

    asInline(f)    // unboxed: used as Foo itself
    asGeneric(f)   // boxed: used as generic type T
    asInterface(f) // boxed: used as type I
    asNullable(f)  // boxed: used as Foo?, which is different from Foo

    // below, 'f' first is boxed (while being passed to 'id') and then unboxed (when returned from 'id')
    // In the end, 'c' contains unboxed representation (just '42'), as 'f'
    val c = id(f)
}
```
因为内联类既可以表示为基础值，也可以表示为包装器，因此引用相等对它们没有意义，因此被禁止。

内联类也可以有一个泛型类型参数作为基础类型。 在这种情况下，编译器将其映射到 Any? 或者，通常是类型参数的上限。
```
@JvmInline
value class UserId<T>(val value: T)

fun compute(s: UserId<String>) {} // compiler generates fun compute-<hashcode>(s: Any?)
```

### 重整
由于内联类被编译为它们的底层类型，它可能会导致各种模糊的错误，例如意外的平台签名冲突：
```
@JvmInline
value class UInt(val x: Int)

// Represented as 'public final void compute(int x)' on the JVM
fun compute(x: Int) { }

// Also represented as 'public final void compute(int x)' on the JVM!
fun compute(x: UInt) { }
```
为了缓解此类问题，使用内联类的函数会通过在函数名称中添加一些稳定的哈希码来破坏。 因此，fun compute(x: UInt) 将表示为 public final void compute-<hashcode>(int x)，从而解决了冲突问题。

Kotlin 1.4.30 中更改了修改方案。 使用 -Xuse-14-inline-classes-mangling-scheme 编译器标志强制编译器使用旧的 1.4.0 修改方案并保持二进制兼容性。


## object 表达式和声明
有时需要创建一个对某个类稍作修改的对象，而无需为它显式声明一个新的子类。 Kotlin 可以使用 object 表达式和 object 声明来处理这个问题。

### object 表达式
object 表达式创建匿名类的对象，即未使用类声明显式声明的类。 这样的类对于一次性使用很有用。 
可以从头开始定义它们、从现有类继承或实现接口。 匿名类的实例也称为匿名对象，因为它们是由表达式而不是名称定义的。

对象表达式以 `object` 关键字开头。

如果只需要一个没有任何重要超类型的对象，请将其成员写在对象后面的花括号中：
```
val helloWorld = object {
    val hello = "Hello"
    val world = "World"
    // object expressions extend Any, so `override` is required on `toString()`
    override fun toString() = "$hello $world"
}
```

要创建从某个（或多个类型）继承的匿名类的对象，在对象和冒号 (:) 之后指定此类型。 然后实现或覆盖这个类的成员。

如果超类型具有构造器，则将适当的构造器参数传递给它。 多个超类型可以指定为冒号后的逗号分隔列表。

当匿名对象用作本地或私有类型但不是内联声明（函数或属性）时，它的所有成员都可以通过此函数或属性访问：
```
class C {
    private fun getObject() = object {
        val x: String = "x"
    }

    fun printX() {
        println(getObject().x)
    }
}
```
如果此函数或属性是公有或私有内联，则其实际类型为：
* Any 如果匿名对象没有声明的超类型
* 匿名对象的声明超类型，如果只有一种这样的类型
* 如果有多个声明的超类型，则显式声明的类型

在所有这些情况下，添加到匿名对象中的成员都是不可访问的。 如果在函数或属性的实际类型中声明了被覆盖的成员，则它们是可访问的：
```
interface A {
    fun funFromA() {}
}
interface B

class C {
    // The return type is Any. x is not accessible
    fun getObject() = object {
        val x: String = "x"
    }

    // The return type is A; x is not accessible
    fun getObjectA() = object: A {
        override fun funFromA() {}
        val x: String = "x"
    }

    // The return type is B; funFromA() and x are not accessible
    fun getObjectB(): B = object: A, B { // explicit return type is required
        override fun funFromA() {}
        val x: String = "x"
    }
}
```
object 表达式中的代码可以从封闭范围访问变量：
```
fun countClicks(window: JComponent) {
    var clickCount = 0
    var enterCount = 0

    window.addMouseListener(object : MouseAdapter() {
        override fun mouseClicked(e: MouseEvent) {
            clickCount++
        }

        override fun mouseEntered(e: MouseEvent) {
            enterCount++
        }
    })
    // ...
}
```

### object 声明
单例模式在多种情况下很有用，而 Kotlin 使声明单例变得容易：
```
object DataProviderManager {
    fun registerDataProvider(provider: DataProvider) {
        // ...
    }

    val allDataProviders: Collection<DataProvider>
        get() = // ...
}
```
这称为 object 声明，它总是在对象关键字后面有一个名称。 就像变量声明一样，object 声明不是表达式，它不能用在赋值语句的右侧。

对象声明的初始化是线程安全的，并在首次访问时完成。

这样的 object 可以有超类：
```
object DefaultListener : MouseAdapter() {
    override fun mouseClicked(e: MouseEvent) { ... }

    override fun mouseEntered(e: MouseEvent) { ... }
}
```

object 声明不能是局部的（也就是说，它们不能直接嵌套在函数内部），但它们可以嵌套到其他 object 声明或非内部类中。

### Companion object
类中的 object 声明可以用 companion 关键字标记。

可以通过使用类名作为限定符简单地调用伴生对象的成员。

伴随对象的名称可以省略，在这种情况下将使用名称 Companion：
```
class MyClass {
    companion object { }
}

val x = MyClass.Companion
```

类成员可以访问相应伴随对象的私有成员。

自己使用的类的名称（不是作为另一个名称的限定符）充当对类的伴随对象的引用（无论是否命名）：

需要注意的是，即使伴随对象的成员在其他语言中看起来像静态成员，在运行时它们仍然是真实对象的实例成员，并且可以例如实现接口：
```
interface Factory<T> {
    fun create(): T
}

class MyClass {
    companion object : Factory<MyClass> {
        override fun create(): MyClass = MyClass()
    }
}

val f: Factory<MyClass> = MyClass
```
但是，在 JVM 上，如果使用 @JvmStatic 注释，可以将伴随对象的成员生成为真正的静态方法和字段。




















