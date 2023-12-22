Android Gradle打包配置

# 背景

1. 在打APK包时，通常会根据上线的应用市场或渠道的区别，打不同的APP包。他们的主要代码逻辑都是一样的，只是有一些细微差别。（如包名不同、部分图片资源不同等）。

2. 我们有些测试代码逻辑，通常我们期望只是buildType为debug的时候，才将这部分文件打包到APK中，线上环境不包含测试代码和资源。

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

1. [构建变体](https://developer.android.com/studio/build/build-variants?hl=zh-cn)（Build variants）的清单文件
   
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

创建Android新项目时，Android Studio 会自动创建其中的部分文件（如图 1 所示），并为其填充默认值。

![](/Users/mtdp/Library/Application%20Support/marktext/images/2023-12-22-18-47-42-image.png)

Android项目里有些默认的Build配置文件，在构建前，需要了解其作用和用法。

## The Gradle Wrapper file


