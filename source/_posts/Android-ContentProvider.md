---
title: Android - ContentProvider
date: 2022-09-03 08:58:25
tags: Android
---

# ContentProvider 是什么
ContentProvider可以帮助应用程序管理对自己存储的、由其他应用程序存储的数据的访问，并提供一种与其他应用程序共享数据的方式。 
它们封装数据，并提供定义数据安全性的机制。 ContentProvider是将一个进程中的数据与另一个进程中运行的代码连接起来的标准接口。

# 基础

### Content URI
**Content URI** 是标识 provider 中数据的 URI。 Content URI 包括整个provider的符号名称（其**authority**）和指向表的名称（**path**）。 
当调用客户端方法来访问provider中的表时，表的Content URI 是参数之一。

`ContentResolver` 对象解析出 URI 的authority，并使用它通过将authority与已知provider的系统表进行比较来“解析”provider。 
然后 ContentResolver 可以将查询参数分派给正确的provider。

ContentProvider 使用Content URI 的path部分来选择要访问的表。 provider通常为其公开的每个表都有一个**path**。

字符串 `content://`（**scheme**）始终存在，并将其标识为Content URI。

注意： Uri 和 Uri.Builder 类包含用于从字符串构造格式良好的 URI 对象的便捷方法。 ContentUris 类包含用于将 id 值附加到 URI 的便捷方法。

## 从 provider 获取数据
要从provider检索数据，请执行以下基本步骤：
1. 请求provider的读取访问权限。
2. 定义向provider发送查询的代码。

### 请求读取权限
要从provider检索数据，应用程序需要provider的“读取访问权限”。不能在运行时请求此权限；相反，必须使用 `<uses-permission>` 元素和provider定义的确切权限名称在清单中指定需要此权限。
当在清单中指定此元素时，实际上是在为应用程序“请求”此权限。当用户安装应用程序时，他们会隐式授予此请求。

### 构造查询
provider客户端查询类似于 SQL 查询，它包含一组要返回的列、一组选择条件和一个排序顺序。

查询应返回的列集称为 **projection**。

指定要检索的行的表达式被拆分为选择子句和选择参数。 选择子句是逻辑和布尔表达式、列名和值的组合。 如果指定可替换参数`?` 而不是一个值，查询方法从选择参数数组中检索值。

#### 防止恶意输入
如果ContentProvider管理的数据在 SQL 数据库中，将外部不受信任的数据包含在原始 SQL 语句中可能会导致 SQL 注入。

```
// Constructs a selection clause by concatenating the user's input to the column name
var selectionClause = "var = $mUserInput"
```
如果这样做，就允许用户将恶意 SQL 连接到 SQL 语句中。 例如，用户可以输入“nothing; DROP TABLE *;” 对于 mUserInput，这将导致选择子句 `var = nothing; DROP TABLE *;`。 
由于选择子句被视为 SQL 语句，这可能会导致提供程序删除底层 SQLite 数据库中的所有表（除非provider设置为捕获 SQL 注入尝试）。

为避免此问题，请使用选择子句，该子句使用 `?` 作为可替换参数和单独的选择参数数组。 执行此操作时，用户输入将直接绑定到查询，而不是被解释为 SQL 语句的一部分。 
因为它不被当作 SQL，所以用户输入不能注入恶意 SQL。

### 从查询结果中获取数据
`Cursor` 实现包含几个“get”方法，用于从对象中检索不同类型的数据。它们还有一个 `getType()` 方法，该方法返回一个指示列的数据类型的值。

## Content provider permission
provider的应用程序可以指定其他应用程序必须具有的权限才能访问provider的数据。这些权限确保用户知道应用程序将尝试访问哪些数据。
根据provider的要求，其他应用程序请求他们需要的权限以访问provider。最终用户在安装应用程序时会看到请求的权限。

如果provider的应用程序未指定任何权限，则其他应用程序无权访问提供者的数据，除非provider is exported。此外，provider应用程序中的组件始终具有完全的读写访问权限，而不管指定的权限如何。

为了获得访问provider所需的权限，应用程序在其清单文件中使用 `<uses-permission>` 元素来请求它们。当 Android 包管理器安装应用程序时，用户必须批准应用程序请求的所有权限。如果用户同意所有这些，包管理器将继续安装；如果用户不同意，包管理器会中止安装。

## 插入、更新和删除数据

### 插入数据
要将数据插入provider，调用 `ContentResolver.insert()` 方法。 此方法将新行插入provider并返回该行的content URI。

要从返回的 Uri 中获取 _ID 的值，调用 `ContentUris.parseId()`。

### 更新数据
要更新行，可以像使用插入一样使用带有更新值的 ContentValues 对象，并像使用查询一样使用选择条件。 使用的客户端方法是 `ContentResolver.update()`。 
只需为要更新的列向 ContentValues 对象添加值。 如果要清除列的内容，请将值设置为 null。

### 删除数据
删除行类似于检索行数据：为要删除的行指定选择标准，客户端方法返回已删除行的数量。

## Provider 数据类型
Provider 为他们定义的每个content URI 维护 MIME 数据类型信息。可以使用 MIME 类型信息来确定应用程序是否可以处理Provider提供的数据，或者根据 MIME 类型选择一种处理类型。
当使用包含复杂数据结构或文件的provider时，通常需要 MIME 类型。例如，Contacts Provider 中的 ContactsContract.Data 表使用 MIME 类型来标记存储在每一行中的联系人数据的类型。
要获取与content URI 对应的 MIME 类型，调用 `ContentResolver.getType()`。

## provider访问的替代形式
provider访问的三种替代形式在应用程序开发中很重要：
* Batch access(批量访问)：可以使用 `ContentProviderOperation` 类中的方法创建批量访问调用，然后使用 `ContentResolver.applyBatch()` 应用它们。
* 异步查询：应该在单独的线程中进行查询。
* 通过intent访问数据：虽然不能直接向provider发送意图，但可以向provider的应用程序发送intent，这通常是修改provider数据的最佳设备。

### Batch access
对provider的批量访问对于插入大量行，或在同一方法调用中在多个表中插入行，或通常用于跨进程边界执行一组操作作为事务（原子操作）很有用。

要以“批处理模式”访问提供者，需要创建一个 `ContentProviderOperation` 对象数组，然后使用 `ContentResolver.applyBatch()` 将它们分派给content provider。 
将content provider的 authority 传递给此方法，而不是特定的content URI。 这允许数组中的每个 ContentProviderOperation 对象针对不同的表工作。 
对 ContentResolver.applyBatch() 的调用会返回一个结果数组。

### 通过intent访问数据
intent可以提供对content provider的间接访问。 即使应用程序没有访问权限，也允许用户访问provider中的数据，方法是从具有权限的应用程序返回结果intent，或者通过激活具有权限的应用程序并让用户在其中工作。

#### 使用临时权限获取访问权限
即使没有适当的访问权限，也可以访问content provider中的数据，方法是向具有权限的应用程序发送intent并接收包含“URI”权限的结果intent。 
这些是特定content URI 的权限，持续到接收它们的activity finish。 具有永久权限的应用程序通过在结果intent中设置标志来授予临时权限：
* Read permission: FLAG_GRANT_READ_URI_PERMISSION
* Write permission: FLAG_GRANT_WRITE_URI_PERMISSION

注意：这些标志不授予权限包含在content URI 中的provider的一般读取或写入访问权限。 访问仅针对 URI 本身。

当你将content URI 发送到另一个应用程序时，请至少包含这些标志中的一个。 这些标志为接收intent并以 Android 11（API 级别 30）或更高版本为目标的任何应用提供以下功能：
* 根据intent中包含的标志，读取或写入content URI 表示的数据。
* 获得对包含与 URI 权限匹配的content provider的应用程序的包可见性。 请注意，发送intent的应用程序和包含content provider的应用程序可能是两个不同的应用程序。

provider使用 `<provider>` 元素的 `android:grantUriPermission` 属性以及 `<provider>` 元素的 `<grant-uri-permission>` 子元素在其清单中定义content URI 的 URI 权限。

例如，即使没有 `READ_CONTACTS` 权限，也可以在 Contacts Provider 中检索联系人的数据。 可能希望在一个应用程序中执行此操作，该应用程序会在联系人生日时向其发送电子问候语。
与请求 READ_CONTACTS 相比，它允许访问所有用户的联系人及其所有信息，您更愿意让用户控制应用程序使用哪些联系人。 为此，使用以下过程：
1. 应用程序使用 `startActivityForResult()` 方法发送包含操作 `ACTION_PICK` 和“contacts”MIME 类型 `CONTENT_ITEM_TYPE` 的intent。
2. 因为这个intent与 People 应用程序的“selection”activity的intent过滤器相匹配，所以activity将出现在前台。
3. 在selection activity中，用户选择要更新的联系人。发生这种情况时，selection activity会调用 `setResult(resultcode, intent)` 来设置一个返回给您的应用程序的intent。
intent包含用户选择的联系人的content URI，以及“extras”标志 `FLAG_GRANT_READ_URI_PERMISSION`。这些标志向您的应用授予 URI 权限，以读取内容 URI 指向的联系人的数据。然后，selection activity调用 finish() 以将控制权返回给您的应用程序。
4. 您的activity返回到前台，系统调用您的activity的 `onActivityResult()` 方法。此方法接收由 People 应用程序中的selection activity创建的结果intent。
5. 使用结果intent中的content URI，可以从Contacts Provider读取联系人的数据，即使没有在清单中请求对provider的永久读取访问权限。
然后，可以获取联系人的生日信息或他们的电子邮件地址，然后发送电子贺卡。

#### 使用另一个应用程序
允许用户修改您没有访问权限的数据的一种简单方法是激活具有权限的应用程序并让用户在那里完成工作。

例如，日历应用程序接受一个 `ACTION_INSERT` intent，它允许您激活应用程序的insert UI。 可以在此intent中传递“extras”数据，应用程序使用这些数据来预填充 UI。 因为重复事件具有复杂的语法，将事件插入日历提供程序的首选方法是使用 ACTION_INSERT 激活日历应用程序，然后让用户在其中插入事件。


## 常量类
常量类定义了帮助应用程序使用content URI、列名、intent action和content provider的其他功能的常量。 常量类不会自动包含在provider中； 
provider的开发者必须定义它们，然后将它们提供给其他开发者。 Android 平台包含的许多provider在 android.provider 包中都有相应的常量类。


## MIME 类型参考
Content provider可以返回标准 MIME 媒体类型或自定义 MIME 类型字符串，或两者兼而有之。

MIME 类型的格式为:
```
type/subtype
```

自定义 MIME 类型字符串，也称为“vendor-specific”MIME 类型，具有更复杂的类型和子类型值。 type值总是
```
vnd.android.cursor.dir
```
对于多行，或
```
vnd.android.cursor.item
```
对于单行。

subtype是provider-specific。 Android 内置提供程序通常有一个简单的子类型。 例如，当联系人应用程序为电话号码创建一行时，它会在该行中设置以下 MIME 类型：
```
vnd.android.cursor.item/phone_v2
```

其他provider开发人员可以根据provider的authority和表名创建自己的subtype模式。 
例如，考虑一个包含火车时刻表的provider。 provider的authority是 com.example.trains，它包含表 Line1、Line2 和 Line3。 响应content URI
```
content://com.example.trains/Line1
```
对于表 Line1，provider返回 MIME 类型
```
vnd.android.cursor.dir/vnd.example.line1
```
响应内容 URI
```
content://com.example.trains/Line2/5
```
对于表 Line2 中的第 5 行，提供程序返回 MIME 类型
```
vnd.android.cursor.item/vnd.example.line2
```


# 创建ContentProvider

## 设计 content URI
### 设计 authority
provider通常只有一个authority，作为其 Android 内部名称。 为避免与其他provider发生冲突，应该使用 Internet 域所有权（反向）作为您的provider authority的基础。 
由于此建议也适用于 Android 包名称，因此可以将provider authority定义为包含provider的包名称的扩展。 例如，如果你的 Android 包名称是 com.example.<appname>，你应该给你的提供者权限 com.example.<appname>.provider。

### 设计 path 结构
通常通过附加指向各个表的路径来从authority创建内容 URI。 例如，如果您有两个表 table1 和 table2，则结合前面示例中的权限以生成内容 URI com.example.<appname>.provider/table1 和 com.example.<appname>.provider/table2。 
Path不限于单个段，并且path的每个级别都不是必须有一个表。

### 处理 content URI ID
按照惯例，provider通过接受内容 URI 来提供对表中单行的访问，该内容 URI 具有 URI 末尾的行的 ID 值。 同样按照惯例，provider将 ID 值与表的 _ID 列匹配，并对匹配的行执行请求的访问。

### content URI 模式
为了帮助选择对传入content URI 采取的操作，provider API 包含便利类 `UriMatcher`，它将content URI“pattern”映射到整数值。 
可以在 `switch` 语句中使用整数值，为与特定模式匹配的content URI 或 URI 选择所需的操作。

content URI 模式使用通配符匹配内容 URI：
* `*`：匹配任意长度的任意有效字符的字符串。
* `#`：匹配任意长度的数字字符串。

## 实现 ContentProvider 类
### 所需方法
抽象类 ContentProvider 定义了六个抽象方法，您必须将它们实现为您自己的具体子类的一部分。 除了 onCreate() 之外的所有这些方法都由试图访问您的内容提供者的客户端应用程序调用：

`query()`  使用参数选择要查询的表、要返回的行和列以及结果的排序顺序。 将数据作为 Cursor 对象返回。

`insert()`  使用参数选择目标表并获取要使用的列值。 返回新插入行的内容 URI。

`update()`  使用参数选择要更新的表和行并获取更新的列值。 返回更新的行数。

`delete()`  使用参数选择要删除的表和行。 返回删除的行数。

`getType()`  返回与内容 URI 对应的 MIME 类型。

`onCreate()`  初始化 Provider。 Android 系统在创建 Provider 后立即调用此方法。 请注意，在 ContentResolver 对象尝试访问它之前，不会创建 Provider。  
实际测试结果是：应用进程启动时就会调用 `onCreate()`，在 `Activity.onCreate()` 之前调用。

对这些方法的实施应考虑以下因素：
* 除了 `onCreate()` 之外的所有这些方法都可以被多个线程同时调用，因此它们必须是线程安全的。
* 避免在 `onCreate()` 中执行冗长的操作。 推迟初始化任务，直到真正需要它们。
* 尽管必须实现这些方法，但您的代码除了返回预期的数据类型外无需执行任何操作。 例如，您可能希望阻止其他应用程序将数据插入某些表中。 为此，您可以忽略对 insert() 的调用并返回 0。

### 实现 query() 
`ContentProvider.query()` 方法必须返回一个 `Cursor` 对象，否则如果失败，则抛出异常。
如果使用 SQLite 数据库作为数据存储，可以简单地返回由 `SQLiteDatabase` 类的 `query()` 方法之一返回的 Cursor。
如果查询不匹配任何行，则应返回其 `getCount()` 方法返回 0 的 Cursor 实例。仅当查询过程中发生内部错误时才应返回 `null`。

如果不使用 SQLite 数据库作为数据存储，请使用 `Cursor` 的具体子类之一。例如，`MatrixCursor` 类实现了一个cursor，其中每一行都是一个 Object 数组。使用此类，使用 `addRow()` 添加新行。

请记住，Android 系统必须能够跨进程边界传递异常。 Android 可以针对以下可能有助于处理查询错误的异常执行此操作：
* `IllegalArgumentException`（如果您的提供者收到无效的内容 URI，您可以选择抛出此异常）
* `NullPointerException`

### 实现 insert() 

### 实现 delete() 
delete() 方法不必从数据存储中物理删除行。 如果您在提供程序中使用同步适配器，则应考虑使用“删除”标志标记已删除的行，而不是完全删除该行。 同步适配器可以检查已删除的行并将它们从服务器中删除，然后再从提供程序中删除它们。

### 实现 update() 

### 实现 onCreate() 
Android 系统在启动提供程序时调用 onCreate()。 您应该在此方法中只执行快速运行的初始化任务，并推迟数据库创建和数据加载，直到提供者实际收到对数据的请求。 如果您在 onCreate() 中执行冗长的任务，您将减慢提供程序的启动速度。 反过来，这将减慢提供程序对其他应用程序的响应。


## 实现 ContentProvider MIME Types
ContentProvider 类有两种返回 MIME 类型的方法：

`getType()`  必须为任何provider实现的必需方法之一。

`getStreamTypes()`  如果provider提供文件，应该实现的方法。

### 表的 MIME 类型
`getType()` 方法返回一个 MIME 格式的`String`，该字符串描述了content URI 参数返回的数据类型。 `Uri` 参数可以是模式而不是特定的 URI； 在这种情况下，应该返回与匹配模式的content URI 关联的数据类型。

对于文本、HTML 或 JPEG 等常见数据类型，`getType()` 应返回该数据的标准 MIME 类型。 IANA MIME 媒体类型网站上提供了这些标准类型的完整列表。

对于指向一行或多行表数据的content URI，`getType()` 应返回 Android 供应商特定 MIME 格式的 MIME 类型

### 文件的 MIME 类型
如果provider提供文件，请实现 `getStreamTypes()`。 该方法为provider可以为给定content URI 返回的文件返回 MIME 类型的 String 数组。 
应该通过 MIME 类型过滤器参数过滤提供的 MIME 类型，以便只返回客户端想要处理的那些 MIME 类型。

例如，考虑一个以 .jpg、.png 和 .gif 格式的文件提供照片图像的provider。 如果应用程序使用过滤器字符串 image/*（即“图像”）调用 `ContentResolver.getStreamTypes()`，则 `ContentProvider.getStreamTypes()` 方法应返回数组：
```
{ "image/jpeg", "image/png", "image/gif"}
```

如果应用只对 .jpg 文件感兴趣，那么它可以使用过滤字符串 `*\/jpeg` 调用 `ContentResolver.getStreamTypes()`，并且 ContentProvider.getStreamTypes() 应该返回：
```
{"image/jpeg"}
```

如果provider不提供过滤字符串中请求的任何 MIME 类型，则 `getStreamTypes()` 应返回 null。


## 实现权限
所有应用程序都可以读取或写入您的provider，即使底层数据是私有的，因为默认情况下您的provider没有设置权限。 
要更改此设置，请使用 `<provider>` 元素的属性或子元素在清单文件中为provider设置权限。 可以设置适用于整个provider、某些表、甚至某些记录或所有三者的权限。

可以在清单文件中使用一个或多个 `<permission>` 元素为provider定义权限。 要使权限对provider来说是唯一的，请对 `android:name` 属性使用 Java 样式的范围。 
例如，将读取权限命名为 `com.example.app.provider.permission.READ_PROVIDER`。

**Single read-write provider-level permission**  一种控制对整个provider的读写访问的权限，由 `<provider>` 元素的 `android:permission` 属性指定。

**Separate read and write provider-level permission**  整个provider的读取权限和写入权限。 可以使用 `<provider>` 元素的 `android:readPermission` 和 `android:writePermission` 属性来指定它们。 它们优先于 `android:permission` 所需的权限。

**Path-level permission**  provider中content URI 的读取、写入或读取/写入权限。 可以使用 `<provider>` 元素的 `<path-permission>` 子元素指定要控制的每个 URI。 
对于指定的每个content URI，可以指定读/写权限、读权限或写权限，或同时指定这三个。 读写权限优先于读/写权限。 此外，Path级别的权限优先于provider级别的权限。

**Temporary permission**  要打开临时权限，请设置 `<provider>` 元素的 `android:grantUriPermissions` 属性，或者将一个或多个 `<grant-uri-permission>` 子元素添加到 `<provider>` 元素。 
如果使用临时权限，则每当从provider中删除对content URI 的支持并且content URI 与临时权限相关联时，都必须调用 `Context.revokeUriPermission()`。

该属性的值决定了您的provider有多少可以访问。 如果该属性设置为 true，那么系统将向您的整个provider授予临时权限，覆盖您的provider级别或路径级别权限所需的任何其他权限。

如果此标志设置为 false，那么您必须将 `<grant-uri-permission>` 子元素添加到您的 `<provider>` 元素。 每个子元素指定授予临时访问权限的内容 URI 或 URI。

要委托对应用程序的临时访问，意图必须包含 FLAG_GRANT_READ_URI_PERMISSION 或 FLAG_GRANT_WRITE_URI_PERMISSION 标志，或两者兼有。 这些是使用 setFlags() 方法设置的。

如果 android:grantUriPermissions 属性不存在，则假定为 false。


## `<provider>` 元素

**Authority** (`android:authorities`) 标识系统内整个提供者的符号名称

**Provider class name** ( `android:name` ) 实现 ContentProvider 的类

**Permissions**  指定其他应用程序必须具有的权限才能访问provider数据的属性
* `android:grantUriPermissions`: Temporary permission flag.
* `android:permission`: Single provider-wide read/write permission.
* `android:readPermission`: Provider-wide read permission.
* `android:writePermission`: Provider-wide write permission.

**Startup and control attributes**  这些属性决定了 Android 系统启动提供程序的方式和时间、提供程序的进程特征以及其他运行时设置：
* `android:enabled`: 允许系统启动provider的标志。
* `android:exported`: 允许其他应用程序使用此provider的标志。
* `android:initOrder`：这个provider应该启动的顺序，相对于同一进程中的其他provider。
* `android:multiProcess`: 允许系统在与调用客户端相同的进程中启动provider的标志。
* `android:process`：provider应该在其中运行的进程的名称。
* `android:syncable`: 指示provider的数据将与服务器上的数据同步的标志。

















