# URP的变化

博主连接：http://blog.coolcoding.cn/?page_id=627

[原文：https://cyangamedev.wordpress.com/2020/06/05/urp-shader-code/](https://cyangamedev.wordpress.com/2020/06/05/urp-shader-code/)

---

URP中，RenderType可能不太重要了，在内置管线中，是用来做ReplacementShaders用的
但是在URP中不支持ReplacementShader，尽管有个ForwardRenderer的overrideMaterail
每个Pass标签都需要标记特定的LightMode，URP使用 single-pass forward renderer.
所有只有一个“UniversalFoward”的Pass，也不能同时渲染多个对象。
也可以不加Tag，但是会破坏SRP Batcher
建议在单独的MeshRender上使用独立的Shader或者材质
或者使用Forward Renderer上overrideMaterial的“Render Objects 特性

---

URP LightMode Tags：
Tags{“LightMode” = “XXX”}
UniversalForward：前向渲染物件之用
ShadowCaster： 投射阴影之用
DepthOnly：只用来产生深度图
Mata：来用烘焙光照图之用
Universal2D ：做2D游戏用的，用来替代前向渲染
UniversalGBuffer ： 貌似与延迟渲染相关（开发中）

---

Unity想弃用CG，推荐使用HLSL(High level shading language-高级着色语言)
不再有fixed类型,只有half和float
Cg和HLSL被视为相同的语言
如果在URP中使用CG的标签，将与URPShaderLibrary冲突
因为变量和函数会被重复定义
现在请使用 HLSLPROGRAM, HLSLINCLUDE, ENDHLSL
不推荐用CGPROGRAM, ENDCG, CGINCLUDE

---

Pass的Name，应该全大写
可以用UsePass来引用
例如：UsePass “Custom/UnlitShaderExample/MyShader”
为了与SPRBatcher兼容，所有传递必须共享相同的UnityPerMaterial CBUFFER
如果不匹配，则出错

---

HLSL数据类型1 – 基础数据
bool – true / false.
float – 32位浮点数，用在比如世界坐标，纹理坐标，复杂的函数计算
half – 16位浮点数，用于短向量、方向、颜色，模型空间位置
double – 64位浮点数，不能用于输入输出，要使用double，得声明为一对unit再用asuint把double打包到uint对中，再用asdouble函数解包
fixed – 只能用于内建管线，URP不支持，用half替代
real – 好像只用于URP，如果平台指定了用half（#define PREFER_HALF 0），否则就是float类型
int – 32位有符号整形
uint – 32位无符号整形(GLES2不支持，会用int替代)

---

HLSL数据类型2 – 向量
vector类型可以直接在基础数据后添加维度
例如：float4, half3, int2 …
访问可以用xyzw或者rgba访问

---

HLSL数据类型3 – 矩阵
matrix类型可以直接在基础数据后添加 维度 x 维度
例如：float4x4, int4x3, half2x1
即表达 4行4列的float，4行3列的int，2行1列的half
float3x3 m = { 0, 1, 2, 3, 4, 5, 6, 7, 8};
float3 row0 = m[0]; // 0, 1, 2
float r1c2 = m[1][2]; // 5
乘法，使用mul进行
mul(m, row0)
第1个列数必须与第2个行数相同

---

HLSL数据类型4 – 数组
ShaderLab材质属性面板Properties不支持数组，只能从C#中设置
必须在Shader中指定数组的大小，例如
float array[10];
float4x4 array2[10];

---

纹理和采样
定义：
TEXTURE2D(textureName);
SAMPLER(sampler_textureName);
缓存区：
从C#中使用material.SetBuffer或者 Shader.SetGlobalBuffer
例如：StructuredBuffer buffer;

---

宏(macro)
define MUL2(x,y) ((x)*(y))
可以做一些语法糖，例如：
define TRANSFORM_TEX(tex, name) (tex.xy* name##_ST.xy + name##_ST.zw)
o.uv = TRANSFORM_TEX(in.uv, _MainTex) =>
o.uv = (in.uv.xy * _MainTex_ST.xy + _MainTex.ST_zw)

---

每个Pass, UnityPerMaterial CBUFFER都是相同的
CBUFFER需要包含所有公用的属性（即与ShaderLab中的Properties相同）
它不能包含其它未公开的属性以及纹理采样器
虽然不需要通过C# material.SetColor/SetFloat等
但多Material实例具有不同的值，这会产生问题，SRP Batcher会将他们一起批处理
1、如果您有未公开的变量，请始终使用Shader.SetGlobalColor / Float
以使它们在所有材质实例中保持不变。
2、如果每个材料都不同，则通过Shaderlab属性块将它们公开，然后将它们添加到CBUFFER中

---

HLSLINCLUDE
\#include “Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl”
CBUFFER_START(UnityPerMaterial) float4 _BaseMap_ST; float4 _BaseColor; CBUFFER_END
ENDHLSL
Core.hlsl文件是URP的内置核心文件
比如如果要使用灯光，则添加 Lighting.hlsl文件

---

Vertex Shader 阶段
在内置Shader中，使用UnityObjectToClipPos将模型空间转换到裁剪空间
URP中使用TransformObjectToHClip函数（在SpaceTransforms.hlsl中有定义）
URP中也可以使用GetVertexPositionInputs函数（在Core.hlsh中有定义）
得到的VertexPositionInputs 结构体包含以下内容
positionWS = positionWorldSpace
positionVS = positionViewSpace
positionCS = positionClipSpace
positionNDC = position in Normalised Device Coordinates
我们可以直接使用上述代码，即便含有未使用到的变量
因为Shader编译器会自动对上述代码进行优化。
法线&切线
VertexNormalInputs normalInputs = GetVertexNormalInputs(IN.normalOS, IN.tangentOS);
GetVertexNormalInputs将模型空间法线和切线转换为世界空间
包含normalWS, tangentWS, bitangetWS。

Fragment Shader 阶段

可以使用 SAMPLE_TEXTURE2D 宏进行采样

---

光照1

URP不支持SurfaceShader

URP的一些光照文件在 Lighting.hlsl 中

可以用来处理例如 UniversalFragmentPBR 

以下代码展示了阴影相关的宏

```C#
#pragma multi_compile _ _MAIN_LIGHT_SHADOWS

#pragma multi_compile _ _MAIN_LIGHT_SHADOWS_CASCADE

#pragma multi_compile _ _SHADOWS_SOFT

#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

------------------------------------------------------------------

光照2 - 示例

struct Attributes {

  ...

  float4 normalOS   : NORMAL;

};

struct Varyings {

  ...

  float3 normalWS   : NORMAL;

  float3 positionWS  : TEXCOORD2;

};

...

Varyings vert(Attributes IN) {

  Varyings OUT;

  VertexPositionInputs positionInputs = GetVertexPositionInputs(IN.positionOS.xyz);

  ...

  OUT.positionWS = positionInputs.positionWS;

  VertexNormalInputs normalInputs = GetVertexNormalInputs(IN.normalOS.xyz);

  OUT.normalWS = normalInputs.normalWS;

  return OUT;

}

half4 frag(Varyings IN) : SV_Target {

  half4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);

  half4 color = baseMap * _BaseColor * IN.color;

  float4 shadowCoord = TransformWorldToShadowCoord(IN.positionWS.xyz);

  Light light = GetMainLight(shadowCoord);

  half3 diffuse = LightingLambert(light.color, light.direction, IN.normalWS);

  return half4(color.rgb * diffuse * light.shadowAttenuation, color.a);

}
```



\------------------------------------------------------------------

光照-3

```C#
如果要继续扩展例如 ambient、bakedGI 以及 多光源

可以参考

https://github.com/Unity-Technologies/Graphics/blob/fcacf7661c3976921d4f5dfbdc17569db7ea77a3/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl#L832

（UniversalFragmentBlinnPhong method in Lighting.hlsl ）

------------------------------------------------------------------

PBR 光照

在URP中使用”Lit”Shader

在shadergraph中使用PBR Master 节点

URP不支持SurfaceShader，所以只能使用ShaderLibrary中的函数（在Lighting.hlsl中）

重点为UniversalFragmentPBR 

PBR 示例

Properties {

  _BaseMap ("Base Texture", 2D) = "white" {}

  _BaseColor ("Example Colour", Color) = (0, 0.66, 0.73, 1)

  _Smoothness ("Smoothness", Float) = 0.5

 

  [Toggle(_ALPHATEST_ON)] _EnableAlphaTest("Enable Alpha Cutoff", Float) = 0.0

  _Cutoff ("Alpha Cutoff", Float) = 0.5

 

  [Toggle(_NORMALMAP)] _EnableBumpMap("Enable Normal/Bump Map", Float) = 0.0

  _BumpMap ("Normal/Bump Texture", 2D) = "bump" {}

  _BumpScale ("Bump Scale", Float) = 1

 

  [Toggle(_EMISSION)] _EnableEmission("Enable Emission", Float) = 0.0

  _EmissionMap ("Emission Texture", 2D) = "white" {}

  _EmissionColor ("Emission Colour", Color) = (0, 0, 0, 0)

  }

...

// And need to adjust the CBUFFER to include these too

CBUFFER_START(UnityPerMaterial)

  float4 _BaseMap_ST; // Texture tiling & offset inspector values

  float4 _BaseColor;

  float _BumpScale;

  float4 _EmissionColor;

  float _Smoothness;

  float _Cutoff;

CBUFFER_END

// Material Keywords

\#pragma shader_feature _NORMALMAP

\#pragma shader_feature _ALPHATEST_ON

\#pragma shader_feature _ALPHAPREMULTIPLY_ON

\#pragma shader_feature _EMISSION

//#pragma shader_feature _METALLICSPECGLOSSMAP

//#pragma shader_feature _SMOOTHNESS_TEXTURE_ALBEDO_CHANNEL_A

//#pragma shader_feature _OCCLUSIONMAP

 

//#pragma shader_feature _SPECULARHIGHLIGHTS_OFF

//#pragma shader_feature _ENVIRONMENTREFLECTIONS_OFF

//#pragma shader_feature _SPECULAR_SETUP

\#pragma shader_feature _RECEIVE_SHADOWS_OFF

 

// URP Keywords

\#pragma multi_compile _ _MAIN_LIGHT_SHADOWS

\#pragma multi_compile _ _MAIN_LIGHT_SHADOWS_CASCADE

\#pragma multi_compile _ _ADDITIONAL_LIGHTS_VERTEX _ADDITIONAL_LIGHTS

\#pragma multi_compile _ _ADDITIONAL_LIGHT_SHADOWS

\#pragma multi_compile _ _SHADOWS_SOFT

\#pragma multi_compile _ _MIXED_LIGHTING_SUBTRACTIVE

 

// Unity defined keywords

#pragma multi_compile _ DIRLIGHTMAP_COMBINED

#pragma multi_compile _ LIGHTMAP_ON

#pragma multi_compile_fog

 

// Some added includes, required to use the Lighting functions

#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

// And this one for the SurfaceData struct and albedo/normal/emission sampling functions.

// Note : It also defines the _BaseMap, _BumpMap and _EmissionMap textures for us, so we should use these as Shaderlab Properties too.

\#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/SurfaceInput.hlsl"

 

struct Attributes {

  float4 positionOS  : POSITION;

  float3 normalOS   : NORMAL;

  float4 tangentOS  : TANGENT;

  float4 color    : COLOR;

  float2 uv      : TEXCOORD0;

  float2 lightmapUV  : TEXCOORD1;

};

 

struct Varyings {

  float4 positionCS        : SV_POSITION;

  float4 color          : COLOR;

  float2 uv            : TEXCOORD0;

  DECLARE_LIGHTMAP_OR_SH(lightmapUV, vertexSH, 1);

  // Note this macro is using TEXCOORD1

\#ifdef REQUIRES_WORLD_SPACE_POS_INTERPOLATOR

  float3 positionWS        : TEXCOORD2;

\#endif

  float3 normalWS         : TEXCOORD3;

\#ifdef _NORMALMAP

  float4 tangentWS        : TEXCOORD4;

\#endif

  float3 viewDirWS        : TEXCOORD5;

  half4 fogFactorAndVertexLight  : TEXCOORD6;

  // x: fogFactor, yzw: vertex light

\#ifdef REQUIRES_VERTEX_SHADOW_COORD_INTERPOLATOR

  float4 shadowCoord       : TEXCOORD7;

\#endif

};

 

//TEXTURE2D(_BaseMap);

//SAMPLER(sampler_BaseMap);

// Removed, since SurfaceInput.hlsl now defines the _BaseMap for us

\#if SHADER_LIBRARY_VERSION_MAJOR < 9

  // This function was added in URP v9.x.x versions

  // If we want to support URP versions before, we need to handle it instead.

  // Computes the world space view direction (pointing towards the viewer).

  float3 GetWorldSpaceViewDir(float3 positionWS) {

​    if (unity_OrthoParams.w == 0) {

​      // Perspective

​      return _WorldSpaceCameraPos - positionWS;

​    } else {

​      // Orthographic

​      float4x4 viewMat = GetWorldToViewMatrix();

​      return viewMat[2].xyz;

​    }

  }

\#endif

 

Varyings vert(Attributes IN) {

  Varyings OUT;

 

  // Vertex Position

  VertexPositionInputs positionInputs = GetVertexPositionInputs(IN.positionOS.xyz);

  OUT.positionCS = positionInputs.positionCS;

\#ifdef REQUIRES_WORLD_SPACE_POS_INTERPOLATOR

  OUT.positionWS = positionInputs.positionWS;

\#endif

  // UVs & Vertex Colour

  OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);

  OUT.color = IN.color;

 

  // View Direction

  OUT.viewDirWS = GetWorldSpaceViewDir(positionInputs.positionWS);

 

  // Normals & Tangents

  VertexNormalInputs normalInputs = GetVertexNormalInputs(IN.normalOS, IN.tangentOS);

  OUT.normalWS = normalInputs.normalWS;

\#ifdef _NORMALMAP

  real sign = IN.tangentOS.w * GetOddNegativeScale();

  OUT.tangentWS = half4(normalInputs.tangentWS.xyz, sign);

\#endif

 

  // Vertex Lighting & Fog

  half3 vertexLight = VertexLighting(positionInputs.positionWS, normalInputs.normalWS);

  half fogFactor = ComputeFogFactor(positionInputs.positionCS.z);

  OUT.fogFactorAndVertexLight = half4(fogFactor, vertexLight);

 

  // Baked Lighting & SH (used for Ambient if there is no baked)

  OUTPUT_LIGHTMAP_UV(IN.lightmapUV, unity_LightmapST, OUT.lightmapUV);

  OUTPUT_SH(OUT.normalWS.xyz, OUT.vertexSH);

 

  // Shadow Coord

\#ifdef REQUIRES_VERTEX_SHADOW_COORD_INTERPOLATOR

  OUT.shadowCoord = GetShadowCoord(positionInputs);

\#endif

  return OUT;

}

InputData InitializeInputData(Varyings IN, half3 normalTS){

  InputData inputData = (InputData)0;

 

\#if defined(REQUIRES_WORLD_SPACE_POS_INTERPOLATOR)

  inputData.positionWS = IN.positionWS;

\#endif

​         

  half3 viewDirWS = SafeNormalize(IN.viewDirWS);

\#ifdef _NORMALMAP

  float sgn = IN.tangentWS.w; // should be either +1 or -1

  float3 bitangent = sgn * cross(IN.normalWS.xyz, IN.tangentWS.xyz);

  inputData.normalWS = TransformTangentToWorld(normalTS, half3x3(IN.tangentWS.xyz, bitangent.xyz, IN.normalWS.xyz));

\#else

  inputData.normalWS = IN.normalWS;

\#endif

 

  inputData.normalWS = NormalizeNormalPerPixel(inputData.normalWS);

  inputData.viewDirectionWS = viewDirWS;

 

\#if defined(REQUIRES_VERTEX_SHADOW_COORD_INTERPOLATOR)

  inputData.shadowCoord = IN.shadowCoord;

\#elif defined(MAIN_LIGHT_CALCULATE_SHADOWS)

  inputData.shadowCoord = TransformWorldToShadowCoord(inputData.positionWS);

\#else

  inputData.shadowCoord = float4(0, 0, 0, 0);

\#endif

 

  inputData.fogCoord = IN.fogFactorAndVertexLight.x;

  inputData.vertexLighting = IN.fogFactorAndVertexLight.yzw;

  inputData.bakedGI = SAMPLE_GI(IN.lightmapUV, IN.vertexSH, inputData.normalWS);

  return inputData;

}

 

SurfaceData InitializeSurfaceData(Varyings IN){

  SurfaceData surfaceData = (SurfaceData)0;

  // Note, we can just use SurfaceData surfaceData; here and not set it.

  // However we then need to ensure all values in the struct are set before returning.

  // By casting 0 to SurfaceData, we automatically set all the contents to 0.

​     

  half4 albedoAlpha = SampleAlbedoAlpha(IN.uv, TEXTURE2D_ARGS(_BaseMap, sampler_BaseMap));

  surfaceData.alpha = Alpha(albedoAlpha.a, _BaseColor, _Cutoff);

  surfaceData.albedo = albedoAlpha.rgb * _BaseColor.rgb * IN.color.rgb;

 

  // Not supporting the metallic/specular map or occlusion map

  // for an example of that see : https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.universal/Shaders/LitInput.hlsl

 

  surfaceData.smoothness = _Smoothness;

  surfaceData.normalTS = SampleNormal(IN.uv, TEXTURE2D_ARGS(_BumpMap, sampler_BumpMap), _BumpScale);

  surfaceData.emission = SampleEmission(IN.uv, _EmissionColor.rgb, TEXTURE2D_ARGS(_EmissionMap, sampler_EmissionMap));

  surfaceData.occlusion = 1;

  return surfaceData;

}

 

half4 frag(Varyings IN) : SV_Target {

  SurfaceData surfaceData = InitializeSurfaceData(IN);

  InputData inputData = InitializeInputData(IN, surfaceData.normalTS);

​         

  // In URP v10+ versions we could use this :

  // half4 color = UniversalFragmentPBR(inputData, surfaceData);

 

  // But for other versions, we need to use this instead.

  // We could also avoid using the SurfaceData struct completely, but it helps to organise things.

  half4 color = UniversalFragmentPBR(inputData, surfaceData.albedo, surfaceData.metallic, 

   surfaceData.specular, surfaceData.smoothness, surfaceData.occlusion, 

   surfaceData.emission, surfaceData.alpha);

​         

  color.rgb = MixFog(color.rgb, inputData.fogCoord);

 

  // color.a = OutputAlpha(color.a);

  // Not sure if this is important really. It's implemented as :

  // saturate(outputAlpha + _DrawObjectPassData.a);

  // Where _DrawObjectPassData.a is 1 for opaque objects and 0 for alpha blended.

  // But it was added in URP v8, and versions before just didn't have it.

  // And I'm writing thing for v7.3.1 currently

  // We could still saturate the alpha to ensure it doesn't go outside the 0-1 range though :

  color.a = saturate(color.a);

 

  return color;

}

\------------------------------------------------------------------
```

阴影 Pass

添加 "LightMode" = "ShadowCaster" 

使用示例ShadowCast

Pass {

  Name "ShadowCaster"

  Tags { "LightMode"="ShadowCaster" }

 

  ZWrite On

  ZTest LEqual

 

  HLSLPROGRAM

  // Required to compile gles 2.0 with standard srp library

  \#pragma prefer_hlslcc gles

  \#pragma exclude_renderers d3d11_9x gles

  //#pragma target 4.5

 

  // Material Keywords

  \#pragma shader_feature _ALPHATEST_ON

  \#pragma shader_feature _SMOOTHNESS_TEXTURE_ALBEDO_CHANNEL_A

 

  // GPU Instancing

  \#pragma multi_compile_instancing

  \#pragma multi_compile _ DOTS_INSTANCING_ON

​       

  \#pragma vertex ShadowPassVertex

  \#pragma fragment ShadowPassFragment

   

  \#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/CommonMaterial.hlsl"

  \#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/SurfaceInput.hlsl"

  \#include "Packages/com.unity.render-pipelines.universal/Shaders/ShadowCasterPass.hlsl"

 

  ENDHLSL

}

或者自己写

\#pragma vertex vert

// function copied from ShadowCasterPass and edited slightly.

Varyings vert(Attributes input) {

  Varyings output;

  UNITY_SETUP_INSTANCE_ID(input);

   // Example Displacement

  input.positionOS += float4(0, _SinTime.y, 0, 0);

   output.uv = TRANSFORM_TEX(input.texcoord, _BaseMap);

  output.positionCS = GetShadowPositionHClip(input);

  return output;

}

\------------------------------------------------------------------

深度 Pass 添加 "LightMode" = "DepthOnly"

示例：

Pass {

  Name "DepthOnly"

  Tags { "LightMode"="DepthOnly" }

 

  ZWrite On

  ColorMask 0

 

  HLSLPROGRAM

  // Required to compile gles 2.0 with standard srp library

  \#pragma prefer_hlslcc gles

  \#pragma exclude_renderers d3d11_9x gles

  //#pragma target 4.5

 

  // Material Keywords

  \#pragma shader_feature _ALPHATEST_ON

  \#pragma shader_feature _SMOOTHNESS_TEXTURE_ALBEDO_CHANNEL_A

 

  // GPU Instancing

  \#pragma multi_compile_instancing

  \#pragma multi_compile _ DOTS_INSTANCING_ON

​       

  \#pragma vertex DepthOnlyVertex

  \#pragma fragment DepthOnlyFragment

​       

  \#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/CommonMaterial.hlsl"

  \#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/SurfaceInput.hlsl"

  \#include "Packages/com.unity.render-pipelines.universal/Shaders/DepthOnlyPass.hlsl"

 

  // Again, using this means we also need _BaseMap, _BaseColor and _Cutoff shader properties

  // Also including them in cbuffer, except _BaseMap as it's a texture.

 

  ENDHLSL

}

\------------------------------------------------------------------

Built-in中

float4 lightColor = _LightColor0;

float3 lightDir = WorldSpaceLightDir(worldPos);

UNITY_LIGHT_ATTENUATION(atten, i, worldPos.xyz);

URP中

Lighting.hlsl:

  GetMainLight();

​    _MainLightPosition; 

Input.hlsl:

  //Constant Buffers

  float4 _MainLightPosition; // 主光方向(dir)

  half4 _MainLightColor; // 主光颜色

\------------------------------------------------------------------

GI

Built-in中

float sh = 0;

float3 SH = ShadeSHPerVertex(worldNormal, sh);

URP中

half3 SH = SampleSH(normalWS);

half3 SH = SampleSHVertex(normalWS);

half3 SH = SampleSHPixel(L2Term, normalWS);

\------------------------------------------------------------------

常用顶点函数的改动

VertexPositionInputs vertexInput = GetVertexPositionInputs(input.positionOS.xyz);

VertexNormalInputs normalInput = GetVertexNormalInputs(input.normalOS, input.tangentOS);

half3 viewDirWS = GetCameraPositionWS() - vertexInput.positionWS;

half3 vertexLight = VertexLighting(vertexInput.positionWS, normalInput.normalWS);

half fogFactor = ComputeFogFactor(vertexInput.positionCS.z);

output.fogFactorAndVertexLight = half4(fogFactor, vertexLight);

inputData.fogCoord = input.fogFactorAndVertexLight.x;

output.normal = half4(normalInput.normalWS, viewDirWS.x);

output.tangent = half4(normalInput.tangentWS, viewDirWS.y);

output.bitangent = half4(normalInput.bitangentWS, viewDirWS.z);

output.normalWs = NormalizeNormalPerVertex (normalInput.normalws);

\------------------------------------------------------------------

常用采样的改动

half4 col = SAMPLE_TEXTURE2D(_MainTex,sampler_MainTex, uv) ;

\------------------------------------------------------------------

总结

1、Use the “RenderPipeline”=”UniversalPipeline” tag on the Subshader block

URP uses the following “LightMode” tags :

UniversalForward

ShadowCaster

DepthOnly

Meta

Universal2D

2、

URP uses a single-pass forward rendered, 

so only the first supported UniversalForward pass will be rendered. 

While you can achieve multi-pass shaders with other passes untagged, 

be aware that it will break batching with the SRP Batcher. 

It is instead recommended to use separate shaders/materials, 

either on separate MeshRenderers or use the Render Objects feature on the Forward Renderer

The RenderObjects Forward Renderer feature can be used to re-render 

objects on a specific layer with an overrideMaterial 

(which is similar to replacement shaders, but the values of properties 

is not retained – Unless you use a material property block, 

but that also breaks batching with the SRP Batcher).

 You can also override stencil and ztest values on the feature. 

 See here for examples of the RenderObjects feature being used 

 (Toon “inverted hull” style outlines and object xray/occlusion effects).

You can also write Custom Forward Renderer features, 

for example a Blit feature like the one here (Blit.cs and BlitPass.cs), 

can be used to achieve custom post processing effects 

(as URP’s post-processing solution currently doesn’t include custom effects).

https://github.com/Unity-Technologies/UniversalRenderingExamples/tree/master/Assets/Scripts/Runtime/RenderPasses

Always use HLSLPROGRAM (or HLSLINCLUDE) and ENDHLSL, 

not the CG versions, otherwise there will be conflicts with the URP ShaderLibrary.

Instead of including UnityCG.cginc, use the URP ShaderLibrary. The main one to include is :

\#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

Your shaders need to have a UnityPerMaterial CBUFFER to be compatible with the SRP Batcher. 

(A UnityPerDraw is also required but the URP ShaderLibrary handles this).

 This buffer must be the same for every pass in the shader, 

 so it’s usually a good idea to put it in a HLSLINCLUDE in the Subshader. 

 It should include every exposed Property which will be used in the shader functions, 

 except textures which don’t have to be in there