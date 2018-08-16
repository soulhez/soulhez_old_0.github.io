---
layout: post
title:  "快速了解Unity2017新功能:Cinemachine(3)"
date:   2018-04-29 17:00:00 +0800
categories: UnityEngine 
tags: Unity2017 NewFeature Cinemachine ClearShot
---

## 简介

[上一篇文章](https://aabao.github.io/Unity2017_Cinemachine_2/)我们更深入地认识了Cinemachine，并且动手实践创建了一个基础的
镜头跟随效果，以及一个2D游戏常见的场景和镜头效果。今天我们再来了解一下`ClearShot`

## 一图流

![ClearShot](http://oxujermt3.bkt.clouddn.com/image/blog/201804291700/Xmind_ClearShot.png)

## 温故知新 

Cinemachine的基础概念这里就不再复述了，不清楚的兄弟可以查看我的[第一篇文章](https://aabao.github.io/Unity2017_Cinemachine_1/)

今天我们来说说ClearShot。他这有什么用呢？试想一个在3D游戏中，我们常会遇到这样的情况，玩家从室外移动到室内，
这时候室内的观察视野就没有野外宽阔了， 我们能看到的内容也会发生变化。如果使用ClearShot就能很轻松地达到这种效果。

## 实践

使用ClearShot，我们可以快速地达到如下的效果。

![rutu](http://oxujermt3.bkt.clouddn.com/image/blog/201804291700/ClearShot效果.gif)

1. 创建一个移动的Cube，和一个固定的wall，如图，cube有一个动画，会向前移动一段距离，穿过wall后再出来

2. 通过Cinemachine->Create ClearShot Camera，创建一个ClearShot VirtualCamera，命名为CM ClearShot1。

3. 通过Cinemachine->Cerate Virtual Camera, 创建一个Virutal Camera，命名为lookTheUp。

4. 将lookTheUp VCam放到CM ClearShot1下，变成clearshot的子物体，这时候会发现clearshot下已经有一个Vcam了，
将这个Vcam改名为lookTheRight<br> 树形结构如图<br>
  ![如图](http://oxujermt3.bkt.clouddn.com/image/blog/201804291700/clearshot树形结构.png)<br>

5. 设置lookTheRight，在cube移动的时候跟随cube，而且注释这cube前进方向的右侧，
设置lookTheUp，在cube移动的时候跟随cube，注释这cube前进方向的上方。

6. 选择CM ClearShot1，修改两个Child VirtualCamera的优先级，设置lookTheRight为12， lookTheUp为11。
这个时候我们会看到一个警告，提示lookTheUp没有添加ColliderExtention。<br>
  ![如图](http://oxujermt3.bkt.clouddn.com/image/blog/201804291700/clearshot警告.png)<br>

7. 为lookTheUp添加ColliderExtention

Play，Enjoy It。

我们可以看到，barin的初始状态由lookTheRight控制，当VCam lookTheRight无法观察到cube的时候，brain会自动选择lookTheUp来控制摄像机。

ClearShot会他的ChildVirtualCamera中选择最优的一个VirtualCamera来控制摄像机，而别这些ChildVirtualCamera都需要有ColliderExtention。

### Randomize Choice

ClearShot里还有一个属性是Randmize Choice，通过查看源码，我们会明白，勾选后我们的ClearShot会从ChildVirtualCamera中随机选择一个VCam来
控制摄像机。当然他的条件也是很严格的，需要所有的ChildVirtualCamera有相同的Priority，还有相同的ShotQuality。

```csharp
private ICinemachineCamera ChooseCurrentCamera(Vector3 worldUp, float deltaTime) 
{
	//not our bussiness this time...

	if (LiveChild != null && !LiveChild.VirtualCameraGameObject.activeSelf)
		LiveChild = null;
	ICinemachineCamera best = LiveChild;
	for (int i = 0; i < childCameras.Length; ++i)
	{
		CinemachineVirtualCameraBase vcam = childCameras[i];
		if (vcam != null && vcam.VirtualCameraGameObject.activeInHierarchy)
		{
			// Choose the first in the list that is better than the current
			if (best == null 
				|| vcam.State.ShotQuality > best.State.ShotQuality
				|| (vcam.State.ShotQuality == best.State.ShotQuality && vcam.Priority > best.Priority)
				|| (m_RandomizeChoice && mRandomizeNow && (ICinemachineCamera)vcam != LiveChild 
					&& vcam.State.ShotQuality == best.State.ShotQuality 
					&& vcam.Priority == best.Priority))
			{
				best = vcam;
			}
		}
	}
	mRandomizeNow = false;

	float now = Time.time;
	if (mActivationTime != 0)
	{
		// Is it active now?
		if (LiveChild == best)
		{
			// Yes, cancel any pending
			mPendingActivationTime = 0;
			mPendingCamera = null;
			return best;
		}
	}

	//not our bussiness this time...
}
```

**而代码中ShotQuality就是ColliderExtention会处理的值**。

查看Randmoize Choice效果的步骤：

1. 隐藏我们的wall

2. 将lookTheUp和lookTheRight的Priority都修改为11

3. 勾选Randomize Choice

Play

每次运行，都会随机使用一个VCamera来控制摄像机。

## 结尾

大家在实践的过程中有遇到什么问题，可以提出来，我很乐意和大家一起分享。

### 这是[Demo的Github地址](https://github.com/aaBaO/DemoRepository)欢迎大家Fork过去参考
