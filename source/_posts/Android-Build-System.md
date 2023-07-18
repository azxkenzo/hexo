---
title: Android - Build System
date: 2023-04-14 11:53:38
tags: Android
---

Android build system 编译应用资源和源代码并将它们打包成 APK 或 Android App Bundle，你可以测试、部署、签名和分发这些应用程序包。

Android Studio 使用高级构建工具包 Gradle 来自动化和管理构建过程，同时允许你定义灵活的自定义构建配置。 每个构建配置都可以定义自己的一组代码和资源，同时重用所有版本的应用程序通用的部分。 Android Gradle plugin 与构建工具包配合使用，提供特定于构建和测试 Android 应用程序的流程和可配置设置。

Gradle 和 AGP 独立于 Android Studio 运行。 这意味着你可以从 Android Studio、你机器上的命令行或未安装 Android Studio 的机器（例如持续集成服务器）上构建你的 Android 应用程序。

## The build process
构建过程涉及许多工具和过程，可将你的项目转换为 Android 应用程序包 (APK) 或 Android App Bundle (AAB)。

AGP 为你完成大部分构建过程，但了解构建过程的某些方面可能很有用，这样你就可以调整构建以满足你的要求。

不同的项目可能有不同的构建目标。 例如，第三方库的构建会生成 AAR 或 JAR 库。 但是，应用是最常见的项目类型，应用项目的构建会生成应用的 debug 或 release APK 或 AAB，你可以部署、测试或发布给外部用户。

## Android build glossary(词汇表)
Gradle 和 AGP 可帮助你配置构建的以下方面：

#### Build types
构建类型定义了 Gradle 在构建和打包你的应用程序时使用的某些属性。 构建类型通常针对开发生命周期的不同阶段进行配置。

例如，debug 构建类型启用 debug 选项并使用 debug 密钥对应用程序进行签名，而 release 构建类型可能会收缩、混淆并使用 release 密钥对你的应用程序进行签名以进行分发。

必须至少定义一种构建类型来构建你的应用程序。 Android Studio 默认创建 debug 和 release 构建类型。

#### Product flavors
Product flavors 代表你可以向用户发布的应用程序的不同版本，例如免费和付费版本。 可以自定义 Product flavor 以使用不同的代码和资源，同时共享和重用所有版本的应用程序通用的部分。 Product flavor 是可选的，你必须手动创建它们。

#### Build variants
Build variants 是 build type 和 product flavor 的交叉产品，是 Gradle 用于构建你的应用程序的配置。 使用构建变体，可以在开发期间构建 product flavors 的 debug 版本，并签名 product flavors 的 release 版本用于分发。 虽然你不直接配置构建变体，但你可以配置构成它们的 build types 和 product flavors。 创建额外的 build types 或 product flavors 也会创建额外的构建变体。

#### Manifest entries
可以在构建变体配置中为清单文件的某些属性指定值。 这些构建值会覆盖清单文件中的现有值。 如果你想使用不同的应用程序名称、最低 SDK 版本或目标 SDK 版本生成应用程序的多个变体，这将非常有用。 当存在多个清单时，清单合并工具合并清单设置。

#### Dependencies
构建系统管理来自本地文件系统和远程存储库的项目依赖项。 这意味着你不必手动搜索、下载依赖项的二进制包并将其复制到项目目录中。

#### Signing
构建系统允许你在构建配置中指定签名设置，并且它可以在构建过程中自动签署你的应用程序。 构建系统使用 使用已知凭证的默认密钥和证书对 debug 版本进行签名，以避免在构建时提示密码。 除非你为此构建明确定义签名配置，否则构建系统不会签署 release 版本。 如果你没有 release 密钥，你可以按照为你的应用程序签名中所述生成一个。 通过大多数应用程序商店分发应用程序需要签名的 release 版本。

#### Code and resource shrinking
构建系统允许你为每个构建变体指定不同的 ProGuard 规则文件。 在构建你的应用程序时，构建系统会应用一组适当的规则来使用其内置的缩减工具（例如 R8）缩减你的代码和资源。 压缩代码和资源有助于减小 APK 或 AAB 的大小。

#### Multiple APK support
构建系统可让你自动构建不同的 APK，每个 APK 仅包含特定屏幕密度或应用程序二进制接口 (ABI) 所需的代码和资源。 

## Build configuration 
创建自定义构建配置需要你更改一个或多个构建配置文件或 build.gradle.kts 文件。 这些纯文本文件使用领域特定语言 (DSL) 来描述和操作使用 Kotlin 脚本的构建逻辑，这是 Kotlin 语言的一种风格。 你还可以使用 Groovy（一种用于 Java 虚拟机 (JVM) 的动态语言）来配置你的构建。 用 Groovy 编写的构建脚本称为 build.gradle 文件。

你无需了解 Kotlin 脚本或 Groovy 即可开始配置你的构建，因为 Android Gradle 插件引入了你需要的大部分 DSL 元素。 要了解有关 Android Gradle 插件 DSL 的更多信息，请阅读 DSL 参考文档。 Kotlin 脚本还依赖于底层的 Gradle Kotlin DSL。

当开始一个新项目时，Android Studio 会自动为你创建其中一些文件并根据合理的默认值填充它们。 项目文件结构具有以下布局：
```
└── MyApp/  # Project
    ├── gradle/
    │   └── wrapper/
    │       └── gradle-wrapper.properties
    ├── build.gradle(.kts)
    ├── settings.gradle(.kts)
    └── app/  # Module
        ├── build.gradle(.kts)
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

### The Gradle Wrapper file
Gradle wrapper (`gradlew`) 是一个包含在源代码中的小型应用程序，用于下载并启动 Gradle 本身。 这将创建更加一致的构建执行。 开发人员下载应用程序源并运行 gradlew。 这将下载所需的 Gradle 发行版，并启动 Gradle 来构建你的应用程序。

`gradle/wrapper/gradle-wrapper.properties` 文件包含一个属性 `distributionUrl`，它描述了用于运行构建的 Gradle 版本。

提示：如果同时处理多个项目，如果可能，请确保所有项目使用相同的 Gradle 版本。 否则，Gradle 会为每个 Gradle 版本创建 Gradle 守护进程的副本，此外还会为用于运行 Gradle 的每个 JDK 创建单独的副本。 这会增加内存和 CPU 使用率，可能会减慢构建速度或影响计算机上的其他工作。

### The Gradle settings file
`settings.gradle.kts` 文件（对于 Kotlin DSL）或 `settings.gradle` 文件（对于 Groovy DSL）位于根项目目录中。 此设置文件定义项目级 repository 设置，并通知 Gradle 在构建你的应用程序时应包含哪些模块。 多模块项目需要指定应该进入最终构建的每个模块。
```
pluginManagement {

    /**
      * The pluginManagement.repositories block configures the
      * repositories Gradle uses to search or download the Gradle plugins and
      * their transitive dependencies. Gradle pre-configures support for remote
      * repositories such as JCenter, Maven Central, and Ivy. You can also use
      * local repositories or define your own remote repositories. The code below
      * defines the Gradle Plugin Portal, Google's Maven repository,
      * and the Maven Central Repository as the repositories Gradle should use to look for its
      * dependencies.
      */

    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
    }
}
dependencyResolutionManagement {

    /**
      * The dependencyResolutionManagement.repositories
      * block is where you configure the repositories and dependencies used by
      * all modules in your project, such as libraries that you are using to
      * create your application. However, you should configure module-specific
      * dependencies in each module-level build.gradle file. For new projects,
      * Android Studio includes Google's Maven repository and the Maven Central
      * Repository by default, but it does not configure any dependencies (unless
      * you select a template that requires some).
      */

  repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
  repositories {
      google()
      mavenCentral()
  }
}
rootProject.name = "My Application"
include(":app")
```

### The top-level build file
顶级 `build.gradle.kts` 文件（对于 Kotlin DSL）或 `build.gradle` 文件（对于 Groovy DSL）位于根项目目录中。 它定义了适用于项目中所有模块的依赖项。 默认情况下，顶级构建文件使用 `plugins` 块来定义项目中所有模块通用的 Gradle 依赖项。 此外，顶级构建文件包含用于清理构建目录的代码。
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
对于包含多个模块的 Android 项目，在项目级别定义某些属性并在所有模块之间共享它们可能很有用。 你可以通过向顶级 build.gradle.kts 文件（对于 Kotlin DSL）或 build.gradle 文件（对于 Groovy DSL）中的 ext 块添加额外的属性来做到这一点：
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
模块级 `build.gradle.kts`（对于 Kotlin DSL）或 `build.gradle` 文件（对于 Groovy DSL）位于每个 `project/module/` 目录中。 它允许你为其所在的特定模块配置构建设置。配置这些构建设置可以让你提供自定义打包选项，例如其他 build types and product flavors，并覆盖 `main/` 应用程序清单或顶级构建脚本中的设置 .
```
/**
 * 构建配置中的第一部分将 Android Gradle 插件应用于此构建，
 * 并使 android 块可用于指定特定于 Android 的构建选项。
 */

plugins {
    id 'com.android.application'
}

/**
 * 找到（并可能下载）用于构建 kotlin 源代码的 JDK。 这也作为 sourceCompatibility、targetCompatibility 和 jvmTarget 的默认值。
 * 请注意，这不会影响使用哪个 JDK 来运行 Gradle 构建本身，并且不需要考虑 Gradle 插件（例如 Android Gradle Plugin）所需的 JDK 版本
 */
kotlin {
    jvmToolchain 11
}


/**
 * android 块是你配置所有 Android 特定构建选项的地方。
 */

android {

    /**
     * The app's namespace. Used primarily to access app resources.
     */

    namespace 'com.example.myapp'

    /**
     * compileSdk 指定 Gradle 应该用来编译你的应用程序的 Android API 级别。
     * 这意味着你的应用程序可以使用此 API 级别及更低级别中包含的 API 功能。
     */

    compileSdk 33

    /**
     * defaultConfig 块封装了所有构建变体的默认设置和条目，
     * 并且可以从构建系统动态覆盖 main/AndroidManifest.xml 中的一些属性。
     * 你可以配置产品 flavor 来覆盖不同版本应用程序的这些值。
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
    
    
    /**
     * 要覆盖源和目标兼容性（如果与toolchain JDK 版本不同），请添加以下内容。
     * 所有这些默认值与 kotlin.jvmToolchain 相同。 如果你对这些值和 kotlin.jvmToolchain 使用相同的版本，则可以删除这些块。
     */
    //compileOptions {
    //    sourceCompatibility JavaVersion.VERSION_11
    //    targetCompatibility JavaVersion.VERSION_11
    //}
    //kotlinOptions {
    //    jvmTarget = '11'
    //}

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
Gradle 还包含两个属性文件，位于你的根项目目录中，你可以使用它们来指定 Gradle 构建工具包本身的设置：

`gradle.properties`

你可以在此处配置项目范围的 Gradle 设置，例如 Gradle 守护程序的最大堆大小。

`local.properties`

为构建系统配置本地环境属性，包括以下内容：
* `ndk.dir` - NDK 的路径。 此属性已被弃用。 NDK 的任何下载版本都安装在 Android SDK 目录中的 ndk 目录中。
* `sdk.dir` - SDK 的路径。
* `cmake.dir` - CMake 的路径。
* `ndk.symlinkdir` - 在 Android Studio 3.5 及更高版本中，创建指向 NDK 的符号链接，该链接可以比安装的 NDK 路径更短。

### Source sets
Android Studio 在逻辑上将每个模块的源代码和资源分组到 source sets 中。 当你创建一个新模块时，Android Studio 会在该模块中创建一个 `main/` source 集。 模块的 `main/` 源集包括其所有构建变体使用的代码和资源。

其他源集目录是可选的，Android Studio 不会在你配置新的构建变体时自动为你创建它们。 但是，与 `main/` 类似，创建源集有助于组织 Gradle 仅在构建特定版本的应用程序时才应使用的文件和资源：

`src/main/`  此源集包括所有构建变体通用的代码和资源。

`src/buildType/`  创建此源集以仅包含特定构建类型的代码和资源。

`src/productFlavor/`  创建此源集以仅包含特定产品风格的代码和资源。

`src/productFlavorBuildType/`  创建此源集以仅包含特定构建变体的代码和资源。

例如，要生成应用程序的“fullDebug”版本，构建系统会合并来自以下源集中的代码、设置和资源：
* `src/fullDebug/` (the build variant source set)
* `src/debug/` (the build type source set)
* `src/full/` (the product flavor source set)
* `src/main/` (the main source set)

注意：当你在 Android Studio 中创建新文件或目录时，请使用 File > New 菜单选项为特定源集创建它。 你可以选择的源集基于你的构建配置，Android Studio 会自动创建所需的目录（如果它们尚不存在）。

如果不同的源集包含同一文件的不同版本，Gradle 在决定使用哪个文件时会使用以下优先级顺序。 左侧的源集覆盖右侧源集的文件和设置：

`build variant > build type > product flavor > main source set > library dependencies `


## Java versions in Android builds
无论你的源代码是用 Java、Kotlin 还是两者编写的，你都必须在多个地方为你的构建选择 JDK 或 Java 语言版本。

### 词汇表
#### Java Development Kit (JDK)
JDK 包含：
* 工具，例如编译器、分析器和存档创建器。 这些在构建过程中在幕后用于创建应用程序。
* 包含可从 Kotlin 或 Java 源代码调用的 API 的库。 请注意，并非所有功能均可在 Android 上使用。
* Java 虚拟机 (JVM)，执行 Java 应用程序的解释器。 你使用 JVM 来运行 Android Studio IDE 和 Gradle 构建工具。 Android 设备或模拟器上不使用 JVM。

#### JetBrains Runtime (JBR)
JetBrains Runtime (JBR) 是一个增强的 JDK，随 Android Studio 一起分发。 它包括在 Studio 和相关 JetBrains 产品中使用的多项优化，但也可用于运行其他 Java 应用程序。

### 如何选择 JDK 来运行 Android Studio？
我们建议你使用 JBR 来运行 Android Studio。 它与 Android Studio 一起部署并用于测试 Android Studio，并包含优化 Android Studio 使用的增强功能。 为确保这一点，请勿设置 STUDIO_JDK 环境变量。

Android Studio 的启动脚本按以下顺序查找 JVM：
1. `STUDIO_JDK` environment variable
2. `studio.jdk` directory (in the Android Studio distribution)
3. `jbr` directory (JetBrains Runtime), in the Android Studio distribution. Recommended.
4. `JDK_HOME` environment variable
5. `JAVA_HOME` environment variable
6. java executable in the `PATH` environment variable

### 如何选择运行我的 Gradle 构建的 JDK？
如果你使用 Android Studio 中的按钮运行 Gradle，则将使用 Android Studio 设置中设置的 JDK 来运行 Gradle。 如果你在 Android Studio 内部或外部的终端中运行 Gradle，JAVA_HOME 环境变量（如果设置）将确定哪个 JDK 运行 Gradle 脚本。 如果未设置 JAVA_HOME，它将使用 PATH 环境变量中的 java 命令。

为了获得最一致的结果，请确保设置 JAVA_HOME 环境变量，并将 Android Studio 中的 Gradle JDK 设置为相同的 JDK。

注意：如果你通过右键单击并选择使用 IDE 运行突出显示的命令来在 Android Studio 终端中运行 Gradle 命令，那么它将使用 Android Studio 设置中的 JDK，而不是 JAVA_HOME。

运行构建时，Gradle 会创建一个称为 daemon 的进程来执行实际构建。 只要构建使用相同的 JDK 和 Gradle 版本，就可以重用此过程。 重用 daemon 可以减少启动新 JVM 和初始化构建系统的时间。

如果你使用不同的 JDK 或 Gradle 版本开始构建，则会创建额外的 daemon，从而消耗更多的 CPU 和内存。

提示：同时处理多个项目时，如果可能，请在其 gradle-wrapper.properties 文件中指定相同的 Gradle 版本，以减少创建的 Gradle daemon 的数量。

#### Set the Gradle JDK in Android Studio
要设置并选择下载 Android Studio 用于运行 Gradle 的 JDK，请转至 Settings > Build, Execution, Deployment > Build Tools > Gradle and edit the Gradle JDK field.

确保选择的 JDK 版本高于或等于你在 Gradle 构建中使用的插件所使用的 JDK 版本。 要确定 Android Gradle 插件 (AGP) 所需的最低 JDK 版本，请参阅发行说明中的兼容性表。

例如，Android Gradle 插件版本 8.x 需要 JDK 17。

### 我可以在 Java 或 Kotlin 源代码中使用哪些 Java API？
Android 应用程序可以使用 JDK 中定义的部分 API，但不是全部。 Android SDK 定义了许多 Java 库函数的实现作为其可用 API 的一部分。 `compileSdk` 属性指定编译 Kotlin 或 Java 源代码时要使用的 Android SDK 版本。

每个版本的 Android 都支持特定版本的 JDK 及其可用 Java API 的子集。 如果你使用的 Java API 在 `compileSdk` 中可用，而在指定的 `minSdk` 中不可用，则你可以通过称为脱糖的过程在早期版本的 Android 中使用该 API。 请参阅通过脱糖提供的 Java 11+ API，了解受支持的 API。

使用下表确定每个 Android API 支持哪个 Java 版本，以及在哪里可以找到有关可用 Java API 的详细信息。

Android 	Java 	API and language features supported  
14 (API 34) 	17 	Core libraries  
13 (API 33) 	11 	Core libraries  
12 (API 32) 	11 	Java API  
11 and lower 		Android versions  

### 哪个 JDK 编译我的 Java 源代码？
Java toolchain JDK 包含用于构建任何 Java 源代码的 Java 编译器。 该 JDK 还运行 Kotlin 编译器（在 JVM 上运行）。

toolchain默认为用于运行 Gradle 的 JDK。 如果你使用默认值并在不同的计算机（例如，本地计算机和单独的持续集成服务器）上运行构建，则使用不同的 JDK 版本时，构建的结果可能会有所不同。

要创建更一致的构建，你可以显式指定 Java toolchain版本。 指定这一点：
* 在运行构建的系统上找到兼容的 JDK。
  * 如果不存在兼容的 JDK（并且定义了toolchain解析器），则下载一个。
* 公开toolchain Java API 以供源代码调用。
* 使用 Java 语言版本编译 Java 源代码。
* 提供 `sourceCompatibility` 和 `targetCompatibility` 的默认值。

我们建议你始终指定 Java toolchain，并确保安装了指定的 JDK，或者将toolchain解析器添加到你的构建中。

无论你的源代码是用 Java、Kotlin 还是两者编写的，你都可以指定toolchain。 在模块的 `build.gradle(.kts)` 文件的顶层指定toolchain。

如果你的源代码仅用 Java 编写，请指定 Java toolchain版本，如下所示：
```
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}
```

如果你的源仅是 Kotlin 或 Kotlin 和 Java 的混合，请指定 Java toolchain版本，如下所示：
```
kotlin {
    jvmToolchain 17
}
```

toolchain JDK 版本可以与用于运行 Gradle 的 JDK 相同，但请记住它们有不同的用途。

### 我可以在 Java 源代码中使用哪些 Java 语言源功能？
`sourceCompatibility` 属性确定在编译 Java 源代码期间哪些 Java 语言功能可用。 它不影响 Kotlin 源。

如果未指定，则默认为用于运行 Gradle 的 Java toolchain或 JDK。 我们建议你始终显式指定toolchain（首选）或 `sourceCompatibility`。

在模块的 build.gradle(.kts) 文件中指定 `sourceCompatibility`。
```
android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
    }
}
```
注意：从 Android Studio Giraffe 开始，导入项目时，`sourceCompatibility` 选项也用作编写 Java 源代码时 IDE 代码辅助和 linting 的默认选项。 某些 Java 语言功能需要库支持，并且在 Android 上不可用。 compileSdk 选项确定哪些库可用。 其他功能（例如 switch 表达式）仅需要 Java 编译器并可在 Android 上运行。

### 编译 Kotlin 或 Java 源代码时可以使用哪些 Java 二进制功能？
指定 `targetCompatibility` 和 `jvmTarget` 分别确定为编译的 Java 和 Kotlin 源生成字节码时使用的 Java 类格式版本。

一些 Kotlin 功能在添加等效的 Java 功能之前就已存在。 早期的 Kotlin 编译器必须创建自己的方式来表示这些 Kotlin 功能。 其中一些功能后来被添加到 Java 中。 在更高的 jvmTarget 级别中，Kotlin 编译器可能会直接使用 Java 功能，这可能会带来更好的性能。

`targetCompatibility` 默认与 `sourceCompatibility` 的值相同，但如果指定，则必须大于或等于 `sourceCompatibility`。

`jvmTarget` 默认为toolchain版本。

不同版本的Android支持不同版本的Java。 你可以通过增加 `targetCompatibility` 和 `jvmTarget` 来利用其他 Java 功能，但这可能会迫使你还增加最低 Android SDK 版本以确保该功能可用。
```
android {
    compileOptions {
        targetCompatibility JavaVersion.VERSION_17
    }
    kotlinOptions {
        jvmTarget '17'
    }
}
```



## Configure the app module

### Set the application ID
每个 Android 应用程序都有一个唯一的应用程序 ID，看起来像 Java 或 Kotlin 包名称，例如 com.example.myapp。 此 ID 在设备上和 Google Play 商店中唯一标识你的应用程序。

重要提示：一旦你发布了你的应用程序，你就永远不应更改应用程序 ID。 如果你更改应用程序 ID，Google Play 商店会将上传视为完全不同的应用程序。 如果你想上传应用的新版本，你必须使用与最初发布时相同的应用程序 ID 和签名证书。

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

当你在 Android Studio 中创建新项目时，applicationId 会自动分配你在设置期间选择的包名称。 从那时起，你可以在技术上独立切换这两个属性，但不建议这样做。

建议你在设置应用程序 ID 时执行以下操作：
* 保持应用程序 ID 与命名空间相同。 这两个属性之间的区别可能有点令人困惑，但如果保持它们相同，就没有什么可担心的。
* 发布应用程序后，请勿更改应用程序 ID。 如果你更改它，Google Play 商店会将后续上传的内容视为新应用。
* 显式定义应用程序 ID。 如果未使用 applicationId 属性显式定义应用程序 ID，它会自动采用与命名空间相同的值。 这意味着更改命名空间会更改应用程序 ID，这通常不是你想要的。

注意：应用程序 ID 过去直接与代码的包名称相关联，因此某些 Android API 在其方法名称和参数名称中使用术语“包名称”。 这实际上是你的应用程序 ID。 例如，Context.getPackageName() 方法返回你的应用程序 ID。 永远不需要在应用程序代码之外共享代码的真实包名称。

### Set the namespace
每个 Android 模块都有一个命名空间，用作其生成的 `R` 和 `BuildConfig` 类的 Kotlin 或 Java 包名称。

你的命名空间由模块的 build.gradle.kts 文件中的命名空间属性定义，如以下代码片段所示。 命名空间最初设置为你在创建项目时选择的包名称。

在将你的应用程序构建到最终应用程序包 (APK) 中时，Android 构建工具使用该命名空间作为你应用程序生成的 `R` 类的命名空间，该类用于访问你的应用程序资源。 例如，在前面的构建文件中，`R` 类是在 `com.example.myapp.R` 中创建的。

你为 build.gradle.kts 文件的命名空间属性设置的名称应始终与项目的基本包名称相匹配，你可以在其中保存活动和其他应用程序代码。 你的项目中可以有其他子包，但这些文件必须使用 `namespace` 属性中的命名空间导入 `R` 类。


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

这声明了对名为“mylibrary”的 Android 库模块的依赖（此名称必须与使用 include: 在你的 settings.gradle.kts 文件中定义的库名称相匹配）。 当你构建应用程序时，构建系统会编译库模块并将生成的编译内容打包到应用程序中。

#### Local binary dependency
`implementation fileTree(dir: 'libs', include: ['*.jar'])`

Gradle 在项目的 `module_name/libs/` 目录中声明对 JAR 文件的依赖项（因为 Gradle 读取相对于 build.gradle.kts 文件的路径）。

或者，你可以指定单个文件，如下所示：

`implementation files('libs/foo.jar', 'libs/bar.jar')`

#### Remote binary dependency
`implementation 'com.example.android:app-magic:12.3'`

#### Native dependencies
从 Android Gradle 插件 4.0 开始，本机依赖项也可以按照本页所述的方式导入。

依赖公开本机库的 AAR 将自动使它们可用于 `externalNativeBuild` 使用的构建系统。 要从你的代码访问这些库，你必须在你的本机构建脚本中链接到它们。 在此页面上，请参阅使用本机依赖项。

### Dependency configurations
在 dependencies 块中，你可以使用几种不同的依赖配置之一来声明库依赖。 每个依赖项配置都为 Gradle 提供了有关如何使用依赖项的不同说明。 下表描述了你可以在 Android 项目中用于依赖项的每个配置。 该表还将这些配置与从 Android Gradle 插件 3.0.0 开始弃用的配置进行了比较。

`implementation`  Gradle 将依赖项添加到编译类路径并将依赖项打包到构建输出。 但是，当你的模块配置 **implementation** 依赖项时，它会让 Gradle 知道你不希望模块在编译时将依赖项泄露给其他模块。 也就是说，依赖项仅在运行时对其他模块可用。

使用此依赖项配置而不是 **api** 或 **compile**（已弃用）可以显着缩短构建时间，因为它减少了构建系统需要重新编译的模块数量。 例如，如果一个 **implementation** 依赖项更改了它的 API，Gradle 只会重新编译该依赖项和直接依赖它的模块。 大多数应用程序和测试模块应使用此配置。

`api`  Gradle 将依赖项添加到编译类路径和构建输出。 当一个模块包含一个 **api** 依赖项时，它会让 Gradle 知道该模块想要将该依赖项传递给其他模块，以便它们在运行时和编译时都可用。

此配置的行为就像 **compile**（现已弃用）一样，但你应该谨慎使用它，并且仅在需要传递到其他上游消费者的依赖项中使用。 这是因为，如果 **api** 依赖项更改了其外部 API，Gradle 会在编译时重新编译有权访问该依赖项的所有模块。 因此，拥有大量的 **api** 依赖项会显着增加构建时间。 除非你想将依赖项的 API 公开给单独的模块，否则库模块应该使用 **implementation** 依赖项。

`compileOnly`  Gradle 仅将依赖项添加到编译类路径（即，它不会添加到构建输出）。 这在你创建 Android 模块并且在编译期间需要依赖项时很有用，但在运行时是否存在它是可选的。

如果你使用此配置，那么你的库模块必须包含一个运行时条件以检查依赖项是否可用，然后优雅地更改其行为，以便在未提供时它仍然可以运行。 这有助于通过不添加不重要的临时依赖项来减小最终应用程序的大小。 此配置的行为与提供的一样（现已弃用）。

`runtimeOnly`  Gradle 仅将依赖项添加到构建输出，以供在运行时使用。 也就是说，它不会添加到编译类路径中。 此配置的行为就像 **apk**（现已弃用）。

`annotationProcessor`  要添加对注解处理器库的依赖，你必须使用 **annotationProcessor** 配置将其添加到注解处理器类路径中。 这是因为使用此配置通过将编译类路径与注释处理器类路径分开来提高构建性能。 如果 Gradle 在编译类路径上找到注解处理器，它会停用避免编译，这会对构建时间产生负面影响（Gradle 5.0 及更高版本会忽略在编译类路径上找到的注解处理器）。

如果 Android Gradle 插件的 JAR 文件包含以下文件，则它假定依赖项是注解处理器： META-INF/services/javax.annotation.processing.Processor

如果插件检测到编译类路径上的注解处理器，它会产生构建错误。

注意：Kotlin 项目应该使用 kapt 来声明注释处理器依赖项。

以上所有配置都将依赖项应用于所有构建变体。 如果你只想为特定构建变体源集或测试源集声明依赖项，则必须将配置名称大写，并在其前面加上构建变体或测试源集的名称。

例如，要仅将 `implementation` 依赖项添加到你的“free” product flavor（使用远程二进制依赖项），它看起来像这样：
```
dependencies {
    freeImplementation 'com.google.firebase:firebase-ads:9.8.0'
}
```

但是，如果要为结合了 product flavor and a build type 的变体添加依赖项，则必须在 `configurations` 块中初始化配置名称。 以下示例将 `runtimeOnly` 依赖项添加到你的“freeDebug”构建变体（使用本地二进制依赖项）。
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
Groovy 允许在方法调用中省略括号，而 Kotlin 需要它们。 要迁移你的配置，请为这些类型的方法调用添加括号。 此代码显示如何在 Groovy 中配置设置：
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

先迁移最小的文件，积累经验，然后继续。 你可以在一个项目中混合使用 Kotlin 和 Groovy 构建文件，因此请花点时间谨慎地进行迁移。

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
Groovy 使用基于属性名称的属性推导逻辑。 对于布尔属性 `foo`，其推导方法可以是 `getFoo`、`setFoo` 或 `isFoo`。 因此，一旦转换为 Kotlin，需要将属性名称更改为 Kotlin 不支持的推导方法。 例如，对于 `buildTypes` DSL 布尔元素，你需要在它们前面加上 `is`。 此代码显示如何在 Groovy 中设置布尔属性：
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
如果你的构建使用 `buildscript {}` 块向项目添加插件，你应该重构为使用 `plugins {}` 块。 `plugins {}` 块使应用插件变得更容易，并且它与 **version catalogs** 配合得很好。

此外，当在构建文件中使用 `plugins {}` 块时，即使构建失败，Android Studio 也会知道上下文。 此上下文有助于修复你的 Kotlin DSL 文件，因为它允许 Studio IDE 执行代码完成并提供其他有用的建议。

### Find the plugin IDs
虽然 `buildscript {}` 块使用插件的 Maven 坐标将插件添加到构建类路径，例如 `com.android.tools.build:gradle:7.4.0`，但 `plugins {}` 块使用插件 ID。

对于大多数插件，插件 ID 是你使用 apply plugin 应用它们时使用的字符串。 例如，以下插件 ID 是 Android Gradle 插件的一部分：
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
1. 如果你仍有在 `buildscript {}` 块中声明的插件存储库，请将它们移至 `settings.gradle` 文件。
2. 将插件添加到顶层 `build.gradle` 文件中的 `plugins {}` 块中。 你需要在此处指定插件的 ID 和版本。 如果插件不需要应用于根项目，请使用 `apply false`。
3. 从顶层 `build.gradle.kts` 文件中删除 `classpath` 条目。
4. 通过将插件添加到模块级 `build.gradle` 文件中的 `plugins {}` 块来应用插件。 你只需要在这里指定插件的 ID，因为版本是从根项目继承的。
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




