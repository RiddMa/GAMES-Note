<span style='color:red;'>
本文部分文字及截图整理自 GAMES104 课程，仅作学习分享，版权归 GAMES104 所有。https://www.bilibili.com/video/BV1oU4y1R7Km
</span>

# 游戏引擎导论

# 引擎架构分层

# 如何构建游戏世界

# 游戏引擎中的渲染实践

## 可见性剪裁 Visibility Culling

- GPU Culling

## 纹理压缩 Texture Compression

- 块压缩 Block Compression：
	- 桌面平台：BC7（Modern）、DXT（Legacy）
	- 移动平台：ASTC

## 建模工具 Modeling Tools

- 多边形建模 PolyModeling：3ds Max、MAYA、Blender
- 雕刻 Sculpting：ZBrush
- 扫描 Scanning
- 程序化建模 Procedural Modeling：Houdini、Unreal Engine

## 新型渲染管线

### Cluster-Based Mesh Pipeline

- GPU-Driven Rendering Pipeline（2015，Ubisoft）
	- Mesh Cluster Rendering
		- 在单次 drawcall 中支持任意数量的 meshes
		- 使用 cluster bounds 进行 GPU-Culling（虚幻引擎，Nanite）
		- Cluster depth sorting
- Geometry Rendering Pipeline Architecture（2021）

### Programable Mesh Pipeline

Task shader 和 mesh shader，在 GPU 中生成新的几何面

# 游戏引擎中的光照、材质、着色器渲染

## 渲染方程及挑战

### Kajiya 的 BRDF 渲染方程

radiance——辐射光强，irradiance——入射光强

$$
L_o(x,\omega_o) = L_e(x,\omega_o) + \int_{H^2} f_r(x,\omega_o,\omega_i) L_i(x,\omega_i) \cos \theta_i d \omega_i
$$

$L_o$：出射光强，$x$：渲染物体表面一点，$\omega_o$：出射角

$L_e$：辐射光强

$\int_{H^2}$：对上半球面积分，$f_r(x,\omega_o,\omega_i)$：散布函数

$L_i$：入射光强，$\omega_i$：入射角

$\theta_i$：入射角与法向夹角

## 简单的渲染模型

> ambient light + simple light = result
>
> 环境贴图反射 Environment Map Reflection

- Main Light：主光
- Ambient Light：模拟球面分布的低频入射环境光
- Environment Map：模拟球面分布的高频入射环境光

## Blinn-Phong 材质

Diffuse + Ambient + Specular = Blinn-Phong Reflection

### 问题

- 能量不保守（Not Energy Conservative），无法用于光线追踪
- 真实世界的复杂材质，难以用经验表达

## 阴影图 Shadow Map

### 原理

1. 从光源处渲染一张投影点到光源距离的深度图 $c$
2. 从摄像机处渲染，对摄像机视线中的每一点 $p_z$，向光源投影得到投影点 $(p_x,p_y)$
3. 比较 $p_z$到光源距离与深度图中 $c(p_x,p_y)$记录的到光源距离
4. 若 $p_z$离光源更远，则其处在阴影中

### 缺陷

- 阴影分辨率受限于材质
- 深度精度受限于材质

通常的 workaround：加 bias

导致结果：游戏中人物阴影与脚底不重合

## 预计算全局光照 Pre-computed Global Illumination

目标：在多项式时间内计算材质 BRDF 积分

解决手段：傅里叶变换 Fourier Transform，获得场景低频光照信息

球面调和函数 Spherical Harmonics：在球面上定义的傅里叶变换模拟，用于探针（Probe）

### 球面调和函数光照贴图 SH Lightmap

使用低面数代理几何体（low-poly proxy geometry），进行 UV 参数化展开

优点：
- 运行时高效
- 烘焙精细的环境漫反射（diffuse）细节

缺点：
- 计算时间长
- 只支持静态场景和光照
- 空间开销大

### 探针 Probe

- 光照探针 Light Probe

	难点：Light Probe 摆放位置，如何自动生成等

- 反射探针 Reflection Probe

	难点：需要高频信息，分辨率高，性能开销大

优点：
- 运行时高效
- 静态与动态物体都适用
- diffuse 和 specular 都能处理

缺点：
- 大量的 SH 光照探针，需要预计算
- 采样稀疏，无法处理 GI 的精细细节，如软阴影和重叠几何结构

## 基于物理的材质 Physical-Based Material

### 微表面理论 Microfacet Theory

核心思想：物体的光滑度由微表面上法向的聚集度决定

### 基于微表面的 BRDF 模型 BRDF Model Based on Microfacet

$$
\begin{aligned}
L_o(x,\omega_o) = L_e(x,\omega_o) + \int_{H^2} f_r(x,\omega_o,\omega_i) L_i(x,\omega_i) \cos \theta_i d \omega_i
\end{aligned}
$$
其中：
$$
\begin{aligned}
f_r = k_d f_{Lambert} + f_{CookTorrance}
\end{aligned}
$$
其中：
$$
\begin{aligned}
f_{Lambert} = \frac{c}{\pi} \qquad \text{漫反射 diffuse 部分}
\end{aligned}
$$
$$
f_{CookTorrance} = \frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)} \qquad \text{高光 specular 部分}
$$
其中 D 为法向分布：
$$
NDF_{GGX}(n,h,\alpha)=\frac{\alpha^2}{\pi ((n \cdot h)^2(\alpha^2-1)+1)^2}, \quad \alpha \text{为材质的 roughness}
$$
G 为几何自遮挡：
$$
G_{Smith}(l,v)=G_{GGX}(l) \cdot G_{GGX}(v)
$$
$$
G_{GGX}(v)=\frac{n \cdot v}{(n \cdot v)(1 - k) + k}, \quad k=\frac{(\alpha + 1)^2}{8}, \quad \alpha \text{为材质的 roughness}
$$
F 为菲涅尔方程：
$$
F_{Schlick}(h,v,F_0)=F_0+(1-F_0)(1-(v \cdot h))^5, \quad F_0\text{为菲涅尔系数}
$$

### 常用的 BRDF 模型

- PBR Specular Glossiness 模型（SG）：灵活强大，容易出错
- PBR Metallic Roughness 模型（MR）：限制了 SG 模型可调参数范围，不易出错

## 基于图像的光照 Image-Based Lighting

Diffuse Irradiance Map + Specular Approximation（略）

详见 GAMES202

## 经典阴影解决方案 Classic Shadow Solution

级联阴影 Cascade Shadow：从摄像机视锥出发，根据距离采用不同采样精度的阴影图

优点：
- 阴影计算不易出错，从视角出发进行反走样
- 深度图生成快
- 效果较好

缺点：
- 高质量区域阴影几乎不可能实现
- 阴影不支持颜色，不支持半透明物体

### Percentage Closer Filter，PCF

### Percentage Closer Soft Shadow，PCSS

### Variance Soft Shadow Map，VSSM

基于切比雪夫不等式

## AAA Rendering 总结

Lightmap + Lightprobe

PBR + IBL

Cascade shadow + VSSM

⬆️ 5 年前的 3A 效果，可找工作（不是）

## 前沿高质量渲染技术

- 实时光线追踪 Real-Time Ray-Tracing
- 实时全局光照 Real-Time Global Illumination
	- Screen-space GI
	- SDF Based GI
	- Voxel-Based GI（SVOGI / VXGI）
	- RSM / RTX GI
- 更复杂材质模型
	- BSDF Hair（Strand-Based Hair）
	- BSSRDF
- 虚拟阴影贴图 Virtual Shadow Maps
- Uber Shader and Variants
- 跨平台着色器编译 Cross Platform Shader Compile

# 游戏中地形、大气、云的渲染

## 地形的几何

- 高度图 Height Map、等高线图 Contour Map
- Adaptive Mesh Tessellation（自适应望各细分）

优化的两个方向：
- 根据到摄像机的距离和 FoV 优化
- 视角空间的几何简化（与 ground truth 对比允许误差在一定范围内）（预计算）

### 实践中的算法

- Triangle-Based Subdivision（基于二叉树）
	- 适用于大规模地形渲染
	- 注意 Subdivision 中出现的 T-Junction 情况
- QuadTree-Based Subdivision（基于四叉树），实践中最常用
	- 四边形网格的 T-Junction 中，使用 Mesh LoD Stitching 方法，将 T-Junction 的顶点移动到相邻顶点消除 T-Junction
- Triangulated Irregular Network（TIN）

### 基于 GPU 的曲面细分 GPU-Based Tessellation

DX11 三部分：Hull Shader、Tessellator、Domain Shader

Mesh -> Triangle Patch Mesh -> Hull Shader -> Tessellator ->Tessellated Mesh -> Domain Shader -> Displaced Mesh

DX12 支持：Amplification Shader、Mesh Shader，Mesh Shader 代替上面 Hull Shader、Tessellator、Domain Shader

GPU-Based Tessellation 功能可以支持实时可变形地形 Real-Time Deformable Terrain（雪地、泥地效果）

全动态地形：体素化表达 Voxel-Based Representation
- **Marching Cube 算法**

## 地形的材质

地形材质 Terrain Materials
- Base Color + Normal + Roughness + Height

材质混合 Texture Splatting
- 使用高度图 + 透明度混合 Blending with Height

```HLSL
float3 blend(float4 texture1, float height1, float4 texture2, float height2){
	return height1 > height2 ? texture1.rgb : texture2.rgb;
}
```

- 带 Bias 的高度图 + 透明度混合

```HLSL
float3 blend(float4 texture1, float height1, float4 texture2, float height2){
	float bias = 0.2;
	float ma = max(texture1.a + height1, texture2.a + height2) - bias;
	float b1 = max(texture1.a + height1 - ma, 0);
	float b2 = max(texture2.a + height2 - ma, 0);
	return (texture1.rgb * b1 + texture2.rgb * b2)/(b1 + b2);
}
```

### Material Texture Array

作用：将大量不同地表材质集中存储在数组中

### 视差与凹凸贴图 Parallax and Displacement Mapping

视差贴图 Parallax Mapping：
- 从摄像机角度微调渲染的像素，达到立体凹凸效果
- 物体的边缘仍然是光滑无立体凹凸效果的

凹凸贴图 Displacement Mapping：
- 借助 GPU Tessellation，直接根据凹凸贴图改变网格 Mesh

### 虚拟纹理 Virtual Texture，VT

为了解决渲染每个像素都需要对非常多纹理贴图进行采样的问题而诞生，已经成为现代游戏引擎事实标准

- 构建一个虚拟的索引的纹理，存储整个场景的混合采样好的材质，并生成多级 mipmap
- 根据摄像机视角，加载需要的对应 LoD 的纹理块
- 将纹理块拼接称为物理纹理，存到显存

虚拟纹理实现 VT Implementation 的前沿技术：DirectStorage & DMA

### 浮点数精度误差解决

IEEE 754 Float 32bit 在实际游戏中大约只支持 2~3km 半径的坐标范围，否则产生严重抖动

解决方法：相对于相机的渲染 Camera-Relative Rendering

## 植被道路贴花等

- 树木渲染 Tree Rendering：Speed Tree 中间件
- 装饰物渲染 Decorator Rendering：面片 +LoDs
- 道路渲染 Road Rendering：Spline 样条线，根据 Spline 雕刻 Height Field，烘焙进 Virtual Texture
- 贴花渲染 Decal Rendering：烘焙进 Virtual Texture

## 大气散射理论

略

### 大气外观的经验建模 Analytic Atmosphere Appearance Modeling

公式略

优点：计算简单高效

缺点：
- 只适用于地面视角
- 大气参数无法自由调整，表现力不足

### 辐射转移方程 Radiative Transfer Equation（RTE）

光与大气中的参与介质交互分为四部分：

吸收 Absorption：
$$
- \sigma_a L(x,\omega) \quad ,\sigma_a \text{为吸收系数}
$$
外散射 Out-scattering：
$$
- \sigma_s L(x,\omega) \quad ,\sigma_s \text{为散射系数}
$$
自发光 Emission：（不常用）
$$
\sigma_a L_e(x,\omega)
$$
内散射 In-scattering：
$$
\sigma_s \int_{S^2} f_p(x,\omega, \omega')L(x,\omega') d\omega' \quad ,f_p \text{为相位函数}
$$
定义“消光系数”为：
$$
\sigma_t(x) = \sigma_a(x)+\sigma_s(x)
$$
于是得到辐射转移方程：
$$
dL(x,\omega)/dx = -\sigma_t L(x,\omega) + \sigma_a L_e(x,\omega) + \sigma_s \int_{S^2} f_p(x,\omega, \omega')L(x,\omega') d\omega'
$$

### 体积渲染方程 Volume Rendering Equation（VRE）

对于不透明表面上的一点 $M$，光线以方向角 $\omega$经过体积传播至摄像机：
$$
L(P,\omega) = \int_{x=0}^d T(x)[\sigma_a L_e(x,\omega) + \sigma_s L_i(x,\omega) + T(M)L(M,\omega)]
$$
其中：
$$
T(x)=e^{-\int_x^P \sigma_t(s)ds} \quad ,\text{Transmittance(通透度)：吸收和外散射导致的光线净减少因子}
$$
$$
L_i(x,\omega)= \int_{S^2} f_p(x,\omega, \omega')L(x,\omega') d\omega' \quad ,\text{内散射的净增加因子}
$$

### 散射类型 Scattering Types

瑞利散射 Rayleigh Scattering
- 光线向前与向后散射对称
- 短波长更易被散射

散射系数：
$$
\sigma_s^{Rayleigh}(\lambda,h)=\frac{8\pi^3(n^2-1)^2}{3} \frac{\rho(h)}{N} \frac{1}{\lambda^4} \quad ,N\text{为密度},\lambda\text{为波长}
$$
相位函数：
$$
F_{Rayleigh}(\theta)=\frac{3}{16\pi} (1+cos^2\theta)
$$
相乘化简，得到瑞利散射方程 Reyleigh Scattering Equation：
$$
S(\lambda,\theta,h)=\frac{\pi^2(n^2-1)^2}{2} \frac{\rho(h)}{N} \frac{1}{\lambda^4} (1+cos^2\theta)
$$

米氏散射 Mie Scattering
- 光线向前散射更多，向后散射更少
- 各种波长的光散射程度几乎相同

拟合方程，公式略 : (

![Pasted image 20220616150947](attachments/Pasted%20image%2020220616150947.png)

g>0 时为米氏散射，g<0 时光线更多向后散射，g=0 为瑞利散射

## 实时大气渲染

### Ray Marching 算法

思想：通过多次 Single Scattering 积分，计算空间中给定点的最终辐射度（radiance）

$$
L_{sun} \int_A^B S(\lambda,\theta,h) \cdot (T(\text{sun} \rightarrow P) + T(\text{sun} \rightarrow A))\text{d} s
$$

#### Precomputed Atmospheric Scattering

空间换时间的优化：

- Transmittance LUT，预计算通透度

$$
T(X_v \rightarrow X_m)=\frac{T(X_v \rightarrow B)}{T(X_m \rightarrow B)}
$$

![Pasted image 20220616204323](attachments/Pasted%20image%2020220616204323.png)

- Single Scattering LUT，预计算单次散射结果
	- 使用 3D Texture 存储 4D Table

![Pasted image 20220616204359](attachments/Pasted%20image%2020220616204359.png)

- 使用 Transmittance LUT 和 Single Scattering LUT，多次计算得到 Multiple Scattering LUT

![Pasted image 20220616204604](attachments/Pasted%20image%2020220616204604.png)

缺陷：
- 预计算开销大
- 动态天气环境变换困难
- 运行时开销依然不小（每个像素都对 4 维表进行插值）

前沿方法：A Scalable and Production Ready Sky and Atmosphere Rendering Technique, Epic Games

## 云的绘制

### 基于网格的云 Mesh-Based Cloud

没人这么做

### 面片云 Billboard Cloud

十几年前的游戏用这个

优点：
- 高效

缺点：
- 视觉效果很一般
- 云的类型受限制

### 体积云建模 Volumetric Cloud Modeling

3A 游戏常用的解决方案

优点：
- 云很真实
- 支持大面积的云层
- 支持动态天气
- 支持体积光照和体积阴影

缺点：
- 算法复杂
- 渲染开销昂贵

天气纹理 Weather Texture

噪声函数 Noise Functions
- 柏林噪声 Perlin Noise
- 细胞噪声 Worley（Voronoi）Noise

云密度模型 Cloud Density Model
- 思想：用一张纹理图 Texture 限定云的范围和云的高度，用各种频率的噪声作为遮罩，得到云的形态

使用 Ray Marching 渲染云 Rendering Cloud by Ray Marching
- 由于云的通透度很低，可以作大量假设，简化计算

# 游戏引擎中的渲染管线、后处理及其他

## 环境光遮蔽 Ambient Occlusion

### 预计算环境光遮蔽 Precomputed AO

使用光线追踪预计算 AO，把结果存储进纹理

- 优点：效果很好
- 缺点：存储开销增加、只适用静态物体

### 屏幕空间环境光遮蔽 Screen Space AO（SSAO）

从摄像机对每个像素周围的小球体范围内进行随机采样，与深度图该像素对应深度比较，计算该点被遮挡程度。（很少用，算法有错）

![Pasted image 20220616223930](attachments/Pasted%20image%2020220616223930.png)

### SSAO+

改进了 SSAO 算法错误，使用半球面采样

![Pasted image 20220616224000](attachments/Pasted%20image%2020220616224000.png)

### HBAO（Horizon-Based AO）

思想：基于半球面的 Ray Marching 方法

![Pasted image 20220616224244](attachments/Pasted%20image%2020220616224244.png)

![Pasted image 20220616224417](attachments/Pasted%20image%2020220616224417.png)

### GTAO（GroundTruth-based AO）

用数值计算拟合，在多项式时间内计算 AO 最终效果

优点：
- 效果非常好，接近 GT
- AO 阴影支持颜色

![Pasted image 20220616224639](attachments/Pasted%20image%2020220616224639.png)

![Pasted image 20220616225122](attachments/Pasted%20image%2020220616225122.png)

### RTAO（Ray-Tracing AO）

- 使用光线追踪，1~4spp 获得 AO 效果
- 通过时序采样优化效果

## 雾效

### 深度雾 Depth Fog

根据屏幕空间深度添加雾效

分为 Linear、Exp、Exp Squared 三种

### 高度雾 Height Fog

根据浓度进行积分

![Pasted image 20220616225746](attachments/Pasted%20image%2020220616225746.png)

### 体积雾 Voxel-Based Volumetric Fog

基于摄像机视锥的体素化

![Pasted image 20220616230134](attachments/Pasted%20image%2020220616230134.png)

## 反走样 Anti-Aliasing

![Pasted image 20220616231032](attachments/Pasted%20image%2020220616231032.png)

### SSAA（Super-Sample AA）和 MSAA（Multi-Sample AA）

SSAA：直接放大多倍画面，力大砖飞

- 4 倍渲染分辨率，4 倍 z-buffer、framebuffer、rasterization、pixel shading

MSAA：只超采必要的（物体边缘）的像素

- 4 倍渲染分辨率，4 倍 z-buffer、framebuffer、rasterization，1 倍 pixel shading
- 硬件支持，速度很快，但空间占用大
- 存在的问题：现代游戏多边形密集，MSAA 完全退化为 SSAA

### FXAA（Fast Approximate Anti-Aliasing）

- 基于 1 倍分辨率
- 3x3 卷积，得到 Compute Offset Direction
- Edge Searching Algorithm，找到边的两个端点
- Calculate Blend Coefficient，计算混合系数
- Blend Nearby Pixels，混合相邻像素

![Pasted image 20220616231617](attachments/Pasted%20image%2020220616231617.png)

![Pasted image 20220616232234](attachments/Pasted%20image%2020220616232234.png)

![Pasted image 20220616232213](attachments/Pasted%20image%2020220616232213.png)

![Pasted image 20220616232004](attachments/Pasted%20image%2020220616232004.png)

![Pasted image 20220616232020](attachments/Pasted%20image%2020220616232020.png)

### TAA（Temporal Anti-Aliasing）

基于运动向量 Motion Vector，混合当前帧与历史帧

![Pasted image 20220616232449](attachments/Pasted%20image%2020220616232449.png)

## 后处理 Post-Processing

### Bloom

- RGB 转 Luminance，再检测高光区域
- 高斯模糊 Guassian Blur，对高光进行卷积运算
- 金字塔运算 Pyramid Guassian Blur

![Pasted image 20220616233056](attachments/Pasted%20image%2020220616233056.png)

![Pasted image 20220616233227](attachments/Pasted%20image%2020220616233227.png)

![Pasted image 20220616233301](attachments/Pasted%20image%2020220616233301.png)

### Tone Mapping

- 将 HDR 图像映射到 SDR 图像
- 使用一条曝光曲线 Exposure Curve 映射像素亮度
- 常用曲线：Filmic Curve、ACES（Academy Color Encoding System）

![Pasted image 20220616233747](attachments/Pasted%20image%2020220616233747.png)

### Color Grading

Look Up Table（LUT）

## 渲染管线 Rendering Pipeline

### 前向渲染 Forward Rendering

Shadow -> Shading -> Post-Process

### 延迟渲染 Deferred Rendering

延迟光照的处理

Rendering G-Buffer（Albedo、Specular、Normals、Depth）-> Deferred Shading

优点：
- 光照只在可见部分计算
- G-Buffer 可用作后处理

缺点：
- 需要大显存、高带宽
- 不支持透明物体
- 对 MSAA 不友好

### 基于图块渲染 Tile-Based Rendering

把画面拆成若干小块渲染，增加空间局部性

### Forward+（Tile-Based Forward）Rendering

采用

### Cluster-Based Rendering

把空间拆分渲染

### Visibility Buffer

虚幻 5 等引擎使用的前沿管线，基于现代游戏几何体远比材质复杂的事实出发

![Pasted image 20220617000251](attachments/Pasted%20image%2020220617000251.png)

## Frame Graph

使用有向无环图（Directed Acyclic Graph，DAG）表示渲染管线

## 渲染到显示器 Render to Monitor

垂直同步 V-Sync

可变刷新率 Variable Refresh Rate（VRR）

# 游戏引擎的动画系统 - 基础

## 2D 动画 2D Animation

### 精灵动画 Sprite Animation

播放序列帧实现 2D 效果

### 现代游戏中的 Sprite Animation

爆炸、火花等粒子特效
- Live2D

## 3D 动画 3D Animation

概念：自由度 Dof（Degrees of Freedom）

### 刚体层级动画 Rigid Hierarchical Animation

早期使用，可表现机械等动画效果

缺点：无法表现人物等软体的动作

### 顶点动画 Per-Vertex Animation

优点：
- 自由度最高，每个顶点 3DoFs
- 可以使用顶点动画纹理 Vertex Animation Texture（VAT）实现
- 适合表现布料、水流等复杂变形

缺点：
- 需要大量存储空间

### 变形目标动画 Morph Target Animation

使用变形目标 Morph Target+LERP 实现的顶点动画，比纯顶点动画更省空间

- 可以用在捏脸系统中

### 3D 蒙皮动画 3D Skinned Animation

也叫 Skeleton Animation

- 每个顶点根据权重受骨骼影响运动

### 2D 蒙皮动画 2D Skinned Animation

类似 Live2D

### 基于物理的动画 Physics-Based Animation

- 布娃娃 Ragdoll
- 布料与流体模拟 Cloth and Fluid Simulation
- 反向运动学 Inverse Kinematics（IK）

## 蒙皮动画的实现 Skinned Animation Implementation

1. 创建用于绑定的 Mesh
2. 为 Mesh 创建骨骼 Skeleton
3. 为 Mesh 每个顶点绘制权重 Skinning Weights
4. 将骨骼运动到需要的 Pose
5. 根据骨骼和权重，运动 Mesh 的各个顶点

### 不同的坐标空间 Different Spaces

- 世界坐标空间 World Space
- 模型坐标空间 Model Space：模型的原点相对于世界原点的坐标
- 局部坐标空间 Local Space：骨骼原点相对于模型原点的坐标

### 不同生物的骨骼 Skeleton for Creatures

- Humanoid Skeleton
- Non-Humanoid Skeleton

### 关节与骨头 Joint Vs. Bone

- Bone 是刚体
- 两个 Joint 定义一个 Bone
- 物理引擎中实际存储计算的是 Joint

### 三维旋转的数学基础 Math of 3D Rotation

二维旋转矩阵：

![Pasted image 20220618162158](attachments/Pasted%20image%2020220618162158.png)

欧拉角：将旋转拆分为三个轴上的旋转合成

![Pasted image 20220618163222](attachments/Pasted%20image%2020220618163222.png)

欧拉角的缺陷
- （矩阵乘法）不支持交换律，因此旋转的结果跟绕 xyz 轴旋转的顺序相关
- Gimbal Lock：由于失去一个自由度导致旋转轴被锁定
- 难以插值 Hard to interpolate
- 难以叠加多次旋转
- 难以绕给定轴旋转

#### 四元数 Quaternion

使用
$$
q=a+bi+cj+dk \quad ,a,b,c,d \in R
$$
$$
i^2=j^2=k^2=ijk=-1
$$
表示。

四元数的定义：

![Pasted image 20220618164625](attachments/Pasted%20image%2020220618164625.png)

![Pasted image 20220618164952](attachments/Pasted%20image%2020220618164952.png)

四元数转为旋转矩阵：

![Pasted image 20220618212738](attachments/Pasted%20image%2020220618212738.png)

四元数的特性：

![Pasted image 20220618212839](attachments/Pasted%20image%2020220618212839.png)

定轴的四元数旋转：

![Pasted image 20220618213210](attachments/Pasted%20image%2020220618213210.png)

### 关节姿态 Joint Pose

具有 Orientation、Position（Translation）、Scale 三种量，共 9DoFs

表示一个骨骼的变换数据：使用一个仿射变换矩阵（一个线性变换接上一个平移）表示

![Pasted image 20220618233401](attachments/Pasted%20image%2020220618233401.png)

实际应用中，将最后一行的 $0,0,0,1$省略，存为 $3 \times 4$的矩阵

### 关节姿态插值 Joint Pose Interpolation

关节姿态插值使用局部坐标空间 Local Space，而非模型坐标空间 Model Space

原因为局部坐标空间下的坐标插值方便计算，结果正确；而模型坐标空间下直接对关节姿态进行插值，会得到错误结果

![Pasted image 20220618234815](attachments/Pasted%20image%2020220618234815.png)

### 蒙皮矩阵 Skinning Matrix

将 Mesh 的 Vertex 绑定到关节 Joint 的实质：在当前相对位置关系（绑定姿态 Bind Pose）创建一个姿态绑定矩阵 $M_{b(j)}^m$，用于表示 Vertex**从模型空间到相对于关节 Joint 局部空间**的仿射变换。

**（TL;DR：由于 Vertex 与 Joint 相对位置不变，对 Vertex 相对 Joint 在模型空间中的坐标，应用 Joint 相对初始状态在模型空间中的变换，即得到 Vertex 相对初始状态在模型空间中的变换。Sknning Matrix 即为 t 时刻 Joint 相对初始状态在模型空间中的变换。）**

![Pasted image 20220618235129](attachments/Pasted%20image%2020220618235129.png)

如何计算当前时刻 $t$，顶点 $V$在模型空间中的坐标：通过蒙皮变换矩阵 $K_j$得到：

- $M_j^m(t)$为从根节点开始，通过仿射变换矩阵左乘叠加得到的 $t$时刻关节 $J$的仿射变换矩阵
- $V_j^l(t)$为顶点 $V$的局部空间坐标，由顶点 $V$的模型空间坐标 $V_b^m$作姿态绑定矩阵的逆运算 $(M_{b(j)}^m)^{-1}$得到
- $V^m(t)$为顶点 $V$的模型空间坐标，由顶点 $V$的局部空间坐标 $V_j^l$作变换 $M_j^m(t)$得到
- 于是，模型空间中顶点姿态绑定矩阵的逆，左乘模型空间中 $t$时刻关节变换矩阵，即为 Skinning Matrix

![Pasted image 20220618235622](attachments/Pasted%20image%2020220618235622.png)

### 蒙皮矩阵调色盘 Skinning Matrix Palette

将所有顶点的 Skinning Matrix 再左乘模型空间到世界空间的变换矩阵，以数组存于显存

![Pasted image 20220619004122](attachments/Pasted%20image%2020220619004122.png)

### 多关节带权重的蒙皮 Weighted Skinning with Multi-Joints

一个顶点相关的骨骼一般不超过 4 个，加权平均

### 加权蒙皮混合 Weighted Skinning Blend

- 顶点 V 绑定的 $N$个关节 J，分别计算顶点 V 经过 J 的姿态矩阵变换之后的模型空间坐标
- 对得到的 $N$个模型空间坐标，**在模型空间下**加权平均，得到插值后的结果

![Pasted image 20220619004455](attachments/Pasted%20image%2020220619004455.png)

### 平移与缩放：简单插值 Simple Interpolation of Translation and Scale

使用 LERP（Linear Interpolation）

![Pasted image 20220619005142](attachments/Pasted%20image%2020220619005142.png)

### 旋转：四元数插值 Quaternion Interpolation of Rotation

使用 NLERP（Normalized LERP）

（图中浅蓝色线为 NLERP 插值结果）

![Pasted image 20220619005314](attachments/Pasted%20image%2020220619005314.png)

### 修复 NLERP 的最短路径问题 Shortest Path Fixing of NLERP

NLERP 默认使用最短路径旋转插值，无法得到转动大于 180 度的插值结果

- 使用两个四元数点乘得到结果与 0 比较，可知需要插值的方向，进行修复

![Pasted image 20220619005808](attachments/Pasted%20image%2020220619005808.png)

### NLERP 的问题 Problem of NLERP

NLERP 插值得到的旋转角速度不恒定！

### 角速度均匀的插值 SLERP：Uniform Rotation Interpolation

使用反三角函数计算两个四元数的夹角，使用夹角插值
- 反三角函数计算开销大
- 旋转角度小时，浮点数精度不够，发生抖动
- ↑也可能发生除以 0 异常

![Pasted image 20220619010133](attachments/Pasted%20image%2020220619010133.png)

### NLERP+SLERP Combined

现代引擎给定一个 magic number，在旋转角大时使用 SLERP，旋转角小时使用 NLERP

### 简单的动画运行时管线 Simple Animation Runtime Pipeline

![Pasted image 20220619010508](attachments/Pasted%20image%2020220619010508.png)

## 动画压缩 Animation Compression

### 最简单的压缩 - 限制自由度 Simplest Compression - DoF Reduction

咱还是别这么干了

### 关键帧 + 插值 Keyframe+Interpolation

Catmull-Rom Spline

![Pasted image 20220619011223](attachments/Pasted%20image%2020220619011223.png)

![Pasted image 20220619011428](attachments/Pasted%20image%2020220619011428.png)

### 浮点数压缩 Float Quantization

可以根据实际对于动作数据精度的需求，使用小于 32bit 的数据存储四元数中的每个值

![Pasted image 20220619011652](attachments/Pasted%20image%2020220619011652.png)

### 四元数压缩 Quaternion Quantization

![Pasted image 20220619012013](attachments/Pasted%20image%2020220619012013.png)

![Pasted image 20220619012025](attachments/Pasted%20image%2020220619012025.png)

### 误差传播 Error Propagation

由于骨骼关节存储局部空间的变换仿射矩阵，计算末端骨骼时需要从根骨骼一级一级作矩阵左乘：压缩动画数据，或者由于浮点数自身精度导致的误差，会随着骨骼一级一级递增

典例：拿着长柄武器的角色在站定不动时，武器尖端不停晃动

### 衡量准确度 Measuring Accuracy

使用两个线性无关的 Fake Vertex，定量衡量动画导致的误差

![Pasted image 20220619012928](attachments/Pasted%20image%2020220619012928.png)

### 误差补偿 Error Compensation

自适应误差范围 Adaptive Error Margins：
- 对精度要求不同的骨骼使用不同的 Error Threshold

![Pasted image 20220619013327](attachments/Pasted%20image%2020220619013327.png)

 原地补偿 In Place Correction：

 ![Pasted image 20220619013413](attachments/Pasted%20image%2020220619013413.png)

## 动画资产的创作 Animation DCC

略

# 游戏引擎的动画系统 - 高级

## 动画混合 Animation Blending

### 1D Blend Space

基于一个输入变量，线性插值

### 2D Blend Space

基于两个输入变量插值

使用 Delaunay 三角化 Delaunay Triangulation 生成空间的三角形划分，使用重心坐标实现三个顶点间的插值

### Skeleton Masked Blending

给动画设置每个骨骼的混合系数，作为遮罩

把多个动画按遮罩混合，即可实现上身用一个动画下身用另一个动画（或者更多）

### Additive Blending

只存储骨骼动画的变化量

在原有动画的基础上添加一部分骨骼的姿态修改，形成新的动作

容易出 Bug，要避免过度叠加

### 动画状态机 Animation State Machine（ASM）

一个用于表征角色运动状态的状态机，由代表状态的 Node 与状态转移条件 Transition Condition 组成

状态转移条件可能很复杂

![Pasted image 20220619235325](attachments/Pasted%20image%2020220619235325.png)

### 状态转移过渡 Cross Fades

- Smooth Transition：在两个播放的循环动画间均匀过渡，要求两个动画对轴
- Frozen Transition：冻结当前动画帧，渐入目标动画（常用于跳跃、着地等动作突变的情况）

![Pasted image 20220620142527](attachments/Pasted%20image%2020220620142527.png)

### 过渡曲线 Cross Fade Curve

![Pasted image 20220620142735](attachments/Pasted%20image%2020220620142735.png)

### 分层动画状态机 Layered ASM

传统经典设计：分层控制角色部分骨骼的运动（类似 Masked）

![Pasted image 20220620143045](attachments/Pasted%20image%2020220620143045.png)

### 动画树 Animation Blend Tree

Animation Blend Tree 是 Layered ASM 的扩展：

![Pasted image 20220620143802](attachments/Pasted%20image%2020220620143802.png)

动画树中的叶节点可以是动画片段 Clip、Blend Space、ASM；非叶节点可以是 LERP、Additive Blend

#### 结点类型

LERP Blend Node：

![Pasted image 20220620143652](attachments/Pasted%20image%2020220620143652.png)

Additive Blend Node：

![Pasted image 20220620143741](attachments/Pasted%20image%2020220620143741.png)

#### 控制参数 Control Parameters

1. 使用变量 Variable，控制 LERP 结点的参数
2. 使用事件 Event，切换动画树（或树内部 ASM）的状态（如持枪 ->拿刀）

![Pasted image 20220620144827](attachments/Pasted%20image%2020220620144827.png)

## 反向运动学 Inverse Kinematics（IK）

### 基础概念 Basic Concepts

末端作用器 End-Effector：需要被移动到理想位置的骨骼关节

IK（Inverse Kinematics）：使用运动学方程，解出关节的姿态参数，从而使 End-Effector 移动到理想位置的过程

FK（Forward Kinematics）：使用运动学方程，从关节的姿态参数，计算出 End-Effector 位置的过程

### Two-Bone IK（Single-Joint）

已知：膝盖关节、髋关节、脚着地点的位置，大腿骨、小腿骨长度固定，着地点到髋关节距离可计算

通过解三角形即可得出三条边的向量表示，通过预定义的 Reference Vector 即可确定膝盖关节位置的唯一解

![Pasted image 20220620145717](attachments/Pasted%20image%2020220620145717.png)

### Multi-Joint IK Solving

难点：
- 实时解高阶非线性方程
- 可能有多组解、唯一解、无解

![Pasted image 20220620150400](attachments/Pasted%20image%2020220620150400.png)

确认是否能够到目标 Check Reachability of Target：

![Pasted image 20220620150909](attachments/Pasted%20image%2020220620150909.png)

关节的限制 Constraints of Joint

![Pasted image 20220620151042](attachments/Pasted%20image%2020220620151042.png)

### 启发式算法 Heuristics Algorithm

#### CCD（Cyclic Coordinate Decent）

- 对关节迭代，每次旋转关节使得 End-Effector 尽可能接近目标点
- Rechability：在若干次迭代后停止，可得到尽可能接近目标的结果
- Constraints：在计算时允许角度限制 Angular Limit

![Pasted image 20220620151354](attachments/Pasted%20image%2020220620151354.png)

#### Optimized CCD：加入一个逐渐缩小的 Tolerance

![Pasted image 20220620151900](attachments/Pasted%20image%2020220620151900.png)

![Pasted image 20220620151925](attachments/Pasted%20image%2020220620151925.png)

#### FABRIK（Forward And Backward Reaching Inverse Kinematics）

![Pasted image 20220620152254](attachments/Pasted%20image%2020220620152254.png)

![Pasted image 20220620152356](attachments/Pasted%20image%2020220620152356.png)

### 多个末端作用器 Multiple End-Effectors

![Pasted image 20220620152549](attachments/Pasted%20image%2020220620152549.png)

![Pasted image 20220620152647](attachments/Pasted%20image%2020220620152647.png)

![Pasted image 20220620153310](attachments/Pasted%20image%2020220620153310.png)

![Pasted image 20220620153652](attachments/Pasted%20image%2020220620153652.png)

### 其他 IK 解决方案 Other IK Solutions

基于物理的方法 Physics-Based Method
- 更自然
- 需要大量计算

基于位置的动力学 PBD（Position Based Dynamics）
- 与传统的基于物理方法不同
- 视觉表现力更强
- 计算量更低

XPBD（Extended PBD）：虚幻五演示全身 IK（FBIK）

### IK 仍然具有许多挑战

- 如何避免蒙皮的自我碰撞 Self Collision Avoidance
- IK with prediction during moving
- 更接近人类的行为 Natural Human Behavior：数据驱动、深度学习

## 动画管线 Animation Pipeline

### 带有混合和 IK 的动画管线 Updated Animation Pipeline with Blending and IK

[](Games104.md#%E7%AE%80%E5%8D%95%E7%9A%84%E5%8A%A8%E7%94%BB%E8%BF%90%E8%A1%8C%E6%97%B6%E7%AE%A1%E7%BA%BF%20Simple%20Animation%20Runtime%20Pipeline)：

![Pasted image 20220619010508](attachments/Pasted%20image%2020220619010508.png)

↓↓↓升级后↓↓↓

![Pasted image 20220620154829](attachments/Pasted%20image%2020220620154829.png)

## 动画图 Animation Graph

## 面部动画 Facial Animation

![Pasted image 20220620155612](attachments/Pasted%20image%2020220620155612.png)

### FACS（Facial Action Coding System）

将人类面部表情定义为 46 个基础单元

<<https://www.paulekman.com/facial-action-coding-system/>>

![Pasted image 20220620155904](attachments/Pasted%20image%2020220620155904.png)

### Action Units Combination

表情可以被表示为 AU 的组合

![Pasted image 20220620160344](attachments/Pasted%20image%2020220620160344.png)

### 28 Core Action Units

Apple 总结了 28 个最常用的核心动作单元 Core Action Units，其中有 23 个对称的

![Pasted image 20220620235913](attachments/Pasted%20image%2020220620235913.png)

### Key Pose Blending

- FACS in Morph Target Animation：对于每个表情，存储顶点与自然状态下的差值，使用 Additive Blending

![Pasted image 20220621001215](attachments/Pasted%20image%2020220621001215.png)

### 面部骨骼 Facial Skeleton

用于控制眼球、下巴等整体运动的面部部分。调整骨骼位置、旋转、缩放即可改变脸部蒙皮，可用于捏脸系统

![Pasted image 20220621001153](attachments/Pasted%20image%2020220621001153.png)

### 纹理动画 UV Texture Facial Animation

- 在面部贴上动态纹理，实现表情，适用于风格化渲染
- 动森、塞尔达使用该技术

### 肌肉模型动画 Muscle Model Animation

目前属于前沿探索领域

![Pasted image 20220621001421](attachments/Pasted%20image%2020220621001421.png)

## 重定向 Retargeting

将一套动画用于多个身材（骨骼参数）不同的角色上

### 术语 Terminology

- Source Character
- Target Character
- Source Animation
- Target Animation

### 简单的重定向

记录 Source Animation 相对于 Source Character 骨骼在自然状态下的差值 Offset，把 Offset 应用到 Target Character 的自然状态骨骼上，形成 Target Animation

对于骨骼位移，记录位移距离相对骨骼长度的比例，进行等比缩放

![Pasted image 20220621002637](attachments/Pasted%20image%2020220621002637.png)

### IK in Retargeting

对于骨骼长度不一的情况，直接简单重定向可能导致错误

需要移动 Pelvis 并使用 IK 将 End-Effector 固定在合适的位置

![Pasted image 20220621002852](attachments/Pasted%20image%2020220621002852.png)

### 重定向不同的骨骼结构 Retargeting with Different Skeleton Hierarchy

1. 用骨骼名称识别可以对应的骨骼
2. 对于不能对应的骨骼，采用右图方法使得不对应段尽可能贴合原动画，并且首位位置能衔接上其它对应的骨骼

![Pasted image 20220621003045](attachments/Pasted%20image%2020220621003045.png)

### 没有解决的问题 Unresolved Problems of Retargeting

- 自碰撞
- 自接触等语义化动作（鼓掌时手掌无法接触）
- 不同角色的动画体态平衡性

![Pasted image 20220621003406](attachments/Pasted%20image%2020220621003406.png)

### Morph Animation Retargeting Problem

不同脸型的角色播放 Morph Target Animation 动画也会出现眼皮无法完全合上等问题
- 采用移动目标顶点 + 拉普拉斯算子平滑周围顶点的方法（略）

![Pasted image 20220621003824](attachments/Pasted%20image%2020220621003824.png)

# 游戏引擎中物理系统的基础理论和算法

## 物理对象与形状

### Actor 分类

- 静态 Static：不能移动，具有碰撞（场景物体）
- 动态 Dynamic：遵循牛顿运动定律
- 触发器 Trigger：与 Static Actor 类似，但没有碰撞。在其他 Actor 进入或离开 Trigger 时会产生 Event
- 运动学 Kinematic：不遵循牛顿运动定律（由 Gameplay Logic 控制的强制移动）

### Actor Shapes

- 球体 Sphere：点 + 半径，计算最简单
- 胶囊体 Capsule：球的扩展，常用于人形 Actor
- 长方体 Box
- 凸包多面体 Convex Mesh：只支持凸多面体，在复杂几何体中较为快速
- 三角网格体 Triangle Mesh：最慢，最精确
- 高程图 Height Field：用于地形

在引擎的物理系统中，使用 Physics Shapes 包裹物体，形成一个 Actor。注意使用尽可能简单的 Actor Shape 类型

![](attachments/Pasted%20image%2020220623011456.png)

### Shape Properties

- 质量和密度 Mass and Density
- 质心 Center of Mass
- 摩擦力和弹性 Friction&Restitution

## 力与运动

### 力 Force

- 重力
- 拉力
- 摩擦力

### 冲量 Impulse

模拟爆炸等效果

### 运动 Movement

#### 牛顿第一定律 Newton's 1st Law of Motion

没有外力时：
$$
\vec{v}(t+\Delta t) = \vec{v}(t) \\
\vec{x}(t+\Delta x) = \vec{x}(t) + \vec{v}(t) \Delta t
$$

#### 牛顿第二定律 Newton's 2nd Law of Motion

$$
\vec{F} = m \vec{a}
$$
$$
\vec{a}(t) = \frac{\text{d}}
$$
$$
\vec{v}(t+\Delta t) = \vec{v}(t) \\
\vec{x}(t+\Delta x) = \vec{x}(t) + \vec{v}(t) \Delta t
$$

牛顿第三定律 Newton's 3rd Law of Motion

# 游戏引擎物理系统的高级应用

# 游戏引擎中的粒子和声效系统
