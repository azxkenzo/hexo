---
title: Android - AIDL
date: 2022-08-31 21:01:48
tags: Android
---

# AIDL 是什么
Android Interface Definition Language (AIDL) 允许定义客户端和服务都同意的编程接口，以便使用进程间通信 (IPC) 相互通信。 
在 Android 上，一个进程不能正常访问另一个进程的内存。 所以说，他们需要将他们的对象分解为操作系统可以理解的原语，并将对象编组跨越该边界。

注意：仅当允许来自不同应用程序的客户端访问您的 IPC 服务并希望在您的服务中处理多线程时，才需要使用 AIDL。 
如果不需要跨不同应用程序执行并发 IPC，则应通过实现 Binder 创建接口，或者如果想执行 IPC 但不需要处理多线程，则使用 Messenger 实现接口。

在开始设计 AIDL 接口之前，请注意对 AIDL 接口的调用是直接函数调用。 不应该对发生调用的线程做出假设。 根据调用是来自本地进程中的线程还是远程进程，发生的情况会有所不同。 具体来说：

* 从本地进程进行的调用在进行调用的同一线程中执行。 如果这是主 UI 线程，则该线程将继续在 AIDL 接口中执行。 
如果它是另一个线程，那就是在服务中执行代码的线程。 因此，如果只有本地线程正在访问服务，可以完全控制在其中执行哪些线程（但如果是这种情况，那么根本不应该使用 AIDL，而应该通过实现 Binder 创建接口 ）。

* 来自远程进程的调用是从平台在您自己的进程内部维护的线程池中分派的。 您必须为来自未知线程的传入呼叫做好准备，同时发生多个呼叫。 
换句话说，AIDL 接口的实现必须是完全线程安全的。 从同一远程对象上的一个线程发出的调用按顺序到达接收端。

* oneway 关键字修改远程调用的行为。 使用时，远程调用不会阻塞； 它只是发送交易数据并立即返回。 接口的实现最终从Binder线程池接收这个作为常规调用作为常规远程调用。 
如果 oneway 与本地调用一起使用，则没有影响，调用仍然是同步的。


# 定义 AIDL 接口
必须使用 Java 语法在 .aidl 文件中定义 AIDL 接口，然后将其保存在托管服务的应用程序和绑定到服务的任何其他应用程序的源代码中（在 src/ 目录中）。

在构建每个包含 .aidl 文件的应用程序时，Android SDK 工具会根据 .aidl 文件生成一个 IBinder 接口，并将其保存在项目的 gen/ 目录中。 服务必须适当地实现 IBinder 接口。 然后客户端应用程序可以绑定到服务并从 IBinder 调用方法来执行 IPC。

要使用 AIDL 创建 bounded 服务，请执行以下步骤：
1. 创建 .aidl 文件。  该文件定义了带有方法签名的编程接口。
2. 实现接口。Android SDK 工具会根据您的 .aidl 文件以 Java 编程语言生成接口。 此接口有一个名为 Stub 的内部抽象类，它扩展 Binder 并实现 AIDL 接口中的方法。 您必须扩展 Stub 类并实现方法。
3. 向客户端公开接口。实现一个 Service 并覆盖 onBind() 以返回 Stub 类的实现。

### 创建 .aidl 文件
AIDL 使用一种简单的语法，允许使用一个或多个可以接受参数和返回值的方法来声明接口。 参数和返回值可以是任何类型，甚至是其他 AIDL 生成的接口。

必须使用 Java 语言构建 .aidl 文件。 每个 .aidl 文件必须定义一个接口，并且只需要接口声明和方法签名。

默认情况下，AIDL 支持以下数据类型：

* Java 中的所有基本类型（如 int、long、char、boolean 等）
* 原始类型数组，例如 int[]
* String
* CharSequence
* List  
    List 中的所有元素必须是此列表中受支持的数据类型之一，或者是您声明的其他 AIDL 生成的接口或 parcelables 之一。 List 可以选择用作参数化类型类（例如，List<String>）。 对方接收到的实际具体类始终是一个 ArrayList，尽管生成该方法是为了使用 List 接口。
* Map  
    Map 中的所有元素必须是此列表中支持的数据类型之一，或者是您声明的其他 AIDL 生成的接口或 parcelables 之一。 不支持参数化类型映射（例如 Map<String,Integer> 形式的映射）。 对方接收到的实际具体类始终是HashMap，虽然生成方法是为了使用Map接口。 考虑使用 Bundle 作为 Map 的替代品。

必须为上面未列出的每个附加类型包含一个 import 语句，即使它们与您的接口在同一个包中定义。

定义服务接口时，请注意：

* 方法可以接受零个或多个参数，并返回一个值或 void。
* 所有非原始参数都需要一个方向标签来指示数据的走向。 in、out或inout。  
  警告：应该将方向限制为真正需要的方向，因为编组参数很昂贵。
* .aidl 文件中包含的所有代码注释都包含在生成的 IBinder 接口中（除了 import 和 package 语句之前的注释）。
* 可以在 AIDL 接口中定义 String 和 int 常量。 例如：const int VERSION = 1;。
* 方法调用由 transact() 代码分派，该代码通常基于接口中的方法索引。 因为这使得版本控制变得困难，您可以手动将事务代码分配给一个方法：void method() = 10;。
* 可以为 Null 的参数和返回类型必须使用 @nullable 进行注释。

只需将 .aidl 文件保存在项目的 src/ 目录中，当构建应用程序时，SDK 工具会在项目的 gen/ 目录中生成 IBinder 接口文件。 生成的文件名与 .aidl 文件名匹配，但具有 .java 扩展名（例如，IRemoteService.aidl 生成 IRemoteService.java）。

### 实现接口
当您构建应用程序时，Android SDK 工具会生成一个以您的 .aidl 文件命名的 .java 接口文件。 生成的接口包括一个名为 Stub 的子类，它是其父接口的抽象实现，并声明了 .aidl 文件中的所有方法。

注意：Stub 还定义了一些辅助方法，最值得注意的是 asInterface()，它接受一个 IBinder（通常是传递给客户端的 onServiceConnected() 回调方法的那个）并返回一个 stub 接口的实例。

在实现 AIDL 接口时，您应该注意一些规则：

* 传入调用不能保证在主线程上执行，因此您需要从一开始就考虑多线程并正确构建您的服务以实现线程安全。
* 默认情况下，RPC 调用是同步的。 如果您知道该服务需要多于几毫秒来完成一个请求，您不应该从 Activity 的主线程调用它，因为它可能会挂起应用程序（Android 可能会显示“应用程序未响应”对话框）——您应该 通常从客户端中的单独线程调用它们。
* 只有 Parcel.writeException() 的参考文档中列出的异常类型会被发送回调用者。

客户端还必须能够访问接口类，因此如果客户端和服务位于不同的应用程序中，那么客户端的应用程序必须在其 src/ 目录中拥有 .aidl 文件的副本（生成 android.os.Binder 接口） — 提供客户端访问 AIDL 方法）。

当客户端在 onServiceConnected() 回调中收到 IBinder 时，必须调用 YourServiceInterface.Stub.asInterface(service) 将返回的参数转换为 YourServiceInterface 类型。

### 通过 IPC 传递对象
在 Android 10（API 级别 29）中，可以直接在 AIDL 中定义 Parcelable 对象。 这里也支持作为 AIDL 接口参数和其他 parcelables 支持的类型。 这避免了手动编写编组代码和自定义类的额外工作。 但是，这也会创建一个裸结构。 如果需要自定义访问器或其他功能，则应实现 Parcelable。

您还可以通过 IPC 接口将自定义类从一个进程发送到另一个进程。 但是，您必须确保您的类的代码可用于 IPC 通道的另一端，并且您的类必须支持 Parcelable 接口。 支持 Parcelable 接口很重要，因为它允许 Android 系统将对象分解为可以跨进程编组的基元。


Stub类的声明：
```java
public static abstract class Stub extends Binder implements IRemoteService
```

Stub.asInterface() 的实现：
```java
public static IRemoteService asInterface(IBinder obj) {
  if (obj==null) {
    return null;
  }
  
  IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
  if ((iin!=null) && (iin instanceof IRemoteService)) {
    return ((IRemoteService)iin);
  }
  // 
  return new IRemoteService.Stub.Proxy(obj);
}

public @Nullable IInterface queryLocalInterface(@NonNull String descriptor) {
    if (mDescriptor != null && mDescriptor.equals(descriptor)) {
        return mOwner;
    }
    return null;
}
```

Stub.Proxy 中方法的实现：
```java
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
try {
  _data.writeInterfaceToken(DESCRIPTOR);
  _data.writeInt(anInt);
  _data.writeLong(aLong);
  _data.writeInt(((aBoolean)?(1):(0)));
  _data.writeFloat(aFloat);
  _data.writeDouble(aDouble);
  _data.writeString(aString);
  // 
  boolean _status = mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
  if (!_status && getDefaultImpl() != null) {
    getDefaultImpl().basicTypes(anInt, aLong, aBoolean, aFloat, aDouble, aString);
    return;
  }
  _reply.readException();
} finally {
  _reply.recycle();
  _data.recycle();
}
```

Binder.transact() 的实现：
```java
public final boolean transact(int code, @NonNull Parcel data, @Nullable Parcel reply,
        int flags) throws RemoteException {
    if (false) Log.v("Binder", "Transact: " + code + " to " + this);

    if (data != null) {
        data.setDataPosition(0);
    }
    boolean r = onTransact(code, data, reply, flags);
    if (reply != null) {
        reply.setDataPosition(0);
    }
    return r;
}
```

Stub.onTransact() 的实现：
```java
String descriptor = DESCRIPTOR;
switch (code)
{
  case INTERFACE_TRANSACTION:
  {
    reply.writeString(descriptor);
    return true;
  }
  case TRANSACTION_basicTypes:
  {
    data.enforceInterface(descriptor);
    int _arg0;
    _arg0 = data.readInt();
    long _arg1;
    _arg1 = data.readLong();
    boolean _arg2;
    _arg2 = (0!=data.readInt());
    float _arg3;
    _arg3 = data.readFloat();
    double _arg4;
    _arg4 = data.readDouble();
    String _arg5;
    _arg5 = data.readString();
    // 
    this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
    reply.writeNoException();
    return true;
  }
  default:
  {
    return super.onTransact(code, data, reply, flags);
  }
}
```