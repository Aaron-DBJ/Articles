## 一、插桩技术简介

插桩是什么？

所谓的插桩就是在代码编译期间修改已有的代码或者生成新代码。

与之相似的是另外一个概念AOP。

*Aspect Oriented Programming（AOP）*，面向切面编程。这个概念的提出主要是相对于*OOP*（面向对象编程）。*OOP*能够将项目划分为多个模块，但有些功能是各模块都需要的，例如性能监控、日志管理等，***AOP*便是一刀切入（切开并织入）多个模块，为这些模块提供功能，也为这些功能提供统一的管理**。

实现*AOP*的一些常见方案有：

- 动态AOP方式：动态代理，*epic*和*cglib*

- 静态AOP方式，如*ASM*（字节码插桩），*javassist*和*AspectJ*

每种方案都有各自的适用场景和优劣点，如动态代理每次只能针对一个类进行，如果要同时改动几百处，该方式显然就不合适。

一些常见概念介绍：

- 插桩：在代码编译期间修改已有的代码或者生成新代码

- *Transform API*：利用*Transform API*，第三方的插件可以在.*class*文件转为*dex*文件之前，对一些.class 文件进行处理。它简化了这个处理过程，且使用起来很灵活。

## 1.1 Transform

*Transform API* 是 *Android Gradle Plugin 1.5* 就引入的特性，主要用于在 *Android* 构建过程中，在 *Class* → *Dex* 这个节点修改 *Class* 字节码。利用 *Tranxsform API*，我们可以拿到所有参与构建的 Class 文件，借助 Javassist 或 *ASM* 等字节码编辑工具进行修改，插入自定义逻辑。一般来说，这些自定义逻辑是与业务逻辑无关的。<sup>1</sup>

使用 *Transform* 的常见的应用场景有：

- **埋点统计：** 在页面展现和退出等生命周期中插入埋点统计代码，以统计页面展现数据；

- **耗时监控：** 在指定方法的前后插入耗时计算，以观察方法执行时间；

- **方法替换：** 将方法调用替换为调用另一个方法。

# 二、插桩的原理

插桩就是在java代码在被编译过程中从.class到.dex文件这个环节，通过修改已有代码或生成新代码，来实现特定的功能。插桩时机如下图所示👇🏻：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f804eea0f32a42bfb8e6b40f98863545~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

# 三、插桩的实现

通常我们实现插桩的载体是构建一个*gradle*插件，利用*Transform API*和一些插桩框架来实现。目前常见的插桩框架有*Javassist*、*ASM*等，其中*Javassist*具有上手简单，学习成本低的特点，对新手学习插桩来说比较友好。但是相对来说，其性能表现是弱于*ASM*的。

本文以*Transform API*和*Javassist*为例来说明下插桩实现的流程。

## 3.1 实现插桩的步骤

1、新建一个*Java or Kotlin module*来实现插件

<img src="file:///Users/mtdp/Library/Application%20Support/marktext/images/2022-11-21-17-47-59-image.png" title="" alt="" width="348">

2、删除main目录下的其他文件，新建groovy和resources目录

<img src="file:///Users/mtdp/Library/Application%20Support/marktext/images/2022-11-21-17-49-21-image.png" title="" alt="" width="349">

3、由于要实现插件和使用*Javassist*库，在*module*的*build*文件中依赖基础插件和引入*Javassist*依赖

```groovy
plugins {
    id 'groovy' // （必选）用于实现插件
    id 'maven' // 用于发布插件
}

// 省略部分配置
......

dependencies {
    implementation localGroovy()
    implementation gradleApi()
    implementation 'com.android.tools.build:gradle:4.1.3'
    implementation 'org.javassist:javassist:3.21.0-GA'
}
```

这里依赖了2个插件，groovy是实现自定义插件必备的插件；maven是用于发布我们自己实现的插件，这样其他工程就可以远程依赖并使用自定义插件的能力。

另外，还需要依赖*Javassist*库，这里我们使用的版本是*3.21.0-GA*

4、准备工作完成后，就可以开始在groovy目录下实现插桩代码，新建Transform文件，继承自Transform，代码如下👇：

```groovy
package com.aaron.plugin.arronplugin

import com.android.SdkConstants
import com.android.build.api.transform.*
import com.android.build.gradle.internal.pipeline.TransformManager
import javassist.ClassClassPath
import javassist.ClassPool
import org.apache.commons.io.FileUtils
import org.gradle.api.Project

class AaronTransform extends Transform {
    ClassPool mClassPool
    File mOutFile
    Project mProject

    AaronTransform(Project mProject) {
        this.mProject = mProject
    }

    @Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        // 省略，插桩代码逻辑实现处
        ......
    }

    @Override
    String getName() {
        return "AaronTransform"
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    @Override
    boolean isIncremental() {
        return false
    }
}
```

继承*Transform*后，有一些重写的方法：

- `getName`：自定义插件的名称

- `getInputTypes`：自定义插件可以处理的输入数据类型，这里一般是写类

- `getScopes`：*Transform*可以处理的数据范畴，通常填`TransformManager.SCOPE_FULL_PROJECT`

- `isIncremental`：是否是增量编译，*false*表明每次都重新编译全部文件，*true*是只编译增量部分

- `transform`：这是最重要的一个方法，是我们实现插桩代码的入口。

5、**插桩逻辑的实现**

作为演示，我们在一个方法开始位置增加一条日志打印和新建一个方法弹个*toast*。

在`transform`方法中，接受一个参数*TransformInvocation*，它包含所有我们指定范围的输入数据，我们所做的就是遍历所有输入数据，然后根据需求逐一添加自定义的逻辑。

```groovy
@Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        // 清除上次编译内容
        transformInvocation.outputProvider.deleteAll()

        // 初始化一些常规配置
        init(transformInvocation)

        ArrayList<String> classNames = new ArrayList<>()
        // 输入数据有2类，DirectoryInput和JarInput，前者表示工程中所有的文件，后者表示依赖的jar、aar文件
        transformInvocation.inputs.each { TransformInput input ->
            input.directoryInputs.each { DirectoryInput directoryInput ->
                def dirPath = directoryInput.getFile().getAbsolutePath()
                // 添加下class path，避免找不到类的错误
                mClassPool.appendClassPath(dirPath)
                // 打印DirectoryInput路径
                println("DirectoryInput 绝对路径：$dirPath")
                FileUtils.listFiles(directoryInput.file, null, true).each { file ->
                    println("文件绝对路径：$file.absolutePath")

                    if (file.absolutePath.endsWith(SdkConstants.DOT_CLASS)) {
                        def className = file.absolutePath.substring(dirPath.length() + 1, file.absolutePath.length() - SdkConstants.DOT_CLASS.length()).replaceAll('/', '.')
                        println("类名：$className")
                        classNames.add(className)
                    }
                }
            }

            input.jarInputs.each { JarInput jarInput ->
                // 和处理DirectoryInput的逻辑一致
            }
        }

        handleDirectoryInputs(classNames)
    }

    void init(TransformInvocation transformInvocation) {
        // 使用javassist框架hook原程序
        mClassPool = ClassPool.getDefault()
        // 添加搜索路径，这样hook时才能找到对应类
        //project.android.bootClasspath 加入android.jar，不然找不到android相关的所有类
        mClassPool.appendClassPath(mProject.android.bootClasspath[0].toString())
        mClassPool.importPackage('android.os.Bundle')
        mClassPool.insertClassPath(new ClassClassPath(this.getClass()))

        // 设置hook后的文件输出路径
        mOutFile = transformInvocation.outputProvider.getContentLocation("main", outputTypes, scopes, Format.DIRECTORY)
        println("插桩后的文件输出路径：$mOutFile.absolutePath")
    }

    public void handleDirectoryInputs(List<String> classNames) {
        if (classNames == null || classNames.isEmpty()) {
            return
        }

        CodeInjector.injectHookCode(mClassPool, classNames, mOutFile)
    }
```

实现插桩时通常步骤：

- 清除上次编译的内容，防止插桩结果不合预期

- 进行一些初始化的配置，例如创建ClassPool对象，导入包，添加一些类路径（避免找不到类的问题），设置插桩文件的输出路径等

- 遍历输入数据，记录全限定类名，然后对每个类进行插桩

插桩代码：

```groovy
package com.aaron.plugin.arronplugin

import javassist.*

class CodeInjector {
    public static final String VIEW_PAGER_HELPER = "com.example.ffmpegkit.banner.ViewPagerHelper"
    public static final String VIEW_PAGER_ACTIVITY = "com.example.ffmpegkit.activity.ViewPagerActivity"

    public static void injectHookCode(ClassPool classPool, List<String> classNames, File outDir) {
        if (classPool == null || classNames == null || classNames.isEmpty()) {
            return
        }

        classNames.each { className ->
            CtClass ctClass = classPool.getCtClass(className)
            if (ctClass.isFrozen()) {
                ctClass.defrost()
            }

            injectLog(ctClass)
            injectToast(classPool, ctClass)

            ctClass.writeFile(outDir.absolutePath)
            ctClass.detach()
        }
    }

    protected static void injectLog(CtClass ctClass) {
        if (ctClass == null) {
            return
        }

        String className = ctClass.name
        if (className != null && className.endsWith(VIEW_PAGER_HELPER)) {
            CtConstructor ctConstructorMethod = ctClass.getDeclaredConstructor(null)
            ctConstructorMethod.insertAfter("android.util.Log.d(\"插桩的日志\", \"ViewPagerHelper无参构造方法被插桩一条日志\");")

            CtMethod ctMethod = ctClass.getDeclaredMethod("startAutoScroll")
            ctMethod.insertBefore("android.util.Log.d(\"插桩的日志\", \"ViewPagerHelper startAutoScroll被执行\");")
        }
    }

    protected static void injectToast(ClassPool classPool, CtClass ctClass) {
        if (ctClass == null || classPool == null) {
            return
        }

        String className = ctClass.name
        if (className != null && className.endsWith(VIEW_PAGER_ACTIVITY)) {
            CtClass[] params = new CtClass[1]
            params[0] = classPool.get("android.content.Context")
            CtMethod createMethod = new CtMethod(CtClass.voidType, "toastHook", params, ctClass)
            createMethod.setModifiers(Modifier.PUBLIC & ~Modifier.ABSTRACT)
            createMethod.setBody("android.widget.Toast.makeText(\$1, \"我是插桩实现的Toast\", android.widget.Toast.LENGTH_SHORT).show();")
            ctClass.addMethod(createMethod)
        }
    }

}
```

- `isFrozen`方法用于检测当前类是否冻结，即是否可以修改，如果不可，调用`defrost`解除，然后可以开始编写

- 这里以修改代码和新增代码为例，通过*Javassist*库提供的一些API来完成，具体可以在网上搜搜*Javassist*的语法。这里分别实现了在已有方法中插入一条日志和新建一个方法弹toast。注意使用API时，需要写类的全限定名，并且要保证类路径已经添加过了（在*ClassPool*中），避免找不到对应的类。

- 修改完后，必学把插桩代码写入文件。然后调用`detach`清除`CtClass`避免内存占用过大，发生*OOM*。

6、**实现自定义插件**。新建一个插件实现*Plugin*接口（该接口必须依赖*plugin*插件），重写其`apply`方法，在该方法中注册我们的*Transform*。

```groovy
class CommonPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        println("******************************************************************************")
        println("******                                                                  ******")
        println("******                      欢迎使用 FFmpegKit 编译插件                    ******")
        println("******                                                                  ******")
        println("******************************************************************************")

        project.android.registerTransform(new AaronTransform(project))
    }
}
```

7、注册声明插件。在*resources*目录下新建*META-INF*目录，在*META-INF*目录下新建*gradle-plugins*目录，然后在该目录下创建*xxx.properties*文件，*xxx*作为文件名，可以任意命名，它将作为插件的名称，**在步骤10我们使用插件时的名称就是这里的xxx。**

<img src="file:///Users/mtdp/Library/Application%20Support/marktext/images/2022-11-22-11-15-42-image.png" title="" alt="" width="308">

properties文件里注册插件的实现类：

```groovy
implementation-class=com.aaron.plugin.arronplugin.CommonPlugin
```

8、上传插件到远程/本地仓库。在步骤3准备工作中，我们依赖的maven插件就是用来上传插件的。写一个上传插件的*groovy task*，为了测试方便我们就在本地建一个repo目录（和app目录同级），将插件传到该目录。

```groovy
uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: uri('../repo')) // 插件的输出路径
            pom.groupId = 'arron_plugin_group'
            pom.artifactId = 'arron_plugin_artifactId'
            pom.version = '1.2.3'
        }
    }
}
```

第4行：就是打包好的插件的输出路径

第5-7行：配置插件的*groupId*、*artifactId*和版本，使用时通过`groupId:artifactId:version`的方式依赖

9、依赖自定义插件。在工程的build.gradle文件中依赖自定义的插件

```groovy
buildscript {
    repositories {
        google()
        jcenter()
        maven { url uri('./repo') }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.0'
        classpath 'com.jakewharton:butterknife-gradle-plugin:9.0.0-rc2'
        classpath 'arron_plugin_group:arron_plugin_artifactId:1.2.3'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
......
```

10、使用自定义插件。至此就能够在我们的项目里运用插件能力了。

```groovy
apply plugin: 'com.aaron.plugin'
```

## 参考资料

1:  [Gradle 系列（8）其实 Gradle Transform 就是个纸老虎 - 掘金](https://juejin.cn/post/7098752199575994405/)

2: [Gradle 系列（1）为什么说 Gradle 是 Android 进阶绕不去的坎 - 掘金](https://juejin.cn/post/7092367604211253256#heading-29)
