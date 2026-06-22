Unity中内置管线用的Shader语言是[Cg](https://zhida.zhihu.com/search?content_id=243391126&content_type=Article&match_order=1&q=Cg&zhida_source=entity), 但其实本质上还是[HLSL](https://zhida.zhihu.com/search?content_id=243391126&content_type=Article&match_order=1&q=HLSL&zhida_source=entity), 至于为什么以'CGPROGRAM'包裹, 更多地还是历史原因.现在主推[SRP](https://zhida.zhihu.com/search?content_id=243391126&content_type=Article&match_order=1&q=SRP&zhida_source=entity)(包括URP), 标准语言变成了HLSL(High Level Shader Language, 高等着色器语言), 虽然Cg依然受支持, 但推荐使用HLSL.

本文将总结从Cg到HLSL中比较重要的变化

---

## Properties

```text
[MainTexture] _BaseMap("Base Map", 2D) = "white" {} 
// 使用[MainTexture]将该Texture作为API中Material.mainTexture的量
```

## 文件包含

```text
// Cg
#include "UnityCG.cginc"
// HLSL
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
```

## CBUFFER

```text
CBUFFER_START(UnityPerMaterial)
	float4 _BaseTex_ST;
	half4 _BaseColor;
CBUFFER_END
```

注意, Texture的sampler变量并没有包含在CBUFFER中.

## Tags

```text
// URP Tags
"RenderPipeline"="UniversalPipeline"
"LightMode"="UniversalForward"
```

## 常见写法变化

### 坐标转换

```text
// 模型空间->裁剪空间
// Cg
o.positionCS = UnityToClipPos(v.positionOS);
// HLSL
o.positionCS = TransformObjectToHClip(v.positionOS.xyz);

// 模型空间->世界空间
// Cg
o.positionWS = mul(unity_ObjectToWorld, v.positionOS);
// HLSL
o.positionWS = TransformObjectToWorld(v.positionOS.xyz);

// 法线: 模型空间->世界空间
// Cg
o.normalWS = UnityObjectToWorldNormal(v.normalOS);
// HLSL
o.normalWS = TransformObjectToWorldNormal(v.normalOS);

// 世界空间下, 物体表面到摄像机的向量
// Cg
float3 viewWS = normalize(WorldSpaceViewDir(v.positionOS));
// HLSL
float3 viewWS = GetWorldSpaceNormalizeViewDir(positionWS);
```

### 宏

```text
// 纹理的Scale和Tilling
// Cg & HLSL 
o.uv = TRANSFORM_TEX(v.uv, _MainTex); // 没有变化
```

### Texture采样

```text
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);

// HLSL, 需要配合以上声明方式
half4 color = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);
```

### 视线方向

### 光照相关

```C#
// HLSL
Tags{"LightMode" = "UniversalForward"}
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
// 得到光照颜色和光照方向
Light mainLight = GetMainLight();
光照颜色: mianLight.color
光照方向: mainLight.direction
// 得到场景的ambient(Skybox和Light probes)
// vertex中
float3 ambient = SampleSHVertex(normalWS);
// fragment中
float3 ambient = SampleSHPixel(normalWS);

// Cg
Tags{"LightMode" = "ForwardBase"}
#include "Lighting.cginc"
// 可以使用变量
光照颜色: _LightColor0
平行光的光照方向: _WorldSpaceLightPos0
// 得到场景的ambient(Skybox和Light probes)
float3 ambient = ShadeSH9(half4(normalWS, 1));
```



## HLSL模板

```c#
Shader "HLSL/HLSL_Template"
{
	Properties
	{
		_BaseTex ("Base Texture", 2D) = "white" {}
		_BaseColor ("Base Color", Color) = (1,1,1,1)
	}
	SubShader
	{
		Tags  // SubShader Tags 定义何时以及在何种条件下执行 SubShader 块或 Pass。
		{
			"RenderType"="Opaque"
			"Queue"="Geometry"
			"RenderPipeline"="UniversalPipeline" //用于指明使用URP来渲染
		}

		Pass
		{
            	Name "Forward"
			// LightMode tag. Using default here as the shader is Unlit
			// Cull, ZWrite, ZTest, Blend, etc
			Tags
			{
				"LightMode"="UniversalForward"
			}
			HLSLPROGRAM //HLSLPROGRAM/ENDHLSL 标签包裹 HLSL 代码块（使用HLSL语言）
			#pragma vertex vert // 这行代码定义了顶点着色器（vertex shader）的名称
			#pragma fragment frag  // 这行代码定义了片元着色器（fragment shader）的名称。
                
                 // Core.hlsl 文件包含常用 HLSL 宏和函数的定义，
            // 并且还包含对其他 HLSL 文件的 #include 引用（例如 Common.hlsl、SpaceTransforms.hlsl 等）。
			#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

			TEXTURE2D(_BaseMap); //贴图采样  
             SAMPLER(sampler_BaseMap);
			CBUFFER_START(UnityPerMaterial) //声明变量
				float4 _BaseTex_ST;
				half4 _BaseColor;
			CBUFFER_END

			struct appdate
			{
				float4 positionOS: POSITION;
				float2 uv: TEXCOORD0;
			};

			struct v2f
			{
				float4 positionCS: SV_POSITION;
				float2 uv: TEXCOORD0;
			};

			v2f vert(appdate v)
			{
				v2f o;
				o.positionCS = TransformObjectToHClip(v.positionOS.xyz);
				o.uv = TRANSFORM_TEX(v.uv, _BaseTex);
				return o;
			}

			half4 frag(v2f i): SV_Target
			{
				half4 color = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);;
				return color * _BaseColor;
			}
			ENDHLSL
		}
	}
	Fallback Off
}
```



1）CBUFFER_START和CBUFFER_END：变量是单个材质独有的时候建议放在这里面，以提高性能。CBUFFER(常量缓冲区)的空间较小，不适合存放纹理贴图这种大量数据的数据类型，适合存放float，half之类的不占空间的数据

2）TEXTURE2D (_BaseMap)和SAMPLER(sampler_BaseMap) ：贴图采样，放在CBUFFER下面

3）SAMPLE_TEXTURE2D(textureName, samplerName, xxx.uv) ：具有三个变量，分别是TEXTURE2D (_MainTex)的变量和SAMPLER(sampler_MainTex)的变量和uv

4）渲染管线的标签为"RenderPipeline"="UniversalRenderPipeline" 
