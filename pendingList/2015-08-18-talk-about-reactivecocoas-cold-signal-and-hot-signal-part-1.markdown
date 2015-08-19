---
layout: post
title: "细说ReactiveCocoa的冷信号与热信号（一）"
date: 2015-08-18 12:21:28 +0800
comments: true
categories: iOS RAC专栏
description: ReactiveCocoa（简称RAC）是一套基于Cocoa的FRP框架，在我们美团客户端中，我们大量使用了这个框架。而在使用的过程中我们发现，冷信号与热信号的概念很容易混淆并且容易造成一定的问题，相信各位在使用的过程中也可能遇到此类问题。所以我在这里与大家讨论下RAC中冷信号与热信号的相关知识点，希望可以加深大家对冷热信号的理解。先介绍下背景和什么是冷信号与热信号。
keywords: iOS ReactiveCocoa RAC 函数响应式编程 hot cold signal
---

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

好的，至此我们知道了什么是冷信号与热信号，了解了它们的特点。下一部分我们来看看为什么要区分冷信号与热信号。