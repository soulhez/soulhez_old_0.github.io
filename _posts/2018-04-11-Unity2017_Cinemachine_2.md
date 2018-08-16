---
layout: post
title:  "快速了解Unity2017新功能:Cinemachine(2)"
date:   2018-04-11 23:15:00 +0800
categories: UnityEngine 
tags: Unity2017 NewFeature Cinemachine 2D
---

## 简介

[上一篇文章](https://aabao.github.io/Unity2017_Cinemachine_1/)我们大概介绍了Cinemachine的概念和一些基本的属性，今天我们来更深入的认识Cinemachine，并且动手实践一番，创建一个基础的
VirtualCamera制作一个镜头跟随动画，以及制作一个2D游戏常见的场景和镜头效果。

## 温故知新 

我们回顾一下Cinemachine的重要概念，CinamachineBrain和VirtualCamera，brain控制激活哪一个虚拟相机来控制真实的Camera，虚拟相机
可以根据我们的不同需求来设置。

## 实践1

1. 创建一个Cube，为这个cube创建一个平移动画。命名为translation

2. 创建一个Cinamachine Virtual Camera，通过菜单栏Cinemachine->Create Virtual Camera。命名为CM Vcam1。

3. 将Cube设置为CM Vcam1的follow对象。

![Transposer设置](http://oxujermt3.bkt.clouddn.com/image/blog/201804112315/transposer设置.png)

完成，点击play查看我们的成果。

![Follow效果图](http://oxujermt3.bkt.clouddn.com/image/blog/201804112315/follow效果图.gif)

我们会看到摄像机会跟随我们的cube一起移动。
OK咱们的位置跟随效果达到了，在上一篇我们说过，Follow和Body是相关联的。那就来仔细看看跟Follow有关的Body属性的参数吧。
Body的第一个属性就是设置摄像机的跟随类型，默认类型是`Transposer`，按照官方文档的解释，Transposer只改变摄像机在空间中的Position。

#### Transposer 

* Follow Offset

	设置摄像机和Follow的物体的相对位置。

* Binding Mode

	对FollowOffset的补充设定。主要有3种类型:

	* LockToTarget 摄像机和物体的位置以`物体的自身坐标系`为标准，假设我们的FollowOffset设定成(0,0,-10),
	当我们转动物体的时候，我们的摄像机也会跟着改变位置以保持(0, 0, -10)的相对位置。

	  ![LockToTarget旋转示意图](http://oxujermt3.bkt.clouddn.com/image/blog/201804112315/LockToTarget示意图.gif)

	* WorldSpace 摄像机和物体的位置以`世界坐标`为标准，这个时候在不改变物体的位置情况下，我们无论怎么转动物体都没有关系了，
	摄像机和物体的位置保持在世界坐标中的相对关系。

	* SimpleFollow 摄像机和物体的位置以`摄像机坐标系`为标准，在摄像机的坐标系中使摄像机和物体保持相对的位置关系。

* 一些影响位置改变响应速度的参数

	如：X Damping，Y Damping，Z Damping等，值越小改变位置的响应速度更快

OK，现在咱已经弄明白了Virual Camera上的Body属性，下面再来看看Aim属性。同样的，我们也先动手来做一做。

1. 将follow对象上的cube物体去除，把我们的cube设置为CM Vcam1的LookAt对象。

![Composer设置](http://oxujermt3.bkt.clouddn.com/image/blog/201804112315/composer设置.png)

完成，点击Play查看我们的成果。

![Lookat效果图](http://oxujermt3.bkt.clouddn.com/image/blog/201804112315/lookat效果图2.gif)

我们会看到摄像机本身的位置不会发生变化，但是会根据cube的移动发生转动，而且Game试图里也出现了很多奇奇怪怪的线框(如果没有，需要打开GameWindowGuides)。
OK咱们的视线跟随效果达到了，在上一篇我们说过，LookAt和Aim是相关联的。那就来仔细看看跟LookAt有关的Aim属性的参数吧。
我们会发现在Aim属性中也有摄像机的实现跟随类型，默认的选择是Composer，按照官方文档的解释，Composer只改变摄像机在空间中的Rotation。

#### Composer

* Tracked Object Offset

	设置摄像机的视线和物体的相对位置关系，如修改这个值为(0, -1, 0)这个时候我们的视线在cube物体下方1个单位的地方。Game视图中的中心黄点就是我们的视线位置。

* Lookahead Time

	可以理解为摄像机对物体进行追踪后的惯性，值越大惯性越大。

* Lookahead Smoothing

	在设置了LookaheadTime后才能看出效果，值越大，在产生惯性后镜头的转动越平滑。

* Horizontal Damping, Vertival Damping

	摄像头发生转动的灵敏度，值越小响应速度越快。

* Screen X, Screen Y

	物体在屏幕中的偏移位置，修改后摄像机会跟随物体发生旋转。

* Dead Zone X, Dead Zone Y

	代表Game视图中透明区域的范围。透明区域的作用是，当物体的移动要超出这个区域时，摄像机开始转动跟随物体。

* Soft Zone X, Soft Zone Y

	代表Game视图中蓝色区域的范围。蓝色区域的作用是，当物体的移动超出透明区域时拥有一个缓冲区域，这个效果在Damping值较大时体现的比较明显。
	当物体的移动超过蓝色区域，进入红色区域时，摄像头会立刻调整视线，对物体进行追踪（这时候的效果相当于Damping=0）。

* Bias X, Bias Y

	对SoftZone的区域进行偏移。

OK, 现在咱们也充分认识了Transposer和Composer，接下来该怎么玩就随大伙了，好好把玩把玩这有意思的摄像机吧。

## 实践2

现在我们制作一个2D游戏常见的场景。如果你忘记了2D场景是什么样子的，那么请你打开FC版超级马里奥或者拳皇97，重温一下儿时的回忆。

* 如图搭建一个快速场景，MainCamera选择Orthographic，我们的球体就是这个场景的主角

![快速场景](http://oxujermt3.bkt.clouddn.com/image/blog/201804112315/快速场景.png)

* 在菜单栏选择Cinemachine-> Create 2D Camera，创建一个新的2D虚拟相机，将相机命名为2DCMVcam

* 将球体设置为2D相机的Follow对象，在Lens设置中，将OrthographicSize的值设置成5，Body属性中，选择FramingTransposer

![2D虚拟相机设置](http://oxujermt3.bkt.clouddn.com/image/blog/201804112315/2D虚拟相机设置.png)

* 点击Play！

![2D场景穿帮效果图](http://oxujermt3.bkt.clouddn.com/image/blog/201804112315/2D相机穿帮效果.gif)

就这么简单的几步，我们就创建了一个基本的2D相机，而且效果看起来还不赖。

但是有一个问题，现在我们可以看到穿帮的场景，这不是我们想要的，我们作为第九艺术怎么能穿帮呢!
不怕，我们还有解决方案。

1. 选择我们的2DCMVcam，在Extensions中添加CinemachineConfiner

2. 为我们的2DCMVcam添加一个边界的碰撞形状。目前的碰撞只支持PloygonCollider

![CinemachineConfiner设置](http://oxujermt3.bkt.clouddn.com/image/blog/201804112315/CinemachineConfiner设置.png)

![碰撞边界](http://oxujermt3.bkt.clouddn.com/image/blog/201804112315/碰撞边界.png)

点击Play，愉快的解决了我们的边界穿帮问题！Cool！

![2D场景效果图](http://oxujermt3.bkt.clouddn.com/image/blog/201804112315/2D相机效果.gif)

## 结尾

其余的相机效果大家可以在阅读文章后自己动手实践一番，当然实践的过程中有遇到什么问题，可以提出来，我很乐意和大家一起分享。

### 这是[Demo的Github地址](https://github.com/aaBaO/DemoRepository)欢迎大家Fork过去参考
