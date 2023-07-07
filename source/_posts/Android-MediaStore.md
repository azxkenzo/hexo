---
title: Android - MediaStore
date: 2023-03-28 17:33:10
tags: Android 
---


系统自动扫描外部存储卷并将媒体文件添加到以下定义明确的集合中：
* **Images**，包括照片和屏幕截图，它们存储在 `DCIM/` 和 `Pictures/` 目录中。 系统将这些文件添加到 `MediaStore.Images` 表中。
* **Videos**，它们存储在 `DCIM/`、`Movies/` 和 `Pictures/` 目录中。 系统将这些文件添加到 `MediaStore.Video` 表中。
* **Audio files**，它们存储在 `Alarms/`、`Audiobooks/`、`Music/`、`Notifications/`、`Podcasts/` 和 `Ringtones/` 目录中。 此外，系统可以识别 `Music/` 或 `Movies/` 目录中的音频播放列表，以及 `Recordings/` 目录中的录音。 系统将这些文件添加到 `MediaStore.Audio` 表中。 recordings 目录在 Android 11（API 级别 30）及更低版本上不可用。
* **Downloaded files**，它们存储在 `Download/` 目录中。 在运行 Android 10（API 级别 29）及更高版本的设备上，这些文件存储在 `MediaStore.Downloads` 表中。 此表不适用于 Android 9（API 级别 28）及更低版本。

media store 还包括一个名为 `MediaStore.Files` 的集合。 其内容取决于应用是否使用分区存储，适用于面向 Android 10 或更高版本的应用：
* 如果启用分区存储，则集合仅显示您的应用创建的照片、视频和音频文件。 大多数开发人员不需要使用 MediaStore.Files 查看来自其他应用程序的媒体文件，但如果您有特定要求，您可以声明 READ_EXTERNAL_STORAGE 权限。 但是，建议您使用 MediaStore API 打开您的应用程序尚未创建的文件。
* 如果分区存储不可用或未被使用，则该集合会显示所有类型的媒体文件。


## 请求必要权限
### 如果只访问自己的媒体文件，则不需要任何权限
在运行 Android 10 或更高版本的设备上，不需要任何与存储相关的权限来访问和修改您的应用拥有的媒体文件，包括 MediaStore.Downloads 集合中的文件。 例如，如果您正在开发相机应用程序，则无需请求与存储相关的权限，因为您的应用程序拥有您正在写入媒体存储的图像。

### 访问其他应用程序的媒体文件
要访问其他应用创建的媒体文件，必须声明适当的存储相关权限，并且这些文件必须位于以下媒体集合之一中：
* `MediaStore.Images`
* `MediaStore.Video`
* `MediaStore.Audio`

只要文件可从 `MediaStore.Images`、`MediaStore.Video` 或 `MediaStore.Audio` 查询查看，它也可以使用 `MediaStore.Files` 查询查看。

以下代码片段演示了如何声明适当的存储权限：
```
<!-- Required only if your app needs to access images or photos
     that other apps created. -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />

<!-- Required only if your app needs to access videos
     that other apps created. -->
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />

<!-- Required only if your app needs to access audio files
     that other apps created. -->
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />

<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>

<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
                 android:maxSdkVersion="29" />
```

如果同时请求 `READ_MEDIA_IMAGES` 权限和 `READ_MEDIA_VIDEO` 权限，系统会显示一个包含这两个权限的运行时权限对话框。


## 检查 media store 的更新
要更可靠地访问媒体文件，特别是如果您的应用缓存 URI 或来自 media store 的数据，请检查 media store 版本与上次同步媒体数据时相比是否发生了变化。 要执行此更新检查，请调用 `MediaStore.getVersion()`。 返回的版本是一个唯一的字符串，只要媒体存储发生重大变化，它就会发生变化。 如果返回的版本与上次同步的版本不同，请重新扫描并重新同步您应用的媒体缓存。

在应用进程启动时完成此检查。 每次查询 media store 时都无需检查版本。

注意：media store 版本号不会因应用端更改而改变，例如当应用添加媒体文件时。 有一种单独的方法可以帮助您检测媒体文件的更新。


## 检测媒体文件的更新
与之前的时间点相比，应用可能需要识别包含应用添加或修改的媒体文件的 storage volume。 要最可靠地检测这些更改，请将感兴趣的 storage volume 传递给 `MediaStore.getGeneration()`。 只要 media store 版本不变，此方法的返回值就会随时间单调增加。

特别是，`getGeneration()` 比媒体列中的日期更可靠，例如 DATE_ADDED 和 DATE_MODIFIED。 这是因为当应用程序调用 `setLastModified()` 或用户更改系统时钟时，这些媒体列值可能会更改。

```
1:
1402:aec29684-646c-4b9c-9c77-750ea496e117
1402:aec29684-646c-4b9c-9c77-750ea496e117
2940796

2:
1402:aec29684-646c-4b9c-9c77-750ea496e117
1402:aec29684-646c-4b9c-9c77-750ea496e117
2940796

拍照后：
1402:aec29684-646c-4b9c-9c77-750ea496e117
1402:aec29684-646c-4b9c-9c77-750ea496e117
2940805

删除后：
1402:aec29684-646c-4b9c-9c77-750ea496e117
1402:aec29684-646c-4b9c-9c77-750ea496e117
2940813
```

## 访问媒体内容时的注意事项
### Cached data
如果您的应用缓存 media store 的 URI 或数据，请定期检查 media store 的更新。 此检查允许您的应用程序端缓存数据与系统端提供程序数据保持同步。

### Performance
当您使用直接文件路径执行媒体文件的顺序读取时，性能可与 MediaStore API 相媲美。

但是，当您使用直接文件路径对媒体文件执行随机读取和写入时，该过程可能会慢两倍。 在这些情况下，我们建议改用 MediaStore API。

### DATA column
当您访问现有媒体文件时，您可以在逻辑中使用 DATA 列的值。 那是因为这个值有一个有效的文件路径。 但是，不要假设该文件始终可用。 准备好处理可能发生的任何基于文件的 I/O 错误。

另一方面，要创建或更新媒体文件，请不要使用 DATA 列的值。 相反，使用 DISPLAY_NAME 和 RELATIVE_PATH 列的值。

### Storage volumes
以 Android 10 或更高版本为目标的应用程序可以访问系统分配给每个外部存储卷的唯一名称。 此命名系统可帮助您有效地组织和索引内容，并让您控制新媒体文件的存储位置。

记住以下几卷特别有用：
* `VOLUME_EXTERNAL` 卷提供设备上所有共享存储卷的视图。 您可以阅读此合成卷的内容，但不能修改其中的内容。
* `VOLUME_EXTERNAL_PRIMARY` 卷表示设备上的主要共享存储卷。 您可以阅读和修改本卷的内容。

可以通过调用 `MediaStore.getExternalVolumeNames()` 发现其他卷




