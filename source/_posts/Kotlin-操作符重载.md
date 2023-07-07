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





# Okio
[Okio](https://square.github.io/okio/) 是一个库，它补充了 java.io 和 java.nio，使访问、存储和处理数据变得更加容易。

## ByteStrings and Buffers
Okio 围绕两种类型构建，将大量功能打包到一个简单的 API 中：
* `ByteString` 是一个不可变的字节序列。 对于字符数据，`String` 是基础。 `ByteString` 是 String 失散多年的兄弟，可以很容易地将二进制数据视为一个值。 这个类符合人体工程学：它知道如何将自己编码和解码为 hex、base64 和 UTF-8。
* `Buffer` 是一个可变的字节序列。 与 `ArrayList` 一样，不需要提前调整 buffer 的大小。 将 buffer 作为队列读写：将数据写入末尾并从前面读取。

在内部，`ByteString` 和 `Buffer` 做了一些聪明的事情来节省 CPU 和内存。 如果将 UTF-8 字符串编码为 `ByteString`，它会缓存对该字符串的引用，这样如果稍后对其进行解码，就无需执行任何操作。

`Buffer` 被实现为 segment 的链表。 当将数据从一个 buffer 移动到另一个 buffer 时，它会重新分配 segment 的所有权，而不是复制数据。 这种方法对多线程程序特别有用：与网络对话的线程可以与工作线程交换数据，而无需任何复制或仪式。

## Sources and Sinks
`java.io` 设计的一个优雅部分是如何对 stream 进行分层以进行加密和压缩等转换。 Okio 包含自己的 stream 类型，称为 `Source` 和 `Sink`，它们的工作方式类似于 `InputStream` 和 `OutputStream`，但有一些关键区别：
* **超时**。 这些 stream 提供对底层 I/O 机制超时的访问。 与 `java.io` 套接字 stream 不同，`read()` 和 `write()` 调用均会超时。
* **易于实施**。 `Source` 声明了三个方法：`read()`、`close()` 和 `timeout()`。 没有诸如 `available()` 或单字节读取之类的危害会导致正确性和性能意外。
* **便于使用**。 尽管 `Source` 和 `Sink` 的实现只有三个方法可以编写，但调用者可以使用 `BufferedSource` 和 `BufferedSink` 接口获得丰富的 API。 这些接口在一处提供所需的一切。
* **没有人为区分字节流和字符流**。 它们都是数据。 以字节、UTF-8 字符串、big-endian 32 位整数、little-endian shorts 的形式读写它； 任何你想要的。 没有更多的 `InputStreamReader`！
* **易于测试**。 `Buffer` 类同时实现了 `BufferedSource` 和 `BufferedSink`，因此测试代码简单明了。

## Okio读写流程

### 逐行读取文本文件
使用 `FileSystem.source(Path)` 打开 source stream 以读取文件。 返回的 `Source` 接口很小，用途有限。 相反，用 buffer 包装源代码。 这有两个好处：
* **它使 API 更强大**。 `BufferedSource` 没有提供 `Source` 提供的基本方法，而是有几十种方法可以简洁地解决最常见的问题。
* **它使程序运行得更快**。 Buffer 允许 Okio 用更少的 I/O 操作完成更多的工作。

每个打开的 `Source` 都需要关闭。 打开流的代码负责确保它已关闭。

这用于自动关闭流。 这可以防止资源泄漏，即使抛出异常也是如此。
```
fun readLines(path: Path) {
    FileSystem.SYSTEM.source(path).use { fileSource ->
        fileSource.buffer().use { bufferedFileSource ->
            while (true) {
                val line = bufferedFileSource.readUtf8Line() ?: break
                if ("square" in line) {
                    println(line)
                }
            }
        }
    }
}
```
`readUtf8Line()` API 读取所有数据，直到下一个行分隔符——`\n`、`\r\n` 或文件末尾。 它将数据作为字符串返回，省略末尾的定界符。 当它遇到空行时，该方法将返回一个空字符串。 如果没有更多数据可供读取，它将返回 null。

可以使用 `FileSystem.read()` 在 block 之前缓冲 source，在之后关闭 source。 在 block 的主体中，`this` 是一个 `BufferedSource`。
```
fun readLines1(path: Path) {
    FileSystem.SYSTEM.read(path) {
        while (true) {
            val line = readUtf8Line() ?: break
        }
    }
}
```
`readUtf8Line()` 方法适用于解析大多数文件。 对于某些用例，还可以考虑 `readUtf8LineStrict()`。 它是相似的，但它要求每行以 `\n` 或 `\r\n` 结束。 如果在此之前遇到文件末尾，它将抛出 `EOFException`。 strict 变体还允许字节限制以防止格式错误的输入。

### 写文本文件
上面使用了 `Source` 和 `BufferedSource` 来读取文件。 为了写入，使用 `Sink` 和 `BufferedSink`。 缓冲的优点是相同的：更强大的 API 和更好的性能。
```
public void writeEnv(Path path) throws IOException {
  try (Sink fileSink = FileSystem.SYSTEM.sink(path);
       BufferedSink bufferedSink = Okio.buffer(fileSink)) {

    for (Map.Entry<String, String> entry : System.getenv().entrySet()) {
      bufferedSink.writeUtf8(entry.getKey());
      bufferedSink.writeUtf8("=");
      bufferedSink.writeUtf8(entry.getValue());
      bufferedSink.writeUtf8("\n");
    }

  }
}
```
没有用于编写一行输入的 API； 相反，手动插入自己的换行符。 大多数程序应该将`“\n”`硬编码为换行符。 在极少数情况下，可以使用 `System.lineSeparator()` 而不是`“\n”`：它在 Windows 上返回`“\r\n”`，在其他任何地方返回`“\n”`。

可以使用 `FileSystem.write()` 在我们的块之前缓冲 sink 并在之后关闭 sink。 在块的主体中，`this` 是一个 `BufferedSink`。
```
fun writeEnv(path: Path) {
    FileSystem.SYSTEM.write(path) {
        for ((key, value) in System.getenv()) {
            writeUtf8(key)
            writeUtf8("=")
            writeUtf8(value)
            writeUtf8("\n")
        }
    }
}
```
在上面的代码中，对 `writeUtf8()` 进行了四次调用。 进行四次调用比下面的代码更有效，因为 VM 不必创建和垃圾收集临时字符串。  
`sink.writeUtf8(entry.getKey() + "=" + entry.getValue() + "\n"); // Slower!`

### UTF-8
在上面的 API 中可以看到 Okio 非常喜欢 UTF-8。 早期的计算机系统存在许多不兼容的字符编码：ISO-8859-1、ShiftJIS、ASCII、EBCDIC 等。编写支持多字符集的软件非常糟糕，我们甚至没有表情符号！ 今天我们很幸运，世界各地都在 UTF-8 上进行了标准化，在遗留系统中很少使用其他字符集。

如果需要其他字符集，可以使用 `readString()` 和 `writeString()`。 这些方法要求指定一个字符集。 否则可能会不小心创建只能由本地计算机读取的数据。 大多数程序应该只使用 UTF-8 方法。

在对字符串进行编码时，需要注意字符串表示和编码的不同方式。 当字形具有重音或其他装饰时，它可以表示为单个复杂代码点 (`é`) 或简单代码点 (`e`) 后跟其修饰符 (`´`)。 当整个字形是单个代码点时, 称为 **NFC**； 当它是多个时，它是 **NFD**。

尽管在 I/O 中读取或写入字符串时都使用 UTF-8，但当它们在内存中时，Java 字符串使用称为 UTF-16 的过时字符编码。 这是一种糟糕的编码，因为它对大多数字符使用 16 位 `char`，但有些字符不适合。 特别是，大多数表情符号使用两个 Java 字符。 这是有问题的，因为 `String.length()` 返回了一个令人惊讶的结果：UTF-16 字符的数量而不是字形的自然数量。

在大多数情况下，Okio 让您忽略这些问题并专注于您的数据。 但是当您需要它们时，可以使用方便的 API 来处理低级 UTF-8 字符串。

使用 `Utf8.size()` 计算将字符串编码为 UTF-8 而不实际编码所需的字节数。 这在像协议缓冲区这样的长度前缀编码中很方便。

使用 `BufferedSource.readUtf8CodePoint()` 读取单个可变长度代码点，使用 `BufferedSink.writeUtf8CodePoint()` 写入一个。
```
fun dumpStringData(s: String) {
  println("                       " + s)
  println("        String.length: " + s.length)
  println("String.codePointCount: " + s.codePointCount(0, s.length))
  println("            Utf8.size: " + s.utf8Size())
  println("          UTF-8 bytes: " + s.encodeUtf8().hex())
  println()
}
```

### 写二进制文件
编码二进制文件与编码文本文件没有什么不同。 Okio 对两者使用相同的 `BufferedSink` 和 `BufferedSource` 字节。 这对于包含字节和字符数据的二进制格式很方便。

编写二进制数据比文本更危险，因为如果犯了错误，通常很难诊断。 通过小心这些陷阱来避免此类错误：
* **每个字段的宽度**。 这是使用的字节数。 Okio 不包含发出部分字节的机制。 如果你需要，需要在写入之前进行自己的位移和屏蔽。
* **每个字段的字节顺序**。 所有超过一个字节的字段都具有字节顺序：字节是按从最重要到最不重要（大端）还是从最不重要到最重要（小端）排序。 Okio 对 little-endian 方法使用 `Le` 后缀； 没有后缀的方法是大端法。
* **签名与未签名**。 Java 没有无符号原始类型（除了 `char`！），所以处理这个问题通常发生在应用程序层。 为了使这更容易一些，Okio 接受 `writeByte()` 和 `writeShort()` 的 `int` 类型。 你可以传递一个“无符号”字节，比如 255，Okio 会做正确的事情。

此代码按照 BMP 文件格式对位图进行编码。
```
fun encode(bitmap: Bitmap, sink: BufferedSink) {
  val height = bitmap.height
  val width = bitmap.width
  val bytesPerPixel = 3
  val rowByteCountWithoutPadding = bytesPerPixel * width
  val rowByteCount = (rowByteCountWithoutPadding + 3) / 4 * 4
  val pixelDataSize = rowByteCount * height
  val bmpHeaderSize = 14
  val dibHeaderSize = 40

  // BMP Header
  sink.writeUtf8("BM") // ID.
  sink.writeIntLe(bmpHeaderSize + dibHeaderSize + pixelDataSize) // File size.
  sink.writeShortLe(0) // Unused.
  sink.writeShortLe(0) // Unused.
  sink.writeIntLe(bmpHeaderSize + dibHeaderSize) // Offset of pixel data.

  // DIB Header
  sink.writeIntLe(dibHeaderSize)
  sink.writeIntLe(width)
  sink.writeIntLe(height)
  sink.writeShortLe(1) // Color plane count.
  sink.writeShortLe(bytesPerPixel * Byte.SIZE_BITS)
  sink.writeIntLe(0) // No compression.
  sink.writeIntLe(16) // Size of bitmap data including padding.
  sink.writeIntLe(2835) // Horizontal print resolution in pixels/meter. (72 dpi).
  sink.writeIntLe(2835) // Vertical print resolution in pixels/meter. (72 dpi).
  sink.writeIntLe(0) // Palette color count.
  sink.writeIntLe(0) // 0 important colors.

  // Pixel data.
  for (y in height - 1 downTo 0) {
    for (x in 0 until width) {
      sink.writeByte(bitmap.blue(x, y))
      sink.writeByte(bitmap.green(x, y))
      sink.writeByte(bitmap.red(x, y))
    }

    // Padding for 4-byte alignment.
    for (p in rowByteCountWithoutPadding until rowByteCount) {
      sink.writeByte(0)
    }
  }
}
```
该程序最棘手的部分是格式所需的填充。 BMP 格式要求每一行都以 4 字节为边界开始，因此有必要添加零以保持对齐。

编码其他二进制格式通常非常相似。 一些技巧：
* 编写具有黄金价值的测试！ 确认您的程序发出预期结果可以使调试更容易。
* 使用 `Utf8.size()` 计算编码字符串的字节数。 这对于以长度为前缀的格式是必不可少的。
* 使用 `Float.floatToIntBits()` 和 `Double.doubleToLongBits()` 对浮点值进行编码。

### 在套接字上通信
通过网络发送和接收数据有点像写入和读取文件。 使用 `BufferedSink` 对输出进行编码，使用 `BufferedSource` 对输入进行解码。 与文件一样，网络协议可以是文本、二进制或两者的混合。 但是网络和文件系统之间也有一些实质性的区别。


## 源码分析

### Source & Sink
```
interface Source : Closeable {
  /**
   * Removes at least 1, and up to `byteCount` bytes from this and appends them to `sink`. Returns
   * the number of bytes read, or -1 if this source is exhausted.
   */
  @Throws(IOException::class)
  fun read(sink: Buffer, byteCount: Long): Long

  /** Returns the timeout for this source.  */
  fun timeout(): Timeout

  /**
   * Closes this source and releases the resources held by this source. It is an error to read a
   * closed source. It is safe to close a source more than once.
   */
  @Throws(IOException::class)
  override fun close()
}

actual interface Sink : Closeable, Flushable {
  @Throws(IOException::class)
  actual fun write(source: Buffer, byteCount: Long)

  @Throws(IOException::class)
  actual override fun flush()

  actual fun timeout(): Timeout

  @Throws(IOException::class)
  actual override fun close()
}
```
`Source` 和 `Sink` 是 Okio 中最基础的两个接口，分别对应 `java.io` 中的 `InputStream` 和 `OutputStream`。

### BufferedSource & BufferedSink
BufferedSource 和 BufferedSink 接口对 `Source` 和 `Sink` 进行了扩展，并添加了缓冲的能力。

### Segment 和 SegmentPool
Buffer 中的每个 segment 都是一个循环链表节点，引用 Buffer 中的前后 segment。  
池中的每个 segment 都是一个单链表节点，引用池中的其余 segment。  
segment 的底层字节数组可以在 Buffer 和字节串之间共享。 当一个 segment 的字节数组被共享时，该 segment 不能被回收，它的字节数据也不能被改变。 唯一的例外是允许所有者 segment 追加到该 segment，以限制或超出限制写入数据。 每个字节数组都有一个拥有的 segment。 不共享位置、限制、上一个和下一个参考。

### Buffer




