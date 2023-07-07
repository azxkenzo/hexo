---
title: Android - Build System
date: 2023-04-14 11:53:38
tags: Android
---

Android build system 编译应用资源和源代码并将它们打包成 APK 或 Android App Bundle，您可以测试、部署、签名和分发这些应用程序包。

Android Studio 使用高级构建工具包 Gradle 来自动化和管理构建过程，同时允许您定义灵活的自定义构建配置。 每个构建配置都可以定义自己的一组代码和资源，同时重用所有版本的应用程序通用的部分。 Android Gradle plugin 与构建工具包配合使用，提供特定于构建和测试 Android 应用程序的流程和可配置设置。

Gradle 和 AGP 独立于 Android Studio 运行。 这意味着您可以从 Android Studio、您机器上的命令行或未安装 Android Studio 的机器（例如持续集成服务器）上构建您的 Android 应用程序。

## The build process
构建过程涉及许多工具和过程，可将您的项目转换为 Android 应用程序包 (APK) 或 Android App Bundle (AAB)。

AGP 为您完成大部分构建过程，但了解构建过程的某些方面可能很有用，这样您就可以调整构建以满足您的要求。

不同的项目可能有不同的构建目标。 例如，第三方库的构建会生成 AAR 或 JAR 库。 但是，应用是最常见的项目类型，应用项目的构建会生成应用的 debug 或 release APK 或 AAB，您可以部署、测试或发布给外部用户。

## Android build glossary(词汇表)
Gradle 和 AGP 可帮助您配置构建的以下方面：

#### Build types
构建类型定义了 Gradle 在构建和打包您的应用程序时使用的某些属性。 构建类型通常针对开发生命周期的不同阶段进行配置。

例如，debug 构建类型启用 debug 选项并使用 debug 密钥对应用程序进行签名，而 release 构建类型可能会收缩、混淆并使用 release 密钥对您的应用程序进行签名以进行分发。

必须至少定义一种构建类型来构建您的应用程序。 Android Studio 默认创建 debug 和 release 构建类型。

#### Product flavors
Product flavors 代表您可以向用户发布的应用程序的不同版本，例如免费和付费版本。 可以自定义 Product flavor 以使用不同的代码和资源，同时共享和重用所有版本的应用程序通用的部分。 Product flavor 是可选的，您必须手动创建它们。

#### Build variants
Build variants 是 build type 和 product flavor 的交叉产品，是 Gradle 用于构建您的应用程序的配置。 使用构建变体，可以在开发期间构建 product flavors 的 debug 版本，并签名 product flavors 的 release 版本用于分发。 虽然您不直接配置构建变体，但您可以配置构成它们的 build types 和 product flavors。 创建额外的 build types 或 product flavors 也会创建额外的构建变体。

#### Manifest entries
可以在构建变体配置中为清单文件的某些属性指定值。 这些构建值会覆盖清单文件中的现有值。 如果您想使用不同的应用程序名称、最低 SDK 版本或目标 SDK 版本生成应用程序的多个变体，这将非常有用。 当存在多个清单时，清单合并工具合并清单设置。

#### Dependencies
构建系统管理来自本地文件系统和远程存储库的项目依赖项。 这意味着您不必手动搜索、下载依赖项的二进制包并将其复制到项目目录中。

#### Signing
构建系统允许您在构建配置中指定签名设置，并且它可以在构建过程中自动签署您的应用程序。 构建系统使用 使用已知凭证的默认密钥和证书对 debug 版本进行签名，以避免在构建时提示密码。 除非您为此构建明确定义签名配置，否则构建系统不会签署 release 版本。 如果您没有 release 密钥，您可以按照为您的应用程序签名中所述生成一个。 通过大多数应用程序商店分发应用程序需要签名的 release 版本。

#### Code and resource shrinking
构建系统允许您为每个构建变体指定不同的 ProGuard 规则文件。 在构建您的应用程序时，构建系统会应用一组适当的规则来使用其内置的缩减工具（例如 R8）缩减您的代码和资源。 压缩代码和资源有助于减小 APK 或 AAB 的大小。

#### Multiple APK support
构建系统可让您自动构建不同的 APK，每个 APK 仅包含特定屏幕密度或应用程序二进制接口 (ABI) 所需的代码和资源。 

## Build configuration 
创建自定义构建配置需要您更改一个或多个构建配置文件或 build.gradle.kts 文件。 这些纯文本文件使用领域特定语言 (DSL) 来描述和操作使用 Kotlin 脚本的构建逻辑，这是 Kotlin 语言的一种风格。 您还可以使用 Groovy（一种用于 Java 虚拟机 (JVM) 的动态语言）来配置您的构建。 用 Groovy 编写的构建脚本称为 build.gradle 文件。

您无需了解 Kotlin 脚本或 Groovy 即可开始配置您的构建，因为 Android Gradle 插件引入了您需要的大部分 DSL 元素。 要了解有关 Android Gradle 插件 DSL 的更多信息，请阅读 DSL 参考文档。 Kotlin 脚本还依赖于底层的 Gradle Kotlin DSL。

当开始一个新项目时，Android Studio 会自动为您创建其中一些文件并根据合理的默认值填充它们。 项目文件结构具有以下布局：
```
└── MyApp/  # Project
    ├── build.gradle.kts
    ├── settings.gradle
    └── app/  # Module
        ├── build.gradle.kts
        └── build/
            ├── libs/
            └── src/
                └── main/  # Source set
                    ├── java/
                    │   └── com.example.myapp
                    ├── res/
                    │   ├── drawable/
                    │   ├── values/
                    │   └── ...
                    └── AndroidManifest.xml
```
有一些 Gradle 构建配置文件是 Android 应用程序的标准项目结构的一部分。 在开始配置构建之前，了解每个文件的范围和用途以及它们定义的基本 DSL 元素非常重要。

### The Gradle settings file
settings.gradle.kts 文件（对于 Kotlin DSL）或 settings.gradle 文件（对于 Groovy DSL）位于根项目目录中。 此设置文件定义项目级 repository 设置，并通知 Gradle 在构建您的应用程序时应包含哪些模块。 多模块项目需要指定应该进入最终构建的每个模块。

### The top-level build file
顶级 build.gradle.kts 文件（对于 Kotlin DSL）或 build.gradle 文件（对于 Groovy DSL）位于根项目目录中。 它定义了适用于项目中所有模块的依赖项。 默认情况下，顶级构建文件使用 `plugins` 块来定义项目中所有模块通用的 Gradle 依赖项。 此外，顶级构建文件包含用于清理构建目录的代码。
```
plugins {

    /**
     * Use `apply false` in the top-level build.gradle file to add a Gradle
     * plugin as a build dependency but not apply it to the current (root)
     * project. Don't use `apply false` in sub-projects. For more information,
     * see Applying external plugins with same version to subprojects.
     */

    id 'com.android.application' version '8.0.0' apply false
    id 'com.android.library' version '8.0.0' apply false
    id 'org.jetbrains.kotlin.android' version '1.8.10' apply false
}

```

#### Configure project-wide properties
对于包含多个模块的 Android 项目，在项目级别定义某些属性并在所有模块之间共享它们可能很有用。 您可以通过向顶级 build.gradle.kts 文件（对于 Kotlin DSL）或 build.gradle 文件（对于 Groovy DSL）中的 ext 块添加额外的属性来做到这一点：
```
// This block encapsulates custom properties and makes them available to all
// modules in the project. The following are only a few examples of the types
// of properties you can define.
ext {
    sdkVersion = 33
    // You can also create properties to specify versions for dependencies.
    // Having consistent versions between modules can avoid conflicts with behavior.
    appcompatVersion = "1.6.1"
    ...
}

```
要从同一项目中的模块访问这些属性，请在模块级构建脚本中使用以下语法。
```
android {
    // Use the following syntax to access properties you defined at the project level:
    // rootProject.ext.property_name


    compileSdk rootProject.ext.sdkVersion
    ...
}
...
dependencies {
    implementation "androidx.appcompat:appcompat:${rootProject.ext.appcompatVersion}"
    ...
}
```

### The module-level build file
模块级 build.gradle.kts（对于 Kotlin DSL）或 build.gradle 文件（对于 Groovy DSL）位于每个 `project/module/` 目录中。 它允许您为其所在的特定模块配置构建设置。配置这些构建设置可以让您提供自定义打包选项，例如其他 build types and product flavors，并覆盖 `main/` 应用程序清单或顶级构建脚本中的设置 .
```
/**
 * The first line in the build configuration applies the Android Gradle plugin
 * to this build and makes the android block available to specify
 * Android-specific build options.
 */

plugins {
    id 'com.android.application'
}

/**
 * The android block is where you configure all your Android-specific
 * build options.
 */

android {

    /**
     * The app's namespace. Used primarily to access app resources.
     */

    namespace 'com.example.myapp'

    /**
     * compileSdk specifies the Android API level Gradle should use to
     * compile your app. This means your app can use the API features included in
     * this API level and lower.
     */

    compileSdk 33

    /**
     * The defaultConfig block encapsulates default settings and entries for all
     * build variants and can override some attributes in main/AndroidManifest.xml
     * dynamically from the build system. You can configure product flavors to override
     * these values for different versions of your app.
     */

    defaultConfig {

        // Uniquely identifies the package for publishing.
        applicationId 'com.example.myapp'

        // Defines the minimum API level required to run the app.
        minSdk 21

        // Specifies the API level used to test the app.
        targetSdk 33

        // Defines the version number of your app.
        versionCode 1

        // Defines a user-friendly version name for your app.
        versionName "1.0"
    }

    /**
     * The buildTypes block is where you can configure multiple build types.
     * By default, the build system defines two build types: debug and release. The
     * debug build type is not explicitly shown in the default build configuration,
     * but it includes debugging tools and is signed with the debug key. The release
     * build type applies ProGuard settings and is not signed by default.
     */

    buildTypes {

        /**
         * By default, Android Studio configures the release build type to enable code
         * shrinking, using minifyEnabled, and specifies the default ProGuard rules file.
         */

        release {
              minifyEnabled true // Enables code shrinking for the release build type.
              proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }

    /**
     * The productFlavors block is where you can configure multiple product flavors.
     * This lets you create different versions of your app that can
     * override the defaultConfig block with their own settings. Product flavors
     * are optional, and the build system does not create them by default.
     *
     * This example creates a free and paid product flavor. Each product flavor
     * then specifies its own application ID, so that they can exist on the Google
     * Play Store, or an Android device, simultaneously.
     *
     * If you declare product flavors, you must also declare flavor dimensions
     * and assign each flavor to a flavor dimension.
     */

    flavorDimensions "tier"
    productFlavors {
        free {
            dimension "tier"
            applicationId 'com.example.myapp.free'
        }

        paid {
            dimension "tier"
            applicationId 'com.example.myapp.paid'
        }
    }
}

/**
 * The dependencies block in the module-level build configuration file
 * specifies dependencies required to build only the module itself.
 * To learn more, go to Add build dependencies.
 */

dependencies {
    implementation project(":lib")
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation fileTree(dir: 'libs', include: ['*.jar'])
}

```

### Gradle properties files
Gradle 还包含两个属性文件，位于您的根项目目录中，您可以使用它们来指定 Gradle 构建工具包本身的设置：

`gradle.properties`

您可以在此处配置项目范围的 Gradle 设置，例如 Gradle 守护程序的最大堆大小。

`local.properties`

为构建系统配置本地环境属性，包括以下内容：
* `ndk.dir` - NDK 的路径。 此属性已被弃用。 NDK 的任何下载版本都安装在 Android SDK 目录中的 ndk 目录中。
* `sdk.dir` - SDK 的路径。
* `cmake.dir` - CMake 的路径。
* `ndk.symlinkdir` - 在 Android Studio 3.5 及更高版本中，创建指向 NDK 的符号链接，该链接可以比安装的 NDK 路径更短。

### Source sets
Android Studio 在逻辑上将每个模块的源代码和资源分组到 source sets 中。 当您创建一个新模块时，Android Studio 会在该模块中创建一个 `main/` source 集。 模块的 `main/` 源集包括其所有构建变体使用的代码和资源。

其他源集目录是可选的，Android Studio 不会在您配置新的构建变体时自动为您创建它们。 但是，与 `main/` 类似，创建源集有助于组织 Gradle 仅在构建特定版本的应用程序时才应使用的文件和资源：

`src/main/`  此源集包括所有构建变体通用的代码和资源。

`src/buildType/`  创建此源集以仅包含特定构建类型的代码和资源。

`src/productFlavor/`  创建此源集以仅包含特定产品风格的代码和资源。

`src/productFlavorBuildType/`  创建此源集以仅包含特定构建变体的代码和资源。

例如，要生成应用程序的“fullDebug”版本，构建系统会合并来自以下源集中的代码、设置和资源：
* `src/fullDebug/` (the build variant source set)
* `src/debug/` (the build type source set)
* `src/full/` (the product flavor source set)
* `src/main/` (the main source set)

注意：当您在 Android Studio 中创建新文件或目录时，请使用 File > New 菜单选项为特定源集创建它。 您可以选择的源集基于您的构建配置，Android Studio 会自动创建所需的目录（如果它们尚不存在）。

如果不同的源集包含同一文件的不同版本，Gradle 在决定使用哪个文件时会使用以下优先级顺序。 左侧的源集覆盖右侧源集的文件和设置：

`build variant > build type > product flavor > main source set > library dependencies `



## Configure the app module

### Set the application ID
每个 Android 应用程序都有一个唯一的应用程序 ID，看起来像 Java 或 Kotlin 包名称，例如 com.example.myapp。 此 ID 在设备上和 Google Play 商店中唯一标识您的应用程序。

重要提示：一旦您发布了您的应用程序，您就永远不应更改应用程序 ID。 如果您更改应用程序 ID，Google Play 商店会将上传视为完全不同的应用程序。 如果您想上传应用的新版本，您必须使用与最初发布时相同的应用程序 ID 和签名证书。

应用程序 ID 由模块的 build.gradle.kts 文件中的 applicationId 属性定义，如下所示：
```
android {
    defaultConfig {
        applicationId "com.example.myapp"
        minSdkVersion 15
        targetSdkVersion 24
        versionCode 1
        versionName "1.0"
    }
    ...
}
```
尽管应用程序 ID 看起来像传统的 Kotlin 或 Java 包名称，但应用程序 ID 的命名规则更加严格：
* 它必须至少有两个段（一个或多个点）。
* 每个段必须以字母开头。
* 所有字符都必须是字母数字或下划线 [a-zA-Z0-9_]。

当您在 Android Studio 中创建新项目时，applicationId 会自动分配您在设置期间选择的包名称。 从那时起，您可以在技术上独立切换这两个属性，但不建议这样做。

建议您在设置应用程序 ID 时执行以下操作：
* 保持应用程序 ID 与命名空间相同。 这两个属性之间的区别可能有点令人困惑，但如果保持它们相同，就没有什么可担心的。
* 发布应用程序后，请勿更改应用程序 ID。 如果您更改它，Google Play 商店会将后续上传的内容视为新应用。
* 显式定义应用程序 ID。 如果未使用 applicationId 属性显式定义应用程序 ID，它会自动采用与命名空间相同的值。 这意味着更改命名空间会更改应用程序 ID，这通常不是您想要的。

注意：应用程序 ID 过去直接与代码的包名称相关联，因此某些 Android API 在其方法名称和参数名称中使用术语“包名称”。 这实际上是您的应用程序 ID。 例如，Context.getPackageName() 方法返回您的应用程序 ID。 永远不需要在应用程序代码之外共享代码的真实包名称。

### Set the namespace
每个 Android 模块都有一个命名空间，用作其生成的 `R` 和 `BuildConfig` 类的 Kotlin 或 Java 包名称。

您的命名空间由模块的 build.gradle.kts 文件中的命名空间属性定义，如以下代码片段所示。 命名空间最初设置为您在创建项目时选择的包名称。

在将您的应用程序构建到最终应用程序包 (APK) 中时，Android 构建工具使用该命名空间作为您应用程序生成的 `R` 类的命名空间，该类用于访问您的应用程序资源。 例如，在前面的构建文件中，`R` 类是在 `com.example.myapp.R` 中创建的。

您为 build.gradle.kts 文件的命名空间属性设置的名称应始终与项目的基本包名称相匹配，您可以在其中保存活动和其他应用程序代码。 您的项目中可以有其他子包，但这些文件必须使用 `namespace` 属性中的命名空间导入 `R` 类。


## Add build dependencies

### Dependency types
要向项目添加依赖项，请在模块的 build.gradle.kts 文件的依赖项块中指定依赖项配置，例如 `implementation`。

例如，应用模块的以下 build.gradle.kts 文件包含三种不同类型的依赖项：
```
plugins {
    id 'com.android.application'
}

android { ... }

dependencies {
    // Dependency on a local library module
    implementation project(':mylibrary')

    // Dependency on local binaries
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    // Dependency on a remote binary
    implementation 'com.example.android:app-magic:12.3'
}
```
这些中的每一个都请求不同类型的库依赖性，如下所示：

#### Local library module dependency
`implementation project(':mylibrary')`

这声明了对名为“mylibrary”的 Android 库模块的依赖（此名称必须与使用 include: 在您的 settings.gradle.kts 文件中定义的库名称相匹配）。 当您构建应用程序时，构建系统会编译库模块并将生成的编译内容打包到应用程序中。

#### Local binary dependency
`implementation fileTree(dir: 'libs', include: ['*.jar'])`

Gradle 在项目的 `module_name/libs/` 目录中声明对 JAR 文件的依赖项（因为 Gradle 读取相对于 build.gradle.kts 文件的路径）。

或者，您可以指定单个文件，如下所示：

`implementation files('libs/foo.jar', 'libs/bar.jar')`

#### Remote binary dependency
`implementation 'com.example.android:app-magic:12.3'`

#### Native dependencies
从 Android Gradle 插件 4.0 开始，本机依赖项也可以按照本页所述的方式导入。

依赖公开本机库的 AAR 将自动使它们可用于 `externalNativeBuild` 使用的构建系统。 要从您的代码访问这些库，您必须在您的本机构建脚本中链接到它们。 在此页面上，请参阅使用本机依赖项。

### Dependency configurations
在 dependencies 块中，您可以使用几种不同的依赖配置之一来声明库依赖。 每个依赖项配置都为 Gradle 提供了有关如何使用依赖项的不同说明。 下表描述了您可以在 Android 项目中用于依赖项的每个配置。 该表还将这些配置与从 Android Gradle 插件 3.0.0 开始弃用的配置进行了比较。

`implementation`  Gradle 将依赖项添加到编译类路径并将依赖项打包到构建输出。 但是，当您的模块配置 **implementation** 依赖项时，它会让 Gradle 知道您不希望模块在编译时将依赖项泄露给其他模块。 也就是说，依赖项仅在运行时对其他模块可用。

使用此依赖项配置而不是 **api** 或 **compile**（已弃用）可以显着缩短构建时间，因为它减少了构建系统需要重新编译的模块数量。 例如，如果一个 **implementation** 依赖项更改了它的 API，Gradle 只会重新编译该依赖项和直接依赖它的模块。 大多数应用程序和测试模块应使用此配置。

`api`  Gradle 将依赖项添加到编译类路径和构建输出。 当一个模块包含一个 **api** 依赖项时，它会让 Gradle 知道该模块想要将该依赖项传递给其他模块，以便它们在运行时和编译时都可用。

此配置的行为就像 **compile**（现已弃用）一样，但您应该谨慎使用它，并且仅在需要传递到其他上游消费者的依赖项中使用。 这是因为，如果 **api** 依赖项更改了其外部 API，Gradle 会在编译时重新编译有权访问该依赖项的所有模块。 因此，拥有大量的 **api** 依赖项会显着增加构建时间。 除非您想将依赖项的 API 公开给单独的模块，否则库模块应该使用 **implementation** 依赖项。

`compileOnly`  Gradle 仅将依赖项添加到编译类路径（即，它不会添加到构建输出）。 这在您创建 Android 模块并且在编译期间需要依赖项时很有用，但在运行时是否存在它是可选的。

如果您使用此配置，那么您的库模块必须包含一个运行时条件以检查依赖项是否可用，然后优雅地更改其行为，以便在未提供时它仍然可以运行。 这有助于通过不添加不重要的临时依赖项来减小最终应用程序的大小。 此配置的行为与提供的一样（现已弃用）。

`runtimeOnly`  Gradle 仅将依赖项添加到构建输出，以供在运行时使用。 也就是说，它不会添加到编译类路径中。 此配置的行为就像 **apk**（现已弃用）。

`annotationProcessor`  要添加对注解处理器库的依赖，您必须使用 **annotationProcessor** 配置将其添加到注解处理器类路径中。 这是因为使用此配置通过将编译类路径与注释处理器类路径分开来提高构建性能。 如果 Gradle 在编译类路径上找到注解处理器，它会停用避免编译，这会对构建时间产生负面影响（Gradle 5.0 及更高版本会忽略在编译类路径上找到的注解处理器）。

如果 Android Gradle 插件的 JAR 文件包含以下文件，则它假定依赖项是注解处理器： META-INF/services/javax.annotation.processing.Processor

如果插件检测到编译类路径上的注解处理器，它会产生构建错误。

注意：Kotlin 项目应该使用 kapt 来声明注释处理器依赖项。

以上所有配置都将依赖项应用于所有构建变体。 如果您只想为特定构建变体源集或测试源集声明依赖项，则必须将配置名称大写，并在其前面加上构建变体或测试源集的名称。

例如，要仅将 `implementation` 依赖项添加到您的“free” product flavor（使用远程二进制依赖项），它看起来像这样：
```
dependencies {
    freeImplementation 'com.google.firebase:firebase-ads:9.8.0'
}
```

但是，如果要为结合了 product flavor and a build type 的变体添加依赖项，则必须在 `configurations` 块中初始化配置名称。 以下示例将 `runtimeOnly` 依赖项添加到您的“freeDebug”构建变体（使用本地二进制依赖项）。
```
configurations {
    // Initializes a placeholder for the freeDebugRuntimeOnly dependency configuration.
    freeDebugRuntimeOnly {}
}

dependencies {
    freeDebugRuntimeOnly fileTree(dir: 'libs', include: ['*.jar'])
}
```



# Migrate build configuration from Groovy to Kotlin
注意：Kotlin 是从 Android Studio Giraffe 开始的构建配置的默认语言。

## Convert the syntax

### 在方法调用中添加括号
Groovy 允许在方法调用中省略括号，而 Kotlin 需要它们。 要迁移您的配置，请为这些类型的方法调用添加括号。 此代码显示如何在 Groovy 中配置设置：
```
compileSdkVersion 30
```
这是用 Kotlin 编写的相同代码：
```
compileSdkVersion(30)
```

### 添加 = 到赋值调用
Groovy DSL 允许在分配属性时省略赋值运算符 =，而 Kotlin 需要它。 此代码显示如何在 Groovy 中分配属性：
```
java {
    sourceCompatibility JavaVersion.VERSION_17
    targetCompatibility JavaVersion.VERSION_17
}
```
此代码显示如何在 Kotlin 中分配属性：
```
java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}
```

### Convert strings
以下是 Groovy 和 Kotlin 之间的字符串差异：
* **字符串的双引号**：虽然 Groovy 允许使用单引号定义字符串，但 Kotlin 需要双引号。
* **点分表达式上的字符串插值**：在 Groovy 中，可以仅使用 `$` 前缀对点分表达式进行字符串插值，但 Kotlin 要求用大括号括起点分表达式。 例如，在 Groovy 中，可以使用 `$project.rootDir`，如以下代码片段所示：
```
myRootDirectory = "$project.rootDir/tools/proguard-rules-debug.pro"
```
然而，在 Kotlin 中，前面的 `project` 在项目上调用 `toString()`，而不是在 `project.rootDir` 上。 要获取根目录的值，请将 `${project.rootDir}` 表达式用花括号括起来：
```
myRootDirectory = "${project.rootDir}/tools/proguard-rules-debug.pro"
```

### Rename file extensions
在迁移其内容时将 `.kts` 附加到每个构建文件。 例如，选择一个构建文件，如 `settings.gradle` 文件。 将文件重命名为 `settings.gradle.kts` 并将文件的内容转换为 Kotlin。 确保项目在每个构建文件迁移后仍然可以编译。

先迁移最小的文件，积累经验，然后继续。 您可以在一个项目中混合使用 Kotlin 和 Groovy 构建文件，因此请花点时间谨慎地进行迁移。

### Replace def with val or var
这是 Groovy 中的变量声明：
```
def building64Bit = false
```
这是用 Kotlin 编写的相同代码：
```
val building64Bit = false
```

### 布尔属性前缀为 is
Groovy 使用基于属性名称的属性推导逻辑。 对于布尔属性 `foo`，其推导方法可以是 `getFoo`、`setFoo` 或 `isFoo`。 因此，一旦转换为 Kotlin，需要将属性名称更改为 Kotlin 不支持的推导方法。 例如，对于 `buildTypes` DSL 布尔元素，您需要在它们前面加上 `is`。 此代码显示如何在 Groovy 中设置布尔属性：
```
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            ...
        }
        debug {
            debuggable true
            ...
        }
    ...
```
以下是 Kotlin 中的相同代码。 请注意，属性以 is 为前缀。
```
android {
    buildTypes {
        getByName("release") {
            isMinifyEnabled = true
            isShrinkResources = true
            ...
        }
        getByName("debug") {
            isDebuggable = true
            ...
        }
    ...
```

### Convert lists and maps
Groovy 和 Kotlin 中的列表和映射是使用不同的语法定义的。 Groovy 使用 `[]`，而 Kotlin 使用 `listOf` 或 `mapOf` 显式调用集合创建方法。 确保在迁移时将 `[]` 替换为 `listOf` 或 `mapOf`。

下面是在 Groovy 和 Kotlin 中定义列表的方法：
```
jvmOptions += ["-Xms4000m", "-Xmx4000m", "-XX:+HeapDumpOnOutOfMemoryError</code>"]
```
这是用 Kotlin 编写的相同代码：
```
jvmOptions += listOf("-Xms4000m", "-Xmx4000m", "-XX:+HeapDumpOnOutOfMemoryError")
```
下面是在 Groovy 和 Kotlin 中定义映射的方法：
```
def myMap = [key1: 'value1', key2: 'value2']
```
这是用 Kotlin 编写的相同代码：
```
val myMap = mapOf("key1" to "value1", "key2" to "value2")
```

### Configure build types
在 Kotlin DSL 中，只有 debug and release 构建类型是隐式可用的。 所有其他自定义构建类型必须手动创建。

在 Groovy 中，可以使用 debug, release 和某些其他构建类型，而无需先创建它们。 以下代码片段显示了 Groovy 中 `debug`, `release`, and `benchmark` 构建类型的配置。
```
buildTypes {
 debug {
   ...
 }
 release {
   ...
 }
 benchmark {
   ...
 }
}
```
要在 Kotlin 中创建等效配置，必须显式创建 `benchmark` 构建类型。
```
buildTypes {
 debug {
   ...
 }

 release {
   ...
 }
 register("benchmark") {
    ...
 }
}
```

## 从构建脚本迁移到插件块
如果您的构建使用 `buildscript {}` 块向项目添加插件，您应该重构为使用 `plugins {}` 块。 `plugins {}` 块使应用插件变得更容易，并且它与 **version catalogs** 配合得很好。

此外，当在构建文件中使用 `plugins {}` 块时，即使构建失败，Android Studio 也会知道上下文。 此上下文有助于修复您的 Kotlin DSL 文件，因为它允许 Studio IDE 执行代码完成并提供其他有用的建议。

### Find the plugin IDs
虽然 `buildscript {}` 块使用插件的 Maven 坐标将插件添加到构建类路径，例如 `com.android.tools.build:gradle:7.4.0`，但 `plugins {}` 块使用插件 ID。

对于大多数插件，插件 ID 是您使用 apply plugin 应用它们时使用的字符串。 例如，以下插件 ID 是 Android Gradle 插件的一部分：
* com.android.application
* com.android.library
* com.android.lint
* com.android.test

Kotlin 插件可以被多个插件 ID 引用。 我们建议使用带命名空间的插件 ID，并通过下表从速记重构为带命名空间的插件 ID：
Shorthand plugin IDs 	Namespaced plugin IDs  
kotlin 	org.jetbrains.kotlin.jvm  
kotlin-android 	org.jetbrains.kotlin.android  
kotlin-kapt 	org.jetbrains.kotlin.kapt  
kotlin-parcelize 	org.jetbrains.kotlin.plugin.parcelize

### 执行重构
知道所用插件的 ID 后，请执行以下步骤：
1. 如果您仍有在 `buildscript {}` 块中声明的插件存储库，请将它们移至 `settings.gradle` 文件。
2. 将插件添加到顶层 `build.gradle` 文件中的 `plugins {}` 块中。 您需要在此处指定插件的 ID 和版本。 如果插件不需要应用于根项目，请使用 `apply false`。
3. 从顶层 `build.gradle.kts` 文件中删除 `classpath` 条目。
4. 通过将插件添加到模块级 `build.gradle` 文件中的 `plugins {}` 块来应用插件。 您只需要在这里指定插件的 ID，因为版本是从根项目继承的。
5. 从模块级 `build.gradle` 文件中删除对插件的 `apply plugin` 调用。

例如，此设置使用 `buildscript {}` 块：
```
// Top-level build.gradle file
buildscript {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
    dependencies {
        classpath("com.android.tools.build:gradle:7.4.0")
        classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:1.8.0")
        ...
    }
}

// Module-level build.gradle file
apply(plugin: "com.android.application")
apply(plugin: "kotlin-android")
```
这是使用 `plugins {}` 块的等效设置：
```
// Top-level build.gradle file
plugins {
   id 'com.android.application' version '7.4.0' apply false
   id 'org.jetbrains.kotlin.android' version '1.8.0' apply false
   ...
}

// Module-level build.gradle file
plugins {
   id 'com.android.application'
   id 'org.jetbrains.kotlin.android'
   ...
}

// settings.gradle
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
```

## Convert the plugins block
从 `plugins {}` 块应用插件在 Groovy 和 Kotlin 中是类似的。 以下代码展示了在使用 **version catalogs** 时如何在 Groovy 中应用插件：
```
// Top-level build.gradle file
plugins {
   alias libs.plugins.android.application apply false
   ...
}

// Module-level build.gradle file
plugins {
   alias libs.plugins.android.application
   ...
}
```
以下代码显示了如何在 Kotlin 中执行相同的操作：
```
// Top-level build.gradle.kts file
plugins {
   alias(libs.plugins.android.application) apply false
   ...
}

// Module-level build.gradle.kts file
plugins {
   alias(libs.plugins.android.application)
   ...
}
```
以下代码展示了在不使用 **version catalogs** 时如何在 Groovy 中应用插件：
```
// Top-level build.gradle file
plugins {
   id 'com.android.application' version '7.3.0' apply false
   ...
}

// Module-level build.gradle file
plugins {
   id 'com.android.application'
   ...
}
```
以下代码显示了如何在 Kotlin 中执行相同的操作：
```
// Top-level build.gradle.kts file
plugins {
   id("com.android.application") version '7.3.0' apply false
   ...
}

// Module-level build.gradle.kts file
plugins {
   id("com.android.application")
   ...
}
```

## 其他
有关其他功能的 Kotlin 代码示例，请参阅以下文档页面：
* If you have a ProGuard configuration, refer to [Enable shrinking, obfuscation, and optimization](https://developer.android.com/build/shrink-code#enable).
* If you have a signingConfig {} block, refer to [Remove signing information from your build files](https://developer.android.com/studio/publish/app-signing#secure-shared-keystore).
* If you use project-wide properties, refer to [Configure project-wide properties](https://developer.android.com/build#project_wide_properties).




