# Vysor 原理代码实现

看过[vysor原理以及Android同屏方案](https://juejin.im/entry/57fe39400bd1d00058dd4652) , 我突然想到整个过程应该如何验证的问题。于是反编译了vysor 最新的apk, 其中的代码逻辑依然具有很强的借鉴意义。其中通过 `shell` 环境下调用 `adb` 获取截屏权限成为了全篇的亮点所在。以下文字简要地记录了个人的理解过程，同时希望增进对Android Framework 的理解。

## 0. 背景介绍

### 关于App的创建

由于 Zygote 在系统启动时冷启动了一个`Dalvik` / `ART` `VM`, 并开启对创建新APP请求的监听。随后所有新的应用进程都由Zygote 执行 fork 操作而创建的。具体流程如下所述：

Linux内核启动后，就开始了初始化 Android 系统（init process）的过程。/system/bin/app_process 运行并启动了 Android运行时（AndroidRuntime.start()），在这期间运行时启动了Dalvik 虚拟机，并且创建了zygote进程，以及开启com.android.server.SystemServer 系统服务进程。Zygote将在有新的应用启动时被激活，为了加速应用启动的过程，Zygote会预加载公用的Java类和资源到RAM中，以供应用在实际运行时使用。最终，Zygote将fork自己并启动这一新的应用进程。

![](http://upload-images.jianshu.io/upload_images/5227029-f8c4886a84216943.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) Android Boot Sequence (from [Embedded Android](http://www.amazon.com/Embedded-Android-Porting-Extending-Customizing/dp/1449308295/))

### Java 应用与 Android app的差异

![典型 Android 应用模块的构建流程](http://upload-images.jianshu.io/upload_images/5227029-5a6bffc282428f11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从以上Android APP的编译流程上，我们也不难看出：由于Android 平台使用了一个不同于一般 JVM 的虚拟机，这就使得Java class 文件需要额外的处理（即 dex化）之后才能运行。

作为一个"推进器"，上述 `app_process` 除了启动 `Zygote`进程外，还可以创建其它进程。有兴趣的读者可以进一步参考链接中的 [Run a Java main on Android](https://blog.rom1v.com/2018/03/introducing-scrcpy/) 部分, 在命令行中实际编译Java代码，`dex` 处理以及通过 `adb shell` 命令打印出 Android 平台上的"Hello  World"。

## 1. 实现

先上一个截取屏幕并在浏览器中显示的效果图：
![Screen Shot.png](https://upload-images.jianshu.io/upload_images/5227029-4acde91b3b766bdd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.0 与截屏的相关API

在OS 4.3 之前有标注为(`@hide`)的API `android.view.Surface.screenshot ()`; 而4.3之后API变为 `android.view.SurfaceControl.screenshot()`. 非root的设备上，一般的APP是没有权限调用以上的接口的。而在`shell` 环境下确实具备权限的，而这正一点好成为了一个突破口。

### 1.1 代码入口方法

Java 类`Main`的静态方法 `main()` 中简单实现了一个 `HandlerThread` (可类比源码中[ActivityThread.main()](http://grepcode.com/file/repo1.maven.org/maven2/org.robolectric/android-all/5.0.0_r2-robolectric-1/android/app/ActivityThread.java#ActivityThread.main%28java.lang.String%5B%5D%29))。在looper 正在开始处理消息前，启动本地的server, 并设置对screenshot GET请求的回调方法。回调处理过程使用上述的 `screenshot()` 方法进行屏幕截图，并设置Bitmap数据为对应的 HTTP response。最后设置 ```adb forward tcp:53516 tcp:53516``` 将PC上所有 53516 端口通信数据重定向到手机端 53516 端口server上。

### 1.2 调用隐藏的API

在App内部，通常在有 **Context** 的情况下我们可以很方便地获取系统服务:

```
WindowManager window = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
```

而此时的入口 Main 是由 app_process 启动的一个独立进程。于是问题就出现了，如何获取当前屏幕的宽和高呢？想想框架中的 Java 部分代码是如何进行进程间通信的，常见的 [AIDL](https://developer.android.google.cn/guide/components/aidl.html) 成为了一个较好的方案。同样利用框架提供的```WINDOW_SERVICE```, 我们可以将系统源码中的 [IWindowManager.aidl](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/view/IWindowManager.aidl) 拷贝到工程目录中，利用对应生成的 [local stub](https://stackoverflow.com/questions/31908205/what-exactly-does-androids-hide-annotation-do/31908373#31908373) 通过编译，运行时通过反射调用对应所需的服务。

### 1.3 自动化ADB设置的命令行工具

~~类Unix系统上的工具 [cmd_runner.c]  (https://github.com/rayworks/DroidCast/blob/master/cmd_tool/cmd_runner.c#L284).~~ 

为达到跨平台功能，最近更新的 [python 脚本 (Python 2.7.15)](https://github.com/rayworks/DroidCast/blob/master/scripts/automation.py)
已实现了自动化启用截图功能，其中包括 adb forward/unforward PC网络请求，通过管道 (pipe) 跨进程通信得到已安装APP的实际路径，以及 shell 环境下调用 `app_process` 启动内部截图服务等。

对于已连接的单台设备/模拟器，默认调用：

```
python scripts/automation.py
```

即可在自动打开的默认浏览器中查看已连接设备的瞬时截图。

## 2. 源码

目前代码已经放在github [DroidCast](https://github.com/rayworks/DroidCast)，欢迎大家 star 和 fork，并与我交流。

## 3. 最近更新

* 2019-8-22 利用Python 2to3 转换工具，支持Python 3.6+

* 2019-5-30 Python 脚本中支持在有多个连接的设备时，对选定设备进行操作。

* 2019-5-18 用 Python 脚本自动化实现 `adb` 命令相关的设定和自动重置，以及打开默认浏览器查看截图

* 2019-5-7 增加对 websocket 的支持，在屏幕旋转后 web 页面自动刷新显示新截图

* 2019-4-14 支持不同格式的image请求（png, jpeg, webp）以及屏幕旋转后的截图 

* 2019-2-28 适配 Android Pie

* 2019-2-16 更新命令行工具，实现自动打开默认浏览器查看截图

* 2018-11-7 支持通过指定的大小截屏并显示图片

* 2018-10-30 增加`adb`设置说明，支持（相同网段WIFI环境下）无线使用场景

* 2018-9-5 更新了命令行工具，使其能定位安装到设备上的 apk 位置，解决 OS 4.3及以下的设备上出现的无法找到 class 导致的 crash。

* 2018-4-5 增加 *nix 环境下 command line tool (C 程序) 简化对 `adb` 命令相关的设定和自动重置

* 2018-3-28 解决OS 8.0 下 [加载 base.apk 失败的问题](https://developer.android.google.cn/about/versions/oreo/android-8.0-changes.html#security-all)。

> * You can no longer assume that APKs reside in directories whose names end in -1 or -2\. Apps should use [sourceDir](https://developer.android.google.cn/reference/android/content/pm/ApplicationInfo.html#sourceDir) to get the directory, and not rely on the directory format directly.

## 4. 参考引用

* 隐藏的API
[截屏相关(以OS 5.1.1为例)](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/view/SurfaceControl.java#SurfaceControl.screenshot%28int%2Cint%29)

* zygote有关
[zygote](https://anatomyofandroid.com/2013/10/15/zygote/)
[understanding android zygote and dalvik vm](http://stackoverflow.com/questions/9153166/understanding-android-zygote-and-dalvikvm)

* adb forward 命令 
http://blog.csdn.net/mars5337/article/details/6395232

* [Network library: AndroidAsync](https://github.com/koush/AndroidAsync)

## 5. 声明

本文首发于[简书](https://www.jianshu.com/p/791a3ed2a348)，转载请注明出处。
