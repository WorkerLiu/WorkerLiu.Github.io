```js
Shader "URPExample/光照贴图fog"
{
    Properties
    {
        _BaseColor("Color",Color) = (1,1,1,1)
        _BaseMap("Texture",2D) = "white"{}
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
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"        // 类似UnityCG.iginc
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"    // 光照hlsl

        #pragma multi_compile _ _ADDITIONAL_LIGHTS_VERTEX _ADDITIONAL_LIGHTS  // 附加灯光宏
        #pragma multi_compile _ _MAIN_LIGHT_SHADOWS         // 主光源阴影宏
        #pragma multi_compile _ _MAIN_LIGHT_SHADOWS_CASCADE // 主光源阴影宏
        #pragma multi_compile _ _SHADOWS_SOFT           // 软阴影宏
        #pragma multi_compile LIGHTMAP_OFF LIGHTMAP_ON  // 光照贴图
        #pragma multi_compile_fog   // fog灯光雾

        TEXTURE2D(_BaseMap); SAMPLER(sampler_BaseMap);

        CBUFFER_START(UnityPerMaterial)
        float4 _BaseMap_ST;
        half4 _BaseColor;
        float _Cutoff;
        CBUFFER_END
        ENDHLSL
        
        pass
        {
            Tags
            {
                "LightMode"="UniversalForward"
                "RenderType"="TransparentCutout"
                "Queue"="AlphaTest"
            }
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            struct a2v
            {
                float4 positionOS : POSITION;
                float3 normalOS : NORMAL;
                float2 texcoord : TEXCOORD;
                float2 lightmapUV   : TEXCOORD1;
            };
            struct v2f
            {
                float4 positionCS : SV_POSITION;
                float3 positionWS : TEXCOORD0;
                float3 normalWS : TEXCOORD1;
                // 光照贴图
                #ifdef LIGHTMAP_ON
                    DECLARE_LIGHTMAP_OR_SH(lightmapUV, vertexSH, 2);
                #endif
                
                float fogCoord : TEXCOORD3; // 灯光雾
                float2 texcoord : TEXCOORD4;

            };

            v2f vert(a2v v)
            {
                v2f o;
                o.positionCS = TransformObjectToHClip(v.positionOS.xyz);    // To裁剪空间
                o.positionWS = TransformObjectToWorld(v.positionOS.xyz);    // To世界空间
                o.texcoord = TRANSFORM_TEX(v.texcoord,_BaseMap);            // UV计算
                o.normalWS =TransformObjectToWorldNormal(v.normalOS.xyz);   // To世界空间
                // 光照贴图
                #ifdef LIGHTMAP_ON
                    OUTPUT_LIGHTMAP_UV(v.lightmapUV, unity_LightmapST, o.lightmapUV);
                #endif
                // fog灯光雾
                o.fogCoord = ComputeFogFactor(o.positionCS.z);
                return o;
            }

            half4 frag(v2f i):SV_TARGET
            {
                Light light = GetMainLight(TransformWorldToShadowCoord(i.positionWS));  // 获取主光源、阴影
                float3 lightDir = normalize(light.direction);       // 光源向量
                float3 normalWS = normalize(i.normalWS);            // 归一化
                half3 ambient = SampleSH(normalWS);                 // 球谐光照 计算环境光

                // 固有色
                half4 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, i.texcoord) * _BaseColor;

                // 漫反射
                half3 diffuse = (light.color * _BaseColor.rgb) * max(0,dot(normalWS, lightDir) * 0.5 + 0.5);

                // 辅助照明
                half4 lightColor ;
                int lightList = GetAdditionalLightsCount();
                for(int f = 0 ; f < lightList; f++)
                {
                    Light addlight = GetAdditionalLight(f,i.positionWS);  
                    float3 lightDir = normalize(addlight.direction);
                    float lightAttenuation = addlight.distanceAttenuation * addlight.shadowAttenuation;
                    lightColor += half4(addlight.color,1) * (dot(normalWS, lightDir) * 0.5 + 0.5) * lightAttenuation * albedo;
                }

                // 光照贴图混合计算
                #ifdef LIGHTMAP_ON
                    half3 bakedGI = 1;
                    bakedGI = SAMPLE_GI(i.lightmapUV,i.vertexSH, normalWS);
                    albedo.rgb *= light.shadowAttenuation;
                    albedo.rgb *= diffuse + bakedGI;
                #elif LIGHTMAP_OFF
                    float lightAttenuation = light.distanceAttenuation * light.shadowAttenuation;
                    half3 radiance = (diffuse * lightAttenuation ) + ambient;
                    albedo.rgb *= radiance + lightColor.rgb;
                #endif

                // fog灯光雾
                albedo.rgb = MixFog(albedo.rgb,i.fogCoord);
                
                return float4(albedo.rgb,1);
            }
            ENDHLSL
        }
        
        //阴影Pass
        pass
        {
            Tags{"LightMode"="ShadowCaster"}
            HLSLPROGRAM
            #pragma vertex vertshadow
            #pragma fragment fragshadow

            struct a2v
            {
                float4 positionOS : POSITION;
                float3 normalOS : NORMAL;
                float2 texcoord : TEXCOORD;
            };
            struct v2f
            {
                float4 positionCS : SV_POSITION;
                float3 positionWS : TEXCOORD0;
                float3 normalWS : TEXCOORD1;
            };
            
            v2f vertshadow(a2v v)
            {
                v2f o;
                Light MainLight = GetMainLight();
                o.positionWS = TransformObjectToWorld(v.positionOS.xyz);
                o.normalWS = TransformObjectToWorldNormal(v.normalOS.xyz);
                o.positionCS = TransformWorldToHClip(ApplyShadowBias(o.positionWS,o.normalWS,MainLight.direction));
                #if UNITY_REVERSED_Z
                    o.positionCS.z = min(o.positionCS.z,o.positionCS.w * UNITY_NEAR_CLIP_VALUE);
                #else
                    o.positionCS.z = max(o.positionCS.z,o.positionCS.w * UNITY_NEAR_CLIP_VALUE);
                #endif
                return o;
            }

            half4 fragshadow(v2f i):SV_Target
            {
                return 0;
            }
            ENDHLSL
        }
    }
}
```

