## 完整代码

```js
Shader "Example01/01Shader"
{
    Properties
    {
        
    }
    SubShader
    {
        Pass
        {		
            CGPROGRAM
            //定义:顶点着色器
            #pragma vertex vert
            //定义:片源做色器
            #pragma fragment frag
            //顶点着色器
            fixed4 vert (fixed4 v :POSITION) :SV_POSITION
            {
                return UnityObjectToClipPos(v);
            }
            //片源着色器
            fixed4 frag () :SV_Target
            {
                return fixed4(1,1,1,1);
            }
            ENDCG
        }
    }
}
```

复制上述代码，新建shader文件并且建立材质球指定模型后。会得到一个纯白色的模型：

![](img\示意图.png)

## 结构分析

![](img\Shader结构.png)
上述示意图为Shader最基础的机构，可以粗暴的理解为：

-  Properties 界面属性面板
-  SubShader 界面属性面板
-  Pass 图层

```CGPROGRAM --> ENDCG```区域便是为CG语言。默认固定格式
结构图标注的 ```#pragma vertex vert```和``` #pragma fragment frag ```是Shader中类似编程语言的**函数声明**
对应了后续的``` fixed4 vert () ```和 ``` fixed4 frag () ```及**顶点着色器** & **片元着色器**。

## 语义

```js
    //顶点着色器
    fixed4 vert (fixed4 v:POSITION):SV_POSITION
    {
        return UnityObjectToClipPos(v);
    }
    //片源着色器
    fixed4 frag () : SV_Target
    {
        return fixed4(1,1,1,1);
    }
```

Shader语法特定的语法糖例：```fixed4 v:POSITION``` 这个**:POSITION**叫做语意。是告诉编译器这个属性时什么，可以粗略的理解为C#中的声明类型。

```js
	:POSITION       // 输入模型顶点信息
    :SV_POSITION    // 顶点输出裁剪空间顶点信息
    :SV_Target      // 输出到默认的帧缓存
```

语义类型有很多，但都有固定格式套路写法。

> **笔者拙见**：日常标准的Shader写法都大相径庭，初学者``不刨根问底``这样不会入门及劝退。等学习到一定阶段时。针对性的为实现某些效果时，再利用各大搜索引擎深究其原理。