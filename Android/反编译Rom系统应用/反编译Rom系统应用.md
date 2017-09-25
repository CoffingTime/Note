## 反编译Rom系统应用

### 基础知识

**dex**  android平台可执行文件

**odex** 将dex文件优化生成一个单独文件存放

**smali** 虚拟机指令语言，反编译dex文件可以看到相关指令集

### Rom层应用分析

**odex与apk**

> rom厂商会把系统级别的app源码和资源文件做分离

### ![img](http://upload-images.jianshu.io/upload_images/1110736-8e0d455022210de1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从右边图可以看出，系统app的没有看到dex文件

> 寻找系统app比较麻烦。。。

apk通常会包含一个Resources.arsc文件，这个文件是存在app的资源文件索引表，包括xml、图片、文件等。而系统应用除了在Resources.arsc有索引外，还会跟frameWokr的资源产生关联（不确定）。

**框架文件** 是指 `framework的framework.apk、core.` 等，![img](http://upload-images.jianshu.io/upload_images/1110736-42b3a996cc17db91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1110736-4a7c9babe27c42fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



rom层对系统应用进行了`odex`优化，其中就包括了资源文件的依赖，有一部分是存放在`framework.apk、core.jar`等。并不像我们平时开发app的时候只存在于`Resources.arsc`文件中。不过因为厂商和系统版本不同，它们的名称可能也有点差别.对应的目录是：`\system\framework`

![img](http://upload-images.jianshu.io/upload_images/1110736-b714d2482d1be616.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到里面并没有包含`dex`文件，只有`res`和`assets`资源文件。



## 逆向工具箱

下面我们一起来看一下现在主流的反编译工具，我会依依简单介绍。

- baksmali
  `baksmali`用于将`odex`解析成`smali`的工具。可能改个名字叫（`odex2dex`更合适）
- dex2jar
  将dex转换成jar文件。
- gd-gui

反编译整个`jar`可以对把smali文件解析成`java`源码查看的一款工具。

- apktools
  拥有对`apk`编译、反编译、签名等功能。
- SVADeodexerForArt
  自动合并框架工具，可以通过简单的配置，就可以合并出`rom`里的`app`和`odex`文件。
- apkDB
  中文名称安卓逆手，整合了反编译多款工具的一个工具集合，功能很强大，几乎你想要的一切操作都能在里面找到对应的工具命令。
- 当前Activity
  这是一款app应用，用于显示当前手机界面上的`Activity`等包名信息。

**工具下载地址：**
**baksmali**
**dex2jar**
**JD-GUI**
**Apktool**
**SVADeodexerForArt**
**apkDB**
**当前Activity** (需要过墙)



## 实战逆向MIUI系统设置WIFI

首先打开系统设置，跳转到WIFI设置界面。
它是酱样子的。

![img](http://upload-images.jianshu.io/upload_images/1110736-1ab716a957afeac6.png?imageMogr2/auto-orient/strip%7CimageView2/2/h/400)

接下来我们需要知道这个`Activity`的包名和`fragment`的叫什么。
打开Android ADB shell
输入：**dumpsys activity top**
如下图所示，我们可以清晰的看到我们需要的信息。
**Activity:** com.android.settings.SubSettings
**Fragment:** MiuiWifiSettings

![img](http://upload-images.jianshu.io/upload_images/1110736-5932dd50ae4d8312.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1.下载rom包
2.-->解压
3.--> `SVADeodexerForArt`合并`odex`
4.-->`apktools`反编译
5.-->`smali.jar`打包`smali`成`dex`
6.-->`dex2jar.jar`转换`dex`成`jar`
7.-->`jd-jui`工具加载jar查看



至此一共七个步骤，其实这个操作并不难，只是需要借助多款工具来完成就略显步骤复杂。然后中间可能会有些小坑，卡住的话就容易让人失去信心。新手别去强记这些，用多了就自然就熟悉了。

