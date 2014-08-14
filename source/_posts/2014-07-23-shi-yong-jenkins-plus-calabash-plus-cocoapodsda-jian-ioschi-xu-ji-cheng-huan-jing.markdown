---
layout: post
title: "使用jenkins+calabash+cocoapods搭建ios持续集成环境"
date: 2014-07-23 16:19:58 +0800
comments: true
categories: 杂谈 教程
description: 持续集成的介绍
keywords: BDD Jenkins iOS Test
---

## 持续集成
持续集成究竟是什么呢？根据敏捷大师Martin Fowler的定义:

> 持续集成是一种软件开发实践。在持续集成中，团队成员频繁集成他们的工作成果，一般每人每天至少集成一次，也可以多次。每次集成会经过自动构建（包括自动测试）的检验，以尽快发现集成错误。许多团队发现这种方法可以显著减少集成引起的问题，并可以加快团队合作软件开发的速度。

只要是开发就有分工，哪怕是自己一个写也要分成多个模块。随着项目越来越大，模块也越来越多，各个模块是否可以征程协作就成了问题，有了持续集成，可以有如下好处：

1. 持续集成中的任何一个环节都是自动完成的，无需太多的人工干预，有利于减少重复过程以节省时间、费用和工作量；  
2. 持续集成保障了每个时间点上团队成员提交的代码是能成功集成的。换言之，任何时间点都能第一时间发现软件的集成问题，使任意时间发布可部署的软件成为了可能；  
4. 持续集成还能利于软件本身的发展趋势，这点在需求不明确或是频繁性变更的情景中尤其重要，持续集成的质量能帮助团队进行有效决策，同时建立团队对开发产品的信心。  

下面就给大家介绍，如何使用Jenkins+Calabash搭建持续集成开发环境。

## 环境
XCode 5.0

Mac OS X 10.9.2

## Cocoapods

###CocoaPods简介

CocoaPods是一个负责管理iOS项目中第三方开源代码的工具。CocoaPods项目的源码在Github上管理。该项目开始于2011年8月12日，经过一年多的发展，现在已经超过1000次提交，并且持续保持活跃更新。开发iOS项目不可避免地要使用第三方开源库，CocoaPods的出现使得我们可以节省设置和更新第三方开源库的时间。
### 安装Cocoapods

#### 安装Homebrew
[Homebrew](http://brew.sh)是Mac下著名的包管理工具，RVM和以后用到xctool都需要用这个来安装，相当于Ubuntu的Apt-get。

安装方法是在命令行中键入

	ruby -e "$(curl -fsSL https://raw.github.com/Homebrew/homebrew/go/install)"
	
之后执行环境检查

	brew doctor
	
检查没有错误就可以使用了，如果出现错误，请参考提示进行修正。

确认无误后，可以安装第一个应用curl，一个用来下载的工具。使用命令

	brew install curl

#### 安装RVM

虽然Mac默认都带有Ruby，但是有些时候使用起来很麻烦(例如必须使用sudo来安装gem)并且只有一个版本，所以我们使用[RVM](http://rvm.io)来管理ruby的版本，ruby是自动化测试工具calabash的运行环境，所以必须要有。

安装方法是命令行中键入

	\curl -sSL https://get.rvm.io | bash -s stable
	
_过程中可能需要输入sudo密码。_

使用淘宝源替换

	sed -i .bak 's!cache.ruby-lang.org/pub/ruby!ruby.taobao.org/mirrors/ruby!' $rvm_path/config/db
	
#### 安装Ruby
使用rvm下载ruby2.0版本

	rvm install 2.0.0
	
选用2.0.0版本的ruby，并设置为默认

	rvm use 2.0.0 --default

使用淘宝源替换gem源

	rvm source --add http://ruby.taobao.org/
	rvm source --remove https://rubygems.org/
		
#### 安装Cocoapods

CocoaPods是一个用来帮助我们管理第三方依赖库的工具。它可以解决库与库之间的依赖关系，下载库的源代码，同时通过创建一个Xcode的workspace来将这些第三方库和我们的工程连接起来，供我们开发使用。

通过Gem安装Cocoapods

	gem install cocoapods
	
执行cocoapods的初始化

	pod setup
	
_该过程需要到github上拉取specs，速度很慢，可以喝杯咖啡慢慢等_

### 使用Cocoapods

首先创建一个普通项目来演示下如何使用Cocoapods。

![创建项目1](/images/创建项目1.png)  
![创建项目2](/images/创建项目2.png)

之后在命令行里面,进入到你的项目路径

	cd /path/to/your/project
	pod init
	
之后会在项目根目录下创建好Podfile，修改下Podfile的内容

```
 # #为Podfile的注释行，Podfile实际上是一个ruby代码段
 platform :ios, "6.0" # platform后面跟平台和版本号，这里是ios6平台
 
 # pod 'MKNetworkKit' 像这样写就可以引入第三方库了，为了简化，这里没有引入任何库
```	

在目录执行pod插件install命令

	pod install 

_每次使用pod install，它都会到github上更新spec库，耗费了不少时间，可以使用下面的命令跳过这个过程_

	pod install --no-repo-update
	
执行之后，会提示没有引入任何的第三方库，不要担心(因为我们真的没有引入)。你会发现目录上多了integration_test.xcworkspace这个工作区文件，以后我们就都使用这个打开项目了。

打开后如图所示  
![引入Pod后的工程](/images/引入Pod后的工程.png)

恭喜您，已经可以正常使用Cocoapods了。下一步就是使用Calabash进行自动化测试了。

## Calabash
[Calabash](http://calaba.sh)是一款开源的跨平台UI测试工具，目前支持iOS和Android。它使用[Cucumber](http://cukes.info)作为测试核心，Cucumber是一个在敏捷团队十分流行的自动化的功能测试工具，它使用接近于自然语言的特性文档进行用例的书写和测试，支持多语言和多平台。

### 安装Calabash

	gem install calabash-cucumber

### 安装Calabash中文支持包
	
	gem install calabash-cucumber-ios-cn

### 新建集成测试的Target

重新打开工作区，然后选择integration_test这个工程，打开配置，targets中integration_test上右键进行复制。  
![新建集成测试目标](/images/新建集成测试目标.png)  
_如果出现Duplicate iPhone Target对话框，选择Duplicate Only就可以，另外一个选项是复制并转换成iPad程序。_

之后修改目标的名称  
![修改目标名称](/images/修改目标名称.png)

修改项目配置
![修改项目配置](/images/修改项目配置.png)

修改scheme  
![修改scheme1](/images/修改scheme1.png)  
![修改scheme2](/images/修改scheme2.png)

共享scheme，目的是在版本管理中，让其他用户也可以获取到这些scheme  
![共享scheme](/images/共享scheme.png)

这样新的测试目标就创建好了，为什么要创建新的目标呢？

1. 不希望在发布的产品中包含测试代码
2. calabash默认启动-cal结尾的目标

### 引入Calabash包

修改Podfile文件，加入新的pod

```
target 'integration_test-cal', exclusive: false do
  pod 'Calabash'
end
```

到命令行里，进入到自己的目录，执行

	pod install --no-repo-update

执行成功后，创建用例模板

	calabash-ios gen
	
屏幕会出现

```
----------Question----------
I'm about to create a subdirectory called features.
features will contain all your calabash tests.
Please hit return to confirm that's what you want.
---------------------------
```

按回车确认，就生成了features文件夹，我们的用例和测试配置都在这里了。你可以把features这个文件夹拖到项目中，方便使用xcode直接编辑，**注意不要选择任何目标**，以为这些文件根本没有必要编译和存到app中。

### 编写用例

Cucumber是使用[gherkin](https://github.com/cucumber/gherkin)来进行用例描述的，这是一种近乎自然语言的脚本，并且对多语言有很好的支持。具体的语法可以查阅它的[官方wiki](https://github.com/cucumber/cucumber/wiki/Gherkin)。

这里我们先写一个简单用例，修改features/my_first.feature

```
# language: zh-CN  
功能: 运行基准测试
  做为一个iOS开发者
  我希望有一个简单的基准测试
  使我可以快速的开启测试

场景: 基准测试
  假如 应用正在运行
  那么 我把应用切到后台3秒
```

是的，就是这样的用例！你可以书写自然语言来描述一个功能，calabash就使用cucumber帮您测试了，神奇吧。

接下来还需要修改features/step_definitions/calabash_steps.rb，在这里包含中文解析，在最下面加上

	require 'calabash-cucumber-ios-cn/calabash_steps.rb'

这个包里面带有中文的功能说明，具体可以看[文档](https://github.com/cpunion/calabash-cucumber-cn/blob/master/PredefinedSteps.md)。

### 执行用例

激动人心的时刻终于到了，首先编译integration_test-cal这个scheme，然后使用模拟器运行一下，在模拟器打开Accessibility Inspector。模拟器->设置(Settings)->通用(General)->辅助功能(Accessibity)->Accessibility Inspector开启。

打开命令行，进到目录中执行命令

	cucumber
	
可能需要输入密码，之后就看到模拟器重新加载，并按照我们的用例开始自动执行了。

执行结束后，会有下图的结果。

![结果截图](/images/结果截图.png)

恭喜我们的2个步骤都成功了。快点用更多的功能和用例来测试吧^_^。

到这里，每个开发人员都可以通过cucumber命令来对自己写的内容进行测试了，这和我们的持续集成还有一段距离，那么接下来，我们介绍Jenkins这个持续集成web工具，实现真正的持续集成。

## Jenkins

Jenkins 是一个开源项目，提供了一种易于使用的持续集成系统，使开发者从繁杂的集成中解脱出来，专注于更为重要的业务逻辑实现上。同时 Jenkins 能实施监控集成中存在的错误，提供详细的日志文件和提醒功能，还能用图表的形式形象地展示项目构建的趋势和稳定性。

### XCTool

使用Jenkins进行持续集成之前，还有一个前提，就是编译这个过程需要自动化，中途用xcode手动点的不行，所以我们需要有命令可以一次编译我们的工程，这里我们使用[xctool](https://github.com/facebook/xctool)这个工具，是facebook写的一个集成工具，用来编译和打包程序的。

安装方法是使用homebrew，在命令行执行

	brew install xctool
	
安装好在程序目录下测试一下是否可以编译

	 xctool -workspace integration_test.xcworkspace -scheme integration_test-cal -sdk iphonesimulator7.1 clean build
	 
_注意这里的sdk每个人可能不同，要根据本机安装的sdk来写_ ， 查看的方法是执行命令

	xcodebuild -showsdks
	
如果显示`** BUILD SUCCEEDED **`那么可以进入下一步了。

### Jenkins

安装jenkins还是使用brew

	brew install jenkins
	
安装好之后，可以通过使用命令行启动

	java -jar /usr/local/opt/jenkins/libexec/jenkins.war
	
如果想自动启动，需要先执行以下命令，创建启动项

	ln -sfv /usr/local/opt/jenkins/*.plist ~/Library/LaunchAgents
	
可以编辑一下~/Library/LaunchAgents/homebrew.mxcl.jenkins.plist这个文件

	open ~/Library/LaunchAgents/homebrew.mxcl.jenkins.plist
	 
想要让局域网都可以访问，需要把--httpListenAddress=127.0.0.1改成自己的局域网IP

手动启动启动项可以执行

	launchctl load ~/Library/LaunchAgents/homebrew.mxcl.jenkins.plist

之后用浏览器就可以访问`http://localhost:8080/`来登录jenkins了

![Jenkins网页截图](/images/Jenkins网页截图.png)

Jenkins启动之后，可以配置用户权限，但是我们为了简单，先不配置用户。

### Jenkins Plugin

Jenkins有一个很方便的功能，就是可以通过插件形式进行扩展，为了支持我们的持续集成，我们需要先安装必要的插件。

进入Jenkins网页的系统管理->插件管理->高级，找到右下角的“立即获取”就可以获得所有的插件信息了。

![Jenkins获取最新插件](/images/Jenkins获取最新插件.png)

更新好之后，在可选插件里面，安装如下插件

	Git Server Plugin   			#Git的支持，如果用svn就不需要了
	Git Client Plugin   			#Git的支持，如果用svn就补需要了
	Rvm								#加载RVM环境变量以实用ruby的cucumber命令
	Cucumber Test Result Plugin		#解析Cucumber的测试报告
	
_记得安装时勾选更新完自动重启_

至此，我们持续集成的所有环境应该都满足了。

### 托管你的项目

Jenkins一定要从一个地方获得一份软件副本的，所以，要想使用持续集成，必须要有一个版本管理工具，在Jenkins中成为scm，我们的例子使用git，并且我已经将测试工程上传到[CODE](https://code.csdn.net/)服务器上，地址在这里:[https://code.csdn.net/zangcw/integration_test](https://code.csdn.net/zangcw/integration_test)

### 创建一个项目

当你的源代码已经在代码托管服务器上之后，现在就可以在jenkins创建一个项目了。  
我们创建一个自由风格的软件项目  
![创建Jenkins项目](/images/创建Jenkins项目.png)

并且对其配置  
![配置Jenkins项目](/images/配置Jenkins项目.png)

主要配置如下内容：  

* 源码管理，示例中配置为[https://code.csdn.net/zangcw/integration_test.git](https://code.csdn.net/zangcw/integration_test.git)  
* 构建环境，要勾选RVM，否则没有办法在脚本中执行`cucumber`这个命令  
* 构建脚本，选择Execute shell，内容如下，请根据需要自行修改

```
cd $WORKSPACE
/usr/local/bin/xctool -workspace integration_test.xcworkspace -scheme integration_test-cal -sdk iphonesimulator7.1 clean build
mkdir -p test-reports
cucumber --format json -o test-reports/cucumber.json
```

* 构建后的操作，选择Publish Cucumber test result report，指定报告的目录`test-reports/cucumber.json`

之后点击应用，即完成了配置  
![项目创建完毕](/images/项目创建完毕.png)



### 立即构建

还在等什么？马上点击立即构建吧。。。  
![立即构建](/images/立即构建.png)

等待构建的过程中，我们可以查看控制台输出  
![控制台输出](/images/控制台输出.png)

_模拟器也会在中途弹出，然后自动关闭_

构建结束后，我们可以看到构建结果  
![构建结果](/images/构建结果.png)

结果展示了变更、由谁触发的构建和测试报告，更多的信息大家可以自行挖掘。总之构建是完成了。

想要进行持续构建，需要设置成每个一段时间自动构建，在Build periodically中配置即可。

### 下一步该做什么？

在淌通了这一整套流程之后，其实还是有很多事情等着我们来做的，下面是几个例子：

1. 为Jenkins创建用户管理
2. 修改脚本，自动存放ipa并上传到特定服务器
3. 配置构建策略，每日1次，或者多次，或者监听git变化，有上传就构建
4. 配置邮件策略，使大家及时获得反馈

总之，拥抱集成测试吧。