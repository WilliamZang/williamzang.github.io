---
layout: post
title: "细说ReactiveCocoa的冷信号与热信号"
date: 2015-08-18 12:21:28 +0800
comments: true
categories: iOS RAC专栏
description: ReactiveCocoa（简称RAC）是一套基于Cocoa的FRP框架，在我们美团客户端中，我们大量使用了这个框架。而在使用的过程中我们发现，冷信号与热信号的概念很容易混淆并且容易造成一定的问题，相信各位在使用的过程中也可能遇到此类问题。所以我在这里与大家讨论下RAC中冷信号与热信号的相关知识点，希望可以加深大家对冷热信号的理解。先介绍下背景和什么是冷信号与热信号。
keywords: iOS ReactiveCocoa RAC 函数响应式编程 hot cold signal
---

版权说明
-------------
本文为 [美团点评技术团队博客](http://tech.meituan.com/) 特供稿件，[首发地址在此](http://tech.meituan.com/talk-about-reactivecocoas-cold-signal-and-hot-signal-part-1.html)。如需转载，请与 美团点评技术团队博客 联系。

## 背景

[ReactiveCocoa](http://www.github.com/ReactiveCocoa/ReactiveCocoa)（简称RAC）是一套基于Cocoa的FRP框架，在我们美团客户端中，我们大量使用了这个框架。而在使用的过程中我们发现，冷信号与热信号的概念很容易混淆并且容易造成一定的问题，相信各位在使用的过程中也可能遇到此类问题。所以我在这里与大家讨论下RAC中冷信号与热信号的相关知识点，希望可以加深大家对冷热信号的理解。

p.s. 以下代码和示例基于[ReactiveCocoa v2.5](https://github.com/ReactiveCocoa/ReactiveCocoa/releases/tag/v2.5)

## 什么是冷信号与热信号

冷热信号的概念源于C#的MVVM框架[Reactive Extensions](https://msdn.microsoft.com/en-us/library/hh242985.aspx)中的Hot Observables和Cold Observables:

> Hot Observables和Cold Observables的区别：
> 
> 1. Hot Observables是主动的，尽管你并没有订阅事件，但是它会时刻推送，就像鼠标移动；而Cold Observables是被动的，只有当你订阅的时候，它才会发布消息。
> 
> 2. Hot Observables可以有多个订阅者，是一对多，集合可以与订阅者共享信息；而Cold Observables只能一对一，当有不同的订阅者，消息是重新完整发送。

这里面的Observables可以理解为RACSignal。为了加深理解，请大家关注这样的几组代码：

```objective-c
    RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:@1];
        [subscriber sendNext:@2];
        [subscriber sendNext:@3];
        [subscriber sendCompleted];
        return nil;
    }];
    NSLog(@"Signal was created.");
    [[RACScheduler mainThreadScheduler] afterDelay:0.1 schedule:^{
        [signal subscribeNext:^(id x) {
            NSLog(@"Subscriber 1 recveive: %@", x);
        }];
    }];
    
    [[RACScheduler mainThreadScheduler] afterDelay:1 schedule:^{
        [signal subscribeNext:^(id x) {
            NSLog(@"Subscriber 2 recveive: %@", x);
        }];
    }];    
```

以上简单的创建了一个信号，并且依次发送@1，@2，@3作为值。下面分别有两个订阅者在不同的时间段进行了订阅，运行的结果如下：

```
2015-08-11 18:33:21.681 RACDemos[6505:1125196] Signal was created.
2015-08-11 18:33:21.793 RACDemos[6505:1125196] Subscriber 1 recveive: 1
2015-08-11 18:33:21.793 RACDemos[6505:1125196] Subscriber 1 recveive: 2
2015-08-11 18:33:21.793 RACDemos[6505:1125196] Subscriber 1 recveive: 3
2015-08-11 18:33:22.683 RACDemos[6505:1125196] Subscriber 2 recveive: 1
2015-08-11 18:33:22.683 RACDemos[6505:1125196] Subscriber 2 recveive: 2
2015-08-11 18:33:22.683 RACDemos[6505:1125196] Subscriber 2 recveive: 3

```

我们可以看到，信号在18:33:21.681时被创建，18:33:21.793依次接到1、2、3三个值，而在18:33:22.683再依次接到1、2、3三个值。说明了变量名为`signal`的这个信号，在两个不同时间段的订阅过程中，分别完整的发送了所有的消息。

我们再对这段代码进行一个小的改动：

```objective-c
    RACMulticastConnection *connection = [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [[RACScheduler mainThreadScheduler] afterDelay:1 schedule:^{
            [subscriber sendNext:@1];
        }];

        [[RACScheduler mainThreadScheduler] afterDelay:2 schedule:^{
            [subscriber sendNext:@2];
        }];
        
        [[RACScheduler mainThreadScheduler] afterDelay:3 schedule:^{
            [subscriber sendNext:@3];
        }];
        
        [[RACScheduler mainThreadScheduler] afterDelay:4 schedule:^{
            [subscriber sendCompleted];
        }];
        return nil;
    }] publish];
    [connection connect];
    RACSignal *signal = connection.signal;
    
    NSLog(@"Signal was created.");
    [[RACScheduler mainThreadScheduler] afterDelay:1.1 schedule:^{
        [signal subscribeNext:^(id x) {
            NSLog(@"Subscriber 1 recveive: %@", x);
        }];
    }];
    
    [[RACScheduler mainThreadScheduler] afterDelay:2.1 schedule:^{
        [signal subscribeNext:^(id x) {
            NSLog(@"Subscriber 2 recveive: %@", x);
        }];
    }];
```

稍微有些复杂，我们来一一分析下：

* 创建了一个信号，在1秒、2秒、3秒分别发送1、2、3这三个值，4秒发送结束信号。
* 对这个信号调用publish方法得到一个RACMulticastConnection。
* 将connection进行连接操作。
* 获得connection的信号。
* 分别在0.1秒和2秒订阅获得的信号。

抛开RACMulticastConnection是个什么东东，我们先来看下结果：

```
2015-08-12 11:07:49.943 RACDemos[9418:1186344] Signal was created.
2015-08-12 11:07:52.088 RACDemos[9418:1186344] Subscriber 1 recveive: 2
2015-08-12 11:07:53.044 RACDemos[9418:1186344] Subscriber 1 recveive: 3
2015-08-12 11:07:53.044 RACDemos[9418:1186344] Subscriber 2 recveive: 3
```

首先告诉大家`-[RACSignal publish]`、`- [RACMulticastConnection connect]`、`- [RACMulticastConnection signal]`这几个操作生成了一个热信号。
我们再来关注下输出结果的一些细节：

* 信号在11:07:49.943被创建
* 11:07:52.088时订阅者1才收到2这个值，说明1这个值没有接收到，时间间隔是2秒多
* 11:07:53.044时订阅者1和订阅者2同时收到3这个值，时间间隔是3秒多

参考一开始的Hot Observables的论述和两段小程序的输出结果，我们可以确定冷热信号的如下特点：

* 一、热信号是主动的，即使你没有订阅事件，它仍然会时刻推送。（如第二个例子，信号在50秒被创建，51秒的时候1这个值就推送出来了，但是当时还没有订阅者。）而冷信号是被动的，只有当你订阅的时候，它才会发送消息。（如第一个例子。）
* 二、热信号可以有多个订阅者，是一对多，信号可以与订阅者共享信息（如第二个例子，订阅者1和订阅者2是共享的，他们都能在同一时间接收到3这个值。）而冷信号只能一对一，当有不同的订阅者，消息会从新完整发送。（如第一个例子，我们可以观察到两个订阅者没有联系，都是基于各自的订阅时间开始接收消息的。）

## 为什么要区分冷信号与热信号

也许你看到这里并且看到这一章节的标题就会有疑问，为什么RAC要搞如此复杂的一个概念，直接搞成一种信号不就好了么？要解释这个问题需要绕一些弯路。（前方可能比较难懂，如果不能很好理解，请自行查阅各类文档。）

最前面提到了RAC是一套基于Cocoa的FRP框架，那就来说说FRP，FRP全写是Functional Reactive Programming，中文译作函数响应式编程，是RP（Reactive Programm，响应式编程）的FP（Functional Programming，函数式编程）实现。说起来很拗口。太多的细节不多讨论，我们先关注下它是FP的情况。

FP有几个很重要的概念是和我们的主题相关的：

[纯函数](https://en.wikipedia.org/wiki/Functional_programming#Pure_functions)是指一个函数或者一个表达式不存在任何的[副作用](https://en.wikipedia.org/wiki/Side_effect_\(computer_science\))，就如同数学中的函数：

> f(x) = 5x + 1 

这个函数在调用的过程中产生除了返回值以外的任何作用，也不受任何外界因素的影响。那么副作用都有哪些呢？我来列举以下几个情况：

* 函数的处理过程中，修改了外部的变量，例如全局变量。一个特殊点的例子，就是如果把OC的一个方法看做一个函数，所有的成员变量的赋值都是对外部变量的修改。是的，从FP的角度看OOP是充满副作用的。
* 函数的处理过程中，触发了一些额外的动作，例如发送的全局的一个Notification，在console里面输出的结果，保存了文件，触发了网络，更新的屏幕等。
* 函数的处理过程中，受到外部变量的影响，例如全局变量，方法里面用到的成员变量。注意block中捕获的外部变量也算副作用。
* 函数的处理过程中，受到线程锁的影响算副作用。

由此我们可以看出，在目前的iOS编程中，我们是很难的摆脱副作用的。或者换一种说法，我们iOS编程的目的其实是副作用。（基于用户触摸的外界因素，最终反馈到网络变化和屏幕变化上。）

接下来我们来分析下副作用与冷热信号的关系。既然iOS编程中少不了副作用，那么RAC在实际的使用中也不可避免的接触副作用，下面我列举个业务场景，来看下冷信号中副作用的坑：

```objective-c

    self.sessionManager = [[AFHTTPSessionManager alloc] initWithBaseURL:[NSURL URLWithString:@"http://api.xxxx.com"]];
    
    self.sessionManager.requestSerializer = [AFJSONRequestSerializer serializer];
    self.sessionManager.responseSerializer = [AFJSONResponseSerializer serializer];

    @weakify(self)
    RACSignal *fetchData = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        @strongify(self)
        NSURLSessionDataTask *task = [self.sessionManager GET:@"fetchData" parameters:@{@"someParameter": @"someValue"} success:^(NSURLSessionDataTask *task, id responseObject) {
            [subscriber sendNext:responseObject];
            [subscriber sendCompleted];
        } failure:^(NSURLSessionDataTask *task, NSError *error) {
            [subscriber sendError:error];
        }];
        return [RACDisposable disposableWithBlock:^{
            if (task.state != NSURLSessionTaskStateCompleted) {
                [task cancel];
            }
        }];
    }];
    
    RACSignal *title = [fetchData flattenMap:^RACSignal *(NSDictionary *value) {
        if ([value[@"title"] isKindOfClass:[NSString class]]) {
            return [RACSignal return:value[@"title"]];
        } else {
            return [RACSignal error:[NSError errorWithDomain:@"some error" code:400 userInfo:@{@"originData": value}]];
        }
    }];
    
    RACSignal *desc = [fetchData flattenMap:^RACSignal *(NSDictionary *value) {
        if ([value[@"desc"] isKindOfClass:[NSString class]]) {
            return [RACSignal return:value[@"desc"]];
        } else {
            return [RACSignal error:[NSError errorWithDomain:@"some error" code:400 userInfo:@{@"originData": value}]];
        }
    }];
    
    RACSignal *renderedDesc = [desc flattenMap:^RACStream *(NSString *value) {
        NSError *error = nil;
        RenderManager *renderManager = [[RenderManager alloc] init];
        NSAttributedString *rendered = [renderManager renderText:value error:&error];
        if (error) {
            return [RACSignal error:error];
        } else {
            return [RACSignal return:rendered];
        }
    }];
    
    RAC(self.someLablel, text) = [[title catchTo:[RACSignal return:@"Error"]]  startWith:@"Loading..."];
    RAC(self.originTextView, text) = [[desc catchTo:[RACSignal return:@"Error"]] startWith:@"Loading..."];
    RAC(self.renderedTextView, attributedText) = [[renderedDesc catchTo:[RACSignal return:[[NSAttributedString alloc] initWithString:@"Error"]]] startWith:[[NSAttributedString alloc] initWithString:@"Loading..."]];
    
    [[RACSignal merge:@[title, desc, renderedDesc]] subscribeError:^(NSError *error) {
        UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"Error" message:error.domain delegate:nil cancelButtonTitle:@"OK" otherButtonTitles:nil];
        [alertView show];
    }];
```

不晓得大家有没有被这么一大段的代码吓到，我想要表达的是，在真正的工程中，我们的业务逻辑是很复杂的，而一些坑就隐藏在如此看似复杂但是又很合理的代码之下。所以我尽量模拟了一些需求，使得代码看起来更丰富，下面我们还是来仔细看下这段代码的逻辑吧：

1. 创建了一个`AFHTTPSessionManager`用来做网络接口的数据获取。
2. 创建了一个名为`fetchData`的信号来通过网络获取信息。
3. 创建一个名为`title`的信号从获取的`data`中取得`title`字段，如果没有该字段则反馈一个错误。
4. 创建一个名为`desc`的信号从获取的`data`中取得`desc`字段，如果没有该字段则反馈一个错误。
5. 针对`desc`这个信号做一个渲染，得到一个名为`renderedDesc`的新信号，该信号会在渲染失败的时候反馈一个错误。
6. 把`title`信号所有的错误转换为字符串`@"Error"`并且在没有获取值之前以字符串`@"Loading..."`占位，之后与`self.someLablel`的`text`属性绑定。
7. 把`desc`信号所有的错误转换为字符串`@"Error"`并且在没有获取值之前以字符串`@"Loading..."`占位，之后与`self.originTextView`的`text`属性绑定。
8. 把`renderedDesc`信号所有的错误转换为属性字符串`@"Error"`并且在没有获取值之前以属性字符串`@"Loading..."`占位，之后与`self.renderedTextView`的`text`属性绑定。
9. 把`title`、`desc`、`renderedDesc`这三个信号的任何错误订阅，并且弹出`UIAlertView`。

看到这里我相信很多熟悉RAC的同学应该是对这些代码表示认同的，它也体现了RAC的一些优势例如良好的错误处理和各种链式处理。但是很遗憾的告诉大家这段代码是有很严重的错误的。

如果你去尝试运行这段代码，并且打开Charles查看，你会惊奇的发现，这个网络请求发送了6次。没错，是6次请求。我们也可以想象到类似的代码在其他副作用的问题，重新刷新了6次屏幕，写入6次文件，发了6个全局通知。

下面来分析下，为什么是6次网络请求呢？首先根据上面的知识，我们可以推断出名为`fetchData`信号是一个冷信号。那么这个信号在订阅的时候就会执行里面的过程。那这个信号是在什么时候被订阅了呢？仔细回看了代码，我们发现并没有订阅这个信号，只是调用这个信号的`flattenMap`产生了两个新的信号。

**这里有一个很重要的概念，就是任何的信号转换即是对原有的信号进行订阅从而产生新的信号。**我们可以写出flattenMap的伪代码如下：

```objective-c

- (instancetype)flattenMap_:(RACStream * (^)(id value))block {
{
    return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
       return [self subscribeNext:^(id x) {
           RACSignal *signal = (RACSignal *)block(x);
           [signal subscribeNext:^(id x) {
               [subscriber sendNext:x];
           } error:^(NSError *error) {
               [subscriber sendError:error];
           } completed:^{
               [subscriber sendCompleted];
           }];
       } error:^(NSError *error) {
           [subscriber sendError:error];
       } completed:^{
           [subscriber sendCompleted];
       }];
    }];
}

```

除了没有高度复用和缺少一些disposable的处理以外，上述代码可以大致的给我们`flattenMap`的直观处理，我们可以看到其实是在调用这个方法的时候，生成了一个新的信号，在这个新的信号的执行过程中对`self`进行的了**订阅**。我们还需要注意一个细节，就是这个返回信号在未来订阅的时候，才会间接的订阅了`self`。后续的`startWith`、`catchTo`等都可以这样理解。

回到我们的问题，那就是说，在`fetchData`被`flattenMap`之后，它就会因为名为`title`和`desc`信号的订阅而订阅。而后续我们对`desc`也进行了`flattenMap`得到了`renderedDesc`，那也说明了未来`renderedDesc`被订阅的时候，`fetchData`也会被间接订阅。所以我们解释了在后续我们用RAC宏进行绑定的时候，引发的**3次**`fetchData`的订阅。由于`fetchData`是冷信号，所以3次订阅意味着它的过程被执行了3次，也就是网络的3次请求。

另外的3次订阅来自`RACSignal`类的`merge`方法。根据上述的描述，我们也可以猜测`merge`方法也一定是创建了一个新的信号，在这个信号被订阅的时候，把它包含的所有信号订阅。所以我们又得到了额外的3次网络请求。

由此我们可以深刻的看到不熟悉冷热信号对业务造成的影响。我们可以想象对用户流量的影响，对服务器负载的影响，对统计的影响，如果这是一个点赞的接口，会不会造成多次点赞？后果是不堪的。而着一些都可以通过把`fetchData`转换为热信号来解决。

接下来也许你会问，如果我的整个计算过程中都没有副作用，是否就不会有这个问题，答案是肯定的，试想下刚才那段代码如果没有网络请求，换成一些标准化的计算会怎样。可以肯定的是我们不会出现bug，但是不要忽视的就是其中的运算我们执行了多次。刚才在介绍纯函数的时候，还有一个概念就是[引用透明](https://en.wikipedia.org/wiki/Referential_transparency_\(computer_science\))，我们可以在纯函数式语言（例如[Haskell](https://www.haskell.org)）上进行一定的优化，**也就是说纯函数的调用在相同参数下的返回值第二次不需要计算**，所以在纯函数式语言里面的FRP并没有冷信号的担忧。然而Objective-C语言中并未对纯函数进行优化。所以拥有大规模运算的冷信号对性能也是有一定影响的。

所以如果我们想更好的掌握RAC这个框架，区分冷信号与热信号是十分重要的。

## 正确理解冷信号与热信号

FRP是一种[声明式编程](https://en.wikipedia.org/wiki/Declarative_programming)。与传统的命令式编程的区别是声明式只是描述目标性质，让计算机明确目标，而非流程。而声明式编程不一定是FRP所独有的。例如Autolayout就是一种声明式编程的表现，通过编程声明了约束，而框架来做实际的动作。我们的主角RACSignal也是声明式的。请看下面代码：

```objective-c

    RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
		[subscriber sendNext:@1];
        [subscriber sendCompleted];
    }];
    
    RACSignal *mappedSignal = [signal map:^id(NSNumber *value) {
    	return [NSString stringWithFormat:@"Value is %@", value];
    }];
    
```

上述代码的**声明**了一个信号`signal`，`signal`指明了发送`“1”`这个值后发送`结束`事件。另外**声明**了一个信号`mappedSignal`，`mappedSignal`指明`signal`的值都进行一个字符串的`转换`。如果仅仅写到这里，`sendNext:`和`map:`后面的block其实都没有被执行。

那究竟是何时这些block会执行呢？没错，那就是在订阅之后。订阅`mappedSignal`之后，还会连带的把`signal`订阅了。因而预先**声明**的部分就有了动作。

在搞清楚了信号的声明和信号的订阅之后，再来理解多次订阅的问题。既然创建一个信号只是声明了一段操作，那就说明这个信号本身并无状态可言。可以换个角度来理解，在C语言中，声明了一个函数，这个函数在不同的时间被调用了很多次，函数体肯定会执行相应的次数。因为一个被声明的函数并没有状态，它并不清楚自己被谁在什么时间调用。所以冷信号也是一样，这段操作会在每次订阅的时候都执行，因为冷信号没有状态，它并不清楚自己被谁在什么时候订阅了。

当然一旦信号中存在了`副作用`，等同与一个修改了全局变量的函数，每次执行的时候的效果就是不一样的了，所以才会出现了前面提到的几个问题。

打个比方，冷信号好比一个剧本，它预先把要做的事情约定好。一旦一个导演说开拍，就是订阅了这个剧本，里面说描述的动作也开始一一被执行，而另一个导演拿着这个剧本开拍，显然和这个导演没有什么关系，拍摄的时期也可以不同。但是有可能有略微的关联，那就是演员可能请的相同的（访问相同的外部变量，或者触发网络请求），那可能要穿插着拍戏。另一方面观众可能也是相同的（最终都经过转换被UI订阅），那就会出现观众看两遍相同的剧情。

一旦片子拍好，放到电视上热播，就变成了热信号。它是有状态的，因为所有的观众都共享了播放的时间，大家都在同一时间观看同一片段。所以，把冷信号变为热信号的本质，就是“广播”，“广播”就是我们也在前面的代码中看到了`publish`和`RACMulticastConnection`这些操作。

另外举个例子，就是视频直播与视频点播。点播是无状态的，你不需要关心别人看了多少，每次你点播后都是从你需要观看的时间开始播放。而直播是有状态的，你必须要在指定的开播时间观看，一旦错过，就没法看漏掉的节目了。

## 揭示热信号的本质

好的，回到代码的世界。在RAC中，究竟什么才是热信号呢？冷信号比较常见，`map`一下就会得到一个冷信号。在RAC的世界中，其实所有的热信号都是一个类的，那就是`RACSubject`。接下来我们来看看究竟它为什么这么“神奇”。

在RAC2.5文档的[框架概述](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/v2.5/Documentation/FrameworkOverview.md#subjects)中，有这样一段描述：

> A subject, represented by the RACSubject class, is a signal that can be manually controlled.

> Subjects can be thought of as the "mutable" variant of a signal, much like NSMutableArray is for NSArray. They are extremely useful for bridging non-RAC code into the world of signals.

> For example, instead of handling application logic in block callbacks, the blocks can simply send events to a shared subject instead. The subject can then be returned as a RACSignal, hiding the implementation detail of the callbacks.

> Some subjects offer additional behaviors as well. In particular, RACReplaySubject can be used to buffer events for future subscribers, like when a network request finishes before anything is ready to handle the result.

在这段描述中，我们可以看出Subject这三个特点：

1. Subject是“可变”的。
2. Subject是非RAC到RAC的一个桥梁。
3. Subject可以良好的附加行为，例如`RACReplaySubject`可以缓冲事件给未来的订阅者。

从第三个特点来看，Subject具备将事件缓冲给未来订阅者的能力，那也就说明它是自身是有状态的。由此看来Subject是符合热信号的特点的。为了验证它，我们来做个简单实验：

```objective-c
    RACSubject *subject = [RACSubject subject];
    RACSubject *replaySubject = [RACReplaySubject subject];
    
    [[RACScheduler mainThreadScheduler] afterDelay:0.1 schedule:^{
        // Subscriber 1
        [subject subscribeNext:^(id x) {
            NSLog(@"Subscriber 1 get a next value: %@ from subject", x);
        }];
        [replaySubject subscribeNext:^(id x) {
            NSLog(@"Subscriber 1 get a next value: %@ from replay subject", x);
        }];
        
        // Subscriber 2
        [subject subscribeNext:^(id x) {
            NSLog(@"Subscriber 2 get a next value: %@ from subject", x);
        }];
        [replaySubject subscribeNext:^(id x) {
            NSLog(@"Subscriber 2 get a next value: %@ from replay subject", x);
        }];
    }];
    
    [[RACScheduler mainThreadScheduler] afterDelay:1 schedule:^{
        [subject sendNext:@"send package 1"];
        [replaySubject sendNext:@"send package 1"];
    }];
    
    [[RACScheduler mainThreadScheduler] afterDelay:1.1 schedule:^{
        // Subscriber 3
        [subject subscribeNext:^(id x) {
            NSLog(@"Subscriber 3 get a next value: %@ from subject", x);
        }];
        [replaySubject subscribeNext:^(id x) {
            NSLog(@"Subscriber 3 get a next value: %@ from replay subject", x);
        }];
        
        // Subscriber 4
        [subject subscribeNext:^(id x) {
            NSLog(@"Subscriber 4 get a next value: %@ from subject", x);
        }];
        [replaySubject subscribeNext:^(id x) {
            NSLog(@"Subscriber 4 get a next value: %@ from replay subject", x);
        }];
    }];
    
    [[RACScheduler mainThreadScheduler] afterDelay:2 schedule:^{
        [subject sendNext:@"send package 2"];
        [replaySubject sendNext:@"send package 2"];
    }];
```

按照解读一下上述代码：
1. 0s时创建`subject`与`replaySubject`这两个subject。
2. 0.1s时`订阅者1`分别订阅了`subject`与`replaySubject`。
3. 0.1s时`订阅者2`也分别订阅了`subject`与`replaySubject`。
4. 1s时分别向`subject`与`replaySubject`发送了`"send package 1"`这个字符串作为**值**。
5. 1.1s时`订阅者3`分别订阅了`subject`与`replaySubject`。
6. 1.1s时`订阅者4`也分别订阅了`subject`与`replaySubject`。
7. 2s时再分别向`subject`与`replaySubject`发送了`"send package 2"`这个字符串作为**值**。

接下来看一下输出的结果：
```
2015-09-28 13:35:22.855 RACDemos[13646:1269269] Start
2015-09-28 13:35:23.856 RACDemos[13646:1269269] Subscriber 1 get a next value: send package 1 from subject
2015-09-28 13:35:23.856 RACDemos[13646:1269269] Subscriber 2 get a next value: send package 1 from subject
2015-09-28 13:35:23.857 RACDemos[13646:1269269] Subscriber 1 get a next value: send package 1 from replay subject
2015-09-28 13:35:23.857 RACDemos[13646:1269269] Subscriber 2 get a next value: send package 1 from replay subject
2015-09-28 13:35:24.059 RACDemos[13646:1269269] Subscriber 3 get a next value: send package 1 from replay subject
2015-09-28 13:35:24.059 RACDemos[13646:1269269] Subscriber 4 get a next value: send package 1 from replay subject
2015-09-28 13:35:25.039 RACDemos[13646:1269269] Subscriber 1 get a next value: send package 2 from subject
2015-09-28 13:35:25.039 RACDemos[13646:1269269] Subscriber 2 get a next value: send package 2 from subject
2015-09-28 13:35:25.039 RACDemos[13646:1269269] Subscriber 3 get a next value: send package 2 from subject
2015-09-28 13:35:25.040 RACDemos[13646:1269269] Subscriber 4 get a next value: send package 2 from subject
2015-09-28 13:35:25.040 RACDemos[13646:1269269] Subscriber 1 get a next value: send package 2 from replay subject
2015-09-28 13:35:25.040 RACDemos[13646:1269269] Subscriber 2 get a next value: send package 2 from replay subject
2015-09-28 13:35:25.040 RACDemos[13646:1269269] Subscriber 3 get a next value: send package 2 from replay subject
2015-09-28 13:35:25.040 RACDemos[13646:1269269] Subscriber 4 get a next value: send package 2 from replay subject
```

结合结果可以分析出如下内容：

1. 22.855s时，测试启动，`subject`与`replaySubject`创建完毕。
2. 23.856s时，距离启动大约1s后，`订阅者1`和`订阅者2`**同时**从`subject`接收到了`"send package 1"`这个值。
3. 23.857s时，也是距离启动大约1s后，`订阅者1`和`订阅者2`**同时**从`replaySubject`接收到了`"send package 1"`这个值。
4. 24.059s时，距离启动大约1.2s后，`订阅者3`和`订阅者4`**同时**从`replaySubject`接收到了`"send package 1"`这个值。**注意`订阅者3`和`订阅者4`并没有从`subject`接收`"send package 1"`这个值。**
5. 25.039s时，距离启动大约2.1s后，`订阅者1`、`订阅者2`、`订阅者3`、`订阅者4`**同时**从`subject`接收到了`"send package 2"`这个值。
6. 25.040s时，距离启动大约2.1s后，`订阅者1`、`订阅者2`、`订阅者3`、`订阅者4`**同时**从`replaySubject`接收到了`"send package 2"`这个值。

只关注`subject`，根据时间线，我们可以得到下图：

![RAC冷热信号1](/images/RAC冷热信号1.png) 

经过观察不难发现，4个订阅者实际上是共享`subject`的，一旦这个`subject`发送了值，当前的订阅者就会同时接收到。由于`订阅者3`与`订阅者4`的订阅者时间稍晚，所以错过了第一次值的发送。这与冷信号是截然不同的反应。冷信号的图类似下图：

![RAC冷热信号1](/images/RAC冷热信号2.png) 

对比上面两张图，是不是可以发现，`subject`类似“直播”，错过了就不再处理。而`signal`类似“点播”，每次订阅都会从头开始。所以我们有理由锁定`subject`天然就是热信号。

下面再来看看`replaySubject`，根据时间线，我们能得到另一张图：

![RAC冷热信号1](/images/RAC冷热信号3.png) 

将该图与`subject`那张图对比会发现，`订阅者3`与`订阅者4`在订阅后马上接收到了“历史值”。对于`订阅者3`和`订阅者4`来说，他们只关心“历史的值”而不关心“历史的时间线”，因为实际上`1`与`2`是间隔1s发送的，但是他们接收到的显然不是。举个生动的例子，就好像科幻电影里面主人公穿越时间线后会把所有的回忆快速闪过来到现实一样。（见《X战警：逆转未来》、《蝴蝶效应》）所以我们也有理由锁定`replaySubject`天然也是热信号。

看到这里，我们终于揭开了热信号的面纱，结论便是：

1. `RACSubject`及其子类是**热信号**。
2. `RACSignal`排除`RACSubject`类以外的是**冷信号**。

## 如何将一个冷信号转化成热信号——广播

冷信号与热信号的本质区别在于是否保持状态，冷信号的多次订阅是不保持状态的，而热信号的多次订阅可以保持状态。所以一种将冷信号转换为热信号的方法就是，将冷信号订阅，取得的每一个值再通过`RACSbuject`发送出去。

看一下下面的代码：

```objective-c
    RACSignal *coldSignal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        NSLog(@"Cold signal be subscribed.");
        [[RACScheduler mainThreadScheduler] afterDelay:1.5 schedule:^{
            [subscriber sendNext:@"A"];
        }];
        
        [[RACScheduler mainThreadScheduler] afterDelay:3 schedule:^{
            [subscriber sendNext:@"B"];
        }];
        
        [[RACScheduler mainThreadScheduler] afterDelay:5 schedule:^{
            [subscriber sendCompleted];
        }];
        
        return nil;
    }];
    
    RACSubject *subject = [RACSubject subject];
    NSLog(@"Subject created.");
    
    [[RACScheduler mainThreadScheduler] afterDelay:2 schedule:^{
        [coldSignal subscribe:subject];
    }];
    
    [subject subscribeNext:^(id x) {
        NSLog(@"Subscribe 1 recieve value:%@.", x);
    }];
    
    [[RACScheduler mainThreadScheduler] afterDelay:4 schedule:^{
        [subject subscribeNext:^(id x) {
            NSLog(@"Subscribe 2 recieve value:%@.", x);
        }];

```

执行顺序是这样的：

1. 创建一个冷信号：`coldSignal`。该信号声明了“订阅后1.5秒发送‘A’，3秒发送'B'，5秒发送完成事件”。
2. 创建一个RACSubject：`subject`。
3. 在2秒后使用这个`subject`订阅`coldSignal`。
4. 立即订阅这个`subject`。
5. 4秒后订阅这个`subject`。

如果所料不错的话，通过订阅这个`subject`并不会引起`coldSignal`重复执行block的内容。我们来看下结果：

```
2015-09-28 19:36:45.703 RACDemos[14110:1556061] Subject created.
2015-09-28 19:36:47.705 RACDemos[14110:1556061] Cold signal be subscribed.
2015-09-28 19:36:49.331 RACDemos[14110:1556061] Subscribe 1 recieve value:A.
2015-09-28 19:36:50.999 RACDemos[14110:1556061] Subscribe 1 recieve value:B.
2015-09-28 19:36:50.999 RACDemos[14110:1556061] Subscribe 2 recieve value:B.
```

参考时间线，会得到下图：
![RAC冷热信号4](/images/RAC冷热信号4.png) 

解读一下其中的要点：
1. `subject`是从一开始就创建好的，等到2s后便开始订阅`coldSignal`。
2. `subscribe 1`是`subject`创建后就开始订阅的，但是第一个接收时间与`subject`接收`coldSignal`第一个值的时间是一样的。
3. `subscribe 2`是`subject`创建4s后开始订阅的，所以只能接收到第二个值。

通过观察可以确定，`subject`就是`coldSignal`转化的热信号。所以使用`RACSubject`来将冷信号转化为热信号是可行的。

当然，使用这种`RACSubject`来订阅冷信号得到热信号的方式还是有一些小的瑕疵的。例如`subject`的订阅者提前终止了订阅，而`subject`并不能终止对`coldSignal`的订阅。（`RACDisposable`是一个比较大的话题，我计划在其他的文章中详细阐述它，也希望感兴趣的同学自己来理解。）所以RAC库中对于冷信号转化成热信号有如下标准的包装：

```objective-c
- (RACMulticastConnection *)publish;
- (RACMulticastConnection *)multicast:(RACSubject *)subject;
- (RACSignal *)replay;
- (RACSignal *)replayLast;
- (RACSignal *)replayLazily;
```

这5个方法中，最为重要的就是`- (RACMulticastConnection *)multicast:(RACSubject *)subject;`这个方法了，其他几个方法也是间接调用它的。我们来看看它的真相：

```objective-c
/// implementation RACSignal (Operations)
- (RACMulticastConnection *)multicast:(RACSubject *)subject {
	[subject setNameWithFormat:@"[%@] -multicast: %@", self.name, subject.name];
	RACMulticastConnection *connection = [[RACMulticastConnection alloc] initWithSourceSignal:self subject:subject];
	return connection;
}

/// implementation RACMulticastConnection

- (id)initWithSourceSignal:(RACSignal *)source subject:(RACSubject *)subject {
	NSCParameterAssert(source != nil);
	NSCParameterAssert(subject != nil);

	self = [super init];
	if (self == nil) return nil;

	_sourceSignal = source;
	_serialDisposable = [[RACSerialDisposable alloc] init];
	_signal = subject;
	
	return self;
}

#pragma mark Connecting

- (RACDisposable *)connect {
	BOOL shouldConnect = OSAtomicCompareAndSwap32Barrier(0, 1, &_hasConnected);

	if (shouldConnect) {
		self.serialDisposable.disposable = [self.sourceSignal subscribe:_signal];
	}

	return self.serialDisposable;
}

- (RACSignal *)autoconnect {
	__block volatile int32_t subscriberCount = 0;

	return [[RACSignal
		createSignal:^(id<RACSubscriber> subscriber) {
			OSAtomicIncrement32Barrier(&subscriberCount);

			RACDisposable *subscriptionDisposable = [self.signal subscribe:subscriber];
			RACDisposable *connectionDisposable = [self connect];

			return [RACDisposable disposableWithBlock:^{
				[subscriptionDisposable dispose];

				if (OSAtomicDecrement32Barrier(&subscriberCount) == 0) {
					[connectionDisposable dispose];
				}
			}];
		}]
		setNameWithFormat:@"[%@] -autoconnect", self.signal.name];
}
```

代码比较短，大概来说明一下：
1. 当`RACSignal`类的实例调用`- (RACMulticastConnection *)multicast:(RACSubject *)subject`时，创建一个`RACMulticastConnection`实例，以`self`和`subject`作为构造参数。
2. `RACMulticastConnection`构造的时候，保存`source`和`subject`作为成员变量，创建一个`RACSerialDisposable`对象。
3. 当`RACMulticastConnection`类的实例调用`- (RACDisposable *)connect`这个方法的时候，判断是否是第一次，如果是的话用`_signal`这个成员变量来订阅`sourceSignal`之后返回`self.serialDisposable`；否则直接返回`self.serialDisposable`。
4. `RACMulticastConnection`的`signal`只读属性，就是热信号，订阅它就可以。它会在`- (RACDisposable *)connect`第一次调用后，根据`sourceSignal`的订阅结果来传递事件。
5. 想要确保第一次订阅就能成功订阅`sourceSignal`，可以使用`- (RACSignal *)autoconnect`这个方法，它保证了第一个订阅者触发了`sourceSignal`的订阅，也保证了当返回的信号所有订阅者都关闭连接后`sourceSignal`被正确关闭连接。

所以，正确的使用可以像这样：

```objective-c
	RACSignal *coldSignal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        NSLog(@"Cold signal be subscribed.");
        [[RACScheduler mainThreadScheduler] afterDelay:1.5 schedule:^{
            [subscriber sendNext:@"A"];
        }];
        
        [[RACScheduler mainThreadScheduler] afterDelay:3 schedule:^{
            [subscriber sendNext:@"B"];
        }];
        
        [[RACScheduler mainThreadScheduler] afterDelay:5 schedule:^{
            [subscriber sendCompleted];
        }];
        
        
        return nil;
    }];
    
    RACSubject *subject = [RACSubject subject];
    NSLog(@"Subject created.");
    
    RACMulticastConnection *multicastConnection = [coldSignal multicast:subject];
    RACSignal *hotSignal = multicastConnection.signal;
    
    [[RACScheduler mainThreadScheduler] afterDelay:2 schedule:^{
        [multicastConnection connect];
    }];
    
    [hotSignal subscribeNext:^(id x) {
        NSLog(@"Subscribe 1 recieve value:%@.", x);
    }];
    
    [[RACScheduler mainThreadScheduler] afterDelay:4 schedule:^{
        [hotSignal subscribeNext:^(id x) {
            NSLog(@"Subscribe 2 recieve value:%@.", x);
        }];
    }];
```

或者这样：

```objective-c
    RACSignal *coldSignal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        NSLog(@"Cold signal be subscribed.");
        [[RACScheduler mainThreadScheduler] afterDelay:1.5 schedule:^{
            [subscriber sendNext:@"A"];
        }];
        
        [[RACScheduler mainThreadScheduler] afterDelay:3 schedule:^{
            [subscriber sendNext:@"B"];
        }];
        
        [[RACScheduler mainThreadScheduler] afterDelay:5 schedule:^{
            [subscriber sendCompleted];
        }];
        
        
        return nil;
    }];
    
    RACSubject *subject = [RACSubject subject];
    NSLog(@"Subject created.");
    
    RACMulticastConnection *multicastConnection = [coldSignal multicast:subject];
    RACSignal *hotSignal = multicastConnection.autoconnect;
    
    [[RACScheduler mainThreadScheduler] afterDelay:2 schedule:^{
        [hotSignal subscribeNext:^(id x) {
            NSLog(@"Subscribe 1 recieve value:%@.", x);
        }];
    }];
    
    
    [[RACScheduler mainThreadScheduler] afterDelay:4 schedule:^{
        [hotSignal subscribeNext:^(id x) {
            NSLog(@"Subscribe 2 recieve value:%@.", x);
        }];
    }];
```

以上的两种写法都可以得到和之前相同的结果。

下面再来看看其他几个方法的实现：

```objective-c
/// implementation RACSignal (Operations)
- (RACMulticastConnection *)publish {
	RACSubject *subject = [[RACSubject subject] setNameWithFormat:@"[%@] -publish", self.name];
	RACMulticastConnection *connection = [self multicast:subject];
	return connection;
}

- (RACSignal *)replay {
	RACReplaySubject *subject = [[RACReplaySubject subject] setNameWithFormat:@"[%@] -replay", self.name];

	RACMulticastConnection *connection = [self multicast:subject];
	[connection connect];

	return connection.signal;
}

- (RACSignal *)replayLast {
	RACReplaySubject *subject = [[RACReplaySubject replaySubjectWithCapacity:1] setNameWithFormat:@"[%@] -replayLast", self.name];

	RACMulticastConnection *connection = [self multicast:subject];
	[connection connect];

	return connection.signal;
}

- (RACSignal *)replayLazily {
	RACMulticastConnection *connection = [self multicast:[RACReplaySubject subject]];
	return [[RACSignal
		defer:^{
			[connection connect];
			return connection.signal;
		}]
		setNameWithFormat:@"[%@] -replayLazily", self.name];
}

```

这几个方法的时间都相当简单，只是为了简化代码，具体说明一下：
1. `- (RACMulticastConnection *)publish`就是帮忙创建了`RACSubject`。
2. `- (RACSignal *)replay`就是用`RACReplaySubject`来作为`subject`，并立即执行`connect`操作，返回`connection.signal`。其作用是上面提到的`replay`功能，既后来的订阅者可以收到历史值。
3. `- (RACSignal *)replayLast`就是用`Capacity`为1的`RACReplaySubject`来替换`- (RACSignal *)replay`的`subject。其作用是使后来订阅者只收到最后的历史值。
4. `- (RACSignal *)replayLazily`和`- (RACSignal *)replay`的区别就是`replayLazily`会在第一次订阅的时候才订阅`sourceSignal`。

现在看下之前第二章那个业务场景的例子，其实修改的方法很简单，就是在网络获取的`fetchData`这个信号后面，增加一个`replayLazily`变换，就不会出现网络请求重发6次的问题了。

修改后的代码如下：

```objective-c

    self.sessionManager = [[AFHTTPSessionManager alloc] initWithBaseURL:[NSURL URLWithString:@"http://api.xxxx.com"]];
    
    self.sessionManager.requestSerializer = [AFJSONRequestSerializer serializer];
    self.sessionManager.responseSerializer = [AFJSONResponseSerializer serializer];

    @weakify(self)
    RACSignal *fetchData = [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        @strongify(self)
        NSURLSessionDataTask *task = [self.sessionManager GET:@"fetchData" parameters:@{@"someParameter": @"someValue"} success:^(NSURLSessionDataTask *task, id responseObject) {
            [subscriber sendNext:responseObject];
            [subscriber sendCompleted];
        } failure:^(NSURLSessionDataTask *task, NSError *error) {
            [subscriber sendError:error];
        }];
        return [RACDisposable disposableWithBlock:^{
            if (task.state != NSURLSessionTaskStateCompleted) {
                [task cancel];
            }
        }];
    }] replayLazily];  // modify here!!
    
    RACSignal *title = [fetchData flattenMap:^RACSignal *(NSDictionary *value) {
        if ([value[@"title"] isKindOfClass:[NSString class]]) {
            return [RACSignal return:value[@"title"]];
        } else {
            return [RACSignal error:[NSError errorWithDomain:@"some error" code:400 userInfo:@{@"originData": value}]];
        }
    }];
    
    RACSignal *desc = [fetchData flattenMap:^RACSignal *(NSDictionary *value) {
        if ([value[@"desc"] isKindOfClass:[NSString class]]) {
            return [RACSignal return:value[@"desc"]];
        } else {
            return [RACSignal error:[NSError errorWithDomain:@"some error" code:400 userInfo:@{@"originData": value}]];
        }
    }];
    
    RACSignal *renderedDesc = [desc flattenMap:^RACStream *(NSString *value) {
        NSError *error = nil;
        RenderManager *renderManager = [[RenderManager alloc] init];
        NSAttributedString *rendered = [renderManager renderText:value error:&error];
        if (error) {
            return [RACSignal error:error];
        } else {
            return [RACSignal return:rendered];
        }
    }];
    
    RAC(self.someLablel, text) = [[title catchTo:[RACSignal return:@"Error"]]  startWith:@"Loading..."];
    RAC(self.originTextView, text) = [[desc catchTo:[RACSignal return:@"Error"]] startWith:@"Loading..."];
    RAC(self.renderedTextView, attributedText) = [[renderedDesc catchTo:[RACSignal return:[[NSAttributedString alloc] initWithString:@"Error"]]] startWith:[[NSAttributedString alloc] initWithString:@"Loading..."]];
    
    [[RACSignal merge:@[title, desc, renderedDesc]] subscribeError:^(NSError *error) {
        UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"Error" message:error.domain delegate:nil cancelButtonTitle:@"OK" otherButtonTitles:nil];
        [alertView show];
    }];
```

当然，这样修改，仍然有许多计算上的浪费，例如将`fetchData`转换为`title`的block会执行多次，将`fetchData`转换为`desc`的block也会执行多次。但是由于这些block都是无副作用的，计算量又小，可以忽略不计。
 
至此，我们终于揭开RAC中冷信号与热信号的全部面纱，也知道如何使用了。希望此文可以让大家更好的了解RAC，减少使用RAC遇到的误区。谢谢大家。

