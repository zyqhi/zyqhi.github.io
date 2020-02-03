---
title: Cocoa Frameworks
date: 2020-01-16 19:01 +0800
tags: Cocoa
typora-root-url: "/Users/mutsu/project/blog/zyqhi.github.io"
---

# 引言

在Cocoa/Cocoa Touch框架中，我们经常使用Framework作为代码和资源的共享方式。那么Framework到底是什么呢？其实并不神奇，Framework其实就是一个Bundle。而所谓Bundle，是指一个内部结构按照**标准规则**组织的特殊目录。

值得说明的是Framework和静态库以及动态库的区别。`.a`和`.dylib`分别为静态库和动态库，它们本身就是编译好的二进制代码。而Framework，以`.framework`作为扩展名，它其实是对静态库/动态库的一层包装，目的是为了方便使用。

 除包含二进制代码以外，Framework还可以包含头文件、nib文件、图片、字符串子资源文件。

# Framework分类

常见的Framework有三类，分别为：

- Static Framework：对静态库的包装，可以将静态库简单地理解为头文件、资源文件、二进制代码的合集。Static Framework经过链接/linking以后，会成为目标程序的一部分。因此，当生成目标程序`Target.app`以后，直接查看`Target.app`的目录接口，不会看到对应的Static Framework文件。
- Dynamic Framework：对动态库的包装。iOS上系统提供的动态库用这种封装方式，比如UIKit.framework。可以在多个应用之间共享一份代码。
- Embedded Framework：也是对动态库的包装，在结构上和Dynamic Framework没什么区别，只不过应用场景不同，不同点如下：
  - 受限于iOS系统的安全机制，Embedded Framework只能在App及其Extension之间进行共享。
  - 系统的Dynamic Framework不需要拷贝到目标程序中，而Embedded Framework则需要。

# 解剖Framework

## Framework Bundle结构

Framework不仅仅是库文件的简单封装，同时也支持版本控制。简单来说，一个Framework可以包含多个版本的库文件，下文是一种Framework的示例：

```
MyFramework.framework/
    Headers      -> Versions/Current/Headers
    MyFramework  -> Versions/Current/MyFramework
    Resources    -> Versions/Current/Resources
    Versions/
        A/
            Headers/
                MyHeader.h
            MyFramework
            Resources/
                English.lproj/
                    Documentation
                    InfoPlist.strings
                Info.plist
        B/
            Headers/
                MyHeader.h
            MyFramework
            Resources/
                English.lproj/
                    Documentation
                    InfoPlist.strings
                Info.plist
        Current  -> B
```

其中Headers/MyFramework/Resources分别为：

- Headers：头文件，定义了库的接口；
- MyFramework：预编译的二进制代码，与Framework同名；
- Resources：相关资源，nib文件等。

另外需要指出的是，Headers/MyFramework/Resources等并非真实的目录或文件，而是符号链接，指向Versions目录下的Current目录。Versions目录下每一个子目录代表一个主版本/major version。而Current指向目录B，表明当前Framework的版本为B。

这样只需要修改Current的指向，便可实现引用特定版本库的目的。

## Framework示例

### Static Framework示例

通常我们遇到的Framework没有这么复杂，经常就只有一个版本，下面来看一个具体的例子：腾讯开放平台的SDK `TecentOpenAPI.framework`，可以点击[这里](https://wiki.open.qq.com/index.php?title=mobile/SDK%E4%B8%8B%E8%BD%BD&oldid=47694)下载。

![image-20200203143531871](/../../../../../../../media/2020-01-16-cocoa-frameworks/image-20200203143531871.png)

再来看下`TecentOpenAPI`文件：

``` bash
➜  TencentOpenAPI.framework file TencentOpenAPI
TencentOpenAPI: Mach-O universal binary with 4 architectures: [arm_v7:current ar archive] [i386] [x86_64] [arm64]
TencentOpenAPI (for architecture armv7):	current ar archive
TencentOpenAPI (for architecture i386):	current ar archive
TencentOpenAPI (for architecture x86_64):	current ar archive
TencentOpenAPI (for architecture arm64):	current ar archive
```

会发现就是一个普通的静态库。

> 此处的静态库为FAT-Binary，包含多个平台的静态库。

### Dynamic Framework示例

采用CocoaPods做依赖管理时，可以在`Podfile`中添加`use_framworks!`来声明将依赖库以动态Framework的形式引入当前项目。

添加`use_framworks!`以后，所有的Pod都将编译成Dynamic Framework。以`ReactiveObjC.framework`为例来看下Dynamic Framework的结构，如下：

![image-20200203154532235](/../../../../../../../media/2020-01-16-cocoa-frameworks/image-20200203154532235.png)

再来看下`ReactiveObjC`文件：

```
➜  ReactiveObjC.framework file ReactiveObjC
ReactiveObjC: Mach-O 64-bit dynamically linked shared library x86_64
```

会发现与`TecentOpenAPI`不同，`ReactiveObjC`为动态链接库，并且只包含x86_64架构的代码。

动态链接库同样可以包含多个架构的代码，以Flutter为例，当我们将Flutter以Dynamic Framework的形式引入项目时，会生成`Flutter.framework`文件，观察该Framework下的`Flutter`文件：

```
➜  Flutter.framework file Flutter
Flutter: Mach-O universal binary with 2 architectures: [arm_v7:Mach-O dynamically linked shared library arm_v7] [arm64]
Flutter (for architecture armv7):	Mach-O dynamically linked shared library arm_v7
Flutter (for architecture arm64):	Mach-O 64-bit dynamically linked shared library arm64
```

可以看到该动态链接库包含了多个平台的代码。

# Framework应用

工程模块化/组件化

cocoapods



代码分发

什么情况下需要用Framework？



@import的作用？Modules的概念，以及其于Framework的联系。



# 参考

- [iOS中的动态库，静态库和framework介绍](https://juejin.im/post/5df47cc0e51d45582e21ba99)
- [Creating a Framework for iOS](https://www.raywenderlich.com/5109-creating-a-framework-for-ios)
- [iOS开发里的Bundle是个啥玩意？！](https://www.cnblogs.com/BigPolarBear/archive/2012/03/28/2421802.html)
- [Bundle](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/Bundle.html)
- [Cocoa Core Competencies](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/Introduction.html#//apple_ref/doc/uid/TP40008195-CH68-DontLinkElementID_2)
- [Introduction to Framework Programming Guide](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPFrameworks/Frameworks.html#//apple_ref/doc/uid/10000183-SW1)
- [ios Standard Framework和Umbrella Framework](https://www.cnblogs.com/wenrisheng/p/6603618.html)
- [@import vs #import - iOS 7](https://stackoverflow.com/questions/18947516/import-vs-import-ios-7)
- [Creating iOS/OSX Frameworks: is it necessary to codesign them before distributing to other developers?](https://stackoverflow.com/questions/30963294/creating-ios-osx-frameworks-is-it-necessary-to-codesign-them-before-distributin)
- [Why do we use use_frameworks in CocoaPods?](https://stackoverflow.com/questions/41210249/why-do-we-use-use-frameworks-in-cocoapods)

