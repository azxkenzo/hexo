---
title: Kotlin - String 类 API
date: 2022-05-27 07:24:30
tags: Kotlin
---

### compareTo
将此对象与指定对象进行比较以进行排序。如果此对象等于指定的其他对象，则返回零，如果小于 other，则返回负数，如果大于 other，则返回正数。
``` kotlin
operator fun compareTo(other: String): Int
```


### contains
检查原字符串中时候包含给定字符或字符串或正则表达式
``` kotlin
inline operator fun CharSequence.contains(regex: Regex): Boolean

operator fun CharSequence.contains(char: Char, ignoreCase: Boolean = false): Boolean

operator fun CharSequence.contains(other: CharSequence, ignoreCase: Boolean = false): Boolean
```


### drop
返回移除了前 n 个字符的字符串
``` kotlin
fun String.drop(n: Int): String

"4321".drop(2)   // 21
```


### length
获取字符串的长度
``` kotlin
val length: Int  // 定义

"123".length
```


### plus
返回通过将此字符串与给定其他对象的字符串表示形式连接获得的字符串。
``` kotlin
operator fun plus(other: Any?): String  // 定义

"123".plus("312")  or  "123" + "321"
```


### repeat
返回将原字符串重复 n 次后的新字符串。当 n = 0 时，返回空字符串
``` kotlin
actual fun CharSequence.repeat(n: Int): String
```


### split
使用指定模式或字符串或字符分割原字符串
``` kotlin
fun CharSequence.split(regex: Pattern, limit: Int = 0)

fun CharSequence.split(vararg delimiters: String, ignoreCase: Boolean = false, limit: Int = 0): List<String>

fun CharSequence.split(vararg delimiters: Char, ignoreCase: Boolean = false, limit: Int = 0): List<String>

inline fun CharSequence.split(regex: Regex, limit: Int = 0): List<String>
```




















