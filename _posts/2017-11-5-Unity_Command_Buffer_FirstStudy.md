---
layout: post
title:  "Shader Playground:Graphic Command Buffer初次探究"
date:   2017-10-29 13:51:00 +0800
categories: CG 
tags: Unity Shader CommandBuffer
---

## 简介
理解了一定的Shader渲染原来以后，会发现Shader做的事情是每一个像素，每一个片段都会做的事情，如果我们想要去处理大面积的像素时该怎么办呢？这时候Command Buffer就出场了！

### 什么是Command Buffer
我们先来看看官方文档的解释：
> List of graphics commands to execute.
> Command buffers hold list of rendering commands ("set render target, draw mesh, ..."). They can be set to execute at various points during camera rendering 

简单理解CommandBuffer就是一大串的图形渲染指令，可以在摄像机渲染大量像素上时执行。

似乎这就是我们要的解决方案！那就来试试看吧！

## 理解官方例子-Blurry Refractions
![官方的模糊例子](https://docs.unity3d.com/uploads/Main/RenderingCommandBufferBlurryRefraction.png) 

在[官方的手册](https://docs.unity3d.com/Manual/GraphicsCommandBuffers.html)上有示范工程的下载链接，下载后打开工程一探究竟。
打开BlurryRefractions目录，看到里面相关的文件有5个，一个demo场景，一份readme，一个脚本，两个shader文件。

打开场景查看`_RefractiveGlass`, MeshRenderer上只有一个Material，然后就是一个Component-`CommandBufferBlurRefraction`。

那首先自然是看看这个Material上放了什么Shader，他做了点什么?
```glsl
// Similar to regular FX/Glass/Stained BumpDistort shader
// from standard Effects package, just without grab pass,
// and samples a texture with a different name.

Shader "FX/Glass/Stained BumpDistort (no grab)" {
Properties {
	_BumpAmt  ("Distortion", range (0,64)) = 10
	_TintAmt ("Tint Amount", Range(0,1)) = 0.1
	_MainTex ("Tint Color (RGB)", 2D) = "white" {}
	_BumpMap ("Normalmap", 2D) = "bump" {}
}

Category {

	// We must be transparent, so other objects are drawn before this one.
	Tags { "Queue"="Transparent" "RenderType"="Opaque" }

	SubShader {

		Pass {
			Name "BASE"
			Tags { "LightMode" = "Always" }
			
CGPROGRAM
#pragma vertex vert
#pragma fragment frag
#pragma multi_compile_fog
#include "UnityCG.cginc"

struct appdata_t {
	float4 vertex : POSITION;
	float2 texcoord: TEXCOORD0;
};

struct v2f {
	float4 vertex : POSITION;
	float4 uvgrab : TEXCOORD0;
	float2 uvbump : TEXCOORD1;
	float2 uvmain : TEXCOORD2;
	UNITY_FOG_COORDS(3)
};

float _BumpAmt;
half _TintAmt;
float4 _BumpMap_ST;
float4 _MainTex_ST;

v2f vert (appdata_t v)
{
	v2f o;
	o.vertex = mul(UNITY_MATRIX_MVP, v.vertex);
	#if UNITY_UV_STARTS_AT_TOP
	float scale = -1.0;
	#else
	float scale = 1.0;
	#endif
	o.uvgrab.xy = (float2(o.vertex.x, o.vertex.y*scale) + o.vertex.w) * 0.5;
	o.uvgrab.zw = o.vertex.zw;
	o.uvbump = TRANSFORM_TEX( v.texcoord, _BumpMap );
	o.uvmain = TRANSFORM_TEX( v.texcoord, _MainTex );
	UNITY_TRANSFER_FOG(o,o.vertex);
	return o;
}

sampler2D _GrabBlurTexture;
float4 _GrabBlurTexture_TexelSize;
sampler2D _BumpMap;
sampler2D _MainTex;

half4 frag (v2f i) : SV_Target
{
	// calculate perturbed coordinates
	// we could optimize this by just reading the x & y without reconstructing the Z
	half2 bump = UnpackNormal(tex2D( _BumpMap, i.uvbump )).rg;
	float2 offset = bump * _BumpAmt * _GrabBlurTexture_TexelSize.xy;
	i.uvgrab.xy = offset * i.uvgrab.z + i.uvgrab.xy;
	
	half4 col = tex2Dproj (_GrabBlurTexture, UNITY_PROJ_COORD(i.uvgrab));
	half4 tint = tex2D(_MainTex, i.uvmain);
	col = lerp (col, tint, _TintAmt);
	UNITY_APPLY_FOG(i.fogCoord, col);
	return col;
}
ENDCG
		}
	}

}

}

```

可以看到这个shader利用	
`o.uvgrab.xy = (float2(o.vertex.x, o.vertex.y*scale) + o.vertex.w) * 0.5;`
处理了顶点像素的偏移
然后下面就是普通的绘制颜色和法线操作。但是！这里的`_GrabBlurTexture`从哪里来？

接着我们再来看看SeparableGlassBlur这个shader做了点什么。

```glsl
Shader "Hidden/SeparableGlassBlur" {
	Properties {
		_MainTex ("Base (RGB)", 2D) = "" {}
	}

	CGINCLUDE
	
	#include "UnityCG.cginc"
	
	struct v2f {
		float4 pos : POSITION;
		float2 uv : TEXCOORD0;

		//用一个float4类型存储两组uv信息，不仅减少代码量，还能减少gpu将float2类型转换到float4类型的时间
		float4 uv01 : TEXCOORD1;
		float4 uv23 : TEXCOORD2;
		float4 uv45 : TEXCOORD3;
	};
	
	float4 offsets;
	
	sampler2D _MainTex;
	
	v2f vert (appdata_img v) {
		v2f o;
		o.pos = mul(UNITY_MATRIX_MVP, v.vertex);

		o.uv.xy = v.texcoord.xy;

		o.uv01 =  v.texcoord.xyxy + offsets.xyxy * float4(1,1, -1,-1);
		o.uv23 =  v.texcoord.xyxy + offsets.xyxy * float4(1,1, -1,-1) * 2.0;
		o.uv45 =  v.texcoord.xyxy + offsets.xyxy * float4(1,1, -1,-1) * 3.0;

		return o;
	}
	
	half4 frag (v2f i) : COLOR {
		half4 color = float4 (0,0,0,0);

		color += 0.40 * tex2D (_MainTex, i.uv);
		color += 0.15 * tex2D (_MainTex, i.uv01.xy);
		color += 0.15 * tex2D (_MainTex, i.uv01.zw);
		color += 0.10 * tex2D (_MainTex, i.uv23.xy);
		color += 0.10 * tex2D (_MainTex, i.uv23.zw);
		color += 0.05 * tex2D (_MainTex, i.uv45.xy);
		color += 0.05 * tex2D (_MainTex, i.uv45.zw);
		
		return color;
	}

	ENDCG
	
Subshader {
 Pass {
	  ZTest Always Cull Off ZWrite Off
	  Fog { Mode off }

      CGPROGRAM
      #pragma fragmentoption ARB_precision_hint_fastest
      #pragma vertex vert
      #pragma fragment frag
      ENDCG
  }
}

Fallback off


} // shader
```

一切都是那么的熟悉，没错，这里就是模糊的实现部分，而且还有很多让人惊喜的代码编写技巧。但是！`_MainTex`和`offsets`从哪里来？

大概读了读两个shader，里面都有没有我们熟悉的从material里传入的纹理和数据。

那秘密就藏在这个Component里了，现在来看看CommandBufferBlurRefraction。

```glsl
using UnityEngine;
using UnityEngine.Rendering;
using System.Collections.Generic;

// See _ReadMe.txt for an overview
[ExecuteInEditMode]
public class CommandBufferBlurRefraction : MonoBehaviour
{
	public Shader m_BlurShader;
	private Material m_Material;

	private Camera m_Cam;

	// We'll want to add a command buffer on any camera that renders us,
	// so have a dictionary of them.
	private Dictionary<Camera,CommandBuffer> m_Cameras = new Dictionary<Camera,CommandBuffer>();

	// Remove command buffers from all cameras we added into
	private void Cleanup()
	{
		foreach (var cam in m_Cameras)
		{
			if (cam.Key)
			{
				cam.Key.RemoveCommandBuffer (CameraEvent.AfterSkybox, cam.Value);
			}
		}
		m_Cameras.Clear();
		Object.DestroyImmediate (m_Material);
	}

	public void OnEnable()
	{
		Cleanup();
	}

	public void OnDisable()
	{
		Cleanup();
	}

	// Whenever any camera will render us, add a command buffer to do the work on it
	public void OnWillRenderObject()
	{
		var act = gameObject.activeInHierarchy && enabled;
		if (!act)
		{
			Cleanup();
			return;
		}
		
		var cam = Camera.current;
		if (!cam)
			return;

		CommandBuffer buf = null;
		// Did we already add the command buffer on this camera? Nothing to do then.
		if (m_Cameras.ContainsKey(cam))
			return;

		if (!m_Material)
		{
			m_Material = new Material(m_BlurShader);
			m_Material.hideFlags = HideFlags.HideAndDontSave;
		}

		buf = new CommandBuffer();
		buf.name = "Grab screen and blur";
		m_Cameras[cam] = buf;

		// copy screen into temporary RT
		int screenCopyID = Shader.PropertyToID("_ScreenCopyTexture");
		buf.GetTemporaryRT (screenCopyID, -1, -1, 0, FilterMode.Bilinear);
		buf.Blit (BuiltinRenderTextureType.CurrentActive, screenCopyID);
		
		// get two smaller RTs
		int blurredID = Shader.PropertyToID("_Temp1");
		int blurredID2 = Shader.PropertyToID("_Temp2");
		buf.GetTemporaryRT (blurredID, -2, -2, 0, FilterMode.Bilinear);
		buf.GetTemporaryRT (blurredID2, -2, -2, 0, FilterMode.Bilinear);
		
		// downsample screen copy into smaller RT, release screen RT
		buf.Blit (screenCopyID, blurredID);
		buf.ReleaseTemporaryRT (screenCopyID); 
		
		// horizontal blur
		buf.SetGlobalVector("offsets", new Vector4(2.0f/Screen.width,0,0,0));
		buf.Blit (blurredID, blurredID2, m_Material);
		// vertical blur
		buf.SetGlobalVector("offsets", new Vector4(0,2.0f/Screen.height,0,0));
		buf.Blit (blurredID2, blurredID, m_Material);
		// horizontal blur
		buf.SetGlobalVector("offsets", new Vector4(4.0f/Screen.width,0,0,0));
		buf.Blit (blurredID, blurredID2, m_Material);
		// vertical blur
		buf.SetGlobalVector("offsets", new Vector4(0,4.0f/Screen.height,0,0));
		buf.Blit (blurredID2, blurredID, m_Material);

		buf.SetGlobalTexture("_GrabBlurTexture", blurredID);

		cam.AddCommandBuffer (CameraEvent.AfterSkybox, buf);
	}	
}

```
完美，找到了答案，就是Command Buffer做了上面那些让我们有疑惑的事情。
可以看到这个例子中主要使用了`GetTemporaryRT`,`ReleaseTemporaryRT`,`Blit`,`SetGlobalVector`, `SetGlobalTexture`这里几条指令。

`RenderTargetIdentifier`是RenderTexture的标识，他可以用在`Blit`和`SetRenderTarget`中。从文档中延伸开来会知道，RenderTexture可以有3种标识方式:
1. 直接在编辑器里创建出来的RenderTexture物体 
2. 默认支持的类型，Built-in render textures(BuiltinRenderTextureType) 
3. 临时的RenderTexture，用CommandBuffer.GetTemporaryRT申请的

也就是说`SetGlobalTexture指令`，也可以用RenderTexture。

官方的例子里，用到了2，3两种方式。因为1我们很熟悉。

```glsl
// copy screen into temporary RT
int screenCopyID = Shader.PropertyToID("_ScreenCopyTexture");
buf.GetTemporaryRT (screenCopyID, -1, -1, 0, FilterMode.Bilinear);
buf.Blit (BuiltinRenderTextureType.CurrentActive, screenCopyID);
```

先用`GetTemporaryRT`申请一个临时的RenderTexture，再用`Blit`，将`BuiltinRenderTextureType.CurrentActive`的值赋给申请的临时RenderTexture。完成Grab Texture操作。

更多的API内容可以在官方的手册查看。

## 尝试应用
本来的想法是将CommandBuffer应用到BlurTexture中，但是目前的探究发现CommandBuffer不能达到我的需求。

我想要一个模糊贴图，就像使用UGUI的RawImage一样方便的控制Rect，Depth。如果是用`Camera.AddCommandBuffer`就无法做到，因为CameraEvent并没有精确到某一个物体的事件。

查看官方手册，有`Graphic.ExecuteCommandBuffer`在理论上可以达到需求，但是我们用的`RawImage`使用的是`CanvasRender`，好像无法使用`OnWillRenderObejct()`方法做到渲染前执行。

所以哔哔了这么多，我还没有得到满意的答案=。=跟大家分享学习CommandBuffer的内容，希望能抛砖迎玉，知道答案的朋友也希望能不吝赐教，万分感谢。


