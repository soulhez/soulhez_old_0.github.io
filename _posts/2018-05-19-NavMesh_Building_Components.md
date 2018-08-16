---
layout: post
title:  "探索NavMeshBuildingComponents，让AIAgent飞檐走壁"
date:   2018-05-19 10:50:00 +0800
categories: UnityEngine
tags: NewFeature Navmesh highlevel
---

## 简介
最近在看AI寻路的文章时，发现Unity在5.6的版本中出了新的寻路功能NavMeshBuildingComponents。(-。-其实这个新功能并没有隐藏的很深，只要是5.6版本以上的兄弟，就能在引擎中很明显的位置看到，没发现的我太没用了)用过旧的Navigation的兄弟都知道旧版的寻路有多难受，烘焙出来的NavMesh只能支持一个agentType，而且只能支持静态寻路。

所以以上的这些痛点就催生了NavMeshBuildingComponents。支持多个agentType，支持动态寻路，支持任意方向的表面寻路。

## 一图流

![NavMeshComponents-xmind](http://oxujermt3.bkt.clouddn.com/image/blog/201805191050/NavMeshCuildComponents-xmind.png)

NavMeshComponents有4个主要的组件：
* NavMesh Surface

  主要控件，控制哪些地方（MeshRenderer，Collider，Volume）需要烘焙NavMesh

* NavMesh Modifier

  修改物体的寻路设置，可以覆盖NavMeshSurface的设置

* NavMesh ModifierVolume

  修改范围内的寻路设置，可以覆盖NavMeshSurface的设置

* NavMesh Link

  连接2个NavMeshSurface

## 实践
今天我们的实践内容有2个demo，一个是让agent飞檐走壁，一个是运行时动态生成NavMesh。

### 获取NavMeshComponents
在Github上可以找到NavMeshComponents的[工程链接](https://github.com/Unity-Technologies/NavMeshComponents)，下载或者克隆后就能拿到我们想要的Components。

### 飞檐走壁
我们今天的主角有ThinAgent和FatAgent

1. ThinAgent<br>
  ![ThinAgent](http://oxujermt3.bkt.clouddn.com/image/blog/201805191050/ThinAgent.png)<br>
2. FatAgent<br>
  ![FatAgent](http://oxujermt3.bkt.clouddn.com/image/blog/201805191050/FatAgent.png)<br>
3. 为两个Agent创建一个简单的寻路地形EZGround和Wall。<br>
  ![EZGround&Wall](http://oxujermt3.bkt.clouddn.com/image/blog/201805191050/EZGround&Wall.png)<br>
4. 为EZGround添加两个NavMeshSurface组件，AgentType分别选择FatAgent和ThinAgent，点击bake
5. 为EZWall添加NavMeshSurface组件，我们只允许ThinAgent飞檐走壁，所以AgentType选择ThinAgent，点击bake
6. 添加NavMeshLink，连接EZGround和EZWall。<br>
  ![NavMeshLink](http://oxujermt3.bkt.clouddn.com/image/blog/201805191050/NavMeshLink.png)<br>
7. 为我们的Agent添加Movement，让他们往我们鼠标所选的位置移动。

Play， Enjoy It!

![飞檐走壁预览](http://oxujermt3.bkt.clouddn.com/image/blog/201805191050/%E9%A3%9E%E6%AA%90%E8%B5%B0%E5%A3%81%E9%A2%84%E8%A7%88.gif)

注意NavMeshSurface组件放在一个空物体上时，在烘焙时只默认烘焙法线向世界坐标Y轴方向的面。所以需要为法线方向特殊的面生成NavMesh时需要单独根据MeshRenderer或者Collider创建一个NavMeshSurface。

飞檐走壁的Agent这么快就完成了，是不是很激动！激动之余让我们来看看NavMeshSurface到底做了什么事情。直捣黄龙，看看Bake按钮的奥秘。

**NavMeshSurfaceEditor.cs**
```csharp
//...
if (GUILayout.Button("Bake"))
{
	// Remove first to avoid double registration of the callback
	EditorApplication.update -= UpdateAsyncBuildOperations;
	EditorApplication.update += UpdateAsyncBuildOperations;

	foreach (NavMeshSurface surf in targets)
	{
		var oper = new AsyncBakeOperation();

		oper.bakeData = InitializeBakeData(surf);
		oper.bakeOperation = surf.UpdateNavMesh(oper.bakeData);
		oper.surface = surf;

		s_BakeOperations.Add(oper);
	}
}
//...
static NavMeshData InitializeBakeData(NavMeshSurface surface)
{
	var emptySources = new List<NavMeshBuildSource>();
	var emptyBounds = new Bounds();
	return UnityEngine.AI.NavMeshBuilder.BuildNavMeshData(surface.GetBuildSettings(), emptySources, emptyBounds
	, surface.transform.position, surface.transform.rotation);
}
```

**NavMeshSurface.cs**
```csharp
public AsyncOperation UpdateNavMesh(NavMeshData data)
{
	var sources = CollectSources();

	// Use unscaled bounds - this differs in behaviour from e.g. collider components.
	// But is similar to reflection probe - and since navmesh data has no scaling support - it is the right choice here.
	var sourcesBounds = new Bounds(m_Center, Abs(m_Size));
	if (m_CollectObjects == CollectObjects.All || m_CollectObjects == CollectObjects.Children)
	sourcesBounds = CalculateWorldBounds(sources);

	return NavMeshBuilder.UpdateNavMeshDataAsync(data, GetBuildSettings(), sources, sourcesBounds);
}
```

NavMeshComponent的生成流程主要分成两个步骤，一个是收集原始的数据，然后就是传送数据进行烘焙。然后一切的关键操作都在`NavMeshBuilder.UpdateNavMeshDataAsync`中进行。虽然看了这么多代码但是关键的部分还是没有公开=。=

等NavMesh数据烘焙完成后，调用`NavMesh.AddNavMeshData`，返回NavMesh实例。

### 运行时动态生成NavMesh
利用官方提供的Example中的LocalNavMeshBuilder和NavMeshSourceTag来快速实现。这个版本的实现只是一个粗略的演示，默认的agentTypeID为0，并且area只要walkable。但是在看懂了实现代码后，就可以很方便地得到我们自己需要的动态寻路需求。

1. 创建一个DynamicEnv空物体，添加LocalNavMeshBuilder组件
2. 创建CubeA 和 CubeB为Dynamic的子物体，分别添加NavMeshSourceTag组件

Play, Engjoy it!

![动态生成NavMesh预览](http://oxujermt3.bkt.clouddn.com/image/blog/201805191050/%E5%8A%A8%E6%80%81%E7%94%9F%E6%88%90NavMesh%E9%A2%84%E8%A7%88.gif)

动态生成网格的关键和预先烘焙差不多，也是在运行时不断手机原始数据，再进行烘焙。关键的API也是`NavMeshBuilder.UpdateNavMeshDataAsync`。

在烘焙数据生成后调用NavMesh.AddNavMeshData，返回NavMesh实例。这个实例只在Enable的时候新建，然后保持引用，Disable的时候移除这些烘焙数据。

## 结尾
有了NavMeshComponent这么强大的工具，我们就能轻松愉快的实现心目中的AI了。官方对NavMeshComponents的定位是High-Level NavMesh Building Components，因此才开放给了我们一定的接口来满足我们的特殊需求，而且还给了很很多很好的Sample。

这是工程的Github地址：[https://github.com/aaBaO/DemoRepository](https://github.com/aaBaO/DemoRepository)，欢迎大家Fork过去参考。

### 参考文档
UnityManual:[https://docs.unity3d.com/2018.1/Documentation/Manual/NavMesh-BuildingComponents.html](https://docs.unity3d.com/2018.1/Documentation/Manual/NavMesh-BuildingComponents.html)

