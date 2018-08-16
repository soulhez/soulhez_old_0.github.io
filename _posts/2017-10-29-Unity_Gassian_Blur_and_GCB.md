---
layout: post
title:  "Shader Playground:高斯模糊效果"
date:   2017-10-29 13:51:00 +0800
categories: CG 
tags: Unity Shader GassionBlur 
---

## 简介
上一次我们用GrabPass实现了简单的线性模糊效果。但是效果实在是让人无法恭维=。=看久了眼睛是要彻底报废的。

模糊效果界最有名的效果就是高斯模糊，俗称毛玻璃效果，今天我们来弄明白他并且实现它。

## 高斯模糊
先让我们来看看大名鼎鼎的高斯模糊的效果。

![高斯模糊效果图](http://oxujermt3.bkt.clouddn.com/iamge/shaderplayground/blur/xiaoguotu.jpeg)

是不是比线性模糊效果要平滑很多！

那么高斯模糊到底是何方神圣呢？我们直接搜索，得到`正态分布`，`高斯函数`等的让人望而生畏的结果。

我们要实现高斯模糊，只要明白为什么应用了正态分布，就能达到我们的模糊效果。

我们先来想想什么是模糊效果，假设我们现在显示器分辨率是1920x1080，那么我们同时打开两张图片，一张尺寸是64x64，还有一张尺寸是512x512。如果让我们同时把两个图片到拉升到1024x1024大小的画布上，我们很快就会发现小尺寸的图片模糊的厉害！为什么？因为他只有64x64个有准确颜色信息的像素，强行让他铺满1024x1024大小的画布，他根本不知道其他的像素该是什么颜色。

反过来想，如果我们要让图片变模糊，就是强行让图片的像素携带的颜色信息变少。之前的线性模糊就是这种做法，让一个像素和他周围上下左右的像素颜色一样，这样就会让原先的像素颜色信息丢失。就达到了模糊的目的。

那么问题来了，当我们在让像素失去原有颜色信息的时候用上正态分布，`越接近中心像素的像素可以对中心像素的最后颜色造成更大的影响，距离中心像素越远的像素会中心像素最后颜色造成的影响越小。`然后中心像素周围的像素的颜色的影响力之和就是我们中心像素最后的颜色，如果对一张图片中所有的像素都做同样的处理，那么原先的颜色信息就会丢失，每个像素的颜色信息都会很接近周围像素的信息，使得最后的颜色丢失不会很突兀，而是显得更加平滑。

这样理解的话，那么我们只要在渲染颜色的时候加上影响力权重就能实现高斯模糊的效果了！没错，那么这些权重值从哪里来呢？这些权重值就是由高斯函数计算出来的。

这里我使用了[阮一峰大神的计算结果](http://www.ruanyifeng.com/blog/2012/11/gaussian_blur.html)

修改我们之前的Shader:
```glsl
Shader "ShaderPlayground/GassionBlur" {
	Properties{
		_MainTex("Base (RGB)", 2D) = "white" {}
		_BlurRadius("Raius", Range(0, 10)) = 0
	}
	SubShader{
		//UGUI的RenderQueue在Transparent
		Tags{ "Queue" = "Transparent"}

		GrabPass{}

		Pass{
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			ENDCG
		}
	}

	CGINCLUDE
		#include "UnityCG.cginc"
		uniform sampler2D _MainTex;
		float2 _MainTex_TexelSize;
		sampler2D _GrabTexture;
		float2 _GrabTexture_TexelSize;
		float _BlurRadius;

		struct appdata {
			float4 pos:POSITION;
			float2 uv:TEXCOORD0;
			float4 color:COLOR;
		};

		struct v2f {
			float4 pos:SV_POSITION;
			float2 uv:TEXCOORD0;
			float2 uv1:TEXCOORD1;
			float2 uv2:TEXCOORD2;
			float2 uv3:TEXCOORD3;
			float2 uv4:TEXCOORD4;
			float2 uv5:TEXCOORD5;
			float2 uv6:TEXCOORD6;
			float2 uv7:TEXCOORD7;
			float2 uv8:TEXCOORD8;
			float4 color:COLOR;
		};

		v2f vert(appdata i) {
			v2f o;
			o.pos = mul(UNITY_MATRIX_MVP, i.pos);
			o.uv = i.uv.xy;
#if UNITY_UV_STARTS_AT_TOP
			if (_GrabTexture_TexelSize.y < 0)
				o.uv.y = 1 - o.uv.y;
#endif
			o.uv1 = o.uv.xy + _BlurRadius * _GrabTexture_TexelSize * float2(1, 1);
			o.uv2 = o.uv.xy + _BlurRadius * _GrabTexture_TexelSize * float2(-1, 1);
			o.uv3 = o.uv.xy + _BlurRadius * _GrabTexture_TexelSize * float2(-1, -1);
			o.uv4 = o.uv.xy + _BlurRadius * _GrabTexture_TexelSize * float2(1, -1);
			o.uv5 = o.uv.xy + _BlurRadius * _GrabTexture_TexelSize * float2(0, 1);
			o.uv6 = o.uv.xy + _BlurRadius * _GrabTexture_TexelSize * float2(-1, 0);
			o.uv7 = o.uv.xy + _BlurRadius * _GrabTexture_TexelSize * float2(0, -1);
			o.uv8 = o.uv.xy + _BlurRadius * _GrabTexture_TexelSize * float2(1, 0);
			o.color = i.color;
			return o;
		}

		fixed4 frag(v2f i) : SV_Target{
			fixed4 color = fixed4(0, 0, 0, 0);
			color += 0.14 	* tex2D(_GrabTexture, i.uv);
			color += 0.125	* tex2D(_GrabTexture, i.uv5);
			color += 0.125	* tex2D(_GrabTexture, i.uv6);
			color += 0.125	* tex2D(_GrabTexture, i.uv7);
			color += 0.125	* tex2D(_GrabTexture, i.uv8);
			color += 0.09 	* tex2D(_GrabTexture, i.uv1);
			color += 0.09 	* tex2D(_GrabTexture, i.uv2);
			color += 0.09 	* tex2D(_GrabTexture, i.uv3);
			color += 0.09 	* tex2D(_GrabTexture, i.uv4);
			return color;
		}
	ENDCG
}
```

## 效果对比
OK，高斯模糊done！让我们来看看效果的对比。

![效果对比图片](http://oxujermt3.bkt.clouddn.com/iamge/shaderplayground/blur/duibixiaoguo.png)

效果是不是提升了很多呢？想要查看demo的朋友，可以fork我的demo工程。
[ForkMeIn Github](https://github.com/aaBaO/DemoRepository.git)

现在的效果还不是很完美，之前一直在使用的`BlurRadius`似乎很尴尬，他的值足够大时又要伤害我们的眼睛了=。=
并不是我想要的结果啊！怎么办，下次尝试用Graphic Command Buffer来解决看看。
