---
layout: post
title: "聊一聊RAC"
date: 2014-08-20 11:17:32 +0800
comments: true
categories: iOS RAC专栏
description: 来讲一讲RAC
keywords: iOS ReactiveCocoa RAC 函数响应式编程

---

今天来聊一聊RAC，这个在Github上很火热的开源框架，相信很多关注iOS前沿开发的人都或多或少的知道。

关于RAC的介绍和一些概念，我这里就不再啰嗦了，大家可以看	[limboy](http://limboy.me/about.html)的几篇关于RAC的介绍，很不错。

> [ReactiveCocoa与Functional Reactive Programming](http://limboy.me/ios/2013/06/19/frp-reactivecocoa.html)  
> [说说ReactiveCocoa 2](http://limboy.me/ios/2013/12/27/reactivecocoa-2.html)  
> [基于AFNetworking2.0和ReactiveCocoa2.1的iOS REST Client](http://limboy.me/ios/2014/01/05/ios-rest-client-implementation.html)  
> [ReactiveCocoa2实战](http://limboy.me/tech/2014/06/06/deep-into-reactivecocoa2.html)

此外还有Cocoachina的一系列教程[Reactive Cocoa详解](http://www.cocoachina.com/cms/plus/view.php?aid=8905)

写Blog，在我看来和开发一样，也要讲究重用，引用列位的信息即可，何必黏贴和表达类似的观点。这里我也仅说一下我自己的理解和看法。

相信很多人都尝试过RAC用到一些小的DEMO或者项目中的一个小部分然后就浅尝辄止了。为什么呢？我觉得主要在于，RAC实际推行的一种新的概念大家还没有习惯，那就是函数响应式编程FRP。[staltz](https://github.com/staltz)在一个Gist上倒是给了一个比较全面的解释[The introduction to Reactive Programming you've been missing](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)，可惜是英文的。

这种FRP的核心就在于数据流，FRP把整个的程序认为是数据流的转换，既不是过程、也不是对象。一个系统先产生信号，转换信号，然后业务和界面再接收和响应信号的变化。所以，这是一个新的概念。如果使用别的概念套用，自然用起来就没那么顺手了。那么我们来看看，这样做的好处是什么呢？

* 统一的流处理模式，使得不同的组件可以很好的结合起来，例如iOS的`UserDefault`、`NSNotificationCenter`、`KVO`这些，在RAC库下面，都是相同的封装，这样就使得上层的业务逻辑实现了**大同**，进而一切的信号转换合并都可以有效的结合在一起。
* 处理异步，很多时候，我们对于异步再同步是比较头大的。而RAC中，一个信号的终止，是不局限在一个函数中的。这样我们可以把不同线程、不同时期的状态绑到一个信号上，使得使用者达到一种内聚。和这种内聚，在转换和迭代的过程中是很必要的。
* 统一的错误处理，从古老的C时代的`int DoSomeThine(int input1, int *output1)`这种以返回值返回错误，到后来`SetErrorStatus(int ErrorCode, const char *message)`的线程栈内全局报错机制，还有现在try-catch机制，都有一个很要命的问题，就是错误处理，或者是可以被忽略，或者是让开发变得很烦恼。Java的try-catch机制，相信Java的开发者们一定深有感触。而RAC把错误变得简单了，它对于错误的处理，会随着变化一起传递到顶层，既不会忘记，也不用在中间环节中手动传递来传递去。
* 逻辑的拆分，在FRP中，逻辑变得相对独立，通常是一个模块，根据一定的变化产生一个信号，亦或是一个模块，根据一个传入的信号，产生一定的转换。这就使得，我们可以只返回我们的直接结果，后期的加工和变更是分离的。对于上层模块，也只关注信号的类型，不关注处于那个线程还是何种手法。

总而言之，RAC给与我们以数据的变化作为出发点，界面与之响应的一套框架。详细的一些技巧，我会在后续的blog中为大家慢慢介绍。