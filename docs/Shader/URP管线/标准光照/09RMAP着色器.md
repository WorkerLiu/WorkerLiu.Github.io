##  RMAP渐变纹理

```js
Shader "URPExample/RMAP着色器"
{
    Properties
    {
        _BaseColor("Color",Color) = (1,1,1,1)
        _BaseMap("Texture",2D) = "white"{}
        _RmapMap("Rmap",2D) = "white"{}
    }
    SubShader
    {
        Tags
        {
            "RenderPipeline"="UniversalRenderPipeline"
            "LightMode"="UniversalForward"
            "RenderType"="Opaque"
        }

        HLSLINCLUDE
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"        //类似UnityCG.iginc
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"    //光照hlsl

        TEXTURE2D(_BaseMap);
        SAMPLER(sampler_BaseMap);
        TEXTURE2D(_RmapMap);
        SAMPLER(sampler_RmapMap);

        CBUFFER_START(UnityPerMaterial)
        float4 _BaseMap_ST;
        half4 _BaseColor;
        CBUFFER_END
        ENDHLSL

        pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            struct a2v
            {
                float4 positionOS : POSITION;
                float3 normalOS : NORMAL;
                float2 texcoord : TEXCOORD;
            };

            struct v2f
            {
                float4 positionCS : SV_POSITION;
                float3 normalWS : TEXCOORD0;
                float2 texcoord : TEXCOORD1;
            };

            v2f vert(a2v v)
            {
                v2f o;
                o.positionCS = TransformObjectToHClip(v.positionOS.xyz);    //To裁剪空间
                o.normalWS = TransformObjectToWorldNormal(v.normalOS);      //To世界空间
                o.texcoord = TRANSFORM_TEX(v.texcoord, _BaseMap);           //UV计算
                return o;
            }

            half4 Screen(half4 a, half4 b)
            {
                half4 col = a;
                b.rgb = b.rgb * b.a;
                col.rgb = 1 - (1 - a.rgb) * (1 - b.rgb);
                col.a = a.a + b.a;
                return col;
            }

            half4 frag(v2f i):SV_Target
            {
                Light light = GetMainLight();                   //获取主灯光
                float3 lightDir = normalize(light.direction);   //光源向量
                float3 normalWS = normalize(i.normalWS);        //归一化

                //半兰伯特
                float halfLambert = max(0, dot(lightDir, normalWS) * 0.5 + 0.5);

                //Rmap
                half4 rmap = SAMPLE_TEXTURE2D(_RmapMap, sampler_RmapMap, float2(halfLambert,0.5));
                rmap = Screen(rmap, _BaseColor);    //采用滤色叠加

                //固有色
                half4 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, i.texcoord);

                //混合计算
                albedo *= rmap;
                return float4(albedo.rgb, 1);
            }
            ENDHLSL
        }
    }
}
```

