---
layout: post
title: "如何利用Objective-C写一个精美的DSL"
date: 2017-01-10 13:22:49 +0800
comments: true
categories: iOS Objective-C
description: 如何利用Objective-C写一个精美的DSL
keywords: 链式操作 Objective-C iOS DSL
---
## 背景

在程序开发中，我们总是希望能够更加简洁、更加语义化地去表达自己的逻辑，链式调用是一种常见的处理方式。我们常用的 Masonry、 Expecta 等第三方库就采用了这种处理方式。

``` objective-c
// Masonry
[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(superview.mas_top).with.offset(padding.top);
    make.left.equalTo(superview.mas_left).with.offset(padding.left);
    make.bottom.equalTo(superview.mas_bottom).with.offset(-padding.bottom);
    make.right.equalTo(superview.mas_right).with.offset(-padding.right);
}];
```

``` objective-c
// Expecta
expect(@"foo").to.equal(@"foo"); // `to` is a syntactic sugar and can be safely omitted.
expect(foo).notTo.equal(1);
expect([bar isBar]).to.equal(YES);
expect(baz).to.equal(3.14159);
```

像这种用于特定领域的表达方式，我们叫做 DSL (Domain Specific Language)，本文就介绍一下如何实现一个链式调用的 DSL.

## 链式调用的实现

我们举一个具体的例子，比如我们用链式表达式来创建一个 UIView，设置其 frame、backgroundColor， 并添加至某个父 View。
对于最基本的 Objective-C (在 iOS4 block 出现之前)，如果要实现链式调用，只能是这个样子的：

``` objective-c
UIView *aView = [[[[UIView alloc] initWithFrame:aFrame] bgColor:aColor] intoView:aSuperView];
```

有了 block，我们可以把中括号的这种写法改为点语法的形式

``` objective-c
UIView *aView = AllocA(UIView).with.position(x, y).size(width, height).bgColor(aColor).intoView(aSuperView);

// 当x和y为默认值0和0或者width和height为默认值0的时候，还可以省略
UIView *bView = AllocA(UIView).with.size(width, height).bgColor(aColor).intoView(aSuperView);
```

可以看出，链式语法的语义性很明确，后者的语法更加紧凑，下面我们从两个角度看一下后者的实现。

### 1. 从语法层面来看

链式调用可以用两种方式来实现：

1. 在返回值中使用属性来保存方法中的信息

   比如，Masonry 中的 `.left .right .top .bottom` 等方法，调用时会返回一个 `MASConstraintMaker` 类的实例，里面有 `left/right/top/bottom` 等属性来保存每次调用时的信息；

   ```objective-c
   make.left.equalTo(superview.mas_left).with.offset(15);
   ```

   再比如，Expecta 中的方法 `.notTo` 方法会返回一个 `EXPExpect` 类的实例，里面有个 BOOL 属性 `self.negative` 来记录是否调用了 `.notTo`；

   ```objective-c
   expect(foo).notTo.equal(1);
   ```

   再比如，上例中的 .with 方法，我们可以直接 `return self;` 

2. 使用 block 类型的属性来接受参数

   比如 Masonry 中的 `.offset(15)` 方法，接收一个 CGFloat 作为参数，可以在 `MASConstraintMaker` 类中添加一个 block 类型的属性：

   ```objective-c
   @property (nonatomic, copy) MASConstraintMaker* (^offset)(CGFloat);
   ```

   比如例子中的 `.position(x, y)`，可以给的某类中添加一个属性：

   ```objective-c
   @property (nonatomic, copy) ViewMaker* (^position)(CGFloat x, CGFloat y);
   ```
   在调用 `.position(x, y)` 方法时，执行这个block，返回 ViewMaker 的实例保证链式调用得以进行。


### 2. 从语义层面来看

从语义层面上，需要界定哪些是助词，哪些是需要接受参数的。为了保证链式调用能够完成，需要考虑传入什么，返回什么。

还是以上面的例子来讲：

```objective-c
UIView *aView = AllocA(UIView).with.position(x, y).size(width, height).bgColor(aColor).intoView(aSuperView);
```

分步来看一下，这个 DSL 表达式需要描述的是一个祈使句，以 Alloc 开始，以 intoView 截止。在 intoView 终结语之前，我们对 UIView 进行一定的修饰，利用 `position` `size` `bgColor` 这些。

下面我们分别从四段来看，如何实现这样一个表达式：

#### (1) 宾语

在 AllocA(UIView) 的语义中，我们确定了宾语是 a UIVIew。由于确定 UIView 是在 intoView 截止那时，所以我们需要创建一个中间类来保存所有的中间条件，这里我们用 ViewMaker 类。

``` objective-c
@interface ViewMaker : NSObject
@property (nonatomic, strong) Class viewClass;
@property (nonatomic, assign) CGPoint position;
@property (nonatomic, assign) CGPoint size;
@property (nonatomic, strong) UIColor *color;
@end

```

另外我们可以注意到AllocA是一个函数，而UIView无法直接传递到这个函数中，语法就要变成 `AllocA([UIView class])` 而失去了简洁性。所以我们需要先定义一个宏来“吞”掉中括号和 `class` 这个方法：


```objective-c

#define AllocA(aClass)  alloc_a([aClass class])

ViewMaker* alloc_a(Class aClass){
    ViewMaker *maker = ViewMaker.new;
    maker.viewClass = aClass;
    return maker;
}
```

#### (2) 助词

很多时候，为了让 DSL 的语法看起来更加连贯，我们需要一些助词来帮助，例如 Masonry 里面的 make.top.equalTo(superview.mas_top).**with**.offset(padding.top) 这句中的 with 就是这样一个助词。

而这个助词和我们学过的语法一样，通常没有什么实际效果，简单返回self就可以。

``` objective-c
@interface ViewMaker : NSObject
@property (nonatomic, strong) Class viewClass;
@property (nonatomic, assign) CGPoint position;
@property (nonatomic, assign) CGPoint size;
@property (nonatomic, strong) UIColor *color;
@property (nonatomic, readonly) ViewMaker *with;
@end

@implementation ViewMaker

- (ViewMaker *)with
{
    return self;
}
@end

```

需要注意的是，返回自己，就没有办法阻止用户不断调用自己 `.with.with.with` ，为了避免这种情况，可以新生成一个类，每个类都拥有自己所在层次的方法，避免跃层调用。

```objective-c
@interface ViewMaker : NSObject
@property (nonatomic, strong) Class viewClass;
@property (nonatomic, assign) CGPoint position;
@property (nonatomic, assign) CGPoint size;
@property (nonatomic, strong) UIColor *color;
@end

@interface ViewClassHelper : NSObject
@property (nonatomic, strong) Class viewClass;
@property (nonatomic, readonly) ViewMaker *with;
@end

#define AllocA(aClass)  alloc_a([aClass class])

ViewClassHelper* alloc_a(Class aClass){
    ViewClassHelper *helper = ViewClassHelper.new;
    helper.viewClass = aClass;
    return helper;
}
@implementation ViewClassHelper

- (ViewMaker *)with
{
    ViewMaker *maker = ViewMaker.new;
    maker.viewClass = self.viewClass;
    return maker;
}
@end

```

这样就有效防止了，`.with.with.with`这样的语法。但是实际上，我们要根据真实的需要来进行开发，使用 DSL 的用户是为了更好的表达性，所以并不会写出`.with.with.with`这样的代码，这样的防护性措施就显得有点不必要了。

不过使用类来区分助词还有另外几个小好处，就是它可以确保在语法提示的时候，`ViewClassHelper`这个类只有`.with`这样一个语法提示，而`ViewMaker`不出现`.with`语法提示；并且同时确保`.with`一定要出现。

不过为了简化文章，我们都使用前者，既`.with`返回`self`来继续下文：

``` objective-c
@interface ViewMaker : NSObject
@property (nonatomic, strong) Class viewClass;
@property (nonatomic, assign) CGPoint position;
@property (nonatomic, assign) CGPoint size;
@property (nonatomic, strong) UIColor *color;
@property (nonatomic, readonly) ViewMaker *with;
@end

@implementation ViewMaker

- (ViewMaker *)with
{
    return self;
}
@end

```

#### (3) 修饰部分——定语

像例子中的`position` `size` `bgColor`这些都是定语部分，用来修饰UIView，他们以属性的形势存在于`ViewMaker`的实例中，为了支持链式表达，所以实现的时候，都会继续返回`self`。

我们来试着实现下：

``` objective-c
@interface ViewMaker : NSObject
// ...
@property (nonatomic, copy) ViewMaker* (^position)(CGFloat x, CGFloat y);
@property (nonatomic, copy) ViewMaker* (^size)(CGFloat x, CGFloat y);
@property (nonatomic, copy) ViewMaker* (^bgColor)(UIColor *color);
@end

@implementation ViewMaker

- (instancetype)init
{
    if (self = [super init]) {
        @weakify(self)
        _position = ^ViewMaker *(CGFloat x, CGFloat y) {
            @strongify(self)
            self.position = CGPointMake(x, y);
        };
        _size = ^ViewMaker *(CGFloat x, CGFloat y) {
            @strongify(self)
            self.size = CGPointMake(x, y);
        };
        _bgColor = ^ViewMaker *(UIColor *color) {
            @strongify(self)
            self.color = color;
        };
    }
    return self;
}
@end

```

#### (4) 终结词

“终结词”这个实在是在现代语法里面找不到对应关系了，但是在 DSL 中，这一段尤为重要。`ViewMaker`的实例从头至尾收集了很多的修饰，需要最后的一个表达词语来产生最后的结果，这里就称为”终结词”。例如在 Expecta 这个开源库里面的 `equal` 就是把真正的行为表现出来的时候，`to` 和 `notTo` 都不会真正触发行为。

在我们的例子里，终结词`.intoView(aSuperViwe)`可以这样实现：

``` objective-c
@interface ViewMaker : NSObject
// ...
@property (nonatomic, copy) UIView* (^intoView)(UIView *superView);
@end

@implementation ViewMaker

- (instancetype)init
{
    if (self = [super init]) {
        @weakify(self)
        // ...
        _intoView = ^UIView *(UIView *superView) {
            @strongify(self)
            CGRect rect = CGRectMake(self.position.x, self.position.y,
                         self.size.width, self.size.height);
            UIView *view = [[UIView alloc] initWithFrame:rect];
            view.backgroundColor = self.color;
            [superView addSubView:view];
            return view;
        };
    }
    return self;
}
@end

```

这样，一个终结词就写好了。

最终代码的汇总：

``` objective-c
@interface ViewMaker : NSObject
@property (nonatomic, strong) Class viewClass;
@property (nonatomic, assign) CGPoint position;
@property (nonatomic, assign) CGPoint size;
@property (nonatomic, strong) UIColor *color;
@property (nonatomic, readonly) ViewMaker *with;
@property (nonatomic, copy) ViewMaker* (^position)(CGFloat x, CGFloat y);
@property (nonatomic, copy) ViewMaker* (^size)(CGFloat x, CGFloat y);
@property (nonatomic, copy) ViewMaker* (^bgColor)(UIColor *color);
@property (nonatomic, copy) UIView* (^intoView)(UIView *superView);
@end

#define AllocA(aClass)  alloc_a([aClass class])

ViewMaker* alloc_a(Class aClass){
    ViewMaker *maker = ViewMaker.new;
    maker.viewClass = aClass;
    return maker;
}

@implementation ViewMaker

- (instancetype)init
{
    if (self = [super init]) {
        @weakify(self)
        _position = ^ViewMaker *(CGFloat x, CGFloat y) {
            @strongify(self)
            self.position = CGPointMake(x, y);
        };
        _size = ^ViewMaker *(CGFloat x, CGFloat y) {
            @strongify(self)
            self.size = CGPointMake(x, y);
        };
        _bgColor = ^ViewMaker *(UIColor *color) {
            @strongify(self)
            self.color = color;
        };
        _intoView = ^UIView *(UIView *superView) {
            @strongify(self)
            CGRect rect = CGRectMake(self.position.x, self.position.y,
                         self.size.width, self.size.height);
            UIView *view = [[UIView alloc] initWithFrame:rect];
            view.backgroundColor = self.color;
            [superView addSubView:view];
            return view;
        };
    }
    return self;
}

- (ViewMaker *)with
{
    return self;
}
@end


```
## 总结

这种链式调用能够使程序更加清晰，在特定场景下使程序的可读性更强。这种手段在Swift也是相同道理，大家可以善加利用，让自己的代码更加美观。