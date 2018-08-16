---
layout: post
title:  "秘籍！Unity与iOS平台交互和原生插件开发"
date:   2018-02-24 10:51:00 +0800
categories: Unity
tags: Unity iOS NativePlugin 
---

## 简介
Unity引擎虽然很强大，但是很多的时候还是需要运行平台的原生功能，这时候光靠Unity是做不到的。比如iOS平台上我们要从一个应用唤起另一个应用，在我们的游戏中打开一个网页，或者是直接嵌入一个iOS原生的界面（也就是现在接SDK的时候要做的事情）。
很多兄弟在刚接触的时候一头雾水，不知道从哪里入手。也有很多兄弟搞过一次一段时间后就忘记地一干二净。于是我说，入门的和忘记的人多了，就有了这个文章的诞生！希望能问新手打开新世界的大门，让忘记细节的老兵可以快速回忆。

## Unity Call iOS 
这里我们来实现从Unity调用iOS中OC实现的方法。
1.	在C#文件中，声明一个`extern`方法，如下：

	```csharp
	[DllImport("__Internal")]
	private static extern void CalliOSNativeFunction();
	```

2. 新建我们在iOS原生环境下运行的源文件，如: `iOSBridgePlugin.mm`，`iOSBridgePlugin.h`。
3. 定义我们要调用的方法。

	`iOSBridgePlugin.h`文件：

	```objc
	extern "C" {
		void CalliOSNativeFunction();
	}
	```

	`iOSBridgePlugin.mm`文件：

	```objc
	#import "iOSBridgePlugin.h"

	void CalliOSNativeFunction(){
		NSLog(@"[iOS Native] I am running!");
	}

	```

	在XCode中，`.m`是C或者Object-C类型的文件，`.mm`是C++文件，两者在被编译时有不同的处理。所以我这里使用了`.mm`文件，需要加上：

	```cpp
	extern "C"{
		/* 
		包裹我们要声明的方法。
		*/
	}
	```
4. 将源文件放在`Assets/Plugins/iOS/`目录下，这样源文件在Unity打包iOS工程时会自动将文件拷贝到XCode工程中的`Plugins/iOS/`目录下，并且在工程中添加正确的引用。

5. 打包，跑一跑我们刚才实现的接口。原生插件的开发难度似乎就是个纸老虎。

现在我们已经可以顺利地从Unity调用iOS的方法了，那么剩下来iOS原生系统支持的事情我们都能实现了，开始为所欲为吧！

### 实现“HellWorldSDK”
很多时候我们要接入项目的第三个SDK都有自己的iOS原生界面，我在只需要成功绘制出界面就能完成大部分的工作了。
这里我们实现一个自己的SDK来接入到我们的测试工程里
1. 创建一个界面，叫做HelloWorldSDKViewController，继承UIViewController。
2. 界面上有简单的标题文字，一个矩形图案和一个按钮。
3. 调用我们的SDK。修改我们原先的`iOSBridgePlugin.m`文件。

	```objc
	void CalliOSNativeFunction(){
		NSLog(@"[iOS Native] I am running!");
		[[HelloWorldSDKViewController sharedInstance] ShowHelloWorld];
	}
	```

	```objc
	[GetAppController().rootViewController presentViewController:_sharedInstance
														animated:true
													  completion:nil];
	```
	Unity生成的项目中，所有的场景都是一个ViewController，要绘制我们SDK的界面，就是在Unity的ViewController上绘制一个新的界面。

4. 从我们的SDK返回。
	```objc
	[GetAppController().rootViewController dismissViewControllerAnimated:true completion:nil];
	```
	关闭我们的界面也是一样，从Unity的ViewController上销毁我们的界面。

`HelloWorldSDKViewController.h`:
```objc
#ifndef HelloWorldSDKViewController_h
#define HelloWorldSDKViewController_h
#import <Foundation/Foundation.h>
@interface HelloWorldSDKViewController : UIViewController{
    
}
@end
#endif
```

`HelloWorldSDKViewController.m`:
```objc
#include "HelloWorldSDKViewController.h"

@implementation HelloWorldSDKViewController

// 用static声明一个类的静态实例；
static HelloWorldSDKViewController *_sharedInstance = nil;

//使用类方法生成这个类唯一的实例
+(HelloWorldSDKViewController *)sharedInstance{
    if (!_sharedInstance) {
        _sharedInstance =[[self alloc]init];
    }
    return _sharedInstance;
}

-(void) viewDidLoad{
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];

    UIButton *btnBack = [UIButton buttonWithType:UIButtonTypeSystem];
    btnBack.frame = CGRectMake(0.5f * self.view.bounds.size.width - 100, 300, 200, 40);
    btnBack.layer.borderWidth = 1;
    [btnBack setTitle:@"返回" forState:UIControlStateNormal];
    [btnBack addTarget:self
                action:@selector(BackToUnityScene:)
      forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:btnBack];

    UILabel *title = [[UILabel alloc]init];
    title.frame = CGRectMake(0.5f * self.view.bounds.size.width - 100, 100, 200, 40);
    title.text = @"Hello World";
    [self.view addSubview:title];
}

-(void) ShowHelloWorld{
    [GetAppController().rootViewController presentViewController:_sharedInstance
                                                        animated:true
                                                      completion:nil];
}

-(void) BackToUnityScene:(id)sender {
    NSLog(@"[HelloWorldSDK] Back to unity scene");
    [GetAppController().rootViewController dismissViewControllerAnimated:true completion:nil];
}
@end

```

## iOS Call Unity
现在我们从Unity调用iOS的接口已经成功了，那么下面我们就会想从iOS是否可以调用我们Unity中用C#实现的方法呢？答案是肯定的！
我们可以用`UnitySendMessage`来实现。
```objc
UnitySendMessage("GameObjectName1", "MethodName1", "Message to send");
```
通过这个接口我们可以清楚的知道，我们能调用的接口必须是挂在GameObject上的脚本上的某一个方法。
让我们来动手实现一个方法。
1. 在C#文件中实现我们需要调用的方法
	```csharp
	private void Callback_iOSReturnMessage(string message){
		Debug.Log("[Unity callback] Message is :" + message);
	}
	```
2. 从我们的SDK返回时，发送消息
	```objc
	-(void) BackToUnityScene:(id)sender {
		NSLog(@"[HelloWorldSDK] Back to unity scene");
		UnitySendMessage("BridgeGameObject", "Callback_iOSReturnMessage", "[Return Unity Successful]");
		[GetAppController().rootViewController dismissViewControllerAnimated:true completion:nil];
	}
	```
3. 打包运行！

## iOS 插件开发的关键点
* OC语法！要熟练iOS的开发，OC的语法还是要会一点的，至少要知道类的声明和定义，单例对象的声明和定义，函数的声明和定义，函数的调用。
* iOS框架，比如iOS中UI的框架，iOS中的APPDelegate
	* iOS的UI框架是MVC模式的，所有的界面都继承自`UIViewController`，界面上的UI元素就是`View`。
	* AppDelegate的作用范围是全局的
* URLSchemes
* 自动化打包，能自动化的机械活当然要让计算机自动处理！有两种方式，一个是大神开发的XUPorter，还有是Python，曾经在VuforiaARSDK中见过
* 接入SDK的本质
* `.a`文件的库是和XCode的版本相关的

## 结尾
到此为止，秘籍结束了。总的来说，为Unity开发iOS原生的插件在理解了实现原理后不会很难，即使忘记了很多，在看过秘籍再重新操作一遍以后也能快速的回忆起来，毕竟这些都是当年趟过的坑=。=
希望这篇秘籍可以帮助兄弟们能更好地驾驭Unity，驾驭iOS原生插件开发。
这是工程的地址[ForkMeOnGithub](https://github.com/aaBaO/DemoRepository.git)

如果有什么不正确的，或是表达的不够准确的，希望兄弟们可以评论出来，共同进步~~
