---
layout: post
title: "聊一聊iOS开发中的惰性计算"
date: 2016-11-07 21:56:25 +0800
comments: true
categories: iOS 函数式编程 Objective-C
description: 聊一聊iOS开发中的惰性计算
keywords: 惰性计算 iOS 函数式编程
---
首先给大家讲一个笑话：

> 有一只小白兔，跑到蔬菜店里问老板：“老板，有100个胡萝卜吗？”。老板说：“没有那么多啊。”，小白兔失望的说道：“哎，连100个胡萝卜都没有。。。”。  
> 第二天小白兔又来到蔬菜店问老板：“今天有100个胡萝卜了吧？”，老板尴尬的说：“今天还是缺点，明天就能好了。”，小白兔又很失望的走了。  
> 第三天小白兔刚一推门，老板就高兴的说道：“有了有了，从前天就进货的100个胡萝卜到货了。”，小白兔说：“太好了，我要买2根！”。。。

不晓得笑话是否博您一笑，但是这里面确有一个点是和我们的主题惰性计算相关的。试想一下，假设蔬菜店是一个电商，你是老板，你挂商品数量的时候，是100个，1000个，还是真实的备货2个？显然做过淘宝的同学都知道这其中的玄机，就是先挂大的余量，有卖出再补货。所以，如果这个老板先回答有100个胡萝卜，再等它要2个的时候把自己备货的2个拿给它，是不是就免去了100个胡萝卜的物流？

在程序开发中，我们也会经常的遇到这样的问题，明明创建了很大的一个对象，但是其实只用了一个字段；明明创建了一个500个的数组，其实只用了第0个和第1个元素。遇到这类问题，我们可以尝试使用惰性计算来解决。

关于惰性计算，或者惰性求值。想必大家第一反应就是在getter里动态返回属性了。例如有一个很大的属性，你希望在有人调用的时候才创建，就可以这样写：

``` objective-c
- (id)someBigProperty
{
    if (_someBigProperty == nil) {
        NSMutableArray *someBigProperty = [NSMutableArray array];
        for (int i = 0; i < 100000; ++i) {
            [someBigProperty addObject:@(i)];
        }
        _someBigProperty = [someBigProperty copy];
    }
    
    return _someBigProperty;
}

```

本文当然不拘泥于大家耳熟能详的知识点进行阐述了。上述的代码虽然也能勉强叫惰性求值，但并非足够理想。为什么说是“勉强叫”呢？大家想想上面的笑话，其实这样做和老板的做法并无差别。首先店里没有100个胡萝卜，就好像这个对象没有`_someBigProperty`属性一样。一旦有人需要100个“胡萝卜”，就循环100000次创建这个`_someBigProperty`属性。然后可能使用者只需要第0个。

另外在实际项目中这样的一个手段几乎被大家严重的乱用了，为什么说是乱用呢？除了创建非常大的属性、或者创建对象的时候有一些必要的副作用不能提前创建之外，几乎不应该使用惰性求值来处理类似逻辑。原因如下：

* 如果真的是很大的属性，一般它比较重要，很可能会被访问，要不要在getter中写出来意义不大。
* @property的atomic、nonatomic、copy、strong等描述在有getter方法的属性上会失效，后人修改代码的时候可能只改了@property声明，并不会记得改getter，于是隐患就这样埋下了。
* 代码含有了隐私操作，尤其getter中再混杂了各种逻辑，使得程序出现问题非常不好排查。后人哪会想到`someObj.someProperty`这样一个简简单单的取属性发生了很多奇妙的事。
* 代码多，本来代码只需要在`init`方法中创建用上一两行，结果用了至少7行的一个getter方法才能写出来，想想一个程序轻则数百个属性，都这么搞，得多出多少行代码？另外代码格式几乎完全一样，不符合DRY原则。好的程序员不应该总是写重复的代码，不是么？（某人说一个程序里面，20-30个属性已经算非常多了，只能是眼界问题了）
* 性能损耗，对于属性取值可能会非常的频繁，如果所有的属性取值之前都经过一个`if`判断，这不是平白浪费的性能？

我们回到正题。既然简单改写一下getter不但解决不了问题还有这么多隐患，那我们该如何能够正确优雅的把惰性计算写好？下面给大家一些建议。

观察上面的代码，你会发现_someBigProperty是一个非常规则的NSArray，它的item内容与下标相等。我们可以看出item的结果与index存在如下关系：

> f(x) = x

类似的可以有很多，例如`> 100`的为`@“world”`，`0 <= x <= 100`的为`@“hello”`；item为下标的平方；item为下标的数值转换成的字符串等。所以这类`NSArray`，基本需要一个count和一个函数就可以构成了。那我们现在就基于`NSArray`这个类簇，实现一个特殊的类吧！

关于类簇，相信很多同学都有所了解，大概的说法是不可以直接继承一个`NSArray`、`NSNumber`、`NSString`这样的类。如果要继承需要实现全部的必要方法，在`NSArray`这个类簇来说，就是如下的方法：

``` objective-c
@interface NSArray<__covariant ObjectType> : NSObject <NSCopying, NSMutableCopying, NSSecureCoding, NSFastEnumeration>

@property (readonly) NSUInteger count;
- (ObjectType)objectAtIndex:(NSUInteger)index;
- (instancetype)init NS_DESIGNATED_INITIALIZER;
- (instancetype)initWithObjects:(const ObjectType [])objects count:(NSUInteger)cnt NS_DESIGNATED_INITIALIZER;
- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder NS_DESIGNATED_INITIALIZER;

@end
```

当然除了`NSArray`类的基本方法，还有`NSCopying`、`NSMutableCopying`、`NSSecureCoding`这些协议需要实现，另外`NSFastEnumberation`协议已经默认实现完成，不需要额外处理。与惰性计算无关的细节大家可以自己填补，对于本例，我们只需要关心这几个方法的实现：

``` objective-c
    
typedef id(^ItemBlock)(NSUInteger index);
    
@interface ZDynamicArray : NSArray

- (instancetype)initWithItemBlock:(ItemBlock)block count:(NSUInteger)cnt;
- (id)objectAtIndex:(NSUInteger)index;
- (NSUInteger)count;
@end
```

按照上文的说法，对于这样一个特殊的`NSArray`，我们真正要储存的数据只有一个count值外加一个函数，所以我们用这两个作为`init`参数。实现也很简单：

``` objective-c
@interface ZDynamicArray()

@property (nonatomic, readonly) ItemBlock block;
@property (nonatomic, readonly) NSUInteger cnt;
@end

@implementation ZDynamicArray

- (instancetype)initWithItemBlock:(ItemBlock)block count:(NSUInteger)cnt
{
    if (self = [super init]) {
        _block = block;
        _cnt = cnt;
    }
    return self;
}

- (NSUInteger)count
{
    return self.cnt;
}

- (id)objectAtIndex:(NSUInteger)index
{
    if (self.block) {
        return self.block(index);
    } else {
        return nil;
    }
}

@end 
    
```

瞧，就这么简单的写好了。让我们试一下吧！

``` objective-c
ZDynamicArray *array = [[ZDynamicArray alloc] initWithItemBlock:^id(NSUInteger index) {
    return @(index);
} count:100000];

for (id v in array) {
    NSLog(@"%@", v);
}

NSLog(@"%@", array[15]);
```

一个看似10w数据的数组，其实占用空间微乎其微，但是作用和最开始那样的代码效果一样。很不错吧。大家也可以动手实践，写一些自己需要用到的惰性计算代码，例如一个`Model`的数组，并非所有的`Model`都需要用到，我们也可以做成这样的一个数组，等用到的时候再从`NSDicitonary`转换成`Model`。就像这样：

``` objective-c
NSArray *downloadData = @[@{}, @{}, @{}, @{}];

NSArray *modelArray = [[ZDynamicArray alloc] initWithItemBlock:^id(NSUInteger index) {
    return [SomeModel modelFromDictionary:downloadData[index]];
} count:downloadData.count];    
    
```    

好了，惰性计算就说到这里了。大家善加利用，一定可以写一些好玩的东西的。

