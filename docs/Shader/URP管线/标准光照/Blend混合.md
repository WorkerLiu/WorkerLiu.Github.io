```js
Shader "URPExample/Blend混合"
{
    Properties
    {
        _BaseColor("Color",Color) = (1,1,1,1)
        _BaseMap("Texture",2D) = "white"{}
        _Cutoff("AlphaCutout",float)= 0.5
    }
    SubShader
    {
        Tags
        { 
            "RenderPipeline"="UniversalRenderPipeline"
            "LightMode"="UniversalForward"
            "Queue" = "Transparent"         // 排序
            "RenderType" = "Transparent"    // 类型
        }
        Blend SrcAlpha OneMinusSrcAlpha     // alpha混合
        
        HLSLINCLUDE
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"        // 类似UnityCG.iginc
        
        TEXTURE2D (_BaseMap); SAMPLER(sampler_BaseMap);

        CBUFFER_START(UnityPerMaterial)
        float4 _BaseMap_ST;
        half4 _BaseColor;
        float _Cutoff;
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
                float2 texcoord : TEXCOORD;
            };
            struct v2f
            {
                float4 positionCS : SV_POSITION;
                float2 texcoord : TEXCOORD;
            };

            v2f vert(a2v v)
            {
                v2f o;
                o.positionCS=TransformObjectToHClip(v.positionOS.xyz);  // To裁剪空间
                o.texcoord=TRANSFORM_TEX(v.texcoord,_BaseMap);          // UV计算
                return o;
            }

            half4 frag(v2f i):SV_Target
            {
                // 固有色
                half4 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, i.texcoord) *_BaseColor;
                return albedo;
            }
            ENDHLSL
        }
    }
}
```

## 常用Blend混合模式

```js
Blend SrcAlpha OneMinusSrcAlpha //Alphablending alpha混合
Blend One One                   //Additive 相加混合
Blend One OneMinusDstColor      //Soft Additive柔和相加混合
Blend DstColor Zero             //Multiplicative 相乘混合
BlendDstColor SrcColor          //2x Multiplicative 2倍相乘混合
```

