---
layout: post
title: "介绍一下开源项目FastAnimationWithPOP"
date: 2014-07-23 16:07:04 +0800
comments: true
categories: iOS 开源介绍
description: 我的开源项目FastAnimationWithPOP介绍
keywords: Github POP iOS
---

这是一个很简单的动画框架，基于Facebook的POP库。使用它你就可以在故事版中以0行代码的代价来添加动画了。

Github上地址是 [这里](https://github.com/WilliamZang/FastAnimationWithPOP).

你可以从[这里](https://github.com/WilliamZang/FastAnimationWithPopDemo)下载DEMO查看效果.


**如果你觉得不错，欢迎在到[这里](https://github.com/WilliamZang/FastAnimationWithPOP)点个赞，方便让更多人注意到它**

![Demo](https://raw.githubusercontent.com/WilliamZang/FastAnimationWithPOP/master/Docs/demo.gif)

## 功能
* 使用属性来添加一个动画到任意的View。
* 在nib或者故事版唤醒时自动执行动画。
* 也可以随时手动执行动画。
* 控制动画的细节。
* 给control绑定一些动画，例如按下松开等状态。
* 轻松的扩展新的动画，只需要实现`FastAnimationProtocol`、`ControlFastAnimationProtocol` 和 `FastAnimationReverseProtocol`这几个协议.

## 环境要求

iOS SDK: iOS 6.0+

XCode版本: 5.0+

## 如何安装
最好的办法是使用[CocoaPods](http://cocoadocs.org):

1. 添加这行到你的`podfile`文件 `pod 'FastAnimation'`

2. 安装更新 `pod install`

如果想要尝试最新的版本，你可以添加这个`pod 'FastAnimation', :head`.

## 使用指导

### 1. 在故事板里使用

你可以通过设置用户自定义运行时属性(user defined runtime attributes)给View添加一个动画。

![StroyBoard1](https://raw.githubusercontent.com/WilliamZang/FastAnimationWithPOP/master/Docs/stroyBoard1.png)

![StroyBoard2](https://raw.githubusercontent.com/WilliamZang/FastAnimationWithPOP/master/Docs/stroyBoard2.png)

下面是一些属性的含义：

#### UIView的属性

* animationType
 
	通过这个属性来指定动画的类型，可以是完整的类名，也可以省略`FAAnimation`前缀.
	
* delay

	执行动画的延时，以秒为单位。
	
* animationParams

	这个是各个动画的灵活参数，你可以从动画类的头文件中找到信息，例如下面：

```objective-c
#define kSpringBounciness   (@"animationParams.springBounciness")
#define kSpringSpeed        (@"animationParams.springSpeed")
#define kDynamicsTension    (@"animationParams.dynamicsTension")
#define kDynamicsFriction   (@"animationParams.dynamicsFriction")
#define kDynamicsMass       (@"animationParams.dynamicsMass")
```

* startAnimationWhenAwakeFromNib

	定义是否需要在故事板唤醒的时候就执行动画，默认是`YES`。

#### UIControl的属性

* bindingAnimationType

	通过这个属性来指定控件动画的类型，可以是完整的类名，也可以省略`FAAnimation`前缀.


### 2. 代码写View的应用

在代码写View中使用FastAnimation同样方便。

你可以设置动画类型等属性，然后执行```- (void)startFAAnimation```即可。就像这样：

```objective-c
UIView *view = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 100, 100)];
view.backgroundColor = [UIColor redColor];
view.animationType = @"Shake";
view.animationParams[@"velocity"] = @-7000;
// You can also set params like this
// [view setValue:@-7000 forKeyPath:kShakeVelocity];
[view startFAAnimation];
```

还有这些扩展的用法：

```objective-c
// In UIView instance.
- (void)startFAAnimation;
- (void)reverseFAAnimation;
// In UIControl instance.
- (void)bindingFAAnimation;
- (void)unbindingFAAnimation;
```
### 3. 定义一个新的动画扩展

轻松的扩展新的动画，只需要实现`FastAnimationProtocol`、`ControlFastAnimationProtocol` 和 `FastAnimationReverseProtocol`这几个协议.

就像这样：

```objective-c
// new_animation.h
@interface FAAnimationNewAnimation : NSObject<FastAnimationProtocol, 
FastAnimationReverseProtocol> // Maybe only FastAnimationProtocol

@end
// new_animation.m
@implementation FAAnimationBounceRight

+ (void)performAnimation:(UIView *)view
{
    // some thing you like.
}

+ (void)stopAnimation:(UIView *)view
{
    // some thing you like.
}

+ (void)reverseAnimation:(UIView *)view
{
     // some thing you like.
}

+ (void)stopReverse:(UIView *)view
{
     // some thing you like.
}
@end

```

### 4. 一些控制动画的操作

* 停止动画：

如果想要手动体制，使用下面的方法：

```objective-c
- (void)stopFAAnimation;
- (void)stopReverseFAAnimation;
```

* 嵌套动画：

使用如下方法处理嵌套：

```objective-c
- (void)startFAAnimationNested;
- (void)stopFAAnimationNested;
- (void)reverseFAAnimationNested;
- (void)stopReverseFAAnimationNested;
```

### 目前已经拥有的动画:

* 反弹动画（4方向）: `BounceLeft`,`BounceRight`,`BounceUp`,`BounceDown`
* 放大动画（2方向）：`ZoomInX`,`ZoomInY`
* 颤动动画
* 组动画
* 放大动画
* Button的放大效果绑定
* **更多的动画等着大家的贡献哟！**

## 下一步要做的事

* 把DEMO和库项目和到同一个Workspace里。
* 制作更多更好看的DEMO。
* 假如便捷的转场动画，目前先设法支持iOS7+
* 确保所有的功能都含有单元测试。
* 更多更好的动画。
* 把核心部分和效果部分分离，效果按照iOS5 6 7+来打成不同的包.
* 支持Swift写扩展.

