---
title: Android 面试
date: 2023-04-18 10:54:14
tags:
---

# JVM

### JVM的结构

### volatile

### 内存模型

### GC





# Android

### 绘制流程

### 同步消息屏障
ViewRootImpl 将遍历view树包装成一个Runnable并抛到Choreographer， 在抛之前会向主线程消息队列中抛同步屏障

同步屏障也是一个Message，只不过 target 等于null

取下一条message的算法中，若遇到同步屏障，则会越过同步消息，向后遍历找第一条异步消息找到则返回（Choreographer抛的异步消息），若没有找到则会执行epoll挂起

当执行到遍历View树的 runnable时，ViewRootImpl会移除同步屏障

### Choreographer
将和ui相关的任务与vsync同步的一个类。

每个任务被抽象成CallbackRecord，同类任务按时间先后顺序组成一条任务链CallbackQueue。四条任务链存放在mCallbackQueues[]数组结构中

触摸事件，动画，View树遍历都会被抛到编舞者，并被包装成CallbackRecord并存入链式数组结构，当Choreographer收到一个Vsync就会依次从输入，动画，绘制这些链中取出任务执行

当vsync到来时，会向主线程抛异步消息（执行doFrame）并带上消息生成时间，当异步消息被执行时，从任务链上摘取所有以前的任务，并按时间先后顺序逐个执行。

### 绘制模型
1.软件绘制 2.硬件加速绘制

### 应用冷启动流程

### Surface
它是原始图像缓冲区的一个句柄。即raw buffer的内存地址，raw buffer是保存像素数据的内存区域，通过Surface的canvas 可以将图像数据写入这个缓冲区

Surface类是使用一种称为双缓冲的技术来渲染

这种双缓冲技术需要两个图形缓冲区GraphicBuffer，其中一个称为前端缓冲区frontBuffer，另外一个称为后端缓冲区backBuffer。前端缓冲区是正在渲染的图形缓冲区，而后端缓冲区是接下来要渲染的图形缓冲区，当vsync到来时，交换前后缓冲区的指针

### SurfaceFlinger
SurfaceFlinger 是由 init 进程启动的运行在底层的一个系统进程，它的主要职责是合成和渲染多个Surface，并向目标进程发送垂直同步信号 VSync,并在 vsync 产生时合成帧到frame buffer

### Surfaceview
一个嵌入View树的独立绘制表面，他位于宿主Window的下方，通过在宿主canvas上绘制透明区域来显示自己

虽然它在界面上隶属于view hierarchy，但在WMS及SurfaceFlinger中都是和宿主窗口分离的，它拥有独立的绘制表面，绘制表面在app进程中表现为Surface对象，在系统进程中表现为在WMS中拥有独立的WindowState，在SurfaceFlinger中拥有独立的Layer，而普通view和其窗口拥有同一个绘制表面

### TextureView
SurfaceView 拥有独立的绘制表面，而TextureView和View树共享绘制表面

TextureView通过观察BufferQueue中新的SurfaceTexture到来，然后调用invalidate触发View树重绘，如果有View叠加在TextureView上面，它们的脏区有交集，则会触发不必要的重绘，所以他的刷新操作比SurfaceView更重


### 界面卡顿

### requestLayout() vs invalidate()

### Binder

### 进程通信方式(IPC)

### Bundle

### Parcel

### 持久化方式

### Service

### 广播

### 消息机制

### 触摸事件

### 滑动冲突

### recyclerview

### view生命周期

### Bitmap

### ANR

### Lifecycle

### 恢复数据
onSaveInstanceState()+onRestoreInstanceState():会进行序列化到磁盘，耗时，杀进程依然存在

Fragment+setRetainInstance()：数据保存在内存，配置发生变化时数据依然存在，但杀进程后数据不存在

### 进程优先级

### LruCache

### 启动优化

### LiveData

### ViewModel

### 编译打包流程

### launch mode


# Java

### ArrayMap & HashMap

### SparseArray & HashMap

### 泛型

### java对象生命周期

### 类加载

### 类构造顺序

### HashMap

### WeakHashMap

### LinkedHashMap

### ThreadLocal

### 内存泄漏

### 引用

### 异常

### 注解

### OOM类型

### 内存优化

### equals()

### hashCode()












