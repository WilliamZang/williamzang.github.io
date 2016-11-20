---
layout: post
title: "把玩高阶函数"
date: 2016-11-08 13:19:42 +0800
comments: true
categories: iOS 函数式编程
description: 聊一聊iOS开发中的高阶函数
keywords: 高阶函数 iOS 函数式编程
---

如果你开始接触函数式编程，你一定听说过高阶函数。在维基百科它的中文解释是这样的：

> 在数学和计算机科学中，高阶函数是至少满足下列一个条件的函数：  
>
> * 接受一个或多个函数作为输入  
>
> * 输出一个函数

看起它就是ObjC语言中入参或者返回值为block的block或者函数，在Swift语言中即为入参或者返回值为函数的函数。那它们在实际的开发过程中究竟起着什么样的作用呢？我们将从入参、返回值和综合使用三部分来看这个问题：

## 函数作为入参

函数作为入参似乎无论在ObjC时代还是Swift时代都是司空见惯的事情，例如`AFNetworking`就用两个入参block分别回调成功与失败。Swift中更是加了一个尾闭包的语法（最后一个参数为函数，可以不写括号或者写到括号外面直接跟随方法名），例如下面这样：

``` Swift
[1, 2, 3].forEach { item in  
    print(item)  
}  

```

我们可以将入参为函数的函数分为两类，escaping函数入参和noescape函数入参，区别在于这个入参的函数是在执行过程内被调用还是在执行过程外被调用。执行过程外被调用的一般用于callback用途，例如：

``` Swift
Alamofire.request("https://httpbin.org/get").responseJSON { response in  
    print(response.request)  // original URL request  
    print(response.response) // HTTP URL response  
    print(response.data)     // server data  
    print(response.result)   // result of response serialization  

    if let JSON = response.result.value {  
        print("JSON: \(JSON)")  
    }  
}

```

这个response的入参函数就作为网络请求回来的一个callback，并不会在执行responseJSON这个函数的时候被调用。另外我们来观察`forEach`的代码，可以推断入参的函数一定会在`forEach`执行过程中使用，执行完就没有利用意义，这类就是noescape函数。

callback的用法大家应该比较熟悉了，介绍给大家noescape入参的一些用法：

### 1. 自由构造器

看过GoF设计模式的同学不知道是否还记得构造器模式，Android中的构造器模式类似如下：

``` Java
new AlertDialog.Builder(this)  
  .setIcon(R.drawable.find_daycycle_icon)  
  .setTitle("提醒")  
  .create()  
  .show();
```

如果你想要做成这样的代码，你需要将`setIcon`、`setTitle`、`create`等方法都实现成返回`this`才行。这样就无法直接利用无返回值的`setter`了。
为什么需要这样的方式呢？如果你同时有如下需求：

1. 构造一个对象需要很多的参数
2. 这些参数里面很多有默认值
3. 这些参数对应的属性未来不希望被修改

那么用这样的模式就可以直观又精巧的展示构建过程。

如果使用noescape入参函数还可以更简单的构造出这种代码，只需要传入一个入参为builder的对象就可以了，如下：

``` Swift
// 实现在这里  
class SomeBuilder {  
    var prop1: Int  
    var prop2: Bool  
    var prop3: String  
    init() {  
        // default value  
        prop1 = 0  
        prop2 = true  
        prop3 = "some string"  
    }  
}  
  
class SomeObj {  
    private var prop1: Int  
    private var prop2: Bool  
    private var prop3: String  
    init(_ builderBlock:(SomeBuilder) -> Void) {  
        let someBuilder = SomeBuilder()  
        builderBlock(someBuilder) // noescape 入参的使用  
        prop1 = someBuilder.prop1  
        prop2 = someBuilder.prop2  
        prop3 = someBuilder.prop3  
    }  
}  
  
// 使用的时候  
let someOjb = SomeObj { builder in  
    builder.prop1 = 15  
    builder.prop2 = false  
    builder.prop3 = "haha"  
}
```

### 2. 自动配对操作

很多时候，我们开发过程中都会遇到必须配对才能正常工作的API，例如打开文件和关闭文件、进入edit模式退出edit模式等。虽然swift语言给我们defer这样的语法糖避免大家忘记配对操作，但是代码看起来还是不那么顺眼

``` Swift
func updateTableView1() {  
    self.tableView.beginUpdates()  
  
    self.tableView.insertRows(at: [IndexPath(row: 2, section: 0)], with: .fade)  
    self.tableView.deleteRows(at: [IndexPath(row: 5, section: 0)], with: .fade)  
  
	self.tableView.endUpdates() // 容易漏掉或者上面出现异常  
	
}  
  
func updateTableView2() {  
    self.tableView.beginUpdates()  
    defer {  
        self.tableView.endUpdates()  
    }  
  
    self.tableView.insertRows(at: [IndexPath(row: 2, section: 0)], with: .fade)  
    self.tableView.deleteRows(at: [IndexPath(row: 5, section: 0)], with: .fade)  
}  
```

利用noescape入参，我们可以将要操作的过程封装起来，使得上层看起来更规整

``` Swift
// 扩展一下UITableView  
extension UITableView {  
    func updateCells(updateBlock: (UITableView) -> Void) {  
        beginUpdates()  
        defer {  
            endUpdates()  
        }  
        updateBlock(self)  
    }  
}  
  
func updateTableView() {  
	// 使用的时候  
    self.tableView.updateCells { (tableView) in  
        tableView.insertRows(at: [IndexPath(row: 2, section: 0)], with: .fade)  
        tableView.deleteRows(at: [IndexPath(row: 5, section: 0)], with: .fade)  
    }  
}  

```

函数作为入参就简单介绍到这里，下面看看函数作为返回值。

## 函数作为返回值

在大家的日常开发中，函数作为返回值的情况想必是少之又少。不过，如果能简单利用起来，就会让代码一下子清爽很多。

首先没有争议的就是我们有很多的API都是需要函数作为入参的，无论是上一节提到过的escaping入参还是noescape入参。所以很多的时候，大家写的代码重复率会很高，例如：

``` Swift
let array = [1, 3, 55, 47, 92, 77, 801]  
  
let array1 = array.filter { $0 > 3 * 3}  
let array2 = array.filter { $0 > 4 * 4}  
let array3 = array.filter { $0 > 2 * 2}  
let array4 = array.filter { $0 > 5 * 5}  
```

一段从数组中找到大于某个数平方的代码，如果不封装，看起来应该是这样的。为了简化，通常会封装成如下的两个样子：

``` Swift
func biggerThanPowWith(array: [Int], value: Int) -> [Int] {  
    return array.filter { $0 > value * value}  
}  
  
let array1 = biggerThanPowWith(array: array, value: 3)  
let array2 = biggerThanPowWith(array: array, value: 4)  
let array3 = biggerThanPowWith(array: array, value: 2)  
let array4 = biggerThanPowWith(array: array, value: 5)  

```

如果用高阶函数的返回值函数，可以做成这样一个高阶函数：

``` Swift
// 一个返回(Int)->Bool的函数  
func biggerThanPow2With(value: Int) -> (Int) -> Bool {  
    return { $0 > value * value }  
}  
  
let array1 = array.filter(biggerThanPow2With(value: 3))  
let array2 = array.filter(biggerThanPow2With(value: 4))  
let array3 = array.filter(biggerThanPow2With(value: 2))  
let array4 = array.filter(biggerThanPow2With(value: 5))  
```

你一定会说，两者看起来没啥区别。所以这里面需要讲一下使用高阶返回函数的几点好处

#### 1. 不需要wrapper函数也不需要打开原始类

如同上面的简单封装，其实就是一个wrapper函数，把`array`作为入参带入进来。这样写代码和看代码的时候就稍微不爽一点，毕竟大家都喜欢OOP嘛。如果要OOP，那就势必要对原始类进行扩展，一种方式是加extension，或者直接给类加一个新的方法。

#### 2. 阅读代码的时候一目了然

使用简单封装的时候，看代码的人并不知道内部使用了`filter`这个函数，必须要查看源码才能知道。但是用高阶函数的时候，一下子就知道了使用了系统库的`filter`。

#### 3. 更容易复用

这也是最关键的一点，更细粒度的高阶函数，可以更方便的复用，例如我们知道Set<Int>也是有`filter`这个方法的，复用起来就这样：
	
``` Swift
let set = Set<Int>(arrayLiteral: 1, 3, 7, 9, 17, 55, 47, 92, 77, 801)  
let set1 = set.filter(biggerThanPow2With(value: 3))  
let set2 = set.filter(biggerThanPow2With(value: 9))	  
```

回忆下上面的简单封装，是不是就无法重用了呢？

类似的返回函数的高阶函数还可以有很多例子，例如上面说过的builder，假如每次都需要定制成特殊的样子，但是某个字段不同，就可以用高阶函数很容易打造出来：

``` Swift
func builerWithDifferentProp3(prop3: String) -> (SomeBuilder) -> Void {  
    return { builder in  
        builder.prop1 = 15  
        builder.prop2 = true  
        builder.prop3 = prop3  
    }  
}  
  
let someObj1 = SomeObj.init(builerWithDifferentProp3(prop3: "a"))  
let someObj2 = SomeObj.init(builerWithDifferentProp3(prop3: "b"))  
let someObj3 = SomeObj.init(builerWithDifferentProp3(prop3: "c"))  

```

介绍完入参与返回值，还有另外的一个组合模式，那就是入参是一个函数，返回值也是一个函数的情况，我们来看看这种情况。

## 入参函数 && 返回值函数

这样的一个函数看起来会很恐怖，swift会声明成`func someFunc<A, B, C, D>(_ a: (A) -> B)-> (C) -> D`，objective-c会声明成`- (id (^)(id))someFunc:(id (^)(id))block`。让我们先从一个小的例子来讲起，回忆一下我们刚刚做的`biggerThanPow2With`这个函数，如果我们要一个`notBiggerThanPow2With`怎么办呢？你知道我一定不会说再写一个。所以我告诉你我会这样写：

``` Swift
func not<T>(_ origin_func: @escaping (T) -> Bool) -> (T) -> Bool {  
    return { !origin_func($0) }  
}  
  
let array5 = array.filter(not(biggerThanPow2With(value: 9)))  
  
  
```

并不需要一个`notBiggerThanPow2With`函数，我们只需要实现一个`not`就可以了。它的入参是一个`(T) -> Bool`，返回值也是`(T) -> Bool`，只需要在执行block内部的时候用个取反就可以了。这样不单可以解决刚才的问题，还可以解决任何`(T) -> Bool`类型函数的取反问题，比如我们有一个`odd(_: int)`方法来过滤奇数，那我们就可以用`even=not(odd)`得到一个过滤偶数的函数了。

``` Swift
func odd(_ value: Int) -> Bool {  
    return value % 2 == 1  
}  
  
let array6 = array.filter(odd)  
let array7 = array.filter(not(odd))  
  
let even = not(odd)  
let array8 = array.filter(even)  
```

大家可以看下上面的`biggerThanPow2With`时我们讨论过的，如果`biggerThanPow2With`不是一个返回函数的高阶函数，那它就不太容易用`not`函数来加工了。

综上，如果一个入参和返回值都是函数的函数就是这样的一个转换函数，它能够让我们用更少的代码组合出更多的函数。另外需要注意一下，如果返回的函数里面闭包了入参的函数，那么入参函数就是escaping入参了。

下面再展示给大家两个函数，一个交换参数的函数`exchangeParam`，另一个是柯里化函数`currying`：

``` Swift
func exchangeParam<A, B, C>(_ block: @escaping (A, B) -> C) -> (B, A) -> C {  
    return { block($1, $0) }  
}  
  
func currying<A, B, C>(_ block: @escaping (A, B) -> C, _ value: A) -> (B) -> C {  
    return { block(value, $0) }  
}  


```

第一个函数`exchangeParam`是交换一个函数的两个参数，第二个函数`currying`是给一个带两个参数的函数和一个参数，返回一个带一个参数的函数。那这两个函数究竟有什么用途呢？看一下下面的例子：

``` Swift
let array9 = array.filter(currying(exchangeParam(>), 9))
```

swift语言里面`>`是一个入参(a, b)的函数，所以`>(5, 3) == true`。我们使用`exchangeParam`交换参数就变成了(b, a)，这时`exchangeParam(>)(5, 3)`就等于`false`了。

而currying函数又把参数`b`固定为一个常量9，所以`currying(exchangeParam(>), 9)`就是大于9的函数意思。

这个例子里就利用了全部的预制函数和通用函数，没有借助任何的命令与业务函数声明实现了一个从数组中过滤大于9的子数组的需求。试想一下，如果我们更多的使用这样的高阶函数，代码中是不是很多的逻辑可以更容易的互相组合，而这就是函数式编程的魅力。

## 总结

高阶函数的引入，无论是从函数式编程还是从非函数式编程都带给我们代码一定程度的简化，使得我们的API更加简易可用，复用更充分。然而本文的例子不过是冰山一角，更多的内容还需要大家的不断尝试和创新，也可以通过学习更多的函数式编程范式来加深理解。







