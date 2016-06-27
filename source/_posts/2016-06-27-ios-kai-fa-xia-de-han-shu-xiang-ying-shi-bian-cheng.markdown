---
layout: post
title: "iOS开发下的函数响应式编程"
date: 2016-06-27 10:25:19 +0800
comments: true
categories: iOS RAC专栏
description: 随着移动互联网的蓬勃发展， iOS App 的复杂度呈指数增长。美团·大众点评两个App随着用户量不断增加、研发工程师数量不断增多，可用性的要求也随之不断提升。在这样的一个背景之下，我们面临了很多的问题和挑战。美团和大众点评的iOS工程师们面对挑战，想出了很多的策略和方针来应对，引入函数响应式编程就是美团App中重要的一环。
keywords: iOS ReactiveCocoa RAC 函数响应式编程
---
版权说明
-------------
本文为刊登于《程序员》杂志2016年5月刊。如需转载，请与《程序员》杂志联系。

### 背景和面临的问题

随着移动互联网的蓬勃发展，iOS App的复杂度呈指数增长。美团·大众点评两个App随着用户量不断增加、研发工程师数量不断增多，可用性的要求也随之不断提升。在这样的一个背景之下，我们面临了很多的问题和挑战。美团和大众点评的iOS工程师们面对挑战，想出了很多的策略和方针来应对，引入函数响应式编程就是美团App中重要的一环。

### 函数响应式编程简介

函数式编程想必您一定听过，但响应式编程的说法就不大常见了。与响应式编程对应的命令式编程就是大家所熟知的一种编程范式，我们先来看一段代码：

```
    int a = 3;
    int b = 4;
    int c = a + b;
    NSLog(@"c is %d", c); // => 12
    a = 5;
    b = 7;
    NSLog(@"c is %d", c); // 仍然是12
```

命令式编程就是通过表达式或语句来改变状态量，例如c = a + b就是一个表达式，它创建了一个名称为c的状态量，其值为a与b的加和。下面的a = 5是另一个语句，它改变了a的值，但这时c是没有变化的。所以命令式编程中c = a + b只是一个瞬时的过程，而不是一个关系描述。在传统的开发中，想让c跟随a和b的变化而变化是比较困难的。而让c的值实时等于a与b的加和的编程方式就是响应式编程。

实际上，在日常的开发中我们会经常使用响应式编程的思想来进行开发。最典型的例子就是Excel，当我们在一个B1单元格上书写一个公式“=A1+5”时，便声明了一种对应关系，每当A1单元格发生变化时，单元格B2都会随之改变。

![image](/images/iOSinMeituan/1.png)  
图1 Excel中的响应式

iOS开发中也有响应式编程的典型例子，例如Autolayout。我们通过设置约束描述了各个视图的位置关系，一旦其中一个变化，另一个就会响应其变化。类似的例子还有很多。

函数响应式编程（英文Functional Reactive Programming，以下简称FRP，）正是在函数式编程的基础之上，增加了响应式的支持。

简单来讲，FRP是基于异步事件流进行编程的一种编程范式。针对离散事件序列进行有效的封装，利用函数式编程的思想，满足响应式编程的需要。

区别于面向过程编程范式以过程单元作为核心组成部分，面向对象编程范式以对象单元作为核心组成部分，函数式编程范式以函数和高阶函数作为核心组成部分。FRP则以离散有序序列作为核心组成部分，也可将其定义为信号。其特点是具备可迭代特性并且允许离散事件节点有时间联系，计算机科学中称为Monad。

严格意义上来讲，下文提及的iOS开发下的函数响应式编程，并不能算完全的FRP，这一点，本文就不做学术上的讨论了。

接来下会为您介绍iOS相关的FRP内容，我们先从选型开始。

### iOS项目的函数响应式编程选型

很长一段时间以来，iOS项目并没有很好的FRP支持，直到iOS 4.0 SDK中增加了Block语法才为函数式编程提供了前置条件，FRP开源库也逐步健全起来。

最先与大家见面的莫过于[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)这样一个库了，ReactiveCocoa是Github在制作Github客户端时开源的一个副产物，缩写为RAC。它是Objective-C语言下FRP思想的一个优秀实例，后续版本也支持了Swift语言。

Swift语言的推出为iOS界的函数式编程爱好者迎来了曙光。著名的FRP开源库Rx系列也新增了[RxSwift](https://github.com/ReactiveX/RxSwift)，保持其接口与ReactiveX.net、RxJava、RxJS接口保持一致。

下面对不同厂商几个版本的FRP库进行简单的对比：

   _      | Objective-C 支持 | Swift 支持 | Cocoa框架支持 | 其他
:--------:|:---------------:|:-----------:|:-----------:|----------         
 RAC 2.5 | √				| ×			| 完善           | 迭代周期长，稳定
 RAC 3.0+| √				| √			| 继承2.5版本           | 开始全面支持Swift
 RxSwift | ×				| √			| 不完善           | 符合Rx标准
 
 表1 iOS下几种FRP库的对比

美团App由于历史原因仍然沿用ReactiveCocoa 2.5版本。下文也主要会针对ReactiveCocoa 2.5版本做介绍，但各位可以根据自己项目的需要来选择FRP库，其思想和主要的API大同小异。

为什么需要在iOS项目中引入FRP这样厚重的库呢？

iOS的项目主要以客户端项目为主，主要的业务场景就是进行页面交互和与服务器拉取数据，这里面会包含多种事件和异步处理逻辑。FRP本身就是面向事件流的一种开发方式，又擅长处理异步逻辑。所以从逻辑上是满足iOS客户端业务需要的。

然而能够把一个理念融合到实际的项目中，需要一个漫长的过程。所以接下来就根据美团App在FRP上的实践，具体讲述下融入FRP的过程。希望能给大家一些参考。

### 一步一步进行函数响应式编程

众所周知，FRP的学习是存在一定门槛的，想必这也是大家对FRP、ReactiveCocoa这些概念比较畏惧的主要原因。美团App在推行FRP的过程中，是采用分步骤的形式，逐步演化的。其演化的过程可以分为初探、入门、提高、进阶这样四个阶段。

#### 初探

美团App是在2014年5月第一次将ReactiveCocoa这个开源库纳入到工程中的，当时iOS工程师数量还不是很多，但是已经遇到了写法不统一、代码量膨胀等问题了。

写法不统一聚焦在回调形式的不统一上，iOS中的回调方式有非常多的种类：UIKit主要进行的事件处理target-action、跨类依赖推荐的delegate模式、iOS 4.0纳入的block、利用通知中心（Notifcation Center）进行松耦合的回调、利用键值观察（Key-Value Observe，简称KVO）进行的监听。由于场景不同，选用的规则也不尽相同，并且我们没有办法很好的界定什么场景该写什么样的回调。

这时我们发现ReactiveCocoa这样一个开源库，恰好能以统一的形式来解决跨类调用的问题，也包装了UIKit事件处理、Notifcation Center、KVO等常见的场景。使其代码风格上高度统一。

使用ReactiveCocoa进行统一后，上述的几种回调都可以写成如下形式：

```
// 代替target-action
    [[self.confirmButton rac_signalForControlEvents:UIControlEventTouchUpInside]
     subscribeNext:^(id x) {
        // 回调内容写在这里
     }];
    
// 代替delegate
    [[self.scrollView rac_signalForSelector:@selector(scrollViewDidScroll:) fromProtocol:@protocol(UIScrollViewDelegate)]
     subscribeNext:^(id x) {
        // 回调内容写在这里
     }];
    
// 代替block
    [[self asyncProcess]
     subscribeNext:^(id x) {
        // 回调内容写在这里
     } error:^(NSError *error) {
        // 错误处理写到这里
     }];
    
// 代替notification center
    [[[NSNotificationCenter defaultCenter] rac_valuesForKeyPath:@"Some-key" observer:nil]
     subscribeNext:^(id x) {
        // 回调内容写在这里
     }];

// 代替KVO
    [RACObserve(self, userReportString)
     subscribeNext:^(id x) {
        // 回调内容写在这里
     }];
```

代码1 回调统一

通过观察代码不难发现，ReactiveCocoa使得不同场景下的代码样式上高度统一，使我们在书写代码、维护代码、阅读代码方面的效率大大提高。

经过一定的研究，我们也发现使用`RAC(target, key)`宏可以更好组织代码形式，利用`filter:`和`map:`来代替原有的代码，达到更好复用，例如下面两段代码：

```
// 旧写法
    @weakify(self)
    [[self.textField rac_newTextChannel]
     subscribeNext:^(NSString *x) {
         @strongify(self)
         if (x.length > 15) {
             self.confirmButton.enabled = NO;
             [self showHud:@"Too long"];
         } else {
             self.confirmButton.enabled = YES;
         }
         self.someLabel.text = x;
         
     }];
    
// 新写法
    RACSignal *textValue = [self.textField rac_newTextChannel];
    RACSignal *isTooLong = [textValue
                            map:^id(NSString *value) {
                                return @(value.length > 15);
                            }];
    RACSignal *whenItsTooLongMessage = [[isTooLong
                                         filter:^BOOL(NSNumber *value) {
                                             return @(value.boolValue);
                                         }]
                                        mapReplace:@"Too long"];
    [self rac_liftSelector:@selector(showHud:) withSignals:whenItsTooLongMessage, nil];
    RAC(self.confirmButton, enabled) = [isTooLong not];
    RAC(self.someLabel, text) = textValue;
```

代码2 逻辑优化

上述代码修改虽然代码行数有一定的增加，但是结构更加清晰，复用性也做得更好。

综上所述，在这一阶段，我们主要以回调形式的统一为主，不断尝试合适的代码形式来表达绑定这种关系，也寻找一些便捷的小技巧来优化代码。

#### 入门

![image](/images/iOSinMeituan/2.png)  
图2 美团App首页


单纯解决回调风格的统一和树立绑定的思维是远远不够的，代码中更大的问题在于共享变量、异步协同以及异常传递的处理。
列举几个简单的场景，就拿美团App的首页来讲，我们可以看到上面包含很多的区块，而各个区块的访问接口不尽相同，但是渲染的逻辑却又多种多样：

* 有的需要几个接口都返回后才能统一渲染。
* 有的需要一个接口返回后，根据返回的内容决定后续的接口访问，最终才能渲染。
* 有的则是几个接口按照返回顺序依次渲染。

这就导致我们在处理这些逻辑的时候，需要很多的异步处理手段，利用一些中间变量来储存状态，每次返回的时候又判断这些状态决定渲染的逻辑。

更糟糕的是，有的时候对于同时请求多个网络接口，某些出现了网络错误，异常处理也变得越来越复杂。

随着对ReactiveCocoa理解的加深，我们意识到使用信号的组合等“高级”操作可以帮助我们解决很多的问题。例如`merge:`操作可以解决依次渲染的问题，`zip:`操作可以解决多个接口都返回后再渲染的问题，`flattenMap:`可以解决接口串联的问题。大概的示例代码如下：

```
// 依次渲染
    RACSignal *mergedSignal = [RACSignal merge:@[[self fetchData1],
                                                 [self fetchData2]]];
    
// 接口都返回后一起渲染
    RACSignal *zippedSignal = [RACSignal zip:@[[self fetchData1],
                                               [self fetchData3]]];
    
// 接口串联
    @weakify(self)
    RACSignal *finalSignal = [[self fetchData4]
                              flattenMap:^RACSignal *(NSString *data4Result) {
                                  @strongify(self)
                                  return [self fetchData5:data4Result];
                              }];
```

没有用到一个中间状态变量，我们通过这几个“魔法接口”神奇地将逻辑描述了出来。这样写的好处还有很多。

FRP具备这样一个特点，信号因为进行组合从而得到了一个数据链，而数据链的任一节点发出错误信号，都可以顺着这个链条最终交付给订阅者。这就正好解决了异常处理的问题。

![image](/images/iOSinMeituan/3.png)  
图3 错误传递链

由于此项特性，我们可以不再关注错误在哪个环节，只需等待订阅的时候统一处理即可。我们也找到了很多的方法用来更好地支持异常的处理。例如`try:`、`catch:`、`catchTo:`、`tryMap:`等。

简单列举下示例：

```
// 尝试判断并捕捉异常
    RACSignal *signalForSubscriber =
    
      [[[[[self fetchData1]
          try:^BOOL(NSString *data1,
                    NSError *__autoreleasing *errorPtr) {
              if (data1.length == 0) {
                  *errorPtr = [NSError new];
                  return YES;
              }
              return NO;
          }]
         flattenMap:^RACStream *(id value) {
             @strongify(self)
             return [self fetchData5:value];
         }]
        try:^BOOL(id value,
                  NSError *__autoreleasing *errorPtr) {
            if (![value isKindOfClass:[NSString class]]) {
                *errorPtr = [NSError new];
                return YES;
            }
            return NO;
        }]
       catch:^RACSignal *(NSError *error) {
           return [RACSignal return:error.domain];
       }];
```

总结一下，在这个阶段，我们主要尝试解决了异步协同的问题，包括了异常的处理。运用了异常处理模型来解决了很多的实际问题，同时继续寻找了更多的技巧来优化代码。

在初探和入门这两个阶段，美团App还只是谨慎地进行小的尝试，主旨是以代码简化为目的，使用ReactiveCocoa这个开源框架的一些便利功能来优化代码。在代码覆盖程度上尽量只在模块内部使用，避免跨层使用ReactiveCocoa。

#### 提高

随着对ReactiveCocoa这个开源框架的理解不断加深。美团App并不满足于简单的尝试，而是开始在更多的场景下使用ReactiveCocoa，并体现一定的FRP思想。这个阶段最具代表性的实践就是与MVVM架构的融合了，它就是体现了FRP响应式的思想。

Model-View-Controller（简称MVC）是苹果Cocoa框架默认的一个架构。实际上业务场景的复杂度越来越高，而MVC架构自身也存在分层不清晰等诸多问题，最终使得MVC这一架构在实际的使用中渐渐走了样。

Model-View-ViewModel（简称MVVM）便是近几年来十分推崇的一种架构，它解决了MVC架构的一些不足，在层次定义上更为清晰。在MVVM的架构中，最为关键的一环莫过于ViewModel层与View层的绑定了，我们的主角FRP恰好可以解决绑定问题，同时还能处理跨层错误处理的问题。

先来关注下绑定，自初探阶段开始，我们就开始使用`RAC(target, key)`这样的一个宏来表述绑定的关系，并且使用一些简单的信号转换使得原始信号满足视图渲染的需求。在引入MVVM架构后，我们将之前的经验利用起来，并且使用了RACChannel、RACCommand等组件来支持MVVM。

```
// 单向绑定
    RAC(self.someLabel, text) = RACObserve(self.viewModel, someProperty);
    RAC(self.scrollView, hidden) = self.viewModel.someSignal;
    RAC(self.confirmButton, frame) = [self.viewModel.someChannel
                                      map:^id(NSString *str) {
                                          CGRect rect = CGRectMake(0, 0, 0, str.length * 3);
                                          return [NSValue valueWithCGRect:rect];
                                      }];
    
// 双向绑定
    RACChannelTo(self.someLabel, text) = RACChannelTo(self.viewModel, someProperty);
    [self.textField.rac_newTextChannel subscribe:self.viewModel.someChannel];
    [self.viewModel.someChannel subscribe:self.textField.rac_newTextChannel];
    RACChannelTo(self, reviewID) = self.viewModel.someChannel;
    
// 命令绑定
    self.confirmButton.rac_command = self.viewModel.someCommand;
    
    RAC(self.textField, hidden) = self.viewModel.someCommand.executing;
    [self.viewModel.someCommand.errors
     subscribeNext:^(NSError *error) {
         // 错误处理在这里
     }];
```

绑定只是提供了上层的业务逻辑，更为重要的是，FRP的响应式范式恰如其分地体现在MVVM中。一个MVVM中View就会响应ViewModel的变化。我们来根据一副简单的图来分析一下：

![image](/images/iOSinMeituan/4.png)  
图4 MVVM示意图

上述简图列出了View-ViewModel-Model的大致关系，View和ViewModel间通过RACSignal来进行单向绑定，通过RACChannel来进行双向绑定，通过RACCommand进行执行过程的绑定。

ViewModel和Model间通过RACObserve进行监听，通过RACSignal进行回调处理，也可以直接调用方法。

Model有自身的数据业务逻辑，包含请求Web Service和进行本地持久化。

响应式的体现就在于View从一开始就是“声明”了与ViewModel间的关系，就如同A3单元格声明其“=A2+A1”一样。一旦后续数据发生变化，就按照之前的约定响应，彼此之间存在一层明确的定义。View在业务层面也得到了极大简化。

具体的数据流动就如同下图两种形式：

![image](/images/iOSinMeituan/5.png)  
![image](/images/iOSinMeituan/6.png)  
图5&图6 MVVM的数据流向示意

从两张图中可以看出，无论View收到用户修改TextField的文本框内容的事件，还是受到用户点击Button的事件。View层都不需要对此做特殊的逻辑处理，只是将之传递给ViewModel。而ViewModel自身维护逻辑，并体现在某些绑定关系上。这是与MVC中ViewController和Model的关系是截然不同的。FRP的响应式范式很好的帮助我们实现了此类需求。

之前虽然也提到过错误处理，但是也提到美团App在初探和入门阶段，只是小规模的在模块内使用，对外并不会以RACSignal的形式暴露。而这个阶段，我们也尝试了层级间通过RACSignal来进行信息的传递。这也自然可以应用上FRP异常处理的优势。

![image](/images/iOSinMeituan/7.png)  
图7 MVVM的数据流向示意

上图体现了一个按钮绑定了RACCommand收到错误后的一个数据流向。

除了MVVM框架的应用，这个阶段美团App也利用FRP解决另外的一个大问题。那就是多线程。

如果你做过异步拉取数据主线程渲染，那么你一定很熟悉子线程的回调结果转移到主线程的过程。这种操作不但繁琐，重复，关键还容易出错。

RAC提供了很多便于处理多线程的组件，核心类为RACScheduler，使得可以方便的通过`subscirbeOn:`方法、`deliverOn:`方法进行线程控制。

```
// 多线程控制
    RACScheduler *backgroundScheduler = [RACScheduler scheduler];
    RACSignal *testSignal = [[RACSignal
                             createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
                                 // 本例中，这段代码会确保运行在子线程
                                 [subscriber sendNext:@1];
                                 [subscriber sendCompleted];
                                 return nil;
                             }]
                             subscribeOn:backgroundScheduler];
    
    [[RACScheduler mainThreadScheduler]
     schedule:^{
        // 这段代码会在下次Runloop时在主线程中运行
         [[testSignal
          deliverOn:[RACScheduler mainThreadScheduler]]
            subscribeNext:^(id x) {
                // 虽然信号的发出者是在子线程中发出消息的
                // 但是接收者确可以确保在主线程接收消息
                
                // 主线程可以做你想做的渲染工作了！
            }
         ];
     }];
```

这一个阶段也算是大跃进的一个阶段，随着MVVM的框架融入，跨层的使用RAC使得代码整体使用FRP的比重大幅提高，全员对FRP的熟悉程度和思想的理解也变得深刻了许多。同时也真正使用了响应式的一些思想和特性来解决实际的问题。使其不再是纸上空谈。

我们美团App也在这个阶段挖掘了更多的RAC高级组件，继续代码优化的持续之路。

#### 进阶

美团App的iOS工程师们在大规模使用FRP后，也积蓄了很多的问题。很多小伙伴也问起了，既然是叫FRP，为什么一直体现的都是响应式的思想，对于函数式的思想应用体现似乎不是很明显。虽然FRP是F开头，称为函数响应式编程。但是考虑到函数式编程的复杂性，我们也将函数式编程的优化拿到了进阶这一阶段来尝试。

这一阶段面临的问题是RAC的大规模应用，使得代码中包含了大量的框架性质的代码。例如下面的代码：

```
// 冗余的代码
    [[self fetchData1]
     try:^BOOL(id value,
               NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSString class]]) {
             return YES;
         }
         *errorPtr = [NSError new];
         return NO;
     }];
    
    [[self fetchData2]
     tryMap:^id(NSDictionary *value, NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSDictionary class]]) {
             if (value[@"someKey"]) {
                 return value[@"someKey"];
             }
             // 并没有一个名为“someKey”的key
             *errorPtr = [NSError new];
             return nil;
         }
         // 这不是一个Dictionary
         *errorPtr = [NSError new];
         return nil;
     }];
    
    [[self fetchData3]
     tryMap:^id(NSDictionary *value, NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSDictionary class]]) {
             if (value[@"someOtherKey"]) {
                 return value[@"someOtherKey"];
             }
             // 并没有一个名为“someOtherKey”的key
             *errorPtr = [NSError new];
             return nil;
         }
         // 这不是一个Dictionary
         *errorPtr = [NSError new];
         return nil;
     }];
```

上述的几个代码段，我们可以看到功能非常近似，内容稍有不同的部分重复出现，很多的同学在实际的开发中也并没有太好地优化它们，甚至很多的同学表示束手无策。这时候函数式编程就可以派上用场了。

函数式编程是一种良好的编程范式，我们在这里主要利用它的几个特点：高阶函数、不变量和迭代。

先来看高阶函数，高阶函数是入参是函数或者返回值是函数的函数。说起来虽然有些拗口，但实际上在iOS开发中司空见惯，例如典型的订阅其实就是一个高阶函数的体现。

```
// 高阶函数
    [[self fetchData1]
     subscribeNext:^(id x) {
        // 这是一个block，作为一个参数
     }];
```

我们更关心的是返回值是函数的函数，这是上面冗长的代码解决之道。代码7的代码中会发现一些相同的逻辑，例如类型判断。我们就可以先做一个这样的小函数：

```
typedef BOOL (^VerifyFunction)(id value);

VerifyFunction isKindOf(Class aClass)
{
    return ^BOOL(id value) {
        return [value isKindOfClass:aClass];
    };
}
```

瞧，很简单对不对！只要把一个类型传进去，就会得到一个用来判断某个对象是否是这个类型的函数。细心的读者会发现我们实际要的是一个入参为对象和一个NSError对象指针的指针类型，返回值是布尔类型的block，但是这个只能返回入参是对象的，显然不满足条件。很多人第一个想到的就是把这个函数改成返回参数为两个参数返回值为布尔类型的block，但是函数式的解决方法是新增一个这样的函数：

```
typedef BOOL (^VerifyAndOutputErrorFunction)(id value, NSError **error);

VerifyAndOutputErrorFunction verifyAndOutputError(VerifyFunction verify,
                                                  NSError *outputError)
{
    return ^BOOL(id value, NSError **error) {
        if (verify(value)) {
            return YES;
        }
        *error = outputError;
        return NO;
    };
}

```

一个典型的高阶函数，入参带有一个block，返回值也是一个block，组合起来就可以把刚才的几个`try:`代码段优化。可能你会问，为什么要搞成两个呢，一个不是更好？搞成两个的好处就在于，我们可以将任意的VerifyFunction类型的block与一个outputError相结合，来返回一个我们想要的VerifyAndOutputErrorFunction类型block，例如增加一个判断NSDictionary是否包含某个Key的VerifyFunction。下面给出一个优化后的代码，大家可以仔细思考下：


```
// 可以高度复用的函数
typedef BOOL (^VerifyFunction)(id value);

VerifyFunction isKindOf(Class aClass)
{
    return ^BOOL(id value) {
        return [value isKindOfClass:aClass];
    };
}

VerifyFunction hasKey(NSString *key)
{
    return ^BOOL(NSDictionary *value) {
        return value[key] != nil;
    };
}

typedef BOOL (^VerifyAndOutputErrorFunction)(id value, NSError **error);

VerifyAndOutputErrorFunction verifyAndOutputError(VerifyFunction verify,
                                                  NSError *outputError)
{
    return ^BOOL(id value, NSError **error) {
        if (verify(value)) {
            return YES;
        }
        *error = outputError;
        return NO;
    };
}

typedef id (^MapFunction)(id value);

MapFunction dictionaryValueByKey(NSString *key)
{
    return ^id(NSDictionary *value) {
        return value[key];
    };
}

// 与本例关联比较大的函数
typedef id (^MapAndOutputErrorFunction)(id value, NSError **error);
MapAndOutputErrorFunction transferToKeyChild(NSString *key)
{
    return ^id(id value, NSError **error) {
        if (hasKey(key)(value)) {
            return dictionaryValueByKey(key)(value);
        } else {
            *error = [NSError new];
            return nil;
        }
    };
};

- (void)oldStyle
{
    // 冗余的代码
    [[self fetchData1]
     try:^BOOL(id value,
               NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSString class]]) {
             return YES;
         }
         *errorPtr = [NSError new];
         return NO;
     }];
    
    [[self fetchData2]
     tryMap:^id(NSDictionary *value, NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSDictionary class]]) {
             if (value[@"someKey"]) {
                 return value[@"someKey"];
             }
             // 并没有一个名为“someKey”的key
             *errorPtr = [NSError new];
             return nil;
         }
         // 这不是一个Dictionary
         *errorPtr = [NSError new];
         return nil;
     }];
    
    [[self fetchData3]
     tryMap:^id(NSDictionary *value, NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSDictionary class]]) {
             if (value[@"someOtherKey"]) {
                 return value[@"someOtherKey"];
             }
             // 并没有一个名为“someOtherKey”的key
             *errorPtr = [NSError new];
             return nil;
         }
         // 这不是一个Dictionary
         *errorPtr = [NSError new];
         return nil;
     }];
}

- (void)newStyle
{
    
    VerifyAndOutputErrorFunction isDictionary = 
      verifyAndOutputError(isKindOf([NSDictionary class]),
       	                NSError.new);
    VerifyAndOutputErrorFunction isString =      
      verifyAndOutputError(isKindOf([NSString class]),
             	           NSError.new);
 
    [[self fetchData1]
     try:isString];
    
    [[[self fetchData2]
      try:isDictionary]
      tryMap:transferToKeyChild(@"someKey")];
    
    [[[self fetchData3]
      try:isDictionary]
     tryMap:transferToKeyChild(@"someOtherKey")];
}
```

虽然代码有些多，但是从newStyle函数的结果来看，我们在实际的业务代码上非常的简洁，而且还抽离出很多可复用的小函数。在实际的业务中，我们甚至通过这种范式在某些业务场景简化了超过50%的代码量。

除此之外，我们还尝试用迭代来进一步减少临时变量。为什么要减少临时变量呢？因为我们想要遵循不变量原则，这是函数式编程的一个特点。试想下如果我们都是使用一些不变量，就不再会有那么多异步锁和痛苦的多线程问题了。基于以上考虑，我们要求工程师尽量在开发的过程中减少使用变量，从而锻炼用更加函数式的方式来解决问题。

例如下面的简单问题，实现一个每秒发送值为0 1 2 3 … 100的递增整数信号，实现的方法可以是这样：

```
- (void)countSignalOldStyle
{
    RACSignal *signal =
    [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        RACDisposable *disposable = [RACDisposable new];
        __block int i = 0;
        __block void (^recursion)();
        recursion = ^{
            if (disposable.disposed || i > 100) {
                return;
            }
            [subscriber sendNext:@(i)];
            ++i;
            [[RACScheduler mainThreadScheduler]
             afterDelay:1 schedule:recursion];
        };
        recursion();
        return disposable;
    }];
}
```

这样的代码不但用了block自递归，还用了一个闭包的i变量。i变量也在数次递归中进行了修改。代码不易理解且block自递归会存在循环引用。使用迭代和不变量的形式是这样的：

```
- (void)countSignalNewStyle
{
    RACSignal *signal =
    [[[[[[RACSignal return:@1]
         repeat] take: 100]
       aggregateWithStart:@0 reduce:^id(NSNumber *running,
                                        NSNumber *next) {
           return @(running.integerValue + next.integerValue);
       }]
      map:^id(NSNumber *value) {
          return [[RACSignal return:value]
                  delay:1];
      }]
     concat];
}
```

解法是这样的，先用固定返回1的信号生成一个无限重复信号，取前100个值，然后用迭代方法，产生一个递增的迭代，再将发送的密集的递增信号转成一个延时1秒的子信号，最后将子信号进行连接。感兴趣的同学可以自己动手尝试下，也希望大家都去思考不适用变量来解决问题的思路。

这些函数式的写法不仅解决了业务上的问题，也给我们美团App的iOS工程师们开拓了代码优化的新思路。

可以看到，到这一阶段，需要对FRP的理解要求更高。为了追求更好的代码体验，我们朝着FRP的道路又迈进了许多，走到这一步是每一个美团App的iOS工程师共同努力的结果。这是一个尚未完结的阶段，我们的工程师仍然在不选找寻更好的FRP范式。对于开发人员来说，优化之路永远不会停步。

### 总结

单纯靠这样一篇文章来介绍全部的FRP思想是不可能的，这也仅是起到了抛砖引玉的作用。FRP不仅可以解决项目中实际遇到的很多问题，也能锻炼更好的工程师素养。希望大家能够掌握起来，用FRP的思想来解决更多实际的问题。社区和开源库也需要大家的不断投入。谢谢大家！

