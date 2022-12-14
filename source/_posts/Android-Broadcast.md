---
title: Android - Broadcast
date: 2022-09-04 11:50:49
tags: Android
---

应用可以发送或接收来自系统和其他应用的广播消息。当感兴趣的事件发生时发送这些广播。例如，系统会在各种系统事件发生时发送广播，例如系统启动时或设备开始充电时。应用程序还可以发送自定义广播，例如，通知其他应用程序他们可能感兴趣的内容（例如，一些新数据已下载）。

应用程序可以注册以接收特定广播。发送广播时，系统会自动将广播路由到已订阅接收该特定类型广播的应用程序。

一般来说，广播可以用作跨应用程序和正常用户流之外的消息传递系统。但是，必须小心不要滥用响应广播和在后台运行job的机会，这可能会导致系统性能下降。

# 关于系统广播
当各种系统事件发生时，系统会自动发送广播，例如当系统切换到飞行模式时。系统广播被发送到所有订阅接收事件的应用程序。

广播消息本身包装在一个 Intent 对象中，该对象的 action 字符串标识发生的事件（例如 `android.intent.action.AIRPLANE_MODE`）。intent还可能包括捆绑到其 extra 字段中的附加信息。例如，飞行模式intent包括一个布尔 extra，指示飞行模式是否打开。

有关系统广播操作的完整列表，请参阅 Android SDK 中的 `BROADCAST_ACTIONS.TXT` 文件。每个广播动作都有一个与之关联的常量字段。例如，常量 ACTION_AIRPLANE_MODE_CHANGED 的值为 android.intent.action.AIRPLANE_MODE。每个广播动作的文档都在其关联的常量字段中可用。

## 系统广播的变化
#### Android 9
从 Android 9（API 级别 28）开始，NETWORK_STATE_CHANGED_ACTION 广播不会接收有关用户位置或个人身份数据的信息。

此外，如果您的应用安装在运行 Android 9 或更高版本的设备上，则来自 Wi-Fi 的系统广播不包含 SSID、BSSID、连接信息或扫描结果。 要获取此信息，请改为调用 getConnectionInfo()。

#### Android 8.0
从 Android 8.0（API 级别 26）开始，系统对清单声明的接收器施加了额外的限制。

如果应用面向 Android 8.0 或更高版本，则不能使用清单为大多数隐式广播（不专门针对您的应用的广播）声明接收器。 当用户积极使用您的应用程序时，您仍然可以使用上下文注册的接收器。


# 接收广播
应用程序可以通过两种方式接收广播：通过清单声明的接收器和上下文注册的接收器。

### Manifest-declared receivers
如果在清单中声明broadcast receiver，则系统会在发送广播时启动应用程序（如果应用程序尚未运行）。
注意：如果应用以 API 级别 26 或更高级别为目标，则不能使用清单为隐式广播（不专门针对您的应用的广播）声明 receiver，但一些不受该限制的隐式广播除外。 
在大多数情况下，可以改用 scheduled jobs。

要在清单中声明broadcast receiver，执行以下步骤：
1. 在应用的清单中指定 `<receiver>` 元素。
    ```
   <receiver android:name=".MyBroadcastReceiver"  android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
        <action android:name="android.intent.action.INPUT_METHOD_CHANGED" />
    </intent-filter>
    </receiver>
   ```
   intent filters 指定receiver订阅的广播action。
2. 继承 BroadcastReceiver 并实现 onReceive(Context, Intent)。

系统package manager在安装应用程序时注册receiver。 然后receiver成为应用程序的单独入口点，这意味着如果应用程序当前未运行，系统可以启动应用程序并传送广播。

系统创建一个新的 `BroadcastReceiver` 组件对象来处理它接收到的每个广播。 此对象仅在调用 `onReceive(Context, Intent)` 期间有效。 
一旦代码从此方法返回，系统就会认为该组件不再处于活动状态。


### Context-registered receivers
1. 创建 BroadcastReceiver 对象
2. 创建一个 `IntentFilter` 并通过调用 `registerReceiver(BroadcastReceiver, IntentFilter)` 注册receiver
   
   注意：要注册本地广播，请改为调用 LocalBroadcastManager.registerReceiver(BroadcastReceiver, IntentFilter)。

   只要注册上下文有效，上下文注册的receiver就会接收广播。 例如，如果在 Activity 上下文中注册，只要该 Activity 未被销毁，就会收到广播。 
   如果在应用程序上下文中注册，只要应用程序正在运行，就会收到广播。

3. 要停止接收广播，请调用 `unregisterReceiver(android.content.BroadcastReceiver)`。 当不再需要receiver或上下文不再有效时，请务必取消注册receiver。
   请注意注册和取消注册接收器的地方，例如，如果使用activity的上下文在 `onCreate(Bundle)` 中注册接收器，则应在 `onDestroy()` 中取消注册它以防止将receiver泄漏到activity上下文之外。 
   如果你在 `onResume()` 中注册了一个接收器，你应该在 `onPause()` 中取消注册它以防止它被多次注册（如果不想在暂停时接收广播，这可以减少不必要的系统开销）。 
   不要在 onSaveInstanceState(Bundle) 中取消注册，因为如果用户移回历史堆栈，则不会调用它。


### 对进程状态的影响
`BroadcastReceiver` 的状态（无论它是否正在运行）会影响其包含进程的状态，这反过来又会影响其被系统杀死的可能性。
例如，当一个进程执行一个receiver（即当前正在运行其 onReceive() 方法中的代码）时，它被认为是一个前台进程。系统保持进程运行，除非在内存压力很大的情况下。

但是，一旦您的代码从 `onReceive()` 返回，BroadcastReceiver 就不再处于活动状态。receiver的宿主进程变得与运行在其中的其他应用程序组件一样重要。
如果该进程仅托管一个清单声明的receiver（用户从未或最近未与之交互的应用程序的常见情况），则在从 `onReceive()` 返回时，系统将其进程视为低优先级进程，并且可能杀死它以使资源可用于其他更重要的进程。

因此，不应从广播receiver启动长时间运行的后台线程。在 `onReceive()` 之后，系统可以随时终止进程以回收内存，并在此过程中终止进程中运行的衍生线程。
为避免这种情况，应该调用 `goAsync()`（如果想要更多时间在后台线程中处理广播）或使用 `JobScheduler` 从receiver调度 `JobService`，以便系统知道该进程继续执行活动工作。

以下代码片段显示了一个 BroadcastReceiver，它使用 goAsync() 来标记它在 onReceive() 完成后需要更多时间来完成。如果您要在 onReceive() 中完成的工作足够长以导致 UI 线程错过一帧（>16 毫秒），这将特别有用，使其更适合后台线程。
```kotlin
class MyBroadcastReceiver : BroadcastReceiver() {
    private val scope = CoroutineScope(SupervisorJob())

    override fun onReceive(context: Context, intent: Intent) {
        val pendingResult: PendingResult = goAsync()

        scope.launch(Dispatchers.Default) {
            try {
                // Do some background work
                buildString {
                    append("Action: ").append(intent.action).append("\n")
                    append("URI: ").append(intent.toUri(Intent.URI_INTENT_SCHEME)).append("\n")
                }.also { log ->
                    Log.d(TAG, log)
                }
            } finally {
                // Must call finish() so the BroadcastReceiver can be recycled
                pendingResult.finish()
            }
        }
    }

    companion object {
        private const val TAG = "MyBroadcastReceiver"
    }
}
```


# 发送广播
Android 提供了三种应用发送广播的方式：
* `sendOrderedBroadcast(Intent, String)` 方法一次向一个receiver发送广播。当每个receiver依次执行时，它可以将结果传播给下一个receiver，或者它可以完全中止广播，
   这样它就不会被传递给其他receiver。运行的order receiver可以通过匹配intent-filter的`android:priority`属性来控制；具有相同优先级的接收器将以任意顺序运行。
* `sendBroadcast(Intent)` 方法以未定义的顺序向所有接收者发送广播。这称为正常广播。这更有效，但意味着receiver无法从其他receiver读取结果、传播从广播接收到的数据或中止广播。
* `LocalBroadcastManager.sendBroadcast` 方法将广播发送到与发送者在同一应用程序中的receiver。如果不需要跨应用发送广播，请使用本地广播。该实现更加高效（不需要进程间通信），无需担心与其他应用程序能够接收或发送您的广播相关的任何安全问题。

```kotlin
Intent().also { intent ->
    intent.setAction("com.example.broadcast.MY_NOTIFICATION")
    intent.putExtra("data", "Nothing to see here, move along.")
    sendBroadcast(intent)
}
```

广播消息被包装在一个 Intent 对象中。 intent的action字符串必须提供应用程序的 Java 包名称语法并唯一标识广播事件。 
可以使用 `putExtra(String, Bundle)` 将附加信息附加到intent。 还可以通过在 Intent 上调用 `setPackage(String)` 将广播限制为同一组织中的一组应用程序。

# 使用权限限制广播
权限允许将广播限制为拥有某些权限的应用程序集。 可以对广播的发送者或接收者实施限制。

## Sending with permissions
在调用 `sendBroadcast(Intent, String)` 或 `sendOrderedBroadcast(Intent, String, BroadcastReceiver, Handler, int, String, Bundle)` 时，可以指定权限参数。 
只有在清单中使用标签请求许可的接收者（如果有危险，随后被授予许可）才能接收广播。 例如，以下代码发送广播：
```kotlin
sendBroadcast(Intent("com.example.NOTIFY"), Manifest.permission.SEND_SMS)
```
要接收广播，接收应用程序必须请求权限，如下所示：
```
<uses-permission android:name="android.permission.SEND_SMS"/>
```
可以指定现有的系统权限（如 `SEND_SMS`）或使用 `<permission>` 元素定义自定义权限。

注意：自定义权限是在安装应用程序时注册的。 定义自定义权限的应用程序必须在使用它的应用程序之前安装。

## Receiving with permissions
如果在注册广播receiver时指定权限参数（使用 `registerReceiver(BroadcastReceiver, IntentFilter, String, Handler)` 或在清单中的 `<receiver>` 标记中），
则只有使用 `<uses-permission>` 请求权限的广播者 清单中的标签（如果危险，随后被授予许可）可以向receiver发送 Intent。
```
<receiver android:name=".MyBroadcastReceiver"
          android:permission="android.permission.SEND_SMS">
    <intent-filter>
        <action android:name="android.intent.action.AIRPLANE_MODE"/>
    </intent-filter>
</receiver>


var filter = IntentFilter(Intent.ACTION_AIRPLANE_MODE_CHANGED)
registerReceiver(receiver, filter, Manifest.permission.SEND_SMS, null )
```

# 安全注意事项和最佳实践
* 如果您不需要向应用程序外部的组件发送广播，请使用支持库中提供的 LocalBroadcastManager 发送和接收本地广播。 LocalBroadcastManager 效率更高（不需要进程间通信），并且允许您避免考虑与其他应用程序能够接收或发送您的广播相关的任何安全问题。 本地广播可以用作您的应用程序中的通用发布/订阅事件总线，而无需任何系统范围广播的开销。
* 如果许多应用程序在其清单中注册接收相同的广播，则可能导致系统启动大量应用程序，从而对设备性能和用户体验造成重大影响。 为了避免这种情况，最好使用上下文注册而不是清单声明。 有时，Android 系统本身会强制使用上下文注册的接收器。 例如，CONNECTIVITY_ACTION 广播仅传递给上下文注册的接收器。
* 不要使用隐含的意图广播敏感信息。 任何注册接收广播的应用程序都可以读取该信息。有三种方法可以控制谁可以接收您的广播：
    * 在发送广播时指定权限。
    * 在 Android 4.0 及更高版本中，您可以在发送广播时使用 setPackage(String) 指定一个包。 系统将广播限制为与包匹配的应用程序集。
    * 使用 LocalBroadcastManager 发送本地广播。
* 当注册接收器时，任何应用都可以向您应用的接收器发送潜在的恶意广播。 有三种方法可以限制您的应用接收的广播：
    * 在注册广播接收器时指定权限。
    * 对于清单声明的接收者，在清单中将 android:exported 属性设置为“false”。 接收方不接收来自应用程序外部来源的广播。
    * 使用 LocalBroadcastManager 将自己限制为仅本地广播。
* 广播action的命名空间是全局的。 确保动作名称和其他字符串写在您拥有的命名空间中，否则您可能会无意中与其他应用程序发生冲突。
* 因为接收者的 `onReceive(Context, Intent)` 方法在主线程上运行，所以它应该快速执行并返回。 如果您需要执行长时间运行的工作，请注意生成线程或启动后台服务，因为系统可以在 onReceive() 返回后终止整个进程。
* 不要从广播接收器开始activity，因为用户体验不和谐； 特别是如果有不止一个接收器。 相反，请考虑显示通知。







