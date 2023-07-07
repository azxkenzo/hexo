---
title: Android - Bitmap
date: 2023-02-28 14:43:37
tags: Android
---

Bitmap 类的定义：
```
public final class Bitmap implements Parcelable
```

### 主要方法和属性
* mNativePtr: 方便JNI访问
* getDensity(): 返回此 Bitmap 的密度
* setDensity(): 指定此 Bitmap 的密度。 当 Bitmap 被绘制到同样具有密度的 Canvas 时，它会被适当缩放。
* reconfigure(): 修改 Bitmap 使具有指定的宽度、高度和 Bitmap.Config，而不影响支持 bitmap 的底层分配。不会为新配置重新初始化 bitmap 像素数据。此方法可用于避免分配新的 bitmap，而是将现有 bitmap 的分配重新用于大小相等或更小的新配置。
* recycle(): 释放与此 bitmap 关联的 native 对象，并清除对像素数据的引用。 这不会同步释放像素数据； 如果没有其他引用，它只是允许它被垃圾收集。 bitmap 被标记为“ dead”，这意味着如果调用 getPixels() 或 setPixels() 它将抛出异常，并且不会绘制任何内容。 此操作无法撤消，因此只有在确定该位图不再有其他用途时才应调用它。 这是一个高级调用，通常不需要调用，因为正常的 GC 过程将在不再引用此 bitmap 时释放此内存。
* getGenerationId(): 返回此 bitmap 的生成 ID。 每当修改 bitmap 时，生成 ID 都会更改。 这可以用作检查 bitmap 是否已更改的有效方法。
* `copy()`: 尝试根据此 Bitmap 的尺寸制作新 Bitmap，将新 Bitmap 的配置设置为指定的配置，然后将此 Bitmap 的像素复制到新 Bitmap 中。 如果不支持转换，或者分配器失败，则返回 NULL。 返回的 Bitmap 与原始 Bitmap 具有相同的密度和颜色空间，但以下情况除外。 复制到 Bitmap.Config.ALPHA_8 时，颜色空间被丢弃。
* 一系列 `createBitmap()` 重载方法
* `compress()`: 将 Bitmap 的压缩版本写入指定的输出流。 如果返回 true，则可以通过将相应的输入流传递给 BitmapFactory.decodeStream() 来重建位图。 注意：并非所有格式都直接支持所有位图配置，因此从 BitmapFactory 返回的位图可能具有不同的位深度，和/或可能丢失了每像素 alpha（例如 JPEG 仅支持不透明像素）。


### Bitmap.Config
Config 类表示可能的 bitmap 配置。 bitmap 配置描述了像素的存储方式。 这会影响质量（颜色深度）以及显示透明/半透明颜色的能力。

Config 有以下枚举值：
* `ALPHA_8(1)`: 每个像素都存储为单个半透明 (alpha) 通道。 例如，这对于有效存储掩码非常有用。 不存储颜色信息。 使用此配置，每个像素需要 1 个字节的内存。
* `RGB_565(3)`: 每个像素存储在 2 个字节上，仅对 RGB 通道进行编码：红色以 5 位精度存储（32 个可能值），绿色以 6 位精度存储（64 个可能值），蓝色以 5 位精度存储。
* `ARGB_8888(5)`: 每个像素存储在 4 个字节上。 每个通道（半透明的 RGB 和 alpha）都以 8 位精度（256 个可能的值）存储。这种配置非常灵活并提供最佳质量。 应尽可能使用它。
* `RGBA_F16(6)`: 每个像素存储在 8 个字节上。 每个通道（用于半透明的 RGB 和 alpha）都存储为半精度浮点值。 此配置特别适合广色域和 HDR 内容。
* `HARDWARE(7)`: 特殊配置，当位图仅存储在图形内存中时。 此配置中的位图始终是不可变的。 当对位图的唯一操作是在屏幕上绘制它时，它是最佳的情况。
* `RGBA_1010102(8)`: 每个像素存储在 4 个字节上。 每个 RGB 通道都以 10 位精度（1024 个可能值）存储。 有一个额外的 alpha 通道，它以 2 位精度（4 个可能的值）存储。 此配置适用于不需要 alpha 混合的广色域和 HDR 内容，因此内存成本与 ARGB_8888 相同，同时实现更高的颜色精度。


### Bitmap.CompressFormat
CompressFormat 类指定 Bitmap 可以压缩成的已知格式

有这几个值：JPEG、PNG、WEBP_LOSSY、WEBP_LOSSLESS


## BitmapFactory
BitmapFactory 用于从各种来源创建 bitmap 对象，包括文件、流和字节数组。

### 主要方法
* decodeByteArray
* decodeFile
* decodeFileDescriptor
* decodeResource
* decodeStream


### BitmapFactory.Options
Options 是一个解码的配置类

主要参数：
* `inBitmap`: 如果设置，采用 Options 对象的解码方法将在加载内容时尝试重用此 bitmap。 如果解码操作不能使用此 bitmap，则解码方法将抛出 IllegalArgumentException。 当前的实现要求重用 bitmap 是 mutable 的，并且即使在解码通常会导致 bitmap immutable 的资源时，生成的重用 bitmap 也将继续保持 mutable。
* `inJustDecodeBounds`: 如果设置为 true，解码器将返回 null（无位图），但 out... 字段仍将被设置，允许调用者查询位图而无需为其像素分配内存。
* `inSampleSize`: 如果设置为 > 1 的值，则请求解码器对原始图像进行子采样，返回较小的图像以节省内存。 样本大小是与解码位图中的单个像素相对应的任一维度中的像素数。 例如，inSampleSize == 4 返回的图像是原始宽度/高度的 1/4，像素数的 1/16。 任何 <= 1 的值都被视为与 1 相同。注意：解码器使用基于 2 的幂的最终值，任何其他值将向下舍入为最接近的 2 的幂。
* `inTargetDensity`: 此 bitmap 将绘制到的目标的像素密度。 这与 inDensity 和 inScaled 结合使用，以确定在返回位图之前是否以及如何缩放位图。
* `inDensity`: 用于 bitmap 的像素密度。这将始终导致返回的 bitmap 具有为其设置的密度。此外，如果设置了 inScaled（默认设置）并且此密度与 inTargetDensity 不匹配，则位图将在返回之前缩放到目标密度。
* `outColorSpace`: 解码位图将具有的颜色空间。 请注意，output 颜色空间不能保证是位图编码的颜色空间。 如果未知（例如，当配置为 Bitmap.Config.ALPHA_8 时），或出现错误，则将其设置为空。
* `outConfig`
* `outHeight`: 位图的结果高度。 如果 inJustDecodeBounds 设置为 false，这将是应用任何缩放后输出位图的高度。 如果为真，它将是输入图像的高度，不考虑缩放。
* `outMimeType`
* `outWidth`


## BitmapRegionDecoder
`BitmapRegionDecoder` 可用于从图像中解码矩形区域。 当原始图像很大并且只需要图像的一部分时，BitmapRegionDecoder 特别有用。
要创建 BitmapRegionDecoder，调用 newInstance(...)。 给定一个 BitmapRegionDecoder，用户可以重复调用 decodeRegion() 以获得指定区域的解码位图。


## Bitmap 内存占用
Bitmap 内存占用大小 = Bitmap.width * Bitmap.height * 像素点大小

像素点大小由 Bitmap.Config 的值控制。


