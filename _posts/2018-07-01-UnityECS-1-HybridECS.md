---
layout: post
title:  "探索UnityECS系列（一）: Hybrid ECS"
date:   2018-07-01 11:50:00 +0800
categories: UnityEngine
tags: NewFeature ECS Hybird-ECS Data-OrientedModel
---

# 简介

Unity又一个令人激动的新功能ECS，Entity-Component-System。可以让我们开发出性能超强的游戏功能。官方提供的最直观的例子就是一个鱼群的模拟场景，这个场景中可以在运行25000个鱼的实例，并且稳定30FPS。那么接下来就让我们开始探索这个新功能吧。

# 一图流

![HybridECS](http://oxujermt3.bkt.clouddn.com/HybridECS.png)

第一次接触ECS的兄弟大可以先将这张图略过，等看完全文再回过头来看，到到时候会有更大的帮助。

# 什么是ECS？

## 概念
要深入学习ECS，建议分两步走，第一步HybridECS，混合类型ECS，在Monobehaviour和GameObject的基础上快速实现ECS的功能，实现数据和行为逻辑的分离。然后是PureECS，纯粹的ECS，完全抛弃MonoBehaviour和GameObject实现对性能的最大提升。

今天我们主要讲的是HybridECS，从我们熟悉的地方开始，逐渐认识ECS。

# 安装

1. 导入Unity Entities<br>
Unity2018版本，可以直接打开Window->Package Manager，在All标签下选择Entities进行安装，安装完后会自动导入相关的文件。<br>
![ECS-install](http://oxujermt3.bkt.clouddn.com/ECS-install.png)<br>
2017版本的兄弟可以到项目工程目录里找到Packages目录，修改manifest.json文件安装，
在`dependencies`中加入
    ```json
    "com.unity.entities": "0.0.12-preview.6",
    ```
    保存，回到Unity就会自动导入相关文件。

2. 修改PlayerSetting<br>
将Scripting Runtime Version 改为`.Net 4.x Equivalent`
![ECS-setup](http://oxujermt3.bkt.clouddn.com/ECS-Setup.png)

# 实践

1. 创建场景`HybridECS`， 添加2个UI按钮分别为`ToECS`，`ToOriginal`
2. 创建Cube，为了防止图形渲染对我们性能测试的干扰，将MeshRenderer上的Receive Shadows关闭，将Cast Shadows设为Off
3. 新建两个文件`OldMonoBehavour.cs`，`HybridRotatorSystem.cs`
4. `OldMonoBehaviour`中我们按照以往的习惯修改Cube的旋转。
5. `HybridRotatorSystem`中我们用ECS中的ComponentSystem实现Cube的旋转处理。

***OldMonoBehaviour.cs***
```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class OldMonoBehaviour : MonoBehaviour 
{
    public float rotateSpeed = 100;

    void Update () 
    {
        // 我们最熟悉的配方，在Monobehavior中修改transform的rotation。
        transform.rotation *= Quaternion.AngleAxis(Time.deltaTime * rotateSpeed, Vector3.up);	
    }
}
```

***HybridECSRotatorSystem.cs***
```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using Unity.Entities;

public class HybridECSRotator : MonoBehaviour
{
    //monobehavior 只存放数据
    public float rotateSpeed = 100;
}

public class HybridECSRotatorSystem : ComponentSystem
{
    //HybridECSRotatorSystem要处理的数据集合
    //注意这里是结构体，在ECS中数据都保存在结构体中方便进行分块序列存储。
    struct CubeEntity
    {
        public Transform transform;
        public HybridECSRotator rotator;
    }

    override protected void OnUpdate()
    {
        //所有enitity的deltaTime都是相同的，所以这里可以直接让他们公用相同的deltaTime
        float deltaTime = Time.deltaTime;
        foreach (var cubeEntity in GetEntities<CubeEntity>())
        {
            cubeEntity.transform.rotation *= Quaternion.AngleAxis(deltaTime * cubeEntity.rotator.rotateSpeed, Vector3.up);	
        }	
    }
}
```

***HybridECSSample.cs***
```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using Unity.Entities;

public class HybridECSSample : MonoBehaviour 
{
    public GameObject cubePrefab;
    public int count = 15000;

    private Dictionary<int, GameObject> m_gameObjectDic = new Dictionary<int, GameObject>();

    public void SwitchToECS()
    {
        DestroyOldGameObject();
        for(int i = 0; i < count; i++)
        {
            var originalCube = GameObject.Instantiate(cubePrefab) as GameObject;
            originalCube.transform.position = UnityEngine.Random.insideUnitSphere * (count / 100);
            originalCube.AddComponent<HybridECSRotator>();
            //需要给GameObject添加GameObjectEntity组件，才能让原先的GameObject被EntityManger识别为Enity。
            originalCube.AddComponent<GameObjectEntity>();

            m_gameObjectDic.Add(originalCube.GetInstanceID(), originalCube);
        }
    }

    public void SwitchToOriginal()
    {
        DestroyOldGameObject();
        for(int i = 0; i < count; i++)
        {
            var originalCube = GameObject.Instantiate(cubePrefab) as GameObject;
            originalCube.AddComponent<OldMonoBehaviour>();
            originalCube.transform.position = UnityEngine.Random.insideUnitSphere * (count / 100);
    
            m_gameObjectDic.Add(originalCube.GetInstanceID(), originalCube);
        }
    }

    private void DestroyOldGameObject()
    {
        foreach(var pair in m_gameObjectDic)
        {
            Destroy(pair.Value);
        }

        m_gameObjectDic.Clear();
    }
}
```

* 原始的MonoBehaviour运行效果<br>
![OldbehaviourPreview](http://oxujermt3.bkt.clouddn.com/OriginalPreview.gif)

* HybridECS运行效果<br>
![HybridECSPreview](http://oxujermt3.bkt.clouddn.com/HybridECSPreview.gif)

看完上面的例子还是一头雾水啊，什么是Entity？什么是EntityManager？什么是ComponentSystem？CubeEntity结构到底是个啥？他怎么拿到的我的HybridECSRotator中的数据的？而且性能并没有啥优化啊，有意义吗这样做？

先回答最后一个问题，这样做的意义是让我们快速的理解ECS的关键概念，帮助我们理解后面要讲的PureECS。

![GameObjectModel](http://oxujermt3.bkt.clouddn.com/GameObjectModel.png)

在就的GameObject框架中，在一个场景中，我们有很多的GameObject，每个Object上都有相应的Component来处理着各自的逻辑和行为。当我们需要进行统一处理时需要建立额外的辅助对象。

![HybridECSModel](http://oxujermt3.bkt.clouddn.com/HybridECSModel.png)

在HybridECS中我们熟悉的MonoBehaviour不再处理具体行为和逻辑，只负责保存数据。
具体的行为和逻辑转移到System里统一处理，
具体的数据载体不再是GameObject，而是Entity。

现在再来看一遍之前的问题。
* ***什么是Entity？什么是EntityManger?什么是ComponentSystem?*** <br>
在HybridECS的框架模型中，Entity是数据的载体，EntityManager管理场景中所有的Entity，ComponentSystem负责处理对应的Entity集合的逻辑和行为。

* ***CubeEntity结构到底是什么？他怎么就能拿到我的HybridECSRotator数据了？*** <br>
由于我们在创建GameObject的时候，为cube添加了GameObjectEntity组件，这个组件在OnEnable的时候，会自动创建一个Entity，然后Entity会添加这个GameObject上的所有Component。然后我们的HybridECSRotatorSystem就能找到HybridECSRotator的数据。我们在ComponentSystem中声明的CubeEntity，也就是在声明我们的system需要处理的数据是Transform和HybridECSRotator，通过`GetEntities`方法，我们就能拿到所有同时拥有Transform和HybridECSRotator组件的Entity了。

* ***这样做对性能优化有意义吗？*** <br>
虽然在HybridECS中没有太大地提升性能，但是却能很好地帮助我们理解ECS的一些关键概念。让我们明白ECS中要传达的是一种面向数据的编程思想。

# 总结
看到这里相信兄弟们对ECS的思想有了一个大概的了解，接下来让我们回头再看看之前的一图流。回忆一下要点，加深理解。

有不足或者错误的地方，希望大家能联系我改进。或者直接评论，希望得到很多的交流。

同时也感谢你的阅读~~~。

这是工程的[GitHub地址](https://github.com/aaBaO/DemoRepository)，欢迎Fork。

## References
[官方说明文档](https://github.com/Unity-Technologies/EntityComponentSystemSamples/blob/master/Documentation/index.md)