---
layout: post
title: "细说ReactiveCocoa的冷信号与热信号（二）"
date: 2015-08-18 13:51:28 +0800
comments: true
categories: iOS RAC专栏
description: ReactiveCocoa（简称RAC）是一套基于Cocoa的FRP框架，在我们美团客户端中，我们大量使用了这个框架。而在使用的过程中我们发现，冷信号与热信号的概念很容易混淆并且容易造成一定的问题，相信各位在使用的过程中也可能遇到此类问题。所以我在这里与大家讨论下RAC中冷信号与热信号的相关知识点，希望可以加深大家对冷热信号的理解。本章介绍下为什么要区分冷信号与热信号。
keywords: iOS ReactiveCocoa RAC 函数响应式编程 hot cold signal
---

## 为什么要区分冷信号与热信号

也许你看到前一章节的时候就有疑问，为什么RAC要搞如此复杂的一个概念，直接搞成一种信号不就好了么？要解释这个问题需要绕一些弯路。（前方可能比较难懂，如果不能很好理解，请自行查阅各类文档。）

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

所以如果我们想更好的掌握RAC这个框架，区分冷信号与热信号是十分重要的。下一部分我会讲述如何正确理解冷信号与热信号。