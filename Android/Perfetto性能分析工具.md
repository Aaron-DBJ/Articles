# 使用命令行来抓取 Perfetto

## 基本命令 - [adb shell perfetto](https://androidperformance.com/2024/05/21/Android-Perfetto-02-how-to-get-perfetto/#%E5%9F%BA%E6%9C%AC%E5%91%BD%E4%BB%A4-adb-shell-perfetto)

对于之前一直用 Systrace 工具的小伙伴来说，命令行抓取 Trace 非常方便。同样，Perfetto 也提供了简单的命令行来抓取，最简单的使用方法与 Systrace 基本一致。你可以直接连到你的 Android 设备上使用`/system/bin/perfetto`命令来启动跟踪。

例如

```shell
//1. 首先执行命令
adb shell perfetto -o /data/misc/perfetto-traces/trace_file.perfetto-trace -t 20s \
 sched freq idle am wm gfx view binder_driver hal dalvik camera input res memory

// 2. 操作手机，复现场景，比如滑动或者启动等

// 3. 将 trace 文件 pull 到本地
adb pull /data/misc/perfetto-traces/trace_file.perfetto-trace
// 4.在网页打开traceid文件，https://ui.perfetto.dev/#!/viewer
```

## 进阶命令 - adb shell perfetto with config file

Perfetto 可以抓取的信息非常多，其数据来源也非常多，每次都用命令行加一大堆配置的话会很不方便。这时候我们就可以使用一个单独的**配置文件(Config)**，来存储这些信息，每次抓取的时候，指定这个配置文件即可。

### [在 Android 12 及之后的设备上](https://androidperformance.com/2024/05/21/Android-Perfetto-02-how-to-get-perfetto/#%E5%9C%A8-Android-12-%E5%8F%8A%E4%B9%8B%E5%90%8E%E7%9A%84%E8%AE%BE%E5%A4%87%E4%B8%8A)

从 Android 12 开始，可以直接使用`/data/misc/perfetto-configs`目录来存储配置文件，这样就不需要通过 stdin 来传递配置文件了。具体命令如下：

| 1  <br>2 | adb push config.pbtx /data/misc/perfetto-configs/config.pbtx  <br>adb shell perfetto --txt -c /data/misc/perfetto-configs/config.pbtx -o /data/misc/perfetto-traces/trace.perfetto-trace |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

在这个例子中，首先将配置文件`config.pbtx`推送到`/data/misc/perfetto-configs`目录中。然后，直接在 Perfetto 命令中通过`-c`选项指定配置文件的路径来启动跟踪。

### [在 Android 12 之前的设备上](https://androidperformance.com/2024/05/21/Android-Perfetto-02-how-to-get-perfetto/#%E5%9C%A8-Android-12-%E4%B9%8B%E5%89%8D%E7%9A%84%E8%AE%BE%E5%A4%87%E4%B8%8A)

由于 SELinux 的严格规则，直接通过文件路径传递配置文件在非 root 设备上会失败。因此，需要使用标准输入(stdin)来传递配置文件。这可以通过将配置文件的内容`cat`到 Perfetto 命令中实现。具体命令如下：

| 1  <br>2 | adb push config.pbtx /data/local/tmp/config.pbtx  <br>adb shell 'cat /data/local/tmp/config.pbtx \| perfetto -c - -o /data/misc/perfetto-traces/trace.perfetto-trace' |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

这里，`config.pbtx`是你的 Perfetto 配置文件，首先使用`adb push`命令将其推送到设备的临时目录中。然后，使用`cat`命令将配置文件的内容传递给 Perfetto 命令。

## [Config 的来源](https://androidperformance.com/2024/05/21/Android-Perfetto-02-how-to-get-perfetto/#Config-%E7%9A%84%E6%9D%A5%E6%BA%90)

Config 我建议使用 [ui.perfetto.dev](https://ui.perfetto.dev/) 的 [**Record new trace**](https://ui.perfetto.dev/#!/record) 这里进行选择定制，然后再保存到本地的文件里面，不同的场景就加载不同的 Config 即可，文章最后一部分有详细讲到这部分，感兴趣的可以看一下。

官方也提供了 share 按钮，你可以把你自己的 config share 给其他人，非常方便。同时我也会建了一个 Github 的库，方便大家在分享（进行中）。

官方代码库也有一些已经配置好的，各位可以下下来自己使用：https://cs.android.com/android/platform/superproject/main/+/main:external/perfetto/test/configs/



# 资料

[Android Perfetto 系列 3：熟悉 Perfetto View · Android Performance](https://www.androidperformance.com/2024/05/21/Android-Perfetto-03-how-to-analysis-perfetto/#/Perfetto-%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95)
