---
layout: post
title: "从另一个角度介绍下Block"
date: 2014-09-09 10:27:24 +0800
comments: true
categories: iOS 程序员日记
description: 从另一个角度介绍下Block
keywords: block 匿名函数 闭包 iOS

---

群里有个小伙伴问我block的理解，我想网上那么多blog都写过iOS的block的介绍，说明，用法，如果还是不能理解，那就换个角度吧。所以，今天我们来聊一聊Block的前世今生，不谈block如何定义，不谈block有哪些坑，只谈它是怎么来的。

首先block的使用，真的不必多说，想必大家google后也都会用。问题就在于，为什么要有block？有人觉得block方便，到底方便在哪里呢？这一切要从函数指针这个很老的概念谈起了。

在C时代，面向过程一度成为程序开发的主流，在没有对象化的程序设计中，我们难免写出如下的程序：

```c
extern const char *GetMenu0();
extern const char *GetMenu1();
extern const char *GetMenu2();
extern const char *GetMenu3();
extern const char *GetMenu4();
extern const char *GetMenu5();



const char *GetMenuShow(int pos) {
    switch (pos) {
        case 0: return GetMenu0();
        case 1: return GetMenu1();
        case 2: return GetMenu2();
        case 3: return GetMenu3();
        case 4: return GetMenu4();
        case 5: return GetMenu5();
        default:
            return NULL;
    }
}
```

随着程序的复杂度提高，这样的程序变得越来越长，这时函数指针可以来帮忙，于是函数就变成了这样：

```c
extern const char *GetMenu0();
extern const char *GetMenu1();
extern const char *GetMenu2();
extern const char *GetMenu3();
extern const char *GetMenu4();
extern const char *GetMenu5();

typedef const char *(*MenuMethodType)();

static MenuMethodType g_methods[] = {
  &GetMenu0,
  &GetMenu1,
  &GetMenu2,
  &GetMenu3,
  &GetMenu4,
  &GetMenu5
};


const char *GetMenuShow(int pos) {
    if (pos < 0 || pos >= (sizeof(g_methods) / sizeof(MenuMethodType))) {
        return NULL;
    }
    return g_methods[pos]();
}
```

这种写法也叫做跳转表，好处是以后GetMenu*这种函数的增长可以放到GetMenuShow这个函数外，减少了耦合。函数指针的另一个妙用就是回调函数，这个非常的普遍，也不需要再举例子了。

函数指针给我们带来的新的开发思想，就是行为的变量化，因此，我们可以将不同的行为封装到统一的流程之外，作为可替换的组件。总之，它允许你把可变化的行为，注入到稳定的过程中，我们便获得了更好的扩展，把开发的中心放到变化而不是重复上。

到了OC时代，OC有了一种比函数指针还高效而简单的东西，那就是selector。这个被称为选择器的工具，不仅可以让我们得到函数指针的一切便利，还可以动态的替换其指向的内容。于是我们有了很多`addTarget:action:`这样的API，使得我们可以把行为注入到已经非常稳定的Cocoa或CocoaTouch框架中。

虽然有了一定的便利，但是程序员是不容易满足的。我们逐渐发现，这种基于action的写法有时很麻烦。主要是以下几点：

1. 就是很多时候，你注入的内容可能就一次，搞一个函数或者方法，浪费了不少的时间。
2. 你还要起名字，要知道，Phil Karlton就说过:“在计算机科学领域,有两大难题,如何验证缓存和如何给各种东西命名。”
3. 使得你原本可以在一个函数里实现的逻辑，分散到不同的部分，你难以专注的一次把你的逻辑写完。

这时，我们的主角block就来帮助大家了。

其实block还有很多别名，其中一个就是匿名函数。利用block，你可以在一个函数中，写上一小段代码，不用起名字，就可以传递过去。函数指针的用法，几乎都可以用匿名函数来替换。一举解决了碎片化，命名等问题。我们便有了这样的代码：

```objective-c
AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager];
[manager GET:@"http://example.com/resources.json" parameters:nil success:^(AFHTTPRequestOperation *operation, id responseObject) {
    NSLog(@"JSON: %@", responseObject);
} failure:^(AFHTTPRequestOperation *operation, NSError *error) {
    NSLog(@"Error: %@", error);
}];

```

贪婪的程序员不仅仅满足于少起个名字和把代码写到一起，一旦程序员们发现，把一个函数写在另一个函数里很爽，就开始频繁的尝试。这时，又发现了几个小问题（以下仅是匿名函数的问题，block解决了这些问题）：

1. 匿名函数，到底还是一个函数，这个括号外面世界的变量，是不可以在里面使用的。
2. 由于只能传递一个函数，我们就没有了可操作的对象（即没有了self）。

其实这两个是一样的问题，都是因为没有变量的传递。如果可以把self传递下来，第二个问题也解决了。

想要解决这个问题，最简单的方法就是搞到全局域，于是代码就写成了这样：

```objective-c
static int g_sum = 0;

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    g_sum = 0;
    [@[@1, @2, @3] enumerateObjectsUsingBlock:^(NSNumber *obj, NSUInteger idx, BOOL *stop) {
        int val = [obj intValue];
        g_sum += val;
    }];
    NSLog(@"sum = %d", g_sum);
    
    // Override point for customization after application launch.
    return YES;
}
```

把想要在匿名函数中的部分先存到全局变量，然后再到匿名函数中取出来。我们原本想解决的碎片化问题，又回来了。这样情况下，我们不得不搞出好多的全局变量，给全局变量起名字的问题也回来了。聪明的工程师们很快发现了，这种传递有着很规律的行为，于是把这些行为封装起来，做成库。这种手段叫做“闭包”，block的又一个名字。

把外面的对象，包在匿名函数中，封装起来，以备不时之需。OC利用编译器，在静态检查的时候，把用到的外部变量，都封装到block中。

这使得匿名函数，有了新的活力。在大量的尝试后，发现这简直就是一种神奇，闭包不单使得你的逻辑更加的紧凑，还使得开发变得越来越有趣。nodejs尝试用大量的闭包铸成了一种单线程异步的神奇的库。ReactiveCocoa也用block改变了大家开发iOS的思路。

有了block，我们可以更好的把变化抽取出来，可以更专注的实现逻辑，将异步的，碎片化的需求，快速的整合到一起。相比这些优点，block稍许复杂的语法，和一些可能出现的问题，是可以被原谅的。swift中，我们看到更多的闭包，可以看出block的写法对于开发有着多么深远的影响。

此篇只是一个引子，block有很多需要学习的地方，用好容易，用得精巧，还需要大家更多的开阔思维。




