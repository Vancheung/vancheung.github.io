---
title: 【Day 0】关于我是如何干掉Appium和RobotFramework这件事的——开篇
date: 2021-06-10 14:00:00 +/-TTTT
categories: [客户端自动化,框架设计]
tags: [python,自动化,iOS,Android,Appium]
typora-copy-images-to: ../assets/img
typora-root-url: ../../vancheung.github.io
---



​	起了个比较大的标题，我终于也变成了自己所不齿的那种标题党。还是解释一下，最近接手了一个项目，需要优化客户端自动化测试框架，而当前的自动化测试框架是基于Appium和RobotFramework。所以严谨一点来说，是在此记录一下，如何从现有自动化工程中解耦Appium和RobotFramework。毕竟它们经历了这么多年的发展，远非我一人的智慧能及，新的设计并不能取代这两个框架的地位。
​	本人才疏学浅，文中难免存在浅显错漏的认知和大量主观臆断，欢迎交流，批评指正。

### **为什么要做这个？**

​	首先给不熟悉背景的同学提供一点上下文，简单介绍一下，Appium和RobotFramework是什么。

#### Appium是什么？

​	首先看下 [Appium](https://appium.io/docs/cn/about-appium/intro/) 官网的介绍：

>   Appium 是一个开源工具，用于自动化 iOS 手机、 Android 手机和 Windows 桌面平台上的原生、移动 Web 和混合应用。Appium 是跨平台的：它允许你用同样的 API 对多平台（iOS、Android、Windows）写测试。做到在 iOS、Android 和 Windows 测试套件之间复用代码。了解 Appium “支持”这些平台意味着什么、有哪些自动化方式的详细信息，请参见 [Appium 支持的平台](https://appium.io/docs/cn/about-appium/platform-support/index.html)。

​	那Appium是如何运行的呢？同样摘抄一段

>   Appium 的核心一个是暴露 REST API 的 WEB 服务器。它接受来自客户端的连接，监听命令并在移动设备上执行，答复 HTTP 响应来描述执行结果。

>   **Appium 服务器** 是一个用 Node.js 写的服务器。可以从[源码](https://github.com/appium/appium/blob/master/docs/cn/contributing-to-appium/appium-from-source.md)构建安装或者从 [NPM](https://www.npmjs.com/package/appium) 直接安装。

>   **Appium 客户端**

>   有一些客户端程序库（分别在 Java、Ruby、Python、PHP、JavaScript 和 C# 中实现），它们支持 Appium 对 WebDriver 协议的扩展。你需要用这些客户端程序库代替常规的 WebDriver 客户端。

​	简单解释一下，首先，Appium的核心是一个Node.js写的Web服务器，在通过npm构建并启动后Appium服务器后，Appium服务器底层会通过uiautomator、XCUITest等方式与被测设备通信，而上层可以通过任意一个可以发送HTTP消息的客户端与Appium服务器进行通信。原则上来说Appium Client可以是任意一个语言，只要遵循WebDriver协议即可。而通常对于大部分测试开发团队来说，这个选择是Python。

![UML 图 (14)](/assets/img/UML 图 (14)-3308096.png)

#### RobotFramework是什么？

​	同样是一段来自 [RobotFramework](https://robotframework.org/) 官网的介绍：

>   **Robot Framework**是一个通用的开源自动化框架。它可用于测试自动化和机器人过程自动化 (RPA)。

>   Robot Framework 具有简单的语法，使用人类可读的关键字。它的功能可以通过用 Python 或 Java 实现的库来扩展。该框架有一个丰富的生态系统，由作为独立项目开发的[库](https://robotframework.org/#libraries)和[工具](https://robotframework.org/#tools)组成。

>   Robot Framework 独立于操作系统和应用程序。核心框架是使用[Python](http://python.org/)实现的，也可以在[Jython](http://jython.org/) (JVM) 和[IronPython](http://ironpython.net/) (.NET) 上运行。

​	提取一下关键词：`开源`、`免费`、`可扩展`、`Python写的`、`使用人类可读的关键字`。RobotFramework被推荐的原因通常都来源于最后一条：“人类可读的关键字”，简单翻译一下就是“给那些不想写（不会写）代码的业务测试同学提供一个易上手的方式，不需要学编程（手动加粗）就可以做自动化。

​	回到最初的问题，为什么要把它们从现有的框架中干掉？

​	先说RobotFramework 关键字这个设计，单拎出来这个逻辑看似很合理，给不懂代码的业务同学写自动化用例的嘛，但是仔细琢磨一下这个逻辑，其实有点鸡肋。首先，对于不会写代码的业务测试同学来说，RobotFramework搭建、运行的过程中存在大量的坑，我都费这么大劲把环境搭起来，还专门学一下关键字是怎么封装的、用例是怎么编写的，用这个时间去学一下Python基础不香吗？而对于有编程基础的测试开发同学来说，我都会Python了为啥还要再多学一门框架？并且，用RF需要新引入语法规范，Keyword之间不支持跳转，还不能断点调试。工作不饱和吗，要把时间用在这个上面？

​	再说说Appium，跨平台是appium最大的优势，但弊端就是Appium Server的引入增加了一层复杂度。众所周知，如果一个操作有可能出错，即使是拨一下开关这样简单的操作，在足够多的操作次数后也一定会出错。由于Node.js不稳定产生的各种部署、运行问题，实际上是与用例无关的，这些问题可能会导致误报率急速上升，而问题太多就等于没有问题，毕竟项目中能有多少人力用于分析Fail Case？“手工没问题就先上，不管自动化结果了。”想必不少测试同学都经历过这个场景。

#### 自动化测试的初心

​	去璞是为了求真，回到自动化最初的目的：保质、提效。我一直赞同以下观点：

-   质量不是测出来的
-   自动化是手段，不是目的

详细解读一下：

（1）第一个观点乍一看，否定了测试存在的意义。毕竟“质量不是测出来的”，是产品的“固有属性”，与“观测手段”无关。假设开发团队拥有足够的单元测试和白盒测试能力，原则上不需要测试和质量保障团队。而如果开发能力不足，那产品质量烂，有没有测试都烂。

​	既然如此，为什么有一点规模的产品都要上测试呢？软件和互联网、移动互联网快速发展，业务上的高速迭代引入了大量的技术债务，这些没时间做单测或者架构设计难以进行单测的产品，只能通过简单朴素的方式：“堆人力”，来保障线上少出一点弱智问题。引用一个网友的绝妙比喻：“外面都是跑车壳子，里面是几个人骑自行车举着走”。

（2）第二个观点存在的逻辑也很合理，随着业务发展，场景增多，复杂性变高，产品质量下降，于是需要更多测试人员来保障质量，而测试人员规模达到一定数量后，就会明显地呈现出一种边际效益递减状态：堆砌人力，整体质量并没有变好，需要投入人力去培训新人和团队沟通，整体人效反而降低了。

​	质量的边界取决于测试设计人员脑力的边界和测试执行人员能力的边界，既然招人无法解决问题，那就用机器来解决，形形色色的自动化手段开始引入测试流程中。

​	小公司一看，得，大公司都这么搞，我们也有样学样吧，于是开始招测试开发，也来搞自动化吧。人员招进来总要做点什么，总不能白拿工资，一串流程下来，大家都陷入了“为了做自动化而做自动化”的误区，没人关心一开始为什么要做自动化。

​	回到这句话，“自动化是手段，不是目的”，提高人效、提升质量才是目的。

### 怎么做？

#### 奥卡姆剃刀原则

>   *Numquam ponenda est pluralitas sine necessitate.*

>   *如无必要，勿增实体。*

​	当两条路都能走通的时候，选择简单的那个。生搬硬套一下，就是发挥python自身的灵活性，重新封装一个纯python的自动化框架，不再依赖RobotFramework和Appium，在测试框架部分使用Pytest+Allure，而在与移动端交互的部分选用支持纯Python的库，例如uiautomator2、python-wda等。

#### 不同对象的需求

​	在做选型和设计时，考虑到不同人员对于测试框架的核心诉求是不一样的，在这里做出一个简单的划分。这个划分更多的是建立在逻辑而非物理意义上，毕竟对于小规模的产品来说，三类参与者可能是同一个人，而对大公司，三者有可能是分属不同业务线、产品线的团队。

![UML 图 (15)](/assets/img/UML 图 (15)-3308336.png)

#### 代码架构

​	基于以上原则，搭建一个分层的代码架构，满足以下标准

-   业务操作与UI底层解耦

-   测试用例编写与业务操作解耦

-   数据与逻辑解耦

    基于这些标准设计的架构0.0.1版本如下

![image-20210610150019463](/assets/img/image-20210610150019463.png)

代码见：

https://github.com/Vancheung/ClientEngine/commit/1edaab037f3f813af36d6f9a1682f456c9968460

### 总结

​	与其说是重构，更像是重写，从0.0.1开始设计一个新的自动化框架，我不知道这个框架会走向何方，也不知道前面还有多少坑在等着我。毕竟架构这个东西，从写下第一行代码开始就会不可避免地走向腐化。选择使用类似日记的形式来记录这个过程，也是为了留下一些记录可以供踩坑的同学参考，前人之鉴后人之师，毕竟，对与错都是一种进步。
