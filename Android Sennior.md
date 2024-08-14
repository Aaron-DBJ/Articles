## Android类加载

当我们new一个类时，首先是Android的虚拟机（Dalvik/ART虚拟机）通过ClassLoader去加载dex文件到内存。
Android中的ClassLoader主要是[PathClassLoader](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Alibcore%2Fdalvik%2Fsrc%2Fmain%2Fjava%2Fdalvik%2Fsystem%2FPathClassLoader.java "https://cs.android.com/android/platform/superproject/+/master:libcore/dalvik/src/main/java/dalvik/system/PathClassLoader.java")和[DexClassLoader](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Alibcore%2Fdalvik%2Fsrc%2Fmain%2Fjava%2Fdalvik%2Fsystem%2FDexClassLoader.java "https://cs.android.com/android/platform/superproject/+/master:libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java"),这两者都继承自[BaseDexClassLoader](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Alibcore%2Fdalvik%2Fsrc%2Fmain%2Fjava%2Fdalvik%2Fsystem%2FBaseDexClassLoader.java%3Bl%3D38%3Fq%3DBaseDexClass%26sq%3D "https://cs.android.com/android/platform/superproject/+/master:libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java;l=38?q=BaseDexClass&sq=")。它们都可以理解成应用类加载器。
PathClassLoader和DexClassLoader的区别：

- **PathClassLoader只能固定加载apk包路径**，不能指定dex文件解压路径。该路径是写死的在/data/dalvik-cache/路径下。所以只能用于加载已安装的apk。
- DexClassLoader可以指定apk包路径和dex文件解压路径（加载jar、apk、dex文件）
  当ClassLoader加载类时，会调用它的findclass方法去查找该类。  
  下方是BaseDexClassLoader的findClass方法实现：
  
  ```java
  public class BaseDexClassLoader extends ClassLoader {
    ...
    @UnsupportedAppUsage
    private final DexPathList pathList;
    ...
     @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // 首先检查该类是否存在shared libraries中.
        if (sharedLibraryLoaders != null) {
            for (ClassLoader loader : sharedLibraryLoaders) {
                try {
                    return loader.loadClass(name);
                } catch (ClassNotFoundException ignored) {
                }
            }
        }
        //再调用pathList.findClass去查找该类，结果为null则抛出错误。
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException(
                    "Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }
  }
  ```
  
  接下来我们再来看看DexPathList的findClass实现：
  
  ```java
  public DexPathList(ClassLoader definingContext, String librarySearchPath) {
    ...
    /**
     * List of dex/resource (class path) elemnts.
     * 存放dex文件的一个数组
     */
    @UnsupportedAppUsage
    private Element[] dexElements;
    ...
    public Class<?> findClass(String name, List<Throwable> suppressed) {
          //遍历Element数组，去查寻对应的类，找到后就立刻返回了
        for (Element element : dexElements) {
            Class<?> clazz = element.findClass(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
      ...
  }
  ```
  
  ## 插件化
  
  **概述**：Android插件化技术，**可以实现功能模块的按需加载和动态更新，其本质是动态加载未安装的apk。**
  **原理**：插件化要解决的三个核心问题：类加载、资源加载、组件生命周期管理。
  **好处**： 
  1.让用户不用重新安装 APK 就能升级应用功能，减少发版本频率，增加用户体验。 
  2.提供一种快速修复线上 BUG 和更新的能力。 
  3.按需加载不同的模块，实现灵活的功能配置，减少服务器对旧版本接口兼容压力。 
  4.模块化、解耦合、并行开发、 65535 问题。
  
  ### 插件化原理
  
  1、类加载

---

## 热修复

**概述**：就是通过下发补丁包，让已安装的客户端动态更新，让用户可以不用重新安装APP，就能够修复软件缺陷的一种技术
**原理**：修复代码、资源、so
**好处**： 
1.无需重新发版。 
2.快速修复线上bug，修复成功率高，降低损失。 
3.用户无感知修复，无需下载最新的应用，代价小。

## 热修复原理

### **类加载方案**

Android中要加载某一个类，最终都是调动DexPathList中的findClass()方法
加载一个类的时候，都会去循环dexElements数组取出里面的dex文件，然后从dex文件中找目标类，只要目标类找到，则直接退出循环，也就是后面的dex文件就没有被取到的机会。 那我们就可以根据这样的一个原理，将修改后的类编译后统一放在patch.dex补丁文件中，通过反射将patch.dex放在dexElemets这个数组的第一个元素，这样，当加载出现bug的类时，首先会先从patch.dex这个文件中去找，因为我们将修改后的类放在了patch.dex文件中，所以肯定会被找到(此时加载到的是已经修复的类)，一旦被找到，后面dex中的bug类就没办法被加载到。
![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240425145926.png)

#### 采用该方案的框架：QZone、Tinker

##### 1、QZone修复的流程:

1. 通过获取到当前应用的Classloader，即BaseDexClassloader

2. 通过反射获取到他的DexPathList属性对象pathList

3. 通过反射调用pathList的dexElements方法把patch.dex转化为Element[]

4. 两个Element[]进行合并，把patch.dex放到最前面去

5. 加载Element[]，达到修复目的

##### 2、Tinker修复的流程:

微信的Tinker将新旧APK做了diff，得到path.dex，再将patch.dex与手机中APK的classes.dex做合并，生成新的classes.dex，然后在运行时通过反射将classes.dex放在Elements数组的第一个元素。

### **底层替换方案**

在native层将传入的javaMethod在ART虚拟机中对应一个ArtMethod指针，ArtMethod结构体中包含了Java方法的所有信息，包括执行入口、访问权限、所属类和代码执行地址等。
替换ArtMethod结构体中的字段或者替换整个ArtMethod结构体，这就是底层替换方案。
AndFix采用的是替换ArtMethod结构体中的字段，这样会有兼容性问题，因为厂商可能会修改ArtMethod结构体，导致方法替换失败。
Sophix采用的是替换整个ArtMethod结构体，这样不会存在兼容问题。

## 资源热修复

资源修复是很常见的操作，热修复方案中的资源修复很多参考了 **Instant Run** 的实现，**Instant Run** 的资源修复核心流程大致如下：

1. 构建一个新的 **AssetManager** 对象，并调用 *addAssetPath* 添加新的资源包；
2. 修改所有 **Activity** 的 *Activity.mAssets(AssetManager实例)* 引用指向新构建的 **AssetManager** 对象；
3. 修改所有 **Resource** 的 *Resource.mAssets(AssetManager实例)* 引用指向新构建的 **AssetManager** 对象.
   对于任意的资源包，被 *AssetManager#addAssetPath* 添加之后，解析 **resourecs.asrc** 并在 native *mResources* 侧保存起来。可参考 [AssetManager.h](https://android.googlesource.com/platform/frameworks/base/+/master/libs/androidfw/include/androidfw/AssetManager.h) 的实现，实际上 *mResources* 是一个 **ResTable** 结构体,存放 **resourecs.asrc** 信息用的。而且一个进程只会有一个 **ResTable**。
- **ResTable** 可加载多个资源包
- 一个资源包都包含一个 **resourecs.asrc**
- 每一个 **resourecs.asrc** 记录了该包的所有资源信息，每一个资源对应一个 **ResChunk**
- 每一个 **ResChunk** 都有一个唯一的编号，由该编号由三部分构成，比如 **0x7f0e0000**. 可以随便找一个 apk 解包查看 *resourecs.asrc* 文件。
  - 前两位 *0x7f* 为 package id，用于区分是哪个资源包
  - 接着两位 *0x0e* 为 type id，用于区分是哪类型资源，比如 drawable，string 等
  - 最后四位 *0x0000* 为 entry id，用于表示一个资源项，第一个为 *0x0000*，第二个为 *0x0001* 依次递增。
    
    > 值得注意的是，系统的资源包的 package id 为 0x01，我们的 apk 为 0x7f
    > 在应用启动之后，**ResourceManager** 在构建 **AssetManager** 的时候就已经加载了 apk 包的资源和系统的资源。如果补丁下发的资源 package id 也会 *0x7f* ，同时我们使用已有的 **AssetManager** 进行加载，在 **Android L** 版本之后会继续追加到已经解析资源的后面，但是由于相同的 package id 的原因，有可能在获取某个资源的时候，由于原 apk 已经存在近而忽略了补丁的新资源。故类 **Instant Run** 方案下，**AssetManager** 需要被完全替换才有效。
    > https://blog.51cto.com/u_15719342/5475324
    > [Android 补丁技术学习总结(三) 资源修复 | YummyLau](https://yummylau.com/2020/05/02/Android_2020-04-20_Android%20%E8%A1%A5%E4%B8%81%E6%8A%80%E6%9C%AF%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93(%E4%B8%89)%20%E8%B5%84%E6%BA%90%E4%BF%AE%E5%A4%8D/)
    
    ### 热修复和插件化区别
    
    插件化和热修复的原理，都是动态加载 dex／apk 中的类／资源，让宿主正常的加载和运行插件（补丁）中的内容。 **两者的目的不同**:
- 插件化目标是想把需要实现的模块或功能当做一个独立的文件提取出来，减少宿主的规模。所以插件化重在解决组件的生命周期，以及资源的问题
- 热修复目标在修复已有的问题。重在解决替换已有的有问题的类／方法／资源等。

---

# Native Crash监控

Native 程序通过 link 连接后，当发生 Native Crash 时，则 kernel 会发送相应的 signal，当进程捕获致命的 signal，通知 debuggerd 调用 ptrace 来获取有价值的信息(这是发生在 crash 前)。
![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240528161902.png)

1. kernel 发送 signal 给 target 进程(包含 native 代码)；

2. target 进程通过 debuggerd_signal_handler，捕获 signal；
- 建立于 debuggerd 进程的 socket 通道；
- 将 action = DEBUGGER_ACTION_CRASH 的消息发送给 debuggerd 服务端；
- 阻塞等待 debuggerd 服务端的回应数据。
3. debuggerd 作为守护进程，一直在等待 socket client 的连接，此时收到 action = DEBUGGER_ACTION_CRASH 的消息；

4. 执行到 handle_request 时，通过 fork 创建子进程来执行各种 dump 相关操作；

5. 新创建的进程，通过 socket 与 system_server 进程中的 NativeCrashListener 线程建立 socket 通道，并向其发送 native crash 信息；

6. NativeCrashListener 线程通过创建新的名为“NativeCrashReport”的子线程来执行 AMS 的 handleApplicationCrashInner 方法。


[Android基础开发实践：如何分析Native Crash-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1192001)
[Android 平台 Native 代码的崩溃捕获机制及实现 - 掘金](https://juejin.cn/post/6844903486836965389#heading-0)
[Android 平台 Native Crash （一）捕获原理详解 - 掘金](https://juejin.cn/post/7124688655796404254/)

---

# ANR原理

ANR(Application Not responding)，是指应用程序未响应，Android系统对于一些事件需要在一定的时间范围内完成，如果超过预定时间能未能得到有效响应或者响应时间过长，都会造成ANR。一般地，这时往往会弹出一个提示框，告知用户当前xxx未响应，用户可选择继续等待或者Force Close。
那么哪些场景会造成ANR呢？

- Service Timeout: 比如前台服务在20s内未执行完成；
- BroadcastQueue Timeout：比如前台广播在10s内未执行完成
- ContentProvider Timeout：内容提供者,在publish过超时10s;
- InputDispatching Timeout:  输入事件分发超时5s，包括按键和触摸事件。
  触发ANR的过程可分为三个步骤: 埋炸弹, 拆炸弹, 引爆炸弹
  
  ## Input事件发生ANR的原理
  
  ![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240507112029.png)
  [理解Android ANR的触发原理 - Gityuan博客 | 袁辉辉的技术博客](https://gityuan.com/2016/07/02/android-anr/)
  [Input系统—ANR原理分析 - Gityuan博客 | 袁辉辉的技术博客](https://gityuan.com/2017/01/01/input-anr/)
  [深入理解 Android ANR 触发原理以及信息收集过程 - huansky - 博客园](https://www.cnblogs.com/huansky/p/14954020.html)
  [Android ANR 原理](https://segmentfault.com/a/1190000022967452)
  [ANR如何产生之InputDispatching Timeout篇 - 掘金](https://juejin.cn/post/7202598910338400316)
  [Android Input（八）- ANR原理分析 - 简书](https://www.jianshu.com/p/b8b35d3ee052)
  
  # Android APT
  
  ### 编译过程
  
  ![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240528110910.png)
  
  ### Q1:注解处理器processor为什么要在META-INF注册？
  
  `META-INF`的作用  META-INF, 相当于一个信息包，用于存放一些meta information相关的文件。用来配置应用程序、扩展程序、类加载器和服务[manifest.mf](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.baidu.com%2Fs%3Fwd%3Dmanifest.mf%26tn%3DSE_PcZhidaonwhc_ngpagmjz%26rsv_dl%3Dgh_pc_zhidao)文件，在用jar打包时自动生成。
  通过`@AutoService(Processor.class)`注解把注解处理器`ButterKnifeProcessor`注册到META-INF/services中，这里的包名是`META-INF/services/javax.annotation.processing.Processor`,  这个文件的内容是
  
  ```css
  com.zx.processor.ButterKnifeProcessor
  ```
  
  这个包名其实就是`@AutoService(Processor.class)`里面的`Processor`类；而文件内容就是注解处理器。
  **在编译时，java编译器（javac）会去META-INF中查找实现了的AbstractProcessor的子类，并且调用该类的process函数，最终生成`.java`文件。其实就像activity需要注册一样，就是要到META-INF注册 ，javac才知道要给你调用哪个类来处理注解。**
  
  ### Q2：注解处理器processor是如何被系统调用的？
  
  一些细心的同学应该发现了这个问题，我们并没有手动调用`AbstractProcessor`这个注解处理器类，那系统是什么时间调用的？又是如何调用的？这其实就牵扯到apt工作机制。
  在上一问中，我们已经了解到，在编译时javac会查找所有的 在META_INF 中注册的注解处理器来处理注解。
  到这里，好像有点清楚了，大概知道javac会去找到Processor并调用。但是呢还是没找到直接源头，因为它不像我们面向对象编程中可以准确追踪到是哪个对象调用的。
  别着急，先来看这么个东西. 是我项目中使用到注解的`app.Gradle`
  
  ```csharp
  dependencies {
    implementation project(':annotation')
    implementation project(':inject_api')
    //gradle3.0以上apt的实现方式
    annotationProcessor project(':processor')
  }
  ```
  
  这里的`annotationProcessor`有点特别，没错，它是**APT**实现方案的一种。这里简单介绍一下：
  
  > APT实现方案 
  > `android-apt`和`annotationProcessor`功能是一样的，都是apt的实现方案，前者是个人开发者提供，比较早（现在不再维护了），后者是google官方开发的内置在`gradle`里的apt。  
  > 
  > annotationProcessor是APT工具中的一种，是google开发的内置框架，不需要引入，可以直接在`build.gradle`文件中使用。
  > 只有在你使用注解的地方引入了`annotationProcessor`，系统才会主动调用注解处理类`Processor`,才会最终生成如下的`.java`文件
  > ![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240524111719.png)
  >                                                         apt生成类.png
  > 这里先简单总结一下： 
  > 2.1、在完成注解处理类`Processor`之后，需要做2件事情：
- **在META-INF目录下注册`Processor`**
- **在项目中使用注解的地方添加apt工具`annotationProcessor`**
  2.2、APT 4要素 
  　　**注解处理器（AbstractProcess）+ 代码处理（javaPoet）+ 处理器注册（AutoService）+ apt（annotationProcessor）**
  `APT(Annotation Processing Tool)总结` 
  首先，APT是javac提供的一种工具，它在编译时扫描、解析、处理注解。它会对源代码文件进行检测，找出用户自定义的注解，根据注解、注解处理器和相应的apt工具自动生成代码。这段代码是根据用户编写的注解处理逻辑去生成的。**最终将生成的新的源文件与原来的源文件共同编译（注意：APT并不能对源文件进行修改操作，只能生成新的文件，例如往原来的类中添加方法）**。具体流程图如下图所示：  
  ![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240524111846.png)
                                                  apt工作流程.png
  APT技术的使用，需要我们遵守一定的规则。大家先看一下整个APT项目项目构建的一个规则图，具体如下所示：
  ![](https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20240524112059.png)
  
  ### Q3：注解申明和注解处理器为什么要分Module处理？
  
  先来回顾一下之前的项目结构：
- `annotation`：申明注解 （java lib）
- `processor`：注解处理器（java lib）
- `inject_api`：调用处理器中生成的类 （android lib）
- `app`：项目使用 （android lib）
  我们都知道注解处理器都需要继承`AbstractProcessor`类，但是`AbstractProcessor`是JDK中的类，不在android sdk中，所以需要放在单独的java lib中；而`processor`中需要依赖自定义注解，把`annotation`抽成一个独立的lib，便于维护。
  **那注解声明和注解处理为什么要分开呢？可不可以放在一起？** 
  先说结论：可以放在一起，放在一起对功能上没有什么影响；但是一般不放在一起，原因如下：
  我们都知道`processor`的作用是：**在编译器解析注解、生成新的`.java`文件。这个lib只在编译器用到，是不会被打包进apk的。对于调用者来说，你只是想使用这个注解，而不希望你已经编译好的项目中引进注解处理器相关的内容，所以为了不引入没必要的文件，我们一般选择将注解声明和注解处理分开处理。**
  
  ## Javac执行追溯
  
  javac本身也是一个计算机程序，当需要编译java源代码时就需要执行程序。而javac程序的`main`函数定义在**com/sun/tools/javac/Main.java**中：
  
  ```java
  public class Main {
    public static void main(String[] args) throws Exception {
        System.exit(compile(args));
    }
    public static int compile(String[] args) {
        com.sun.tools.javac.main.Main compiler =
            new com.sun.tools.javac.main.Main("javac");
        return compiler.compile(args).exitCode;
    }
  ​
    public static int compile(String[] args, PrintWriter out) {
        com.sun.tools.javac.main.Main compiler =
            new com.sun.tools.javac.main.Main("javac", out);
        return compiler.compile(args).exitCode;
    }
  }
  ```
  
  可以看到，当执行javac程序将会执行上面的`main`方法，而main方法会调用到`compile`方法，在`compile`方法中又会创建`com.sun.tools.javac.main.Main`并执行其`compile`方法。
  打开**com/sun/tools/javac/main/Main.java**文件，其compile实现为：
  
  ```java
  public Result compile(String[] args) {
    Context context = new Context();
    JavacFileManager.preRegister(context); 
    // 调用两参的重载compile方法
    Result result = compile(args, context);
    if (fileManager instanceof JavacileManager) {
        ((JavacFileManager)fileManager).close();
    }
    return result;
  }
  ​
  public Result compile(String[] args, Context context) {
    // 最后一个参数：processors 本次编译要执行的注解处理器集合 直接置为null
    return compile(args, context, List.<JavaFileObject>nil(), null);
  }
  ​
  public Result compile(String[] args,
                       Context context,
                       List<JavaFileObject> fileObjects,
                       Iterable<? extends Processor> processors){
    return compile(args,  null, context, fileObjects, processors);
  }
  public Result compile(String[] args,
                          String[] classNames,
                          Context context,
                          List<JavaFileObject> fileObjects,
                          Iterable<? extends Processor> processors){
    //......
    comp = JavaCompiler.instance(context);
    //......
    comp.compile(fileObjects,
                         classnames.toList(),
                         processors);
    //......
  }
  ```
  
  具体的编译是通过`JavaCompiler#compile`完成。这个方法的第三个参数即为要执行的注解处理器集合，根据执行流程追溯，此处直接传递的为`null`。进一步进入**com/sun/tools/javac/main/JavaCompiler.java**
  
  ```java
  public void compile(List sourceFileObjects, List classnames, Iterable<? extends Processor> processors){ 
      //......
      //初始化
      initProcessAnnotations(processors);
      //执行注解处理器
      delegateCompiler =
            processAnnotations(
                enterTrees(stopIfError(CompileState.PARSE, parseFiles(sourceFileObjects))),
                classnames);
      //......
  }
  ```
  
  ## 注解处理器初始化
  
  终于在`JavaCompiler#compile`方法中找到了javac执行过程中对APT的处理。首先`initProcessAnnotations`方法实现了对APT的初始化。根据源码流程可知此时，该方法参数为要执行的注解处理器集合，当前其实被设置为`null`。
  那`initProcessAnnotations`方法中会怎么初始化我们的APT程序呢？实际上，在一开始我们说**APT程序就是Javac的小插件，由javac在编译时候根据条件调起！** 那么既然javac要调起APT中`AbstractProcessor`的process方法，而process方法是实例方法，自然需要先实现对APT中的`AbstractProcessor（Processor接口）`实现类class对象的加载。
  而这个实现类由我们编写，javac如何得知要加载的APT实现类的类名呢？
  结合到文章最开头处APT的注册，相信基础扎实的同学马上就能够想到：**Java SPI机制**。实际上，javac就是利用`ServiceLoader`加载注册文件，从而得到了APT实现类的类名！
  
  ```java
  public void initProcessAnnotations(Iterable<? extends Processor> processors) {
      //......
      procEnvImpl = JavacProcessingEnvironment.instance(context);
      procEnvImpl.setProcessors(processors); 
      //......
  }
  ```
  
  进入**com/sun/tools/javac/processing/JavacProcessingEnvironment.java**文件:
  
  ```java
  public void setProcessors(Iterable<? extends Processor> processors) {
        Assert.checkNull(discoveredProcs);
        initProcessorIterator(context, processors);
  }
  private void initProcessorIterator(Context context, Iterable<? extends Processor> processors) {
        Log log = Log.instance(context);
        //要执行的注解处理器集合
        Iterator<? extends Processor> processorIterator;
        //....
        // ServiceIterator 使用SPI机制（ServiceLoader）加载注册文件并创建APT实现类实例对象
        processorIterator = new ServiceIterator(processorClassLoader, log);
        //....
        discoveredProcs = new DiscoveredProcessors(processorIterator);
  }
  ```
  
  ## 执行注解处理器
  
  回到`JavaCompiler#compile`，在通过`initProcessAnnotations`初始化注解处理器后，接着执行`processAnnotations`实现对注解的处理。
  
  ```java
  public JavaCompiler processAnnotations(List<JCCompilationUnit> roots,
                                           List<String> classnames) {
    //......
    JavaCompiler c = procEnvImpl.doProcessing(context, roots, classSymbols, pckSymbols,
                        deferredDiagnosticHandler);
    //......
  }
  ```
  
  进入**com/sun/tools/javac/processing/JavacProcessingEnvironment.java**文件:
  
  ```java
  public JavaCompiler doProcessing(Context context,
                                     List<JCCompilationUnit> roots,
                                     List<ClassSymbol> classSymbols,
                                     Iterable<? extends PackageSymbol> pckSymbols,
                                     Log.DeferredDiagnosticHandler deferredDiagnosticHandler) {
    Round round = new Round(context, roots, classSymbols, deferredDiagnosticHandler);
    boolean errorStatus;
    boolean moreToDo;
    do {
        // 第一次执行apt
        round.run(false, false);
        errorStatus = round.unrecoverableError();
        moreToDo = moreToDo(); //执行apt后是否还需要再次执行
        round = round.next(
                    new LinkedHashSet<JavaFileObject>(filer.getGeneratedSourceFileObjects()),
                    new LinkedHashMap<String,JavaFileObject>(filer.getGeneratedClasses()));
        if (round.unrecoverableError())
            errorStatus = true;
  ​
    } while (moreToDo && !errorStatus);
  ​
    // 最后一次执行apt
    round.run(true, errorStatus);
    //......
  }
  ```
  
  ### 第一次执行
  
  第一次执行`process`方法是在**do-while**中调起`round.run(false, false)`完成。
  
  ```java
  void run(boolean lastRound, boolean errorStatus) {
    try {
        if (lastRound) {
            filer.setLastRound(true);
            Set<Element> emptyRootElements = Collections.emptySet(); // immutable
            RoundEnvironment renv = new JavacRoundEnvironment(true,
                            errorStatus,
                            emptyRootElements,
                            JavacProcessingEnvironment.this);
             //只有最后一次执行此处
            discoveredProcs.iterator().runContributingProcs(renv);
        } else {
            //不是最后一次执行此处
            discoverAndRunProcs(context, annotationsPresent, topLevelClasses, packageInfoFiles);
        }
        } catch (Throwable t) {
             //.......
        } finally {
            //.......
        }
  }
  ```
  
  APT实现类的Process方法原型是：
  
  ```java
  public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment)
  ```
  
  在编译过程中第一次APT调用由`discoverAndRunProcs`调起：
  
  ```java
  private void discoverAndRunProcs(Context context,
                                     Set<TypeElement> annotationsPresent,
                                     List<ClassSymbol> topLevelClasses,
                                     List<PackageSymbol> packageInfoFiles) {
        HashMap unmatchedAnnotations = new HashMap(annotationsPresent.size());
          //......
        //调用APT实现类的process方法的参数
        RoundEnvironment renv = new JavacRoundEnvironment(false,
                                                          false,
                                                          rootElements,
                                                          JavacProcessingEnvironment.this);
  ​
        while(unmatchedAnnotations.size() > 0 && psi.hasNext() ) {
            ProcessorState ps = psi.next();
            Set<String>  matchedNames = new HashSet<String>();
            Set<TypeElement> typeElements = new LinkedHashSet<TypeElement>();
  ​
            for (Map.Entry<String, TypeElement> entry: unmatchedAnnotations.entrySet()) {
                String unmatchedAnnotationName = entry.getKey();
                //匹配apt实现类支持的注解
                if (ps.annotationSupported(unmatchedAnnotationName) ) {
                    matchedNames.add(unmatchedAnnotationName);
                    //调用APT实现类的process方法的参数
                    TypeElement te = entry.getValue();
                    if (te != null)
                        typeElements.add(te);
                }
            }
  ​
            if (matchedNames.size() > 0 || ps.contributed) {
                //执行注解处理器
                boolean processingResult = callProcessor(ps.processor, typeElements, renv);
                /**
                 * TODO 问题3 
                 * APT实现类返回值为ture，删除它能处理的注解信息,
                 * 这样其他需要处理相同注解的注解处理器就得不到执行了
                 */
                if (processingResult) {
                    // unmatchedAnnotations : 所有的注解集合
                    // matchedNames:匹配此注解处理器的注解
                    unmatchedAnnotations.keySet().removeAll(matchedNames);
                }
  ​
            }
        }
    //......
  }
  ​
  private boolean callProcessor(Processor proc,Set<? extends TypeElement> tes,
                              RoundEnvironment renv) {
    return proc.process(tes, renv);
  }
  ```
  
  ### process方法返回值
  
  1、在javac执行时可以指定多个APT程序(-processorpath 指定的jar包)，一个APT程序可以包含多个APT实现类，所以javac会将指定的多个APT程序中的所有注册的APT实现类加载并实例化，使用迭代器`Iterator`装载。
  2、在`discoverAndRunProcs`中对所有要执行的APT实现类进行迭代，依次执行APT实现类的`process`方法，顺序由注册顺序决定。
  3、但是执行APT实现类的前提是：有APT实现类声明的支持处理的注解信息。**而若先注册的APT实现类其process方法返回true，则会在执行结束此APT实现类后，通过`unmatchedAnnotations.keySet().removeAll(matchedNames);`将其能处理的注解信息删除。这样后注册的APT实现类将会因为没有匹配处理的注解而得不到执行。**
  
  > 比如AProcessor声明处理@Test注解，而BProcessor也声明处理@Test注解，而AProcessor先于BProcessor注册，AProcessor的process方法返回ture。此时BProcessor不会执行。
  
  ### 第2-N次执行
  
  回到`JavacProcessingEnvironment#doProcessing`
  
  ```java
  public JavaCompiler doProcessing(Context context,
                                     List<JCCompilationUnit> roots,
                                     List<ClassSymbol> classSymbols,
                                     Iterable<? extends PackageSymbol> pckSymbols,
                                     Log.DeferredDiagnosticHandler deferredDiagnosticHandler) {
    Round round = new Round(context, roots, classSymbols, deferredDiagnosticHandler);
    boolean errorStatus;
    boolean moreToDo;
    do {
        // 第一次执行apt
        round.run(false, false);
        errorStatus = round.unrecoverableError();
        moreToDo = moreToDo(); //执行apt后是否还需要再次执行
        round = round.next(
                    new LinkedHashSet<JavaFileObject>(filer.getGeneratedSourceFileObjects()),
                    new LinkedHashMap<String,JavaFileObject>(filer.getGeneratedClasses()));
        if (round.unrecoverableError())
            errorStatus = true;
  ​
    } while (moreToDo && !errorStatus);
  ​
    // 最后一次执行apt
    round.run(true, errorStatus);
    //......
  }
  ```
  
  `round.run(false, false);`是在do-while循环中被调用，这也是为什么本节小标题为：**第2-N次执行**。执行多轮的条件为：`moreToDo && !errorStatus`。
  第二个条件是执行APT实现类时未产生异常，而第一个条件：moreTodo
  
  ```java
  private boolean moreToDo() {
    return filer.newFiles();
  }
  public boolean newFiles() {
    return (!generatedSourceNames.isEmpty())
            || (!generatedClasses.isEmpty());
  }
  ```
  
  如果熟悉APT的同学，应该清楚，一般的我们利用APT实现在编译阶段生成新的Java类：
  
  ```java
  //在apt中生成 Test.java
  JavaFileObject sourceFile = processingEnv.getFiler()
                        .createSourceFile("com.xx.Test");
  OutputStream os = sourceFile.openOutputStream();
  os.write("package com.xx;\n  public class Test{}".getBytes());
  os.close();
  ```
  
  在执行到`os.close()`时就会执行一次`generatedSourceNames.add(typeName)`。
  也就是说APT执行多次的条件是：**在APT执行是生成了一个java源文件(或者class文件)都会导致APT再执行一次，这次执行只会处理新生成的类**
  「只会处理新生成的类」：
  
  ```java
  //只处理新生成类的round
  round = round.next( new LinkedHashSet<JavaFileObject>(filer.getGeneratedSourceFileObjects()),
                    new LinkedHashMap<String,JavaFileObject>(filer.getGeneratedClasses()));
  //执行apt
  round.run(false, false);
  ```
  
  而如果第二轮执行又新生成了类，就会执行第三轮、第四轮.......，直到不再产生新的.java（或.class）
  
  ### 最后一次执行
  
  不产生新的类文件时会退出**do-While**循环，此时会执行一次`round.run(true, errorStatus);`
  
  ```java
  void run(boolean lastRound, boolean errorStatus) {
    try {
        if (lastRound) {
            filer.setLastRound(true);
            Set<Element> emptyRootElements = Collections.emptySet(); // immutable
            RoundEnvironment renv = new JavacRoundEnvironment(true,
                            errorStatus,
                            emptyRootElements,
                            JavacProcessingEnvironment.this);
             //只有最后一次执行此处
            discoveredProcs.iterator().runContributingProcs(renv);
        } else {
            //不是最后一次执行此处
            discoverAndRunProcs(context, annotationsPresent, topLevelClasses, packageInfoFiles);
        }
        } catch (Throwable t) {
             //.......
        } finally {
            //.......
        }
  }
  ```
  
  此时，会通过`discoveredProcs.iterator().runContributingProcs(renv);`执行最后一轮APT。
  
  ```java
  public void runContributingProcs(RoundEnvironment re) {
    if (!onProcInterator) {
        // process方法第一个参数
        Set<TypeElement> emptyTypeElements = Collections.emptySet();
        while(innerIter.hasNext()) {
            ProcessorState ps = innerIter.next();
             //执行最后一轮
            if (ps.contributed)
                callProcessor(ps.processor, emptyTypeElements, re);
        }
    }
  }
  ```
  
  可以看到在执行最后一轮时，会构建一个空Set集合作为process方法的第一个参数。这么做的意义猜测是为了将工作和打扫分开。
  第一轮执行：处理你的工作
  第X轮执行: 处理你的工作
  最后执行：在处理完工作之后，再通知一次，执行收尾。
  
  ## 总结
  
  Q：APT原理是什么，怎么被执行起来的？
  A：在javac编译时，先通过SPI机制加载所有的APT实现类，并将解析到需要编译的java源文件中所有的注解信息，与APT声明的支持处理的注解信息进行匹配，若待编译的源文件中存在APT声明的注解，则调起APT实现类的process方法。
  Q：APT中process方法到底执行几次？为什么这么设计？
  A：最少执行两次，最多无数次。最后一次执行可以视为Finish结束通知，执行收尾工作。
  Q：APT中process方法boolean返回值返回true或者false有什么影响？
  A： true：声明的注解只能被当前APT处理；false：在当前APT处理完声明的注解后，仍能被其他APT处理
  [Android APT工作原理（annotationProcessor） - 简书](https://www.jianshu.com/p/89ac9a2513c4)
  [APT你真的了解吗？解析Javac源码APT执行原理 - 掘金](https://juejin.cn/post/7012588238518894606)
  
  # Java编译流程
  
  [爆爆：Java代码编译流程是怎样的？-java 编译过程](https://www.51cto.com/article/699503.html)
