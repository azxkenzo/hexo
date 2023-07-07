---
title: Android - Dagger
date: 2023-06-30 16:35:01
tags: Android
---

# Dagger 基础
根据项目的大小，Android 应用程序中的手动依赖项注入或服务定位器可能会出现问题。 可以通过使用 [Dagger](https://dagger.dev/) 管理依赖项来限制项目扩展时的复杂性。

Dagger 会自动生成模仿手写代码的代码。 由于代码是在编译时生成的，因此它是可追踪的，并且比 Guice 等其他基于反射的解决方案性能更高。

```
注意：在 Android 上使用 Hilt 进行依赖注入。 Hilt 构建在 Dagger 之上，它提供了将 Dagger 依赖项注入合并到 Android 应用程序中的标准方法。
```

## 使用 Dagger 的好处
Dagger 通过以下方式让你免于编写繁琐且容易出错的样板代码：
* 生成你在手动 DI 部分中手动实现的 `AppContainer` 代码（application graph）。
* 为 application graph 中可用的类创建工厂。 这就是内部满足依赖关系的方式。
* 决定是否重用依赖项或通过使用 scope 创建新实例。
* 使用 Dagger 子组件为特定流创建容器。 这可以通过在不再需要内存中的对象时释放它们来提高应用程序的性能。

只要你声明类的依赖项并指定如何使用注解来满足它们，Dagger 就会在构建时自动执行所有这些操作。 Dagger 生成的代码与你手动编写的代码类似。 
在内部，Dagger 创建一个 graph of objects，它可以引用该 graph of objects 来找到提供类实例的方法。 对于 graph 中的每个类，Dagger 都会生成一个工厂类型类，它在内部使用该类来获取该类型的实例。

在构建时，Dagger 会遍历你的代码并：
* 构建并验证 dependency graph，确保：
  * 每个对象的依赖关系都可以得到满足，因此不存在运行时异常。
  * 不存在依赖循环，因此不存在无限循环。
* 生成在运行时用于创建实际对象及其依赖项的类。


## Dagger 中的一个简单用例：生成工厂
为了演示如何使用 Dagger，让我们为 `UserRepository` 类创建一个简单的工厂，如下图所示：

![](https://developer.android.com/static/images/training/dependency-injection/3-factory-diagram.png)

定义 `UserRepository` 如下：
```kotlin
class UserRepository(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) { ... }
```
将 `@Inject` 注释添加到 `UserRepository` 构造函数中，以便 Dagger 知道如何创建 `UserRepository`：
```kotlin
// @Inject lets Dagger know how to create instances of this object
class UserRepository @Inject constructor(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) { ... }
```

```java
public class UserRepository {

    private final UserLocalDataSource userLocalDataSource;
    private final UserRemoteDataSource userRemoteDataSource;

    // @Inject lets Dagger know how to create instances of this object
    @Inject
    public UserRepository(UserLocalDataSource userLocalDataSource, UserRemoteDataSource userRemoteDataSource) {
        this.userLocalDataSource = userLocalDataSource;
        this.userRemoteDataSource = userRemoteDataSource;
    }
}
```

在上面的代码片段中，你告诉 Dagger：
1. 如何使用 `@Inject` 注解的构造函数创建 `UserRepository` 实例。
2. 它的依赖项是：`UserLocalDataSource` 和 `UserRemoteDataSource`。

现在 Dagger 知道如何创建 `UserRepository` 的实例，但不知道如何创建其依赖项。 如果你也注解其他类，Dagger 知道如何创建它们：
```kotlin
// @Inject lets Dagger know how to create instances of these objects
class UserLocalDataSource @Inject constructor() { ... }
class UserRemoteDataSource @Inject constructor() { ... }
```


## Dagger component
Dagger 可以在项目中创建 graph of the dependencies，以便在需要时找出应该从哪里获取这些依赖关系。 要让 Dagger 执行此操作，您需要创建一个接口并使用 `@Component` 进行注释。 Dagger 创建一个容器，就像手动依赖注入一样。

在 `@Component` 接口内部，您可以定义返回所需类实例的函数（即 `UserRepository`）。 `@Component` 告诉 Dagger 生成一个容器，其中包含满足其公开的类型所需的所有依赖项。 这称为 Dagger component； 它包含一个 graph，其中包含 Dagger 知道如何提供的对象及其各自的依赖项。

```kotlin
// @Component makes Dagger create a graph of dependencies
@Component
interface ApplicationGraph {
    // The return type  of functions inside the component interface is
    // what can be provided from the container
    fun repository(): UserRepository
}
```

当您构建项目时，Dagger 会为您生成 `ApplicationGraph` 接口的实现：`DaggerApplicationGraph`。 通过其注释处理器，Dagger 创建了一个 dependency graph，该 dependency graph 由三个类（`UserRepository`、`UserLocalDatasource` 和 `UserRemoteDataSource`）之间的关系组成，
只有一个入口点：获取 `UserRepository` 实例。 您可以按如下方式使用它：

```kotlin
// Create an instance of the application graph
val applicationGraph: ApplicationGraph = DaggerApplicationGraph.create()
// Grab an instance of UserRepository from the application graph
val userRepository: UserRepository = applicationGraph.repository()
```

Dagger 在每次请求时都会创建一个新的 `UserRepository` 实例。

```kotlin
val applicationGraph: ApplicationGraph = DaggerApplicationGraph.create()

val userRepository: UserRepository = applicationGraph.repository()
val userRepository2: UserRepository = applicationGraph.repository()

assert(userRepository != userRepository2)
```

有时，您需要在容器中拥有唯一的依赖项实例。 您需要这个可能有几个原因：
1. 您希望将此类型作为依赖项的其他类型共享同一实例，例如登录流中使用相同 `LoginUserData` 的多个 `ViewModel` 对象。
2. 创建对象的成本很高，并且您不希望每次将其声明为依赖项（例如，JSON 解析器）时都创建一个新实例。

在示例中，您可能希望 graph 中有一个可用的唯一 `UserRepository` 实例，以便每次请求 `UserRepository` 时，您总是会获得相同的实例。 
这在您的示例中非常有用，因为在具有更复杂的 application graph 的现实应用程序中，您可能有多个取决于 `UserRepository` 的 `ViewModel` 对象，并且您不希望每次需要提供 `UserRepository` 时都创建 `UserLocalDataSource` 和 `UserRemoteDataSource` 的新实例 。

在手动依赖注入中，您可以通过将 `UserRepository` 的相同实例传递给 ViewModel 类的构造函数来实现此目的； 但在 Dagger 中，因为您不是手动编写该代码，所以您必须让 Dagger 知道您想要使用同一个实例。 这可以通过 scope annotation 来完成。

### Scoping with Dagger
您可以使用 scope annotation 将对象的生命周期限制为其 component 的生命周期。 这意味着每次需要提供该类型时都会使用相同的依赖项实例。

要在 `ApplicationGraph` 中请求存储库时拥有唯一的 `UserRepository` 实例，**请对 `@Component` 接口和 `UserRepository` 使用相同的 scope annotation**。 您可以使用 Dagger 使用的 `javax.inject` 包附带的 `@Singleton` 注释：

```kotlin
// @Component 接口上的 scope 注解告知 Dagger
// 用此注解（即 @Singleton）注释的类绑定到 graph 的生命周期
// 因此每次请求该类型时都会提供该类型的相同实例。
@Singleton
@Component
interface ApplicationGraph {
    fun repository(): UserRepository
}

// 使用 @Singleton scope（即 ApplicationGraph）将此类的 scope 限定为组件
@Singleton
class UserRepository @Inject constructor(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) { ... }
```

或者，您可以创建并使用自定义 scope annotation。 您可以按如下方式创建 scope annotation：

```kotlin
// Creates MyCustomScope
@Scope
@MustBeDocumented
@Retention(value = AnnotationRetention.RUNTIME)
annotation class MyCustomScope
```

然后，您可以像以前一样使用它

在这两种情况下，对象都具有用于注释 `@Component` 接口的相同 scope。 因此，每次调用 `applicationGraph.repository()` 时，您都会获得相同的 `UserRepository` 实例。


# Using Dagger in Android apps

## 最佳实践总结
* 只要有可能，就使用构造函数注入和 `@Inject` 将类型添加到 Dagger graph 中。 当它不是时：|
  * 使用 `@Binds` 告诉 Dagger 接口应该具有哪个实现。 
  * 使用 `@Provides` 告诉 Dagger 如何提供您的项目不拥有的类。
* 您应该只在 component 中声明一次模块
* 根据使用注释的生命周期来命名 scope annotation。 示例包括 `@ApplicationScope`、`@LoggedUserScope` 和 `@ActivityScope`。


## Adding dependencies
要在项目中使用 Dagger，请将这些依赖项添加到应用程序的 build.gradle 文件中。 您可以在[此 GitHub 项目](https://github.com/google/dagger/releases)中找到最新版本的 Dagger。
```
// Kotlin
plugins {
  id 'kotlin-kapt'
}

dependencies {
    implementation 'com.google.dagger:dagger:2.x'
    kapt 'com.google.dagger:dagger-compiler:2.x'
}
```

```
// Java
dependencies {
    implementation 'com.google.dagger:dagger:2.x'
    annotationProcessor 'com.google.dagger:dagger-compiler:2.x'
}
```

## Dagger in Android
考虑一个具有图 1 中的依赖关系图的示例 Android 应用程序。

![Figure 1. Dependency graph of the example code](https://developer.android.com/static/images/training/dependency-injection/4-application-graph.png)

在 Android 中，您通常会创建一个位于 application 类中的 Dagger graph，因为您希望只要应用程序正在运行，该 graph 的实例就位于内存中。 
通过这种方式，该 graph 就附加到了应用程序生命周期中。 在某些情况下，您可能还希望 application context 在 graph 中可用。 
为此，您还需要将 graph 放在 `Application` 类中。 这种方法的优点之一是该 graph 可供其他 Android framework 类使用。 此外，它允许您在测试中使用自定义应用程序类，从而简化了测试。

因为生成 graph 的接口是用 `@Component` 注解的，所以你可以将其称为 `ApplicationComponent` 或 `ApplicationGraph`。 
您通常会在自定义 `Application` 类中保留该组件的一个实例，并在每次需要 application graph 时调用它，如以下代码片段所示：

```kotlin
// Definition of the Application graph
@Component
interface ApplicationComponent { ... }

// appComponent lives in the Application class to share its lifecycle
class MyApplication: Application() {
    // Reference to the application graph that is used across the whole app
    val appComponent = DaggerApplicationComponent.create()
}

```

由于某些 Android framework 类（例如 activities and fragments）是由系统实例化的，因此 Dagger 无法为您创建它们。 特别是对于 activity，任何初始化代码都需要进入 `onCreate()` 方法。 这意味着您不能像在前面的示例中那样在类的构造函数中使用 `@Inject` 注释（构造函数注入）。 相反，您必须使用字段注入。

您希望 Dagger 为您填充这些依赖项，而不是在 `onCreate()` 方法中创建 activity 所需的依赖项。 对于字段注入，您可以将 `@Inject` 注释应用于要从 Dagger graph 中获取的字段。

```kotlin
class LoginActivity: Activity() {
    // You want Dagger to provide an instance of LoginViewModel from the graph
    @Inject lateinit var loginViewModel: LoginViewModel
}
```

为简单起见，`LoginViewModel` 不是 Android 架构组件 ViewModel； 它只是一个充当 ViewModel 的常规类。 有关如何注入这些类的更多信息，请查看 dev-dagger 分支中官方 Android Blueprints Dagger 实现中的代码。

Dagger 的考虑因素之一是注入的字段不能是私有的。 它们至少需要具有包私有可见性，如前面的代码所示。

```
注意：字段注入只能用在不能使用构造函数注入的 Android 框架类中。
```

### Injecting activities
Dagger 需要知道 `LoginActivity` 必须访问该 graph 才能提供它所需的 ViewModel。 在 Dagger 基础知识页面中，您使用 `@Component` 接口通过公开具有要从 graph 中获取的内容的返回类型的函数来从 graph 中获取对象。 
在这种情况下，您需要告诉 Dagger 需要注入依赖项的对象（本例中为 `LoginActivity`）。 为此，您公开一个函数，该函数将请求注入的对象作为参数。

```kotlin
@Component
interface ApplicationComponent {
    // This tells Dagger that LoginActivity requests injection so the graph needs to
    // satisfy all the dependencies of the fields that LoginActivity is requesting.
    fun inject(activity: LoginActivity)
}
```

该函数告诉 Dagger `LoginActivity` 想要访问该 graph 并请求注入。 Dagger 需要满足 `LoginActivity` 所需的所有依赖项（`LoginViewModel` 具有自己的依赖项）。 如果您有多个请求注入的类，则必须在组件中明确声明它们的确切类型。 
例如，如果您有 `LoginActivity` 和 `RegistrationActivity` 请求注入，则您将有两个 `inject()` 方法，而不是涵盖这两种情况的通用方法。 
通用的 `inject()` 方法不会告诉 Dagger 需要提供什么。 接口中的函数可以有任何名称，**但在 Dagger 中，当它们接收要注入的对象作为参数时叫它们 `inject()` 是一种约定**。

要在 activity 中注入对象，您可以使用 `Application` 类中定义的 `appComponent` 并调用 `inject()` 方法，传入请求注入的 activity 实例。

**使用 Activity 时，请在调用 `super.onCreate()` 之前在 Activity 的 `onCreate()` 方法中注入 Dagger，以避免 fragment 恢复问题**。 
在 `super.onCreate()` 的恢复阶段，Activity 会附加可能想要访问 activity 绑定的 fragment。

**使用 fragment 时，在 fragment 的 `onAttach()` 方法中注入 Dagger**。 在这种情况下，可以在调用 `super.onAttach()` 之前或之后完成。

```kotlin
class LoginActivity: Activity() {
    // You want Dagger to provide an instance of LoginViewModel from the graph
    @Inject lateinit var loginViewModel: LoginViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        // 让 Dagger 在 LoginActivity 中实例化 @Inject 字段
        (applicationContext as MyApplication).appComponent.inject(this)
        // Now loginViewModel is available

        super.onCreate(savedInstanceState)
    }
}

// @Inject tells Dagger how to create instances of LoginViewModel
class LoginViewModel @Inject constructor(
    private val userRepository: UserRepository
) { ... }
```

让我们告诉 Dagger 如何提供其余的依赖项来构建 graph：

```kotlin
class UserRepository @Inject constructor(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) { ... }

class UserLocalDataSource @Inject constructor() { ... }
class UserRemoteDataSource @Inject constructor(
    private val loginService: LoginRetrofitService
) { ... }
```

### Dagger modules
在此示例中，您使用的是 Retrofit 网络库。 `UserRemoteDataSource` 依赖于 `LoginRetrofitService`。 **但是，创建 `LoginRetrofitService` 实例的方法与您迄今为止所做的不同。 它不是一个类实例化；这是调用 `Retrofit.Builder()` 并传入不同参数来配置登录服务的结果**。

除了 `@Inject` 注释之外，还有另一种方法告诉 Dagger 如何提供类的实例：Dagger 模块内的信息。 Dagger 模块是一个用 `@Module` 注解的类。 在那里，您可以使用 `@Provides` 注释定义依赖项。

```kotlin
// @Module informs Dagger that this class is a Dagger Module
@Module
class NetworkModule {

    // @Provides tell Dagger how to create instances of the type that this function
    // returns (i.e. LoginRetrofitService).
    // Function parameters are the dependencies of this type.
    @Provides
    fun provideLoginRetrofitService(): LoginRetrofitService {
        // Whenever Dagger needs to provide an instance of type LoginRetrofitService,
        // this code (the one inside the @Provides method) is run.
        return Retrofit.Builder()
                .baseUrl("https://example.com")
                .build()
                .create(LoginService::class.java)
    }
}
```

```
注意：您可以在 Dagger 模块中使用 @Provides 注释来告诉 Dagger 如何提供您的项目不拥有的类（例如 Retrofit 的实例）。

对于接口的实现，最佳实践是使用 @Binds
```

**模块是一种从语义上封装有关如何提供对象的信息的方法**。 正如您所看到的，您调用了 `NetworkModule` 类来对提供与网络相关的对象的逻辑进行分组。 如果应用扩展，还可以在这里添加如何提供 `OkHttpClient`，或者如何配置 Gson 或 Moshi。

`@Provides` 方法的依赖项是该方法的参数。 对于前面的方法，`LoginRetrofitService` 可以不提供依赖项，因为该方法没有参数。 如果您已声明 `OkHttpClient` 作为参数，Dagger 需要从 graph 中提供 `OkHttpClient` 实例来满足 `LoginRetrofitService` 的依赖关系。 例如：

```kotlin
@Module
class NetworkModule {
    // Hypothetical dependency on LoginRetrofitService
    @Provides
    fun provideLoginRetrofitService(
        okHttpClient: OkHttpClient
    ): LoginRetrofitService { ... }
}
```

为了让 Dagger graph 了解这个模块，您必须将其添加到 `@Component` 接口，如下所示：
```kotlin
// The "modules" attribute in the @Component annotation tells Dagger what Modules
// to include when building the graph
@Component(modules = [NetworkModule::class])
interface ApplicationComponent {
    ...
}
```

向 Dagger graph 中添加类型的推荐方法是使用构造函数注入（即在类的构造函数上使用 `@Inject` 注释）。 有时，这是不可能的，您必须使用 Dagger 模块。 
一个例子是当您希望 Dagger 使用计算结果来确定如何创建对象的实例时。 每当需要提供该类型的实例时，Dagger 都会运行 `@Provides` 方法内的代码。

这就是示例中的 Dagger graph 现在的样子：

![](https://developer.android.com/static/images/training/dependency-injection/4-graph-login.png)

该 graph 的入口点是 `LoginActivity`。 由于 `LoginActivity` 注入 `LoginViewModel`，Dagger 构建了一个 graph，该 graph 知道如何提供 `LoginViewModel` 的实例，并递归地提供其依赖项。 Dagger 知道如何执行此操作，因为类构造函数上有 `@Inject` 注释。

在 Dagger 生成的 `ApplicationComponent` 内部，有一个工厂类型的方法来获取它知道如何提供的所有类的实例。 在此示例中，Dagger 委托给 `ApplicationComponent` 中包含的 `NetworkModule` 来获取 `LoginRetrofitService` 的实例。


### Dagger scopes
Dagger 基础知识页面上提到了 scope 作为在组件中拥有类型的唯一实例的一种方式。 这就是将类型的 scope 限定到组件的生命周期的含义。

由于您可能希望在应用程序的其他功能中使用 `UserRepository`，并且可能不希望每次需要时都创建一个新对象，因此您可以将其指定为整个应用程序的唯一实例。 
`LoginRetrofitService` 也是如此：创建成本可能很高，而且您还希望重用该对象的唯一实例。 创建 `UserRemoteDataSource` 的实例并不那么昂贵，因此没有必要将其 scope 限定到组件的生命周期。

`@Singleton` 是 `javax.inject` 包附带的唯一 scope 注释。 您可以使用它来注释 `ApplicationComponent` 以及您想要在整个 application 中重用的对象。

```kotlin
@Singleton
@Component(modules = [NetworkModule::class])
interface ApplicationComponent {
    fun inject(activity: LoginActivity)
}

@Singleton
class UserRepository @Inject constructor(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) { ... }

@Module
class NetworkModule {
    // Way to scope types inside a Dagger Module
    @Singleton
    @Provides
    fun provideLoginRetrofitService(): LoginRetrofitService { ... }
}
```

```
注意：使用 scope 注释的模块只能在使用相同 scope 注释的组件中使用。
```

将 scope 应用于对象时，请注意不要引入内存泄漏。 只要 scoped component 位于内存中，创建的对象也位于内存中。 因为 `ApplicationComponent` 是在应用程序启动时创建的（在 `Application` 类中），所以当应用程序被销毁时它也会被销毁。 因此，`UserRepository` 的唯一实例始终保留在内存中，直到应用程序被销毁。

```
注意：使用构造函数注入（使用 @Inject）时在类中添加 scope 注释，并在使用 Dagger 模块时将它们添加到 @Provides 方法中。
```

### Dagger subcomponents
如果您的登录流（由单个 `LoginActivity` 管理）包含多个 fragment，则应在所有 fragment 中重用相同的 `LoginViewModel` 实例。 `@Singleton` 无法注释 `LoginViewModel` 以重用实例，原因如下：
1. 流程完成后，`LoginViewModel` 的实例将保留在内存中。
2. 您需要为每个登录流程使用不同的 `LoginViewModel` 实例。 例如，如果用户注销，您需要一个不同的 `LoginViewModel` 实例，而不是与用户第一次登录时相同的实例。

要将 `LoginViewModel` 的 scope 限定为 `LoginActivity` 的生命周期，您需要为登录流程和新 scope 创建一个新组件（新的 subgraph）。

让我们创建一个特定于登录流程的 graph。

```kotlin
@Component
interface LoginComponent {}
```
现在，`LoginActivity` 应该从 `LoginComponent` 获取注入，因为它具有特定于登录的配置。 这消除了从 `ApplicationComponent` 类注入 `LoginActivity` 的责任。

```kotlin
@Component
interface LoginComponent {
    fun inject(activity: LoginActivity)
}
```

`LoginComponent` 必须能够访问 `ApplicationComponent` 中的对象，因为 `LoginViewModel` 依赖于 `UserRepository`。 告诉 Dagger 您希望新组件使用另一个组件的一部分的方法是使用 Dagger subcomponent。 新组件必须是包含共享资源的组件的 subcomponent。

Subcomponent 是继承并扩展父组件的 object graph 的组件。 因此，父组件中提供的所有对象也在子组件中提供。 这样，子组件中的对象可以依赖于父组件提供的对象。

要创建子组件的实例，您需要父组件的实例。 因此，父组件向子组件提供的对象的 scope 仍然是父组件。

在示例中，您必须将 `LoginComponent` 定义为 `ApplicationComponent` 的子组件。 为此，请使用 `@Subcomponent` 注释 `LoginComponent`：

```kotlin
// @Subcomponent annotation informs Dagger this interface is a Dagger Subcomponent
@Subcomponent
interface LoginComponent {

    // This tells Dagger that LoginActivity requests injection from LoginComponent
    // so that this subcomponent graph needs to satisfy all the dependencies of the
    // fields that LoginActivity is injecting
    fun inject(loginActivity: LoginActivity)
}
```

您还必须在 `LoginComponent` 中定义一个子组件工厂，以便 `ApplicationComponent` 知道如何创建 LoginComponent 的实例。

```kotlin
@Subcomponent
interface LoginComponent {

    // Factory that is used to create instances of this subcomponent
    @Subcomponent.Factory
    interface Factory {
        fun create(): LoginComponent
    }

    fun inject(loginActivity: LoginActivity)
}
```

要告诉 Dagger `LoginComponent` 是 `ApplicationComponent` 的子组件，您必须通过以下方式指示它：
1. 创建一个新的 Dagger 模块（例如 `SubcomponentsModule`），将子组件的类传递给注释的 `subcomponents` 属性。
    ```kotlin
    // The "subcomponents" attribute in the @Module annotation tells Dagger what
    // Subcomponents are children of the Component this module is included in.
    @Module(subcomponents = LoginComponent::class)
    class SubcomponentsModule {}
    ```
2. 将新模块（即 SubcomponentsModule）添加到 ApplicationComponent：
   ```kotlin
    // Including SubcomponentsModule, tell ApplicationComponent that
    // LoginComponent is its subcomponent.
    @Singleton
    @Component(modules = [NetworkModule::class, SubcomponentsModule::class])
    interface ApplicationComponent {
    }
   ```
   请注意，`ApplicationComponent` 不再需要注入 `LoginActivity`，因为该责任现在属于 `LoginComponent`，因此您可以从 `ApplicationComponent` 中删除 `inject()` 方法。

   `ApplicationComponent` 的使用者需要知道如何创建 `LoginComponent` 的实例。 父组件必须在其接口中添加一个方法，以允许使用者从父组件的实例中创建子组件的实例：
3. 在接口中公开创建 `LoginComponent` 实例的工厂：
   ```kotlin
   @Singleton
   @Component(modules = [NetworkModule::class, SubcomponentsModule::class])
   interface ApplicationComponent {
   // This function exposes the LoginComponent Factory out of the graph so consumers
   // can use it to obtain new instances of LoginComponent
   fun loginComponent(): LoginComponent.Factory
   }
   ```

### Assigning scopes to subcomponents
如果构建项目，则可以创建 `ApplicationComponent` 和 `LoginComponent` 的实例。 `ApplicationComponent` 附加到 application 的生命周期，因为只要 application 位于内存中，您就希望使用同一 graph 实例。

`LoginComponent` 的生命周期是怎样的？ 您需要 `LoginComponent` 的原因之一是因为您需要在与登录相关的 fragment 之间共享相同的 `LoginViewModel` 实例。 
而且，每当有新的登录流程时，您都需要 `LoginViewModel` 的不同实例。 `LoginActivity` 是 `LoginComponent` 的正确生命周期：对于每个新 activity，您需要一个新的 `LoginComponent` 实例和可以使用该 `LoginComponent` 实例的 fragment。

由于 `LoginComponent` 附加到 `LoginActivity` 生命周期，因此您必须在 activity 中保留对组件的引用，就像在 `Application` 类中保留对 `applicationComponent` 的引用一样。 这样，fragment 就可以访问它。

```kotlin
class LoginActivity: Activity() {
    // Reference to the Login graph
    lateinit var loginComponent: LoginComponent
    ...
}
```

请注意，变量 `loginComponent` 未使用 `@Inject` 进行注释，因为您不希望 Dagger 提供该变量。

您可以使用 `ApplicationComponent` 获取对 `LoginComponent` 的引用，然后注入 `LoginActivity`，如下所示：

```kotlin
class LoginActivity: Activity() {
  // Reference to the Login graph
  lateinit var loginComponent: LoginComponent

  // Fields that need to be injected by the login graph
  @Inject lateinit var loginViewModel: LoginViewModel

  override fun onCreate(savedInstanceState: Bundle?) {
    // Creation of the login graph using the application graph
    loginComponent = (applicationContext as MyDaggerApplication)
      .appComponent.loginComponent().create()

    // Make Dagger instantiate @Inject fields in LoginActivity
    loginComponent.inject(this)

    // Now loginViewModel is available

    super.onCreate(savedInstanceState)
  }
}
```

`LoginComponent` 是在 `Activity` 的 `onCreate()` 方法中创建的，当 Activity 被销毁时，它会被隐式销毁。

每次请求时，`LoginComponent` 必须始终提供相同的 `LoginViewModel` 实例。 您可以通过创建自定义注释 scope 并用它注释 `LoginComponent` 和 `LoginViewModel` 来确保这一点。 
请注意，您不能使用 `@Singleton` 注释，因为它已被父组件使用，这会使该对象成为 application 单例（整个应用程序的唯一实例）。 您需要创建不同的注释 scope。

```
注意：scope 规则如下：

  当一个类型被标记了 scope 注释时，它只能被具有相同 scope 注释的组件使用。
  当组件使用 scope 注释进行标记时，它只能提供具有该注释的类型或不具有注释的类型。
  子组件不能使用其父组件之一所使用的 scope 注释。

在这种情况下，组件还涉及子组件。
```

在这种情况下，您可以将此 scope 称为 `@LoginScope`，但这不是一个好的做法。 scope 注释的名称不应明确说明其所实现的目的。 
相反，它应该根据其生命周期来命名，因为注释可以被同级组件（例如 `RegistrationComponent` 和 `SettingsComponent`）重用。 
这就是为什么您应该将其命名为 `@ActivityScope` 而不是 `@LoginScope`。

```kotlin
// Definition of a custom scope called ActivityScope
@Scope
@Retention(value = AnnotationRetention.RUNTIME)
annotation class ActivityScope

// Classes annotated with @ActivityScope are scoped to the graph and the same
// instance of that type is provided every time the type is requested.
@ActivityScope
@Subcomponent
interface LoginComponent { ... }

// A unique instance of LoginViewModel is provided in Components
// annotated with @ActivityScope
@ActivityScope
class LoginViewModel @Inject constructor(
    private val userRepository: UserRepository
) { ... }
```

现在，如果您有两个需要 `LoginViewModel` 的 fragment，则它们都提供有相同的实例。 例如，如果您有 `LoginUsernameFragment` 和 `LoginPasswordFragment`，它们需要由 `LoginComponent` 注入：

```kotlin
@ActivityScope
@Subcomponent
interface LoginComponent {

    @Subcomponent.Factory
    interface Factory {
        fun create(): LoginComponent
    }

    // All LoginActivity, LoginUsernameFragment and LoginPasswordFragment
    // request injection from LoginComponent. The graph needs to satisfy
    // all the dependencies of the fields those classes are injecting
    fun inject(loginActivity: LoginActivity)
    fun inject(usernameFragment: LoginUsernameFragment)
    fun inject(passwordFragment: LoginPasswordFragment)
}
```

这些组件访问 `LoginActivity` 对象中的组件实例。 `LoginUserNameFragment` 的示例代码显示在以下代码片段中：

```kotlin
class LoginUsernameFragment: Fragment() {

    // Fields that need to be injected by the login graph
    @Inject lateinit var loginViewModel: LoginViewModel

    override fun onAttach(context: Context) {
        super.onAttach(context)

        // Obtaining the login graph from LoginActivity and instantiate
        // the @Inject fields with objects from the graph
        (activity as LoginActivity).loginComponent.inject(this)

        // Now you can access loginViewModel here and onCreateView too
        // (shared instance with the Activity and the other Fragment)
    }
}
```

图 3 显示了带有新子组件的 Dagger graph的外观。 带白点的类（`UserRepository`、`LoginRetrofitService` 和 `LoginViewModel`）是具有其各自组件范围的唯一实例的类。

![](https://developer.android.com/static/images/training/dependency-injection/4-graph-subcomponent.png)

让我们分解一下 graph 的各个部分：
1. `NetworkModule`（以及 `LoginRetrofitService`）包含在 `ApplicationComponent` 中，因为您在组件中指定了它。
2. `UserRepository` 保留在 `ApplicationComponent` 中，因为它的作用域为 `ApplicationComponent`。 如果项目不断增长，您希望在不同功能（例如注册）之间共享相同的实例。
   由于 `UserRepository` 是 `ApplicationComponent` 的一部分，因此它的依赖项（即 `UserLocalDataSource` 和 `UserRemoteDataSource`）也需要位于该组件中，以便能够提供 `UserRepository` 的实例。
3. `LoginViewModel` 包含在 `LoginComponent` 中，因为只有 `LoginComponent` 注入的类才需要它。 `LoginViewModel` 不包含在 `ApplicationComponent` 中，因为 `ApplicationComponent` 中没有依赖项需要 `LoginViewModel`。
   类似地，如果您没有将 `UserRepository` 范围限定为 `ApplicationComponent`，Dagger 会自动将 `UserRepository` 及其依赖项作为 `LoginComponent` 的一部分包含在内，因为这是当前唯一使用 `UserRepository` 的地方。

除了将对象的范围限定到不同的生命周期之外，创建子组件是将应用程序的不同部分相互封装的一个好习惯。

根据应用程序的流程构建应用程序以创建不同的 Dagger 子图有助于在内存和启动时间方面实现更高性能和可扩展的应用程序。


## Best practices when building a Dagger graph
为您的应用程序构建 Dagger graph 时：
* 创建组件时，您应该考虑哪个元素负责该组件的生命周期。 在本例中，`Application` 类负责 `ApplicationComponent`，`LoginActivity` 负责 `LoginComponent`。
* 仅在有意义时才使用 scope。 过度使用 scope 会对应用程序的运行时性能产生负面影响：只要组件位于内存中，对象就位于内存中，并且获取 scope 对象的成本更高。 当 Dagger 提供对象时，它使用 `DoubleCheck` 锁定而不是工厂类型提供程序。


## Working with Dagger modules
Dagger 模块是一种封装如何以语义方式提供对象的方法。 您可以在组件中包含模块，但也可以在其他模块中包含模块。 这很强大，但很容易被滥用。

一旦一个模块被添加到一个组件或另一个模块中，它就已经在 Dagger graph 中了； Dagger 可以在该组件中提供这些对象。 在添加模块之前，请检查该模块是否已是 Dagger graph 的一部分，方法是检查该模块是否已添加到组件中，或者编译项目并查看 Dagger 是否可以找到该模块所需的依赖项。

良好的实践表明，模块只能在组件中声明一次（在特定的高级 Dagger 用例之外）。

假设您以这种方式配置了 graph。 `ApplicationComponent` 包括 `Module1` 和 `Module2`，`Module1` 包括 `ModuleX`。

```kotlin
@Component(modules = [Module1::class, Module2::class])
interface ApplicationComponent { ... }

@Module(includes = [ModuleX::class])
class Module1 { ... }

@Module
class Module2 { ... }
```

如果现在 Module2 依赖于 ModuleX 提供的类。 不好的做法是将 ModuleX 包含在 Module2 中，因为 ModuleX 在 graph 中包含了两次，如以下代码片段所示：

```kotlin
// Bad practice: ModuleX is declared multiple times in this Dagger graph
@Component(modules = [Module1::class, Module2::class])
interface ApplicationComponent { ... }

@Module(includes = [ModuleX::class])
class Module1 { ... }

@Module(includes = [ModuleX::class])
class Module2 { ... }
```

相反，您应该执行以下操作之一：
1. 重构模块并将公共模块提取到组件中。
2. 使用两个模块共享的对象创建一个新模块，并将其提取到组件中。

不以这种方式重构会导致许多模块相互包含而没有清晰的组织感，并且更难以看出每个依赖项来自何处。

**良好实践（选项 1）**：ModuleX 在 Dagger graph 中声明一次。

```kotlin
@Component(modules = [Module1::class, Module2::class, ModuleX::class])
interface ApplicationComponent { ... }

@Module
class Module1 { ... }

@Module
class Module2 { ... }
```

**良好实践（选项 2）**：`ModuleX` 中 `Module1` 和 `Module2` 的公共依赖项被提取到包含在组件中的名为 `ModuleXCommon` 的新模块中。 然后，使用特定于每个模块的依赖项创建另外两个名为 `ModuleXWithModule1Dependency` 和 `ModuleXWithModule2Dependency` 的模块。 所有模块都在 Dagger graph 中声明一次。

```kotlin
@Component(modules = [Module1::class, Module2::class, ModuleXCommon::class])
interface ApplicationComponent { ... }

@Module
class ModuleXCommon { ... }

@Module
class ModuleXWithModule1SpecificDependencies { ... }

@Module
class ModuleXWithModule2SpecificDependencies { ... }

@Module(includes = [ModuleXWithModule1SpecificDependencies::class])
class Module1 { ... }

@Module(includes = [ModuleXWithModule2SpecificDependencies::class])
class Module2 { ... }
```