---
layout: post
title:  "快速了解Unity2017新功能:Cinemachine(1)"
date:   2018-04-06 21:15:00 +0800
categories: UnityEngine 
tags: Unity2017 NewFeature Cinemachine 
---

## 简介
Unity2017又双叒叕一个令人激动的新功能！Cinemachine！由Unity提供了一套摄像机的解决方案，快速，高效的完成我们需要的镜头效果,
像是轨道移动镜头，镜头跟随，镜头切换，模仿手持抖动效果等等。原先难以控制的镜头，现在再也不是噩梦了。
说了这么多，赶紧让我们来看看Cinemachine吧！

## 如何安装
Cinemachine功能强大，但是我们想要使用它的话还需要到AssetStore里下载他的Package。

Import我们刚才下载的Package，这个时候我们的菜单栏就会增加一个Cinemachine菜单。
![Cinemachine菜单](http://oxujermt3.bkt.clouddn.com/image/unity2017_cinemachine/1_menu.png)

我们可以选择`Import Example Asset Package`来导入官方提供的实例场景和一些资源。
全部导入后我们会得到两个目录`Cinemachine`,`CinamechineExamples`

![导入完成后的目录](http://oxujermt3.bkt.clouddn.com/image/unity2017_cinemachine/1_directory.png)

在`Assets/Cinemachine/`目录中，可以查看官方的手册文件`CINEMACHINE_install`

## 重要概念
安装完成！让我们来创建一个Cinemachine开始我们的创作吧！等等，Cinemachine是这么用的吗？在菜单栏里没有找到创建Cinemachine的选项啊。
没错，这是一个好问题，让我们再仔细阅读一下官方留给我们的手册吧。

> Cinemachine is a modular suite of camera tools for Unity which give AAA game quality controls for every camera in your project. 

Cinemachine是为Unity开发的模块化工具，它可以让我们控制我们项目中的每一个摄像机，帮助我们创造出AAA游戏！

看来我们需要创建的并不是Cinemachine，而是在Cinemachine系统中重要的两个家伙`Cinemachine Brain`，`Cinemachine Virtual Camera`。

* Cinamachine Virtual Camera:可以认为是一个单独的摄像机，但他不是一个真实的摄像机。
在Cinemachine系统中，他保存了控制摄像机行为的重要参数。
* Cinamachine Brain:控制Virtual Camera的大脑，有他来选择我要使用哪一个VirtualCamera的数据来控制我手中的这个真实的摄像机。

## 实践出真知
道理我们都懂了，那我们来创建一个Cinemachine Virtual Camera试试。
点击菜单栏上的Create Virtual Camera，我们会发现我们的Main Camera自动加上了Cinemachine Brain组件，同时，Hierarchy中新建了一个CM vcam1物体。
查看一下Cinemachine Brain组件，发现一个Live Camera属性，那么正如手册所说，当前大脑拿来控制真实摄像机的虚拟相机就是我们新建的这个CM vcam1了。
再来看看CM vcam1，果然上面有一堆的属性，那我们接下来就是瞎倒腾这些属性来实现我们的摄像机效果咯。

但是属性这么多，我们怎么下手呢？我们一个个来看
* Game Window Guides: 选择后会在Game窗口中看到辅助我们查看摄像机参数的线条和框框。
* Save During Play:好东西！在Play模式下的改动也会在结束播放后保存下来。
* Priority:优先级，这个属性会在之后的ClearShot中用到，暂时露个脸混个眼熟。
* Follow:摄像机的Position要跟随的物体
* LookAt:摄像机的视线要对准的物体
* Lens:熟悉的摄像机参数。
* Body:有更多的选项来控制摄像机的Position。可以用英语来方便理解，The body that (...) follow something.
* Aim:有跟过的选线来控制摄像机的视线。同样可以用英语来方便理解，The aim that (...) lookat something.
* Noise:可以选择一些晃动效果
* Add Extension:一些扩展功能，像是摄像机的碰撞，位置矫正等。

好啦，下面我们就先自由地玩一玩这些参数吧，在下面的学习中再深入去理解这些神奇的功能！
