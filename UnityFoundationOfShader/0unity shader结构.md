### 基本结构

```
Shader "Custom/BaseShader"
{
    Properties //属性,暴露在材质面板的可调参数,结构为:[变量名 ("显示名称",类型)=初始值],并需要在Pass中定义一个相同名字的变量
    {
        _Diffuse ("Diffuse", Color) = (1, 1, 1, 1)
		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
    }
    SubShader //对A显卡
	{
		Pass 
		{ 
            // 设置渲染状态
			Tags { "LightMode"="ForwardBase" }
		    //开始Cg代码片段
			CGPROGRAM
			//该片段代码编译指令
			#pragma vertex vert //告诉编译器顶点着色器函数名为vert(可改)
			#pragma fragment frag//告诉编译器片元着色器函数名为frag(可改)
			
			#include "Lighting.cginc"//内置文件,包含了unity内置提供的结构体,函数,方便快速计算((参考:UnityShader入门精要5.3)
			//Cg代码
			fixed4 _Diffuse;//变量,对应外部的属性
			fixed4 _Specular;
			float _Gloss;			
			
			//v:POSITION以及:SV_POSITION 该语法:"a:B",统一理解为a关联引用B,后面大写的参数是shader的固定语义(参考:UnityShader入门精要5.4)
            // v:POSITION表示使用v时,指向其引用的POSITION(模型空间的顶点坐标),获取其值进行计算
            // :SV_POSITION表示vert函数返回的float4,最终赋值给其引用的SV_POSITION(模型顶点的裁剪空间的坐标)
			float4 vert(float4 v:POSITION):SV_POSITION {				
				return UnityObjectToClipPos(v);//模型顶点坐标从模型空间转换到裁剪空间(取代以前的mul(UNITY_MATRIX_MVP,*)')
			}
			
            //: SV_Target 告诉渲染器 输出颜色存储到一个渲染目标中(render target)
			fixed4 frag() : SV_Target {			
				return fixed4(1.0,1.0,1.0, 1.0);//返回纯白(RGBA都为1)
			}
			
			ENDCG
            //其他设置
		}
	} 
    //SubShader// 对B显卡
   // {

    //}
    FallBack "Diffuse"//上述subshader失败后回调的unityShader
}

```

### 结构体

类似C#的结构,可以利用其对函数传入多个参数

```
 Pass {
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag
            
            uniform fixed4 _Color;

            //结构体a2v,v2f,旨在给函数输入或者返回一组参数值
            //使用时直接根据最终引用的值计算(如作为vert函数输入值的 'a2v v'),赋值时同样也是修改最后引用的位置的值(如作为vert函数)
			struct a2v {
                float4 vertex : POSITION;//:POSITION 告诉unity该参数vertex与模型空间的顶点坐标关联
				float3 normal : NORMAL;//:NORMAL 告诉unity参数normal与模型空间的顶点法线关联
				float4 texcoord : TEXCOORD0;
            };
            
            struct v2f {
                float4 pos : SV_POSITION;//:NORMAL 告诉unity参数normal与模型空间的顶点法线关联
                fixed3 color : COLOR0;
            };
            
            v2f vert(a2v v) {
            	v2f o;
            	o.pos = UnityObjectToClipPos(v.vertex);//模型空间转裁剪空间
            	o.color = v.normal * 0.5 + fixed3(0.5, 0.5, 0.5);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
            	fixed3 c = i.color;
            	c *= _Color.rgb;
                return fixed4(c, 1.0);
            }

            ENDCG
        }
```

