## BuitinShader代码

```js
Shader "Unlit/BuitinShader"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed4 col = tex2D(_MainTex, i.uv);
                return col;
            }
            ENDCG
        }
    }
}
```

Unity默认代码模板，为了清晰的对比代码结构删除了fog灯光雾相关代码。

## URPShader代码

```js
Shader "Unlit/URPShader"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags 
        {
            "RenderPipeline"="UniversalRenderPipeline"  //添加URP管线
            "RenderType"="Opaque"
        }
        LOD 100

        Pass
        {
            //CGPROGRAM 替换
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            //#include "UnityCG.cginc"  替换
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl" 

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };

            //sampler2D _MainTex;  替换
            TEXTURE2D (_MainTex); SAMPLER(sampler_MainTex);

            CBUFFER_START(UnityPerMaterial) //URP宏包裹变量
            float4 _MainTex_ST;
            CBUFFER_END

            v2f vert (appdata v)
            {
                v2f o;
                //o.vertex = UnityObjectToClipPos(v.vertex)  替换;
                o.vertex = TransformObjectToHClip(v.vertex.xyz);  //精确到向量位数 不然会警告！
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                return o;
            }

            half4 frag (v2f i) : SV_Target
            {
                //fixed4 col = tex2D(_MainTex, i.uv);  替换
                half4 col = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv);
                return col;
            }
            //ENDCG
            ENDHLSL
        }
    }
}
```

根据URPShader框架升级后的Unity默认代码模板。

## URP代码变更

1. CGPROGRAM <--> ENDCG 变更为 HLSLPROGRAM <--> ENDHLSL。
2. 添加``"RenderPipeline"="UniversalRenderPipeline"``tag标签
3. URP渲染管线不再支持``fixed``值，采用``half``。
4. 引用文件``UnityCG.cginc`` 变跟为引用URP库封装的hlsl文件。
5. 贴图采样、转换等API变跟。
6. 变量包裹在 CBUFFER_START(UnityPerMaterial) <--> CBUFFER_END内

