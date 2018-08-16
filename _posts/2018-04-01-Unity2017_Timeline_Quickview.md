---
layout: post
title: 快速了解Unity2017新功能:Timeline
date:   2018-04-01 14:51:00 +0800
categories: UnityEngine
tags: Unity2017 NewFeature Timeline
---

## 简介

Unity2017面世也有一段时间了，今天我们来学习一下2017版本中的新功能`Timeline`。这个功能乍一看和原来的Animation窗口很像，功能感觉也很像，就是整整动画。
但是仔细研究就会发现，其实`Timeline`比`Animation`强大很多。

Timeline支持多种类型，比如动画，声音，物体激活等，这个功能的感觉就像是在用`Adobe Premiere`一样，使用各种轨道来组合出自己想要的效果。
而Animation只能处理动画。

## 认识Timeline 

Unity官方文档对Timeline功能的解释是：我们可以用Timeline编辑器窗口中与场景相关的游戏物体的可视化轨道和片段，来创作我们的场景过度，电影，游戏流程。
Timeline系统中有两个重要的概念，`Timeline Asset`和`Timeline Instance`。

### Timeline Asset

直观翻译成Timeline资源，会被保存到项目中，资源中保存了轨道和片段信息，不包括具体关联的物体。任何在这个Timeline资源下的片段都会被保存成当前Timeline资源的子物体。

![timeline_overview_asset](http://oxujermt3.bkt.clouddn.com/image/unity2017_timeline_quickview/timeline_overview_asset.png)

红色部分就是被保存的Timeline资源，蓝色部分就是这个资源中保存的片段资源。


### Timeline Instance

直观翻译成Timeline实例，会被保存到场景，包括具体关联的物体。关联着的物体被定义为绑定物体。Timeline实例一定是使用了Timeline资源的。

![timeline_overview_instance](http://oxujermt3.bkt.clouddn.com/image/unity2017_timeline_quickview/timeline_overview_instance.png)

蓝色部分是PlayableDirector组件关联的Timeline资源，红色部分就是当前这个Timeline资源在这个Timeline实例上绑定的游戏物体。

## Timeline 示例

大概了解了Timeline以后我们来创建一个场景，学习一下创建和使用Timeline的具体流程。

1. 创建TimelineAsset和TimeInstance

	在场景中选择我们要控制PlayableDirector的物体，打开Timeline编辑器窗口(菜单Window->TimelineEditor)。如果这个物体还没有PlayableDirector组件和TimelineAsset的话，会在
	Timeline编辑器窗口中看到这个提示。

	![timeline_editor_create](http://oxujermt3.bkt.clouddn.com/image/unity2017_timeline_quickview/timeline_editor_create.png)

	点击创建选择我们的TimelineAsset保存路径，Unity会自动为我们添加PlayableDirector组件和创建一个TimelineAsset。

2. 添加自己的动画片段 

	添加完Timeline资源后，Unity默认创建了一个当前物体的动画轨道，不需要的话可以直接删除。

	我们要快速学习，就加入我们自己的动画片段。在项目中将模型放入场景

	选中我们的Timeline实例，将场景中的模型拖到Timeline编辑器窗口中，选择`AnimationTrack`。此时会创建一个与我们的模型相关的轨道。

	再到项目窗口中，找到我们要用的动画片段拖到我们新加的物体的动画轨道上。

	![timeline_rpgrole_animation](http://oxujermt3.bkt.clouddn.com/image/unity2017_timeline_quickview/timeline_rpgrole_animation.png)

3. 播放
	Enjoy it。

### HumaniodAnimation的偏移解决方案。

我们的Timeline实例能跑出我们的角色的动画了，但是会发现，这几段动画都是单纯的播放而已，动画制作时的坐标和位移是怎么样，播放出来的效果就是怎么样。

比如现在的Demo，我们想要的结果是，角色原地待机后向前移动一段距离，再在移动后的位置向右翻滚。这个效果如何实现？

![timeline_rpgrole_animation_offsetdiff](http://oxujermt3.bkt.clouddn.com/image/unity2017_timeline_quickview/timeline_rpgrole_animation_offsetdiff2.gif)

蓝色角色使我们期望的动画效果，红色角色是原始的动画效果。

Timeline很周到，已经帮我们想好了解决方案。就是选中需要拼接的动画，点击右键，选择`Match Offset To Previous Clip`。

![timeline_rpgrole_animation_matchoffset](http://oxujermt3.bkt.clouddn.com/image/unity2017_timeline_quickview/timeline_rpgrole_animation_matchoffset.png)

当然要实现这个效果，必须是HumaniodAnimation才可以。

## 总结

Timeline还有很多强大的功能需要探索。总的来说用Timeline对非程序开发者来说是很方便的，一切都变得可视化，能很直观地控制效果。而且搭配`Cinemachine`可以达到电影效果一样的镜头控制。

想要查看Demo工程的朋友可以从我的GitHub仓库里克隆到。 demo工程的地址[ForkMeInGithub](https://github.com/aaBaO/DemoRepository.git)

