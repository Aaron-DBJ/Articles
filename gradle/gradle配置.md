# 1、dependencyResolutionManagement仓库配置

## dependencyResolutionManagement vs allprojects 详解

  两者关系

 ┌─────────────────────────────────────────────────────────┐
  │                    Gradle 仓库配置演进                        │
  ├─────────────────────────────────────────────────────────┤
  │  旧方式 (Gradle < 6.8)                                       │
  │  ├── allprojects { repositories { ... } }                   │
  │  └── buildscript { repositories { ... } }                   │
  ├─────────────────────────────────────────────────────────┤
  │  新方式 (**Gradle >= 6.8, Android 推荐使用**)                    │
  │  └── dependencyResolutionManagement { repositories {...} }  │
  └────────────────────────────────────────────────────────┘

  **主要区别**
  特性: 引入版本
  dependencyResolutionManagement: **Gradle 6.8+**
  allprojects.repositories: 早期版本
  ────────────────────────────────────────
  **特性: 配置位置**
  dependencyResolutionManagement: settings.gradle
  allprojects.repositories: 任意 build.gradle
  ────────────────────────────────────────
  **特性: 作用范围**
  dependencyResolutionManagement: 整个项目的所有依赖
  allprojects.repositories: 当前及其子项目
  ────────────────────────────────────────
  **特性: 优先级**
  dependencyResolutionManagement: 高（会覆盖项目级配置）
  allprojects.repositories: 低
  ────────────────────────────────────────
  **特性: Android 推荐**
  dependencyResolutionManagement: ✅ 是
  allprojects.repositories: ❌ 已废弃
  **使用方式**

1. dependencyResolutionManagement（推荐）
   
   在 settings.gradle 中配置：
   
   ```groovy
   // settings.gradle
   
   dependencyResolutionManagement {
    // 模式选择
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
   
   repositories {
    google()        // Android 库
    mavenCentral()  // Java 库
    // 自定义仓库
    maven { url "https://m2.kanzhun-inc.com/..." }
   }
   }
   ```
   
   **三种模式：**
   
   1. FAIL_ON_PROJECT_REPOS - 严格模式（推荐）
      //    禁止在任何 build.gradle 中声明仓库
      repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
   
   2. PREFER_SETTINGS - 优先使用 settings 中的配置
      //    settings.gradle 优先，build.gradle 中声明的会被忽略
      repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)
   
   3. PREFER_PROJECT - 优先使用项目中的配置（默认）
      //    build.gradle 优先，settings.gradle 作为后备
      repositoriesMode.set(RepositoriesMode.PREFER_PROJECT)

2. allprojects.repositories（旧方式，不推荐）
   
   // 根目录 build.gradle
   
   ```groovy
   allprojects {
    repositories {
        google()
        mavenCentral()
    }
   }
   ```

# 2、consumer-rules.pro

## 作用

这是 Android Library 特有的文件，用于为使用这个 library 的 app 提供混淆规则。

## 工作原理

当你的 library 被发布后，依赖它的 app 在打包 release 版本时，Gradle 会自动将 consumer-rules.pro 中的规则合并到 app 的混淆配置中。

## 使用场景

###### 1. 保持 library 的公共 API 不被混淆

  -keep public class com.yourcompany.behaviortracker.BehaviorTracker {
      public *;
  }

###### 2. 保持 JSON 解析类的字段

  -keepclassmembers class com.yourcompany.behaviortracker.model.** {
      <fields>;
  }

###### 3. 如果使用了注解

  -keep @com.yourcompany.behaviortracker.annotation.Keep class * { *; }

###### 4. 保持枚举

  -keepclassmembers enum com.yourcompany.behaviortracker.** {
      public static **[] values();
      public static ** valueOf(java.lang.String);
  }

**与 proguard-rules.pro 的区别**
  ┌────────────────────┬───────────────────────┐
  │        文件        │        作用于         │
  ├────────────────────┼───────────────────────┤
  │ consumer-rules.pro │ 依赖此 library 的 app │
  ├────────────────────┼───────────────────────┤
  │ proguard-rules.pro │ library 自身 的编译   │
  └────────────────────┴───────────────────────┘
  何时需要
  ┌────────────────────────────────────────────┬───────────┐
  │                    场景                    │ 是否需要  │
  ├────────────────────────────────────────────┼───────────┤
  │ library 不需要混淆自身（发布未混淆的 aar） │ ✅ 需要   │
  ├────────────────────────────────────────────┼───────────┤
  │ library 自己处理混淆（发布已混淆的 aar）   │ ❌ 不需要 │
  ├────────────────────────────────────────────┼───────────┤
  │ library 有反射调用                         │ ✅ 需要   │
  ├────────────────────────────────────────────┼───────────┤
  │ library 有序列化的 Java Bean               │ ✅ 需要   │
  └────────────────────────────────────────────┴───────────┘


# pluginManagement插件仓库配置

用于配置 Gradle 插件的解析仓库，它影响的是 plugins {} 块中声明的插件。

## 工作原理

Gradle 有两种插件声明方式，解析方式不同：

```groovy
// 1. plugins DSL - 由 pluginManagement 控制
 plugins {
     id 'com.android.application' version '7.4.2'
 }

// 2. buildscript - 由 buildscript.repositories 控制
 buildscript {
     repositories { mavenCentral() }
     dependencies {
         classpath 'com.android.tools.build:gradle:7.4.2'
     }
 }
```

settings.gradle 结构

```groovy
pluginManagement {
 // 插件解析仓库
 repositories {
     google() // Android Gradle Plugin
     mavenCentral()
     gradlePluginPortal() // Gradle 官方插件
 }


  // 插件版本管理（可选）
  plugins {
      id("com.android.application") version "8.1.0"
  }

}

dependencyResolutionManagement {
     // 项目依赖仓库
     repositories {
         google()
         mavenCentral()
     }
 }

rootProject.name = "MyProject"
```

**使用场景**
场景: 声明 Android Gradle Plugin
配置位置: pluginManagement.repositories
────────────────────────────────────────
场景: 声明 Kotlin 插件
配置位置: pluginManagement.repositories
────────────────────────────────────────
场景: 声明自定义 Gradle 插件
配置位置: 需要添加对应的 Maven 仓库
────────────────────────────────────────
场景: 项目依赖库（如 appcompat）
配置位置: dependencyResolutionManagement.repositories

### 什么时候需要自定义 pluginManagement

1. 使用私有插件仓库
   
   ```grooy
   pluginManagement {
        repositories {
            maven { url "https://your-company.com/plugins" }
            google()
            gradlePluginPortal()
       }
   }
   ```
   
   
2. 统一管理插件版本（配合 libs.versions.toml）
   
   ```grooy
   pluginManagement {
    repositories { google() }
   }
   
   // build.gradle 中使用
   plugins {
    alias(libs.plugins.android.application)
   }
   ```
   
   
3. 使用快照版本插件
   
   ```grooy
   pluginManagement {
    repositories {
   
        maven { url "https://oss.sonatype.org/content/repositories/snapshots/" }                                                                      
        google()
    }
   }
   ```
   
   # 3、alias
   
   ## 作用
   
    alias 是使用 Version Catalog（版本目录）来声明插件的语法，它指向 libs.versions.toml 中定义的插件。
   
    两种插件声明方式对比
   
   ```groovy
   // 方式 1: 使用 alias（推荐）
    plugins {
        alias(libs.plugins.android.library)
    }
   
   // 方式 2: 直接声明（不依赖 catalog）
    plugins {
        id 'com.android.library' version '8.1.0'
    }
   ```
   
   ## 必须要用 alias 吗？
   
   不是必须的，只是一个推荐的现代化写法。
   
   **Version Catalog 的好处**
     ┌───────────────────────────────────┬─────────────────┐
     │            使用 alias             │          直接声明           │
     ├───────────────────────────────────┼──────────────────┤
     │ 版本统一管理在 libs.versions.toml │ 版本分散在各个 build.gradle │
     ├───────────────────────────────────┼──────────────────┤
     │ 升级版本只需改一处                │ 需要改多个文件              │
     ├───────────────────────────────────┼──────────────────┤
     │ IDE 有自动补全和提示              │ 容易写错版本号              │
     └───────────────────────────────────┴─────────────────┘
   **libs.versions.toml 映射关系**
   
     [versions]
     agp = "8.1.0"
   
     [plugins]
     android-library = { id = "com.android.library", version.ref = "agp" }
   
     // build.gradle 中使用
     alias(libs.plugins.android.library)
     //       ^     ^           ^             ^
     //       |     |           |             |
     //   关键字 catalog 插件分组      具体插件名
   
   **总结**
   - alias = 使用 libs.versions.toml 定义的插件
   - 不用 alias = 直接写插件 ID 和版本号
   - 项目中如果想用 alias，确保 libs.versions.toml 中有对应定义
   - 如果没有定义 catalog，就用 id 'xxx' version 'xxx'
