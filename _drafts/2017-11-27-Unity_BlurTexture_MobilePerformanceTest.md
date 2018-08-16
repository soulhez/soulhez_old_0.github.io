---
layout: post
title:  "Shader Playground:GrabPass移动平台性能测试"
date:   2017-11-27 00:51:00 +0800
categories: CG 
tags: Unity GassionBlur GrabPass Mobile
---

## 简介
之前用GrabPass实现了UGUI RawImage简单的高斯模糊效果，一直没有测试在移动平台上的性能。今天就来测试一下性能如何，
看看优化后能否在移动平台上使用。

## 搭建测试环境
首先，设计一下测试场景，最后的结果就是比较出在开启和关闭模糊纹理时FPS的变化。
因为影响最终性能的指标有很多，我想就用FPS来评判这次测试的最终结果。
然后再尽量去模拟可能的使用环境下开关BlurTexture，测试对性能的影响。

OK,开始测试。
在AssetStore上找了一个科幻的场景`Sci-fi Modular Enviroment`。
在场景中间添加一个自动旋转的Camera。
再加一个UI按钮控制BlurTexture的开关。

然后准备我们计算FPS的代码。
```csharp
using UnityEngine;
using UnityEngine.UI;
using System.Collections;

[RequireComponent(typeof(Text))]
public class FPS : MonoBehaviour {

	private Text target;

	private const float updateInterval = 0.5f;
	private float lastTime = 0;
	private int frames;

	private int fps;
	private float ftime;

	void Awake(){
		Application.targetFrameRate = -1;
		target = GetComponent<Text>();
	}

	void Start () {
		lastTime = Time.realtimeSinceStartup;
	}

	void Update () {
		++frames;

		float nowTime = Time.realtimeSinceStartup;
		if(lastTime + updateInterval < nowTime){
			float deltaTime = nowTime - lastTime;
			fps = Mathf.FloorToInt(frames / deltaTime);
			ftime = deltaTime * 1000.0f / frames;
			frames = 0;
			lastTime = nowTime;
		}
		target.text = string.Format("{0:0} FPS\n{1:0.00} ms", fps, ftime);
	}
}
```

啥也不做，先让我们build出来，在Google Nexus5上跑跑看看结果如何。

可以看到在不开启我们的BlurTexture的时候，FPS可以大部分部分时间保持在20，最高可以达到30，最低19
然后开启BlurTexture后，FPS的波动就比较大了，大部分时间在17，最高只有24，最低15
=.=差距还是有的。
那就看看有没有优化的可能吧，让我们的效果高大上起来。

开始对Shader动手。
首先，把暂时没有用的变量去掉，然后既然我们要的就是模糊效果，那自然后是名正言顺的抛弃精度了，把float类型改成fixed类型

