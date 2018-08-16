---
layout: post
title:  "Shader Playground:Unity UGUI模糊贴图纹理效果实现"
date:   2017-10-15 14:51:00 +0800
categories: CG 
tags: CG Unity UGUI Shader
---

## 效果预览
![效果图](http://oxujermt3.bkt.clouddn.com/iamge/shaderplayground/blur/blur.gif)

这是一个简单的线性模糊的效果。不用RenderTexture。
一个shader，一个脚本，一个材质球就能快速实现。
## 实现思路
要达到这个效果，分两步实现，
1. 用Unity ShaderLab里的GrabPass来抓取屏幕像素
2. 重新计算并修改MainTeture的UV，达到只模糊RawImage的Rect范围内的图像。

### GrabPass
下面是来自Unity5.4beta官方文档对GrabPass的描述：
> GrabPass is a special passtype - it grabs the contents of the screen where the object is about to be drawn into a texture. 
This texture can be used in subsequent passes to do advanced image based effects.
GrabPass可以把屏幕内容抓取到一个贴图纹理里，然后这个贴图纹理可以在之后的pass中使用。

他的用法不难，下面是官方给的的例子：
```glsl
Shader "GrabPassInvert"
{
    SubShader
    {
        // Draw ourselves after all opaque geometry
        Tags { "Queue" = "Transparent" }

        // Grab the screen behind the object into _GrabTexture
        GrabPass { }

        // Render the object with the texture generated above, and invert the colors
        Pass
        {
            SetTexture [_GrabTexture] { combine one-texture }
        }
    }
}
```

实际这是一个Pass，可以抓取到屏幕内容。
直接只用GrabPass{}后，会把内容存储在默认的变量`_GrabTexture`中
如果想要将内容存放在另外的变量，可以用GrabPass{"TextureName"}
比如我们的变量是`_MyGrabTexture`这样官方的例子就变成了这样：
```glsl
Shader "GrabPassInvert"
{
    SubShader
    {
        // Draw ourselves after all opaque geometry
        Tags { "Queue" = "Transparent" }

        // Grab the screen behind the object into _MyGrabTexture
        GrabPass { "_MyGrabTexture" }

        // Render the object with the texture generated above, and invert the colors
        Pass
        {
            SetTexture [_MyGrabTexture] { combine one-texture }
        }
    }
}
```

### 重新计算TextureUV
由于GrabPass抓取的是整个屏幕所有的内容，那我们可以通过改变纹理的UV就可以实现将屏幕对应区域的内容进行显示。
UV的坐标系原点屏幕左下
在UGUI中，RectTransform.rect.center的值是当前rect中心的点的坐标，但是他的坐标系的原点RectTransform的Pivot
因此我们要得到纹理的正确UV还需要计算出正确的坐标。
```csharp
using UnityEngine;
using UnityEngine.UI;

[RequireComponent(typeof(RawImage))]
public class BlurTexture : MonoBehaviour {

	private RawImage texture;
	/// <summary>
	/// texture所在的Root Canvas
	/// </summary>
	public Canvas rootCanvas;

	private void Awake(){
		texture = GetComponent<RawImage>();		
	}

	private void Update()
	{
		var screenWidth = rootCanvas.pixelRect.xMax / rootCanvas.scaleFactor;
		var screenHeight = rootCanvas.pixelRect.yMax / rootCanvas.scaleFactor;

		//得到一个从屏幕中心点到texture左下角的点的向量
		var bottomLeft = texture.rectTransform.anchoredPosition + texture.rectTransform.rect.min;

		var normalWidth = texture.rectTransform.rect.width / screenWidth;
		var normalHeight = texture.rectTransform.rect.height  / screenHeight;

		//uv的xy起点为(0,0)，我们在计算坐下角的点时是从屏幕中心点开始，因此要加上0.5
		texture.uvRect = new Rect(0.5f + bottomLeft.x / screenWidth, 0.5f + bottomLeft.y / screenHeight, normalWidth, normalHeight);
	}
}
```

### 简单模糊
这里我实现的是相对简单，但是效果欠佳的线性模糊。
```glsl
Shader "ShaderPlayground/BlurTexture" {
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

			#include "UnityCG.cginc"

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
				o.color = i.color;
				return o;
			}

			fixed4 frag(v2f i) : SV_Target{
				fixed4 color = fixed4(0, 0, 0, 0);
				color += tex2D(_GrabTexture, i.uv);
				color += tex2D(_GrabTexture, i.uv1);
				color += tex2D(_GrabTexture, i.uv2);
				color += tex2D(_GrabTexture, i.uv3);
				color += tex2D(_GrabTexture, i.uv4);
				return color * 0.2 ;
			}
			ENDCG
		}
	}
}
```

### Bingo
这样我们就能得到一个随意移动缩放的马赛克了！

后面我还想实现高斯模糊，动态模糊效果，还有移动端优化，让手机也能看到马赛克。

这是demo工程的地址[ForkMeInGithub](https://github.com/aaBaO/DemoRepository.git)

有不对的地方欢迎大家指正和探讨~~
