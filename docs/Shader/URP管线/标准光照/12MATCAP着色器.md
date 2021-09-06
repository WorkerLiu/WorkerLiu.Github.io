```js
Shader "URPExample/MATCAP着色器"
{
    Properties
    {
        _BaseColor("Color",Color) = (1,1,1,1)
        _BaseMap("Texture",2D) = "white"{}
        _MatCap("MatCap",2D) ="white" {}
        _Value("Value",Range(0,1)) = 0.5
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
        
        TEXTURE2D(_BaseMap);    SAMPLER(sampler_BaseMap);
        TEXTURE2D(_MatCap);    SAMPLER(sampler_MatCap);

        CBUFFER_START(UnityPerMaterial)
        float4 _BaseMap_ST;
        half4 _BaseColor;
        float _Value;
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
                float2 NtoV : TEXCOORD2;
            };
            
            v2f vert(a2v v)
            {
                v2f o;
                o.positionCS = TransformObjectToHClip(v.positionOS.xyz);    //To裁剪空间
                o.normalWS =TransformObjectToWorldNormal(v.normalOS.xyz);   //To世界空间

                o.NtoV.x = mul(UNITY_MATRIX_IT_MV[0].xyz, o.normalWS);  //To视角矩阵
                o.NtoV.y = mul(UNITY_MATRIX_IT_MV[1].xyz, o.normalWS);  //To视角矩阵
                o.texcoord = TRANSFORM_TEX(v.texcoord,_BaseMap);        //UV计算
                return o;
            }

            half4 frag(v2f i):SV_Target
            {
                
                //固有色
                half4 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, i.texcoord) * _BaseColor;

                //MatCap
                half3 matcap = SAMPLE_TEXTURE2D(_MatCap , sampler_BaseMap, i.NtoV * 0.5 + 0.5).xyz;

                //混合运算
                albedo.rgb += matcap *_Value;
                return float4(albedo.rgb,1);
            }
            ENDHLSL
        }
    }
}
```

