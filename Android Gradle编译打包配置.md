# 背景

1. 在打APK包时，通常会根据上线的应用市场或渠道的区别，打不同的APP包。他们的主要代码逻辑都是一样的，只是有一些细微差别。（如包名不同、部分图片资源不同等）。

2. 我们有些测试代码逻辑，通常我们期望只是buildType为debug的时候，才将这部分文件打包到APK中，线上环境不包含测试代码和资源。

想要实现以上目的， 就需要用的变体、产品变种等Gradle配置。

# 构建术语

## *Build types*（构建类型）

构建类型定义 Gradle 在构建和打包应用时使用的某些属性。通常针对开发生命周期的不同阶段进行配置。

例如，debug构建类型会启用调试选项，并会使用debug密钥为应用签名；而Release 构建类型则可能会缩减应用大小（配置项`minifyEnabled`、`shrinkResources`）、对应用进行混淆处理（配置项`proguardFiles`），并使用Release密钥为应用签名。

**配置了Build Type后，Gradle也会默认指定一个同名的目录在项目里面，但是不会把目录自动创建出来，我们可以手动创建一个同名的目录**。

<details> <summary>build Type示例</summary>
<pre>
<code>
...
buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles 'proguard_android.txt',
            //noinspection GroovyAssignabilityCheck
            debuggable false
            multiDexEnabled true
        }

         debug {
            //noinspection GroovyAssignabilityCheck
            signingConfig signingConfigs.debugConfig
            multiDexEnabled true
        }
}
...
</code>
</pre>
</details>

## *Product flavors*（产品变种）

产品变种可以向用户发布的不同应用版本，如应用的免费版和付费版。可以自定义产品变种以使用不同的代码和资源，同时共享并重用所有应用版本共用的部分。产品变种是可选的，必须手动创建。

<details>
<summary>Product Flavor示例</summary>
<pre><code>
android {
    ...
    flavorDimensions "default"
    productFlavors {
        free {
            applicationId 'xxx'
            versionCode 'yyy'
            minSdkVersion 26
            dimension "default"
        },
        paid {
            applicationId 'aaa'
            versionCode 'bbb'
            minSdkVersion 26
            dimension "default"
        }
    }
    ...
}
</code></pre>
</details>

## *Build variants*（变体）

变体是 build type与product flavor的交叉产物，也是 Gradle 用来构建应用的配置。利用变体，可以在开发期间构建产品变种的调试版本，以及构建产品变种的已签名发布版本以供发布。**虽然无法直接配置变体，但可以配置构成变体的build type和product flavor组成变体**。创建额外的 build 类型或产品变种也会产生额外的 build 变体。

## *Manifest entries*

可以在变体配置中为mainifest文件的某些属性指定值。这些 build 配置值会覆盖清单文件中的现有值。如果需要为应用生成多个变体，并且让每一个变体都具有不同的应用名称、最低 SDK 版本或目标 SDK 版本，便可运用这一技巧。当存在多个清单时，清单合并工具会[合并清单设置](https://developer.android.com/studio/build/manage-manifests?hl=zh-cn#merge-manifests)。

### Manifest文件合并优先级

合并工具会根据每个清单文件的优先级按顺序合并，将所有清单文件组合到一个文件中。例如，如果有三个清单文件，则会先将优先级最低的清单合并到优先级第二高的清单中，然后再将合并后的清单合并到优先级最高的清单中。

![](https://developer.android.com/static/studio/images/build/manifest-merger_2x.png?hl=zh-cn)

有三种基本的清单文件可以互相合并，它们的合并优先级如下（**按优先级由高到低的顺序**）

1. **[构建变体](https://developer.android.com/studio/build/build-variants?hl=zh-cn)（Build variants）的清单文件**
   
   如果变体有多个源代码集，则其清单优先级如下：
   
   1. 构建变体（Build variants）清单（如 `src/demoDebug/`）
   
   2. 构建类型（Build types）清单（如 `src/debug/`）
   
   3. 产品变种（Product Flavor）清单（如 `src/demo/`）
      
      如果使用的是变种维度，则清单优先级与每个维度在 `flavorDimensions` 属性中的列示顺序（按优先级由高到低的顺序）对应。

2. **应用模块的主清单文件**

3. **所包含的库中的清单文件**
   
   如果您有多个库，则其清单优先级与依赖顺序（库出现在 Gradle `dependencies` 代码块中的顺序）一致。

**总结**：变体的Manifest（变体Manifest > 构建类型Manifest > 产品变种Manifest）  > 工程的Manifest  > 组件库的Manifest

## *Signing*

构建系统既允许在 build 配置中指定签名设置，也可以在构建流程中自动为应用签名。构建系统通过已知凭据使用默认密钥和证书为*Debug*版本签名，以避免在构建时提示输入密码。

## *Code and resource shrinking*

可以为不同的变体指定混淆规则，当编译APP的时候，构建系统会使用一组合适的规则以使用其内置的缩减工具（如 R8）[缩减您的代码和资源](https://developer.android.com/studio/build/shrink-code?hl=zh-cn)。 缩减代码和资源有助于缩减 APK 或 AAB 大小

# Build配置文件

创建Android新项目时，Android Studio 会自动创建其中的部分文件（如下图所示），并为其填充默认值。

<img src="https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/pics/ad905365-06e7-409b-b4a9-c6ee53fa4051.png" title="" alt="" width="442">

Android项目里有些默认的Build配置文件，在构建前，需要了解其作用和用法。

## The Gradle Wrapper file

Gradle wrapper (`gradlew`) 是一个包含在源代码中的小型应用，用于下载和启动 Gradle 本身。这可以创建更一致的构建执行。开发者下载应用源代码并运行 `gradlew`。这将下载所需的 Gradle 发行版，并启动 Gradle 以构建应用。

`gradle/wrapper/gradle-wrapper.properties` 文件包含一个属性 `distributionUrl`，该属性描述了用于运行 build 的 Gradle 版本。

<details><summary>gradle wrapper示例</summary>
<pre><code>
    distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.0-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
</code></pre>
</details>

## Gradle settings file

`settings.gradle.kts` 文件（对应 Kotlin DSL）或 `settings.gradle` 文件（对应 Groovy DSL）位于项目的根目录下。此设置文件会定义项目级代码库设置，并告知 Gradle 在构建应用时应将哪些模块包含在内。多模块项目需要指定应包含在最终 build 中的每个模块。

# Source sets

这个对我们打包构建非常有帮助，尤其是我们只想针对某个build type添加一些代码逻辑，打该build type的APP时包含对应代码，其他build type不含该代码。

Android studio是以source set来组织源代码和资源的。当你创建一个新*module*，Android studio会自动在该module内创建一个名为`main/`的source set。**一个module中的`main/`**

**目录下所包含的源代码和资源，在编译打包时会被所有的变体（build variant）所使用**。即无论打包选择什么变体，`main/`目录下文件都会参与打包。

除了必选的`main/` 这个source set，其他的source set是可选的，而且在配置新的变体的时候Android studio不会自动创建这些source set。当然，就像`main/`一样，可以手动创建其他source set，Gradle构建某个版本（这里版本指不同变体、构建类型和产品变种）的APP时，会用到对应的source set所包含的源代码和资源。

## source set分类

- `src/main/`：无论构建哪个变体，该目录下的源代码和资源都会被使用到。

- `src/buildType/`：只会在构建对应build type的APP时，会使用这个目录下的代码和资源

- `src/productFlavor/`：只会在构建对应product flavor的APP时，会使用这个目录下的代码和资源

- `src/productFlavorBuildType`/：只会在构建对应变体的APP时，会使用这个目录下的代码和资源

例如，如果要构建`fullDebug`（有2个build type，分别为release和debug；2个product flavor，分别为full和free）变体的APP，构建系统将会合并下列目录里的源代码、配置和资源：

- `src/fullDebug/` (the build variant source set)
- `src/debug/` (the build type source set)
- `src/full/` (the product flavor source set)
- `src/main/` (the main source set)

**构建变体的时候，会用到变体本身对应的目录，build type对应的目录，product flavor对应的目录，以及无论什么变体都会使用的main目录**。

## 合并优先级

上文说明了构建变体时参与的目录，这么多目录中可能包含相同的文件，需要规定好合并时的优先级，才能最终确定构建的APP中使用的是哪个目录的文件。

1. 源代码合并
   
   `kotlin/` 或 `java/` 目录中的所有源代码将一起编译以生成单个输出。对于给定的 build 变体，如果 Gradle 遇到两个或更多个源代码集目录定义了同一个 Kotlin 或 Java 类的情况，就会抛出构建错误（duplicate class）。所以**参与构建的所有source set中源代码文件不允许重复**，否则会报错。

2. Manifest文件
   
   所有的Manifest文件最终会合并成一个Manifest文件。优先级是：
   
   **变体的Manifest（变体Manifest > 构建类型Manifest > 产品变种Manifest） > 工程的Manifest > 组件库的Manifest**

3.  `values/`下的资源文件
   
   同名资源文件合并时
   
   **合并优先级：build variant > build type > product flavor > main**

4. `res/` 和 `asset/`下的资源文件
   
   同名资源文件合并时
   
   **合并优先级：build variant > build type > product flavor > main**
