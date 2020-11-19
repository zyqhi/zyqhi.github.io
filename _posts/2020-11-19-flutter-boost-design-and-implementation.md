---
layout:
title: FlutterBoost设计与实现分析
date: 2020-11-19 13:58 +0800
tags:
typora-root-url: "/Users/mutsu/project/blog/zyqhi.github.io"
---

# 前言

[FlutterBoost](https://github.com/alibaba/flutter_boost)是闲鱼技术团队开发并维护的，面向iOS/Android端的Flutter混编解决方案。FlutterBoost不仅仅是一个Flutter Plugin，那只是它的工程形态。我倾向于认为FlutterBoost定义了Native/Flutter混合栈开发模式下，一套标准的Flutter使用方式，包括页面栈管理与路由、应用/页面生命周期管理、Flutter/Native页面传参方式等核心问题的解决方案。FlutterBoost致力于通过框架建设，标准化Flutter的使用方式，从而降低Flutter的使用门槛，让开发者能够像使用WebView一样使用Flutter。

本文依据官方文档、代码，结合个人理解，Native侧以iOS平台为例，详述FlutterBoost Flutter侧和Native侧的实现原理。期望对使用和学习FlutterBoost的同学有所帮助。由于FlutterBoost处于不断的迭代过程中，本文分析依赖的代码和和环境版本信息如下：

- FlutterBoost版本：master（fa8ab3d）
- Flutter版本：1.22.2
- iOS系统环境：iOS 14.1

阅读本文，读者需要具备以下iOS/Flutter相关的基本的知识，如果对这些概念不熟悉，可以先查阅相关概念的材料，或者先有个大致的了解，等后续用到时再查阅。这些概念包括但不限于：

**iOS侧**

- FlutterEngine概念
- FlutterViewController
- UIViewController生命周期

**Flutter侧**

- Flutter Plugin的基础概念与开发方式
- Method Channel：Native/Flutter通信机制
- Key
- Overlay Widget
- Navigator Widget：Flutter侧页面导航与页面栈管理方案

好了，有了以上前提之后，我们可以来讨论FlutterBoost的设计原理了。FlutterBoost是一套混合栈开发的解决方案。其抽象的核心是围绕『页面』来展开的，将两套不同UI系统的有机的结合是FlutterBoost需要解决的首要问题，架构的设计也围绕着要解决的问题展开。因此在讨论FlutterBoost的架构设计之前，先来看下Flutter的页面管理方案。

# 页面管理方案选型

FlutterBoost的页面管理方案也是随着Flutter的演进而随之改变的。以Flutter SDK 1.5版本作为分水岭，FlutterBoost的页面管理方案发生了较大的变化。先来看现在的方案。

Flutter 1.5之后的版本，对FlutterEngine和FlutterViewController进行了解耦。解耦之后的FlutterEngine和FlutterViewController的关系是**1对N**的关系。FlutterEngine负责管理Dart代码的执行、channel通信、纹理与插件的注册等。而FlutterViewController则作为Flutter UI的展示容器，是架在iOS的UIKit框架和Flutter UI框架之间的一座桥。它们的关系可以用下图来表示：

![image-20201119140444669](/../../../../../../../media/2020-11-19-flutter-boost-design-and-implementation/image-20201119140444669.png)

FlutterBoost的页面管理方案也是基于这种基础模型之上的。先抛开FlutterBoost的页面管理方案，我们先来看下混合栈下多页面支持的的几种实现方案：

1. 单引擎、单FlutterViewController容器方案。通过设定不同的参数，容器渲染不同的Flutter页面，从而实现多页面支持，同时通过截图实现页面转场；
2. 单引擎、多FlutterViewController容器方案。单个容器渲染单个页面，也即容器和页面一对一的关系，通过切换不同的容器来实现多页面支持；
3. 多引擎、多FlutterViewController容器方案。引擎、容器、页面皆为一对一关系，通过切换不同引擎来实现多页面支持。

Flutter 1.5之前的版本，容器FlutterViewController和引擎FlutterEngine耦合在一起，二者的关系是一对一的。因此方案1和方案3是这种架构下的多页面的两种实现方案。彼时的FlutterBoost基于这种关系，在设计上选用了方案1，而非方案3来实现多页面支持。原因在于多引擎模式下，引擎之间相互独立，这种模式的好处是执行环境之间可以很好地隔离，但是随之带来的问题是资源占用上升、插件注册与管理复杂、页面通信复杂等问题。

但方案1也并不完美，由于只存在一个容器，在页面之间进行切换时，页面无法『感知』自己的生命周期，而在实际开发中，感知页面生命周期无论是在业务还是技术上都是必要的，比如一些埋点和监控逻辑。其次必须通过截图的方式来保存上个页面的『状态』，而CPU截图保存与加载的耗时操作带来的资源占用，会影响主线程渲染，出现偶尔的白屏和黑屏问题。不仅如此，截图带来的内存占用则在压入过多页面时，增加了OOM的风险。

Flutter 1.5之后，FlutterBoost也进行了重构和升级，在多页面支持上采用了方案2。方案2通过容器的生命周期赋予页面同等的生命周期感知能力。同时多容器模式也不再需要通过截图来实现老页面还原，避免了因截图引入的黑白屏问题，降低了OOM风险。可谓最佳实践。最新的FlutterBoost页面管理方案如下：

![image-20201119140550567](/../../../../../../../media/2020-11-19-flutter-boost-design-and-implementation/image-20201119140550567.png)

在最新的FlutterBoost页面管理方案中，我们可以看到，从Native侧的视角来看：

- 页面栈管理全部由UINavigationController管理，FlutterViewController容器和普通的UIViewContorller没有任何区别，可以完全相容，对于UINavigationController而言是完全透明的；
- 页面切换时，通过通知FlutterEngine attach不同的FlutterViewController容器，从而实现多页面支持。

而从Flutter侧的视角来看：

- 只需要接收来自FlutterEngine的通知，在容器切换时，渲染不同的Flutter页面即可，职责单一且简单；
- BoostContainerManager本身是一个StatefulWidget，当Native侧切换到不同的容器时，BootContainerManager改变其子Widget，从而实现与容器的同步。

> 注：这里Native侧的页面管理以UINavigationController为例，采用其他页面栈管理方案原理是类似的。

# 架构设计与核心概念

前文阐述了FlutterBoost的页面管理方案与背景。总结下来就是：FlutterBoost采用单引擎、多FlutterViewController容器的模式来管理页面。这种方案下，切换不同的页面，就是切换不同的容器，然后通知引擎渲染不同的Flutter Widget。

这种页面管理方案的设计参考了WebView，所以说FlutterBoost不仅仅是一个Plugin，而是定义了一种采用Flutter进行开发的容器化方案。FlutterBoost的架构图如下：

![image-20201119140650654](/../../../../../../../media/2020-11-19-flutter-boost-design-and-implementation/image-20201119140650654.png)

在这个架构体系下，FlutterBoost在Native侧和Flutter侧分别引入了若干概念，这些概念是理解整个架构的前提，我们先来看下：

## Native侧概念

- Container：Native容器。平台Controller，Activity，ViewController。对于iOS而言，**这里Container就是指UIViewController及其子类**。可以是FLBFlutterViewContainer（FlutterBoost定义的iOS侧Flutter容器，派生自FlutterViewController），也可以不是；
- Container Manager：容器管理者。也就是管理多个UIViewController的方案，比如我们最常见的就是基于UINavigationController的导航栈管理方案；
- Router：路由系统。接收页面URL，操纵页面容器进行页面路由。实际应用中，我们可以对Flutter页面和Native页面采用统一的基于URL的建模方式，在Router中进行区分，从而使得上层调用不需要关心页面的实现方式。

## Flutter侧概念

- Widget：表示一个Flutter页面。这里Widget就是普通的StatelessWidget或者StatefulWidget，对于使用者而言，所看到的『页面』就是这个Widget；
- Container：Flutter侧用来容纳Widget的容器。FlutterBoost中为BoostContainer（派生自Navigator），Widget作为BoostContainer的**根Widget**（即路径为/的Widget）存在；
- Container Manager：Flutter容器管理者。具体为BoostContainerManager类（派生自StatefulWidget），该类管理Flutter侧的页面栈，同时提供Push/Pop/Remove等页面管理操作API。BoostContainerManager的子Widget为BoostContainer；
- Coordinator：协调器，接受消息，负责调用Container Manager的状态管理。

## 架构要点说明

- 架构设计上Native侧和Flutter侧有一定的对称性，Native侧有的模块，Flutter侧都有与之对应的模块，反之亦然，但模块能力又有略微差异；
- 所有的路由逻辑，无论是Native侧的还是Flutter侧的，统一都由Native接管。无论Flutter侧还是Native侧，页面的Push/Pop/Remove操作都映射成Native侧的容器的对应操作，而Flutter侧通过被动接受Native侧的消息，根据消息内容调整Widget树。

FlutterBoost的架构设计中，各个模块的职责和边界还是比较清晰且简单的。

# 详细设计实现分析

前文分析了FlutterBoost页面管理方案以及架构设计，这节我们来接着分析一下FlutterBoost的详细实现。详细实现这部分，主要目的是为了方便对FlutterBoost代码感兴趣的同学分析代码，因此比较细节，不感兴趣的可以跳过。

在分析其详细实现之前，有一点需要明确，就是：FlutterBoost在框架设计时是想让使用者像使用WebView一样使用Flutter。回忆一下我们怎么使用WebView：创建WebView；设置URL并加载特Web页面；通过JSBridge和Web页面进行通信。在这套流程里面，Native只负责WebView容器的管理，包括创建、初始化、销毁等，而WebView负责UI渲染。

采用FlutterBoost时，使用流程基本和使用WebView一样。Native负责Flutter容器的管理，包括创建、初始化、销毁等，而Flutter容器负责UI渲染。

## Native侧实现

先来看Native侧的实现。下图是iOS平台上FlutterBoost Native侧一些关键的对象及其之间的引用关系：

![image-20201119140804404](/../../../../../../../media/2020-11-19-flutter-boost-design-and-implementation/image-20201119140804404.png)

从图中可以看出，这些对象中的处于核心的位置的是FLBFlutterApplication，其他对象都是围绕它来展开。现在来分别介绍。

### FlutterBoostPlugin

FlutterBoostPlugin主要面向的是FlutterBoost的使用者，它提供了使用者需要关心的一切内容，封装了FlutterBoost内部所有复杂的实现细节。是使用者和FlutterBoost内部实现的『交互窗口』。FlutterBoostPlugin只是一层壳，其逻辑都交由其他对象实现。它的功能主要分为以下几部分：

- 提供Flutter及FlutterBoost环境初始化API。主要为startFlutterWithPlatform:系列方法，具体逻辑由FLBFlutterApplication实现。
- 提供页面栈操作API。有open/present/close三种，具体逻辑由FLBFlutterApplication实现。
- 提供Flutter/Native消息发送与接收API。有-addEventListener:forName:和-sendEvent:arguments:两种。具体逻辑由BoostMessageChannel实现。
- 监听Flutter侧发送的消息，主要有openPage和closePage。

### FLBFlutterApplication（核心）

FLBFlutterApplication是FlutterBoost Native侧的核心。它的主要工作是衔接Flutter引擎、平台路由模块以及通信模块，协调各模块之间的交互。其具体功能如下：

- 创建Flutter及FlutterBoost运行环境

主要工作为按需创建FlutterEngine，注册其他Flutter插件等。FlutterBoost允许外部创建FlutterEngine，然后在FlutterBoost初始化时传入，也提可以将FlutterEngine的创建工作交由FlutterBoost实现。

- 统一页面栈操作API，屏蔽平台差异

将页面操作抽象成open/present/close三种，具体的页面栈操作，FLBFlutterApplication交由platfrom代理对象PlatformRouterImp实现。FlutterBoost中platform指的就是宿主环境——iOS或Android。之所以交由platform去实现，是因为不同平台，页面栈操作方式不一样，而且即使平台相同，不同的App也可能采用不同的路由和页面管理方案。因此FLBFlutterApplication此处在设计上仍然是依赖接口，而不依赖实现。

- 提供Flutter页面栈查询API

FLBFlutterApplication内部维护了所有的采用Flutter进行渲染的页面，记录到表中，该表的实现类为FLBFlutterContainerManager。通过维护这些信息，FLBFlutterApplication能够对外提供诸如-contains:、-pageCount等页面栈查询API。

- 提供FlutterEngine操作API

FlutterEngine定义了一个lifecycleChannel属性用于接收App的生命周期变化相关消息，但是该消息基于字符串，比如AppLifecycleState.paused，FlutterBoost在这之上做了二次封装，提供pause/resume/inactive等API。这部分功能由FLBFlutterEngine类实现。

- 页面（容器）数据传递

FlutterBoost定义了FLBFlutterViewContainer作为Flutter页面的容器，为了赋予页面关闭时能够回传结果数据的能力，FLBFlutterApplication内部维护了一个映射表_pageResultCallbacks，该表的Key为页面uniqueId，Value为对应的回调。当调用open/present时传入对应的回调，对应的回调会在调用close时得以执行，并回传对应的数据。

### FLBFlutterEngine

FLBFlutterEngine是对FlutterEngine的简单封装。

### FLBFlutterContainerManager

管理采用Flutter渲染的页面。

### PlatformRouterImp<FLBPlatform>

FlutterBoost定义了FLBPlatform协议，用于约束宿主平台接入FlutterBoost以后，其路由模块需要提供的能力。

前文提到过，FlutterBoost只定义页面的open/present/close操作原语，具体的页面路由，都由平台自行实现。实际开发中，开发者只需要提供一个遵循FLBPlatform协议的类即可，此处以PlatformRouterImp为例。在已有的App中以Hybrid的方式接入Flutter时，我们需要对原来的路由系统进行改造，使之能够兼容处理Flutter页面和Native页面。通常而言，路由融合逻辑便是在此类中实现的。

### FLBFlutterViewContainer（核心）

FLBFlutterViewContainer是另外一个比较核心的类。它除了用于渲染Flutter页面以外，整个容器的创建与释放，以及Flutter侧的渲染逻辑都依赖该类的生命周期。

在PlatformRouterImp中，当路由模块判定一个页面需要使用Flutter渲染时，便创建一个FLBFlutterViewContainer实例，并将其压入到Native的页面栈。随后UIKit会在FLBFlutterViewContainer生命周期的各个节点回调对应的方法。在生命周期相关方法调用时，FLBFlutterViewContainer实例会通过BoostMessageChannel通知Flutter侧，从而完成Flutter侧的Widget树调整。

**页面表示**

在FlutterBoost中，一个『页面』在Native侧对应一个FLBFlutterViewContainer实例，在Flutter侧对应一个OverlayEntry Widget（后文中会讲到）。和页面相关的参数有三个，如下表所示：

| **字段名** | **类型** | **可选/必传** | **含义**                                                     |
| ---------- | -------- | ------------- | ------------------------------------------------------------ |
| pageName   | String   | 必传          | 页面名称。使用者自定义，允许重复，实践中，通常选择该页面对应的URL作为页面名称。pageName类似与WebView中的URL。 |
| uniqueId   | String   | 必传          | 页面唯一ID。FlutterBoost内部维护，每创建一个FLBFlutterViewContainer实例，该值便自增1。该值唯一，因此可以作为容器的唯一标识使用。关闭页面（容器）的操作也是通过传入该id来实现的。 |
| params     | Map      | 可选          | 页面参数                                                     |

在创建容器以后，传入pageName，从而决定当前容器需要渲染的Flutter Widget。

**生命周期**

前文也提到，在容器的生命周期方法调用时，会通过Method Channel通知Flutter侧，从而完成Native和Flutter的协同。如下表，是Push/Present时页面的生命周期，其中Native一栏是ViewController的生命周期回调，Flutter一栏与之对应的是Flutter侧收到的消息通知。通过这种机制，FlutterBoost能够保证，当容器出现在Native的页面栈上时，Flutter侧也绘制对应的页面。

| **Push（Open）/Present**       |                       |
| ------------------------------ | --------------------- |
| **Native**                     | **Flutter**           |
| willMoveToParentViewController | didInitPageContainer  |
| viewWillAppear                 | willShowPageContainer |
| viewDidLayoutSubviews          |                       |
| viewDidAppear                  | didShowPageContainer  |
| didMoveToParentViewController  |                       |

> 注：FlutterBoost要求Present只能是**全屏**的方式，否则会出现异常。这是因为在整个FlutterBoost框架下，Flutter侧的页面渲染强依赖于容器的生命周期。只有将模态ViewController的modalPresentationStyle设置为UIModalPresentationFullScreen，各个生命周期函数才会正确执行，Flutter页面也得以正常显示。

下表则是Close操作时，Native侧的生命周期变化与对应的Flutter侧收到的通知。其中比较关键的点是容器的释放，当容器释放时，Flutter侧也将该页面对应的Widget从Widget树上移除。

| **Close**                      |                            |
| ------------------------------ | -------------------------- |
| **Native**                     | **Flutter**                |
| willMoveToParentViewController |                            |
| viewWillDisappear              | willDisappearPageContainer |
| viewDidDisappear               | didDisappearPageContainer  |
| didMoveToParentViewController  | willDeallocPageContainer   |
| dealloc                        | willDeallocPageContainer   |

FlutterBoost赋予Flutter页面感知生命周期的能力，正是通过这种方式实现的。

### BoostMessageChannel

BoostMessageChannel是一个公共的通信信道，它为一组通信API，底层通过FlutterEngine创建时注册的Method Channel进行通信。前文中提到的所有模块需要向Flutter侧发送消息时，皆通过此类实现。

## Flutter侧实现

Flutter侧与Native侧在设计上是对称的，都有消息模块，页面栈管理模块，但是在具体实现上又有所差异，如下图所示是Flutter侧相关的对象及其引用关系：

![image-20201119140848819](/../../../../../../../media/2020-11-19-flutter-boost-design-and-implementation/image-20201119140848819.png)

在Flutter侧的实现中，以BoostContainerManager为根Widget实现了一棵Widget树。可以简单地理解为，整个Flutter侧的核心逻辑为：接收Native消息，根据消息内容对这棵Widget树进行调整（增删改查），从而完成所定义的功能。下文将对各个对象进行介绍：

### FlutterBoost

FlutterBoost是Flutter侧的入口类，集合了使用者所需要关注的各种API。具体有：

- 提供页面builder注册API。即registerPageBuilder系列方法，具体逻辑由ContainerCoordinator实现。
- 提供添加容器生命周期监听的相关API。
- 页面栈操作API。有open/close系列方法，Flutter侧调用这两种API时，通过BoostChannel发送到Native，然后由Native来执行具体的路由逻辑。open和close对应的Method Channel方法名称和参数分别如下：

**openPage**

| **字段名** | **类型** | **可选/必传** | **含义**                                                     |
| ---------- | -------- | ------------- | ------------------------------------------------------------ |
| url        | String   | 必传          | 页面URL，也即pageName                                        |
| urlParams  | Map      | 可选          | 页面URL参数                                                  |
| exts       | Map      | 可选          | 扩展参数，比如animated、params，控制容器关闭打开关闭时是否需要执行动画等都是通过扩展参数实现 |

**closePage**

| **字段名** | **类型** | **可选/必传** | **含义**                       |
| ---------- | -------- | ------------- | ------------------------------ |
| uniqueId   | String   | 必传          | 页面（容器）的唯一id           |
| result     | Map      | 可选          | 关闭页面时的返回值             |
| exts       | Map      | 可选          | 扩展参数，比如animated、params |

### ContainerCoordinator

ContainerCoordinator负责监听channel，处理来自Native的事件和生命周期相关的方法。

该对象内部维护一个页面构建的映射表_pageBuilders。_pageBuilders是FlutterBoost维护的比较关键的一个映射表。其Key为一个String对象，由使用者自行定义，唯一标识一个Flutter页面，实践中一般可用页面的URL（也即pageName）来作为Key。其Value为一个PageBuilder对象，也即一个构造页面的回调方法。使用者通过调用FlutterBoost对象提供的registerPageBuilders在初始化阶段，将页面构造器注册到FlutterBoost，也即存入到_pageBuilders中。如下述代码所示，分别向FlutterBoost注册了first和second两个页面。

```
  FlutterBoost.singleton.registerPageBuilders(<String, PageBuilder>{
   'first': (String pageName, Map<String, dynamic> params, String _) => FirstRouteWidget(),
   'second': (String pageName, Map<String, dynamic> params, String _) => SecondRouteWidget(),
  });
```

在此例中，first和second为pageName，而FirstRouteWidget和SecondRouteWidget则是这两个页面在Flutter侧的具体实现。

**处理的事件**

| **事件名称**        | **说明**     |
| ------------------- | ------------ |
| backPressedCallback | 后退按钮点击 |
| foreground          | 进入前台     |
| background          | 进入后台     |
| scheduleFrame       |              |

**处理的方法**

这些方法在FLBFlutterViewContainer相关生命周期的函数中调用，前文已述，此处不再赘述。

| **方法名称**               | **说明**       |
| -------------------------- | -------------- |
| didInitPageContainer       | 容器创建       |
| willShowPageContainer      | 容器即将可见   |
| didShowPageContainer       | 容器已可见     |
| willDisappearPageContainer | 容器即将不可见 |
| didDisappearPageContainer  | 容器不可见     |
| willDeallocPageContainer   | 容器即将释放   |
| onNativePageResult         | 弃用           |

### BoostChannel

BoostChannel为Flutter/Native的通信channel。提供消息发送与接收API。

FlutterBoost定义了一类特殊方法，称为事件，方法名__event__。对于其他方法调用，FlutterBoost不做限定，直接透传到Native。但对于FlutterBoost内部定义的方法，比如openPage，closePage等，外部应该避免使用。

对于需要监听方法调用的模块，需要提前注册。如下：

- _eventListeners：事件监听者。调用addEventListener进行注册，监听Native事件。调用sendEvent向Native发送事件。
- _methodHandlers：自定义方法监听者。调用addMethodHandler进行注册，监听Native事件。调用invokeMethod系列方法向Native侧发起事件调用。

**事件类方法**

方法名：**__event__**

| **字段名** | **类型** | **可选/必传** | **含义** |
| ---------- | -------- | ------------- | -------- |
| name       | String   | 必传          | 事件名称 |
| arguments  | Map      | 可选          | 事件参数 |

**自定义方法**

方法名：模块自定义，但不能是**__event__**

### BoostContainerManager

BoostContainerManager派生自StatefulWidget。我们知道，采用Flutter编写UI代码，就是在操作一棵Widget树，所有UI操作无非是对这棵Widget树的增删改查。而采用FlutterBoost作为开发框架时，BoostContainerManager就是这棵UI树的根节点。

如下图所示，是FlutterBoost对UI建模的模型。了解整棵树需要先了解Overlay Widget和Navigator Widget。二者都是容器型Widget，前者类似于Stack Widget，用于显示堆叠功能，允许一个Widget叠在另一个Widget之上，后者是导航Widget，类似于iOS中的UINavigationController。

![image-20201119140958702](/../../../../../../../media/2020-11-19-flutter-boost-design-and-implementation/image-20201119140958702.png)

在这棵UI树中，Overlay Widget是最关键的Widget。每当Native侧创建一个FLBFlutterViewContainer实例，Flutter侧便对应创建一个OverlayEntry，并加入到Overlay的_entries中。因此Native侧的容器和Flutter侧的OverlayEntry是一一对应的。

BoostContainerManager(ContainerManagerState)定义了pushContainer用于在页面栈压入新页面。定义了pop于删除当前页面栈顶部的页面，以及remove用于删除特定页面。BoostContainerManager内部用_onstage表示当前页面栈顶部的页面，而_offstage则用于表示页面栈中不可见的页面。

这里需要指出的是：FlutterBoost在Flutter侧对于页面栈（Widget栈）的建模并不是基于Navigator Widget，而是Overlay Widget。如下图，是Flutter侧的Widget树示例：

![image-20201119141048107](/../../../../../../../media/2020-11-19-flutter-boost-design-and-implementation/image-20201119141048107.png)

这里是我唯一一处我不太明白地方。为什么在Flutter侧Widget不是直接作为Overlay的子Widget，而是要先嵌入BoostContainer后再作为Overlay的子Widget存在？一种猜测是为了保留BoostContainer内的纯Flutter页面路由能力——即Navigator管理的纯Flutter路由——但我个人认为这种能力是不太有必要的，因为这与整个框架的设计的初衷不符，而且采用这种方式进行路由，Flutter侧被路由的页面无法感知对应的生命周期，需要开发者自行管理。

# FlutterBoost使用经验总结

截止到目前，本文分析了FlutterBoost实现相关的内容。而对于大部分开发者而言，我们关注的是使用。我们App在接入Flutter以后，也是采用了FlutterBoost作为了我们的混编页面管理方案。目前我们唯一遇到的问题是和我的页面框架融合时，期初时遇到了容器无法释放的问题。我们App中一个页面的结构是SomeContainerController+SomeNavigationController+SomeCustomViewControllerr这种形式。作为业务ViewController的SomeCustomViewController是在外围两个容器Controller包裹之下的。在采用Flutter以后，对于Flutter渲染的页面，很自然地，Native一侧我们选择的容器方式是

SomeContainerController+SomeNavigationController+FLBFlutterViewContainer这种方案。但由于之前的逻辑由于不依赖UIViewController生命周期，实现时并没有严格调用ViewController相关生命周期方法，尤其是didMoveToParentViewController方法，导致的问题是FlutterBoost内部容器释放的相关逻辑没有执行，从而导致页面退出以后，容器没有释放。

解决方案也比较直接，修正Native侧的之前容器ViewController的实现方案，添加对应的生命周期调用即可。

另外一点便是FlutterBoost的应用范围的问题。FlutterBoost的设计是基于App一次只有一个页面（ViewController）对用户可见的假设来设计的。在Native侧支撑的Push和Present都必须是全屏展示，以使得ViewController的相关生命周期能够正常调用，Flutter侧其实也只有一个引擎和一个Widget占据整个屏幕，即使有多个容器时，也只能一个可见的Widget。这导致的问题是我们无法实现显示半屏的页面。关于这种情况，我觉得解决方案是不是从容器这个角度来支持这种能力，而是通过设计一个半屏的Widget来实现这种需求。

# 参考

本文页面选型和架构部分主要引用以下两篇文章：

- 码上用它开始Flutter混合开发——FlutterBoost：https://my.oschina.net/u/1464083/blog/3038015
- FlutterBoost1.0到2.0，我一共做了这几件事...：https://zhuanlan.zhihu.com/p/114389375

Flutter相关概念：

- https://api.flutter.dev/objcdoc/Classes/FlutterViewController.html
- https://api.flutter.dev/objcdoc/Classes/FlutterEngine.html
- https://api.flutter.dev/objcdoc/Classes/FlutterDartProject.html
- [Learning Flutter’s new navigation and routing system](https://medium.com/flutter/learning-flutters-new-navigation-and-routing-system-7c9068155ade)
- https://dart.dev/codelabs/async-await
- https://github.com/alibaba/flutter_boost
- Flutter笔记——FlutterEngine，简析其基础又重要的知识点：https://juejin.im/post/6844904117081473032
- SchedulerBinding Flutter运行机制-从启动到显示：https://book.flutterchina.club/chapter14/flutter_app_startup.html
- Flutter之使用overlay显示悬浮控件：https://juejin.im/post/6844903749534810119
- 插件开发流程：https://flutterchina.club/developing-packages/