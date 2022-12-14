---
title: Kotlin - Coroutine
date: 2022-05-28 10:00:21
tags: 
- Kotlin
- Coroutine
---

[原文](https://kotlinlang.org/docs/coroutines-guide.html)

## 前言
Kotlin 只在标准库中提供了最低级别的API以使各种其他库能够利用协程。  
在 kotlin 中，`async` 和 `await` 不是关键字，而且甚至不是标准库的一部分。  
此外，与 futures and promises 相比，Kotlin 的挂起函数概念为异步操作提供了更安全且不易出错的抽象。

`kotlinx.coroutines` 是由 JetBrains 开发的丰富的协程库。 它包含本指南涵盖的许多支持协程的高级原语，包括`launch`、`async`等。


## 协程基础
协程是可挂起计算的一个实例。 它在概念上类似于线程，因为它需要一个与其余代码同时工作的代码块来运行。
但是，协程并不绑定到任何特定的线程。 它可以在一个线程中暂停执行并在另一个线程中恢复。

```kotlin
fun main() = runBlocking { // this: CoroutineScope
    launch { // launch a new coroutine and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello") // main coroutine continues while a previous one is delayed
}
```
`launch` 是一个协程构建器。 它与其余代码同时启动一个新的协程，该协程继续独立工作。 这就是首先打印 Hello 的原因。

`delay` 是一个特殊的挂起函数。 它将协程暂停特定时间。 挂起协程不会阻塞底层线程，但允许其他协程运行并使用底层线程来执行它们的代码。

`runBlocking` 也是一个协程构建器，它将常的 `fun main()` 的非协程世界与 `runBlocking { ... }` 花括号内的协程代码联系起来。 这在 IDE 中通过以下方式突出显示：在 `runBlocking` 左大括号之后的 `this: CoroutineScope` 提示。

`runBlocking` 的名称意味着运行它的线程在调用期间被阻塞，直到 `runBlocking { ... }` 中的所有协程完成执行。 你会经常看到在应用程序的最顶层使用 `runBlocking`，而在实际代码中很少见，因为线程是昂贵的资源，阻塞它们效率低下，通常是不希望的。


### 结构化并发
协程遵循**结构化并发**原则，这意味着新的协程只能在限定协程生命周期的特定 CoroutineScope 中启动。 上面的例子表明 runBlocking 建立了相应的范围，这就是为什么前面的例子等到 World! 在延迟一秒后打印，然后才退出。

在实际应用程序中，您将启动大量协程。 结构化并发确保它们不会丢失并且不会泄漏。 在其所有子协程完成之前，外部作用域无法完成。 结构化并发还确保正确报告代码中的任何错误并且永远不会丢失。

挂起函数可以像常规函数一样在协程中使用，但它们的附加特性是它们可以反过来使用其他挂起函数（如本例中的`delay`）来挂起协程的执行。


### Scope 构建器
除了不同构建器提供的协程作用域外，还可以使用 `coroutineScope` 构建器声明自己的作用域。 它创建了一个协程范围，并且在所有启动的子进程完成之前不会完成。

runBlocking 和 coroutineScope 构建器可能看起来很相似，因为它们都在等待自己的 body 及其所有子节点完成。 主要区别在于 runBlocking 方法阻塞当前线程等待，而 coroutineScope 只是挂起，释放底层线程用于其他用途。 由于这个区别，runBlocking 是一个常规函数，而 coroutineScope 是一个挂起函数。

### Scope 构建器 和 并发
可以在任何挂起函数中使用 coroutineScope 构建器来执行多个并发操作

### 显式 job
`launch`协程构建器返回一个 Job 对象，该对象是已启动协程的句柄，可用于显式等待其完成。


## 取消和超时

### 取消协程执行
在长时间运行的应用程序中，可能需要对后台协程进行细粒度控制。 例如，用户可能已经关闭了启动协程的页面，现在不再需要其结果并且可以取消其操作。 `launch` 函数返回一个 Job 可以用来取消正在运行的协程：

### 取消是合作的
协程取消是*合作*的。 协程代码必须合作才能取消。 `kotlinx.coroutines` 中的所有挂起函数都是可以取消的。 他们检查协程的取消并在取消时抛出 `CancellationException`。 但是，如果协程在计算中工作并且不检查取消，则无法取消。

### 使计算代码可取消
有两种方法可以使计算代码可取消。 第一个是定期调用检查取消的挂起函数。 有一个 `yield` 函数，它是一个很好的选择。   
另一种是显式检查取消状态。`isActive` 是通过 `CoroutineScope` 对象在协程内部可用的扩展属性。

### 使用 finally 关闭资源
可取消的挂起函数在取消时会抛出 CancellationException，这可以按通常的方式处理。 例如，`try {...} finally {...}` 表达式和 Kotlin `use` 函数在协程被取消时正常执行它们的终结操作

### 运行不可取消的代码块
任何在上一个示例的 `finally` 块中使用挂起函数的尝试都会导致 CancellationException，因为运行此代码的协程被取消。 
通常，这不是问题，因为所有行为良好的关闭操作（关闭文件、取消job或关闭任何类型的通信通道）通常都是非阻塞的，并且不涉及任何挂起功能。 
但是，在极少数情况下，当需要在取消的协程中挂起时，可以使用 `withContext` 函数和 `NonCancellable` 上下文将相应的代码包装在 `withContext(NonCancellable) {...}` 中。

### 超时
取消协程执行的最明显的实际原因是因为它的执行时间已经超过了一些超时。 虽然可以手动跟踪对相应 Job 的引用并启动一个单独的协程以在延迟后取消跟踪的协程，但有一个准备使用的 `withTimeout` 函数可以做到这一点。

`withTimeout` 抛出的 `TimeoutCancellationException` 是 `CancellationException` 的子类。 我们之前没有在控制台上看到它的堆栈跟踪。 那是因为在取消的协程内部 CancellationException 被认为是协程完成的正常原因。 然而，在这个例子中，我们在 `main` 函数中使用了 withTimeout。

由于取消只是一个异常，所有资源都以通常的方式关闭。 如果需要专门针对任何类型的超时执行一些附加操作，可以可以将超时代码包装在 `try {...} catch (e: TimeoutCancellationException) {...}` 块中，或使用 `withTimeoutOrNull` 函数，它类似于 `withTimeout` ，但在超时时返回 `null` 而不是抛出异常


### 异步超时和资源
`withTimeout` 中的超时事件相对于在其块中运行的代码是异步的，并且可能随时发生，甚至在从超时块内部返回之前。 如果您在块内打开或获取一些需要在块外关闭或释放的资源，请记住这一点。


## 组合挂起函数

### 默认是顺序的
协程中的代码，就像在常规代码中一样，默认是顺序的

### 使用 async 来并发
从概念上讲，`async` 就像 `launch`。 它启动一个单独的协程，它是一个轻量级线程，可与所有其他协程同时工作。 不同之处在于，`launch` 返回一个 `Job` 并且不携带任何结果值，而 `async` 返回一个 `Deferred` - 一个轻量级的非阻塞future，表示稍后提供结果的承诺。 可以在deferred值上使用 `.await()` 来获得其最终结果，但 `Deferred` 也是一个 `job`，因此可以在需要时取消它。

### 懒启动的 async
可选地，可以通过将其 `start` 参数设置为 `CoroutineStart.LAZY` 来使 `async` 变得惰性。 在这种模式下，它仅在 `await` 需要其结果或调用其 `Job` 的 `start` 函数时才启动协程。


### 异步风格的函数
