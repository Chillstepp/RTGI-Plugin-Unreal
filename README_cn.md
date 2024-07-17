# RTGI-Plugin-Unreal

![image-20240615164451874](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240615164451874.png)

## 性能部分

	GPU: NVIDIA GeForce RTX 3060
	CPU: 11th Gen Intel(R) Core(TM) i7-11700K @ 3.60GHz

基于Global SDF和Voxelization的DDGI，包含多种优化技巧，性能表现如下：

- ApplyLighting：0.65ms . `RenderDiffuseIndirectAndAmbientOcclusion` 部分

- UpdateDDGIDatas： 可以根据ProbeBudget调整该项性能。注意：由于Probe每帧通过Classify激活的数量有上限，在Probe Budget Count达到一定程度时性能不会继续提升。以下是在某个室内场景下的测试：

  | Probe Budget Count | Cost  |
  | ------------------ | ----- |
  | 1000               | 0.4ms |
  | 2000               | 0.5ms |
  | 3000               | 0.6ms |
  | 5000               | 0.7ms |
  | 6400(Default)      | 0.8ms |

- InjectLight：屏幕空间注入光照0.07ms

- 由于Lightmap pass不再需要采样，节省了更多时间。



**总结：1.5ms左右可以完成漫反射全局光照计算。**

## 流程概要

![image-20240717200025457](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240717200025457.png)

## Features & Optimizations in DDGI

- Infinite Scroll
- Cascaded probes & Cascade Blend
- Classify
- Progressive Frame Update
- Relocate probe origin when view direction changed (For better view frustum coverage).
- Correct Visibility & Self-shadow bias to avoid light leaking
- Probe Relocation
- sRGB Gamma Blending
- Temporal Filter
- Probe Relocation
- Perception-based exponential encoding
- Indirect Light Scaling

### Infinite Scroll

当相机移动时重新计算所有探测数据会浪费时间。如下图所示，明显我们可以只更新那些位于**蓝色区域**的探针。

![image-20240617150515182](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240617150515182.png)

- 原始探测体积的中心被称为**Scroll Anchor**，Probe Volume会随着相机的移动而更新其位置：

  - 当存在重叠区域（紫色区域）时，为了复用重叠区域，在Probe Volume移动后，记录`ProbesScrollOffsets`。当在新的探针体积中采样探针时，将原坐标加上`ProbesScrollOffsets`即可，访问原理如下图：

    ```c++
    (probeCoords + ProbesCounts + ProbesScrollOffsets[cascadeIndex].xyz) % ProbesCounts
    ```

    ![image-20240619141448349](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240619141448349.png)

- 当Probe Volume远离Scroll Anchor处的Probe Volume并且它们不再重叠时，我们将重置Scroll Anchor的位置。





### Cascaded probes & Cascade Blend

- 我们使用 4 级级联探针体积。4级级联探针体积具有相同的探针数量。它们之间唯一的区别是**探针间距**（两个相邻探针的距离）。
- 通过 4 个级联探针卷，我们可以以更低的开销应用探针来覆盖更大的场景。
  - 我们最大的 `CascadesDistanceScales`是10.0f。 `const float CascadesDistanceScales[] = { 1.0f, 2.0f, 5.0f, 10.0f };`
  - 默认情况下，第一级级联中的探头间距为`200*200*200`。
  - 默认情况下，第一级级联中的探针计数为`20*20*10`。
  - 所以最大的Probe Volume将覆盖 `400m * 400m * 200m`。
  - 您可以根据需要调整 `Probe Spacing`, `Probe Count`, `cascadesDistanceScales[]` 。
- 层级之间光追会有轻微跃变（通过混合相邻层级减轻）



### Classify

> This part is significant to speed up DDGI. You can read `Classify.usf` to get more information.

​	Probe可能卡在几何体内部（即使在Probe Relocation尝试移动它们之后），存在于附近没有几何体的空间中，或者距离相机足够远。在这些情况下，无需花费时间Trace和更新这些probe的radiance和distance的texture图集。

​	我们的probe状态包含四种：

```c++
#define DDGI_PROBE_STATE_INACTIVE 0
#define DDGI_PROBE_STATE_ACTIVATED 1
#define DDGI_PROBE_STATE_ACTIVE 2
#define DDGI_PROBE_STATE_SLEEP 3
```

- `DDGI_PROBE_STATE_INACTIVE`: 不采样也不更新
  - probe距离几何体过远
  - 探针可能卡在几何体内部（即使在Probe Relocation尝试移动它们之后）
- `DDGI_PROBE_STATE_ACTIVATED`: 当一个探针是Active状态，更新后会标记为Activated
  - 这意味着我们最近激活了它，并且最近非常重要。所以我们会尝试更新它（即使它在摄像机背面）。

- `DDGI_PROBE_STATE_ACTIVE`
  - 为了平均分摊开销，**Probe Budget**设置在一帧中。我们使用轮询来每帧更新一部分探针。请求部分中的那些探针将被标记为Active的。

- `DDGI_PROBE_STATE_SLEEP`: 不更新（不做ray tracing）此probe，而是对其进行采样。
  - 摄像机背面
  - 为了平均分摊开销，**Probe Budget**设置在一帧中。我们使用轮询来每帧更新一部分探针。不在请求部分中的那些探针将被标记为Sleep。




![whiteboard_exported_image](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/whiteboard_exported_image.png)

>  在早期，只添加了 DDGI_PROBE_STATE_SLEEP 以分摊开销并阻止更新相机背面的探针。但是那些近期被激活的探针仍然需要进行ray tracing才能获得更好更快的场景变化响应。因此 DDGI_PROBE_STATE_ACTIVATED 也被添加到探针状态机中。



### Progressive Frame Update(Probe Budget & Cascade Budget)

为了更好的性能，我们实现了两个维度上的**分帧更新**，分别是: **级联探针按照层级分帧更新** 和 **探针分帧更新**

**Cascade Budget(对于级联探针，我们实现了不同级拥有不同的更新频率)：**

- 我们通过 `uint64 cascadeFrequencies[] = { 2, 3, 5, 7 };`来控制每个cascade的probe volume的更新频率, 第`i`级的Probe Volume会每隔`cascadeFrequencies[i]`帧更新一次.
- 我们的策略是每次只更新一个cascade以免造成峰值卡顿. 这意味着当前帧数对多个 `cascadeFrequencies`取模等于0时, 我们会选择更小的cascade进行更新，即我们希望优先保证相机近周围的画面质量。
- 调试：使用 `r.rtgi.AlwaysUpdate 1` , 所有的cascade每帧都会更新。



**Probe Budget(对于某个cascade，在探针更新时，每次只更新一部分需要更新的探针):**

- 相当于每帧去轮询需要更新的探针，每帧只更新一定数量，超过预算则下一次再更新。



### Relocate probe origin when view direction changed (For better view frustum coverage)

我们将探针原点移向相机方向，以便在放置探针时获得更好的视锥体覆盖范围，这个特性相当于让相机点不再等于Probe Volume的中心点，以获得视锥内更好的光照效果。 可以通过 `r.rtgi.RelocateProbeVolumeOriginWhenViewDirectionChanged` 开启此特性。

![image-20240617132615340](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240617132615340.png)

打开 `r.rtgi.RelocateProbeVolumeOriginWhenViewDirectionChanged` 后，如下图可以发现视锥内有更多的probe.

| Close                                                        | Open                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image-20240617133408601](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240617133408601.png) | ![image-20240617133432739](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240617133432739.png) |



### Correct Visibility & Self-shadow bias to avoid light leaking

> For more details, please read  `float3 SampleDDGIIrradiance()` function

这部分主要实现了以下两个特性：

- Visibility weight (Chebyshev) 实现Visibility项
- Self-shadow bias 解决漏光



我们用切比雪夫不等式计算一个`Probe`与着色点之间可见的概率，这利用了VSSM的思想:

这需要在记录时额外记录一些信息，包括：

- $E(w):$ probe从$w$方向的半球面接收到的irradiance。
- $r(w):$ probe往$w$方向看到的最近物体与probe距离。 (存储半球面上的均值)
- $r^2(w):$ probe往$w$方向看到的最近物体与probe距离平方。 (存储半球面上的均值)

切比雪夫不等式为：
$$
P(r>d)\leq\frac{\sigma^2}{\sigma^2+\left(d-\mu\right)^2}\quad(d>\mu)
$$
其中$d$为着色点到probe的距离，$\mu$为上述的$r(w)$均值，$\sigma^2$表示距离的方差，我们可以通过下式计算方差和均值：
$$
\begin{array}{c}\mu=r(w)\\\sigma^2=r^2(w)-[r(w)]^2\end{array}
$$
$P(r>d)$表示probe和着色点之间没有遮挡的概率, 当$d$大于$\mu$时认为不会被遮挡，设置$P(r>d) = 1$，否则根据公式进行计算概率(假设总能取到上界):
$$
P(r>d)=\frac{\sigma^2}{\sigma^2+\left(d-\mu\right)^2}\quad(d>\mu)\\
P(r>d) = 1\quad(d<=\mu)
$$
最后算出的概率还会做一个3次幂，这是一个主观的参数，相当于希望漏光概率更低。

```c++
// Visibility weight (Chebyshev)
if (biasedPosToProbeDist > probeDistanceMean)
{
    float probeDistanceVariance = abs(probeDistanceMean * probeDistanceMean - probeDistanceMean2);
    float chebyshevWeight = probeDistanceVariance / (probeDistanceVariance + Square(biasedPosToProbeDist - probeDistanceMean));
    weight *= max(chebyshevWeight * chebyshevWeight * chebyshevWeight, 0.05f);
}
```

到此就解决了**Visibility**的问题。



然而这里还可能出现一些由Self-shadow bias造成的漏光，所以我们需要加上一个bias在着色表面。

$BiasVector=(\mathbf{n}*0.2+\omega_o*0.8)*(0.75*D)*B$

- $n$ — 着色点的法向量
- $\omega_0$ — 从着色点到相机的方向向量
- $D$ — 探针间的距离，即Probe Spacing
- $B$ — 供自己调节的一个标量，推荐0.3，在大部分场景表现不错。

| Without Bias                                                 | With Bias                                                    |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image-20240620204424746](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240620204424746.png) | ![image-20240620204455438](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240620204455438.png) |

### Probe Relocation

​	尽管我们有切比雪夫测试来剔除可能会导致漏光漏阴影的probe，但是当一个probe离墙靠的非常近时，切比雪夫测试会出现以下问题。

- 对于贴近墙面的Probe：会导致在着色时切比雪夫的均值被离群值影响，这会导致可见性测试在锐利角度出现较大误差。同时在极端角度下对光照信息的收集也会不全面如下图会有部分光源因为墙面的突起而无法被probe所捕获。

  ![image-20240621171809904](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240621171809904.png)

- 对于在墙体内的probe，probe在着色时就不会被采样，导致着色点可获取的信息量减少，因此需要将probe移到表面外

所以我们需要尽量避免一个probe离墙面非常近，需要对probe的位置进行偏移，从而让其原理表面，总结下我们的需求：

- 对于物体表面内部的Probe，我们要尽可能让它移动到表面外

- 对于物体表面外部靠近物体的probe，我们要尽可能的让它远离表面。同时需要注意，

- 为了保持8个probe的相对位置，我们需要对probe的偏移做一个限制，该限制不能超过probe间距的一半。

我们的方案是在周围27个点分别尝试偏移probe间距的一半并计算偏移后的sdf值，选择这些偏移中sdf值最大的，即尽可能原理墙面的位置却又不会影响到周围probe相对位置。

如下图，黑色为墙体，黑色虚线框为不能超过probe间距的一半的限制，我们绿色点是sdf值最大的，所以最终会relocate到绿色点处。

![image-20240621164015160](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240621164015160.png)



### sRGB Gamma Blending

​	在三线性插值时，由于人眼对亮度感知是非线性的，需要转到非线性空间去插值。

​	![image-20240621102212887](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240621102212887.png)

如图所示：

 	如果有一个点想要在亮度为0.0的A点和亮度为0.5的B点之间进行插值。  最好是在感知线性亮度上做（以gamma编码），暗区细节更多，更适合人眼感知。

![image-20240621102421094](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240621102421094.png)	

​	着色时，我们将在插值之前对辐照度应用伽玛校正，然后在插值之后恢复到线性空间以匹配渲染管线（渲染管道最终将始终应用色调映射）。



### Temporal Filter

 	在更新Probe的irradiance的数据时，为了重用之前帧的数据，在每次更新的数据和之前更新的数据之间进行插值。这不仅**减少了画面抖动**，还减少了采样数量。我们提供**参数 `ProbeHistroyWeight`** 来调整历史帧的权重。
 	
 	然而，有时照明的快速变化需要视觉的快速响应。因此，我们还提供了一个 **`ProbeChangeThreshold` 参数**；如果变化超过这个阈值，则认为没有必要重用历史帧，或者可能需要适当减少历史帧权重。

```c++
if (GIMaxAxis(previous.rgb - result.rgb) > ProbeChangeThreshold)
{
    // Lower the hysteresis when a large lighting change is detected
    historyWeight = max(0.f, historyWeight - 0.75f);//decrease weight!
}
```

​	我们也同时实现了了基于亮度Luminance的判断，即把RGB颜色差值转换为Luminance作为判断标准。

```c++
/**
 * Convert Linear RGB value to Luminance
 */
float GILinearRGBToLuminance(float3 rgb)
{
	const float3 LuminanceWeights = float3(0.2126, 0.7152, 0.0722);
	return dot(rgb, LuminanceWeights);
}
```

 最后，使用历史帧进行插值：
$$
newValue=(1-\alpha)*curValue+\alpha*oldValue
$$

### Perception-based exponential encoding

由于时域滤波的存在，探针会收敛缓慢，场景中突然的照明变化可能会产生明显的滞后。我们可以用指数编码来加快探针数据的收敛。

Encoding:

```c++
result.rgb = pow(result.rgb, ProbeInvertIrradianceEncodingGamma);
```

Decoding:

```c++
//DDGI.ush SampleDDGIIrradiance()
float3 exponent = ProbeIrradianceEncodingGamma * 0.5f;//0.5 is gamma blending before, now we revert it here.
probeIrradiance = pow(probeIrradiance, exponent);
```

在插值时:
$$
newValue=[(1-\alpha)*curValue^{\frac1{gamma}}+\alpha*oldValue^{\frac1{gamma}}]^{gamma}
$$


### Indirect Light Scaling

​	在这篇演讲中 [GTC China: RTXGI 在游戏剑侠情缘网络版三家园系统中的应用](https://www.nvidia.cn/on-demand/session/gtccn2020-cns20991/) .作者提到，由于Albedo太暗，场景在间接照明下显得较暗。为了解决这个问题，他们提供了一个参数来调整Albedo。在这里，我们还提供了**一个缩放参数`IndirectLightingIntensity`**来调整间接照明。

```c++
//在进行Blend后进行放大，解决场景过暗
Irradiance *= IndirectLightingIntensity;
```

| IndirectLightingIntensity = 1(Default)                       | IndirectLightingIntensity = 2.2                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image-20240623171347302](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240623171347302.png) | ![image-20240623171516980](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240623171516980.png) |

## Infinite Bounce

​	实现无限弹射主要源于, 在Trace一个Probe的radiance时：

- 若Probe的Trace击中：**1-Bounce 间接光照** + **Multi-Bounce 间接光照** 共同更新probe的radiance texture

  - 1-Bounce 间接光照： 直接TraceRay采样LightVoxel

  - Multi-Bounce 间接光照： 采样周围8个Probe

- 若Probe的Trace未击中，说明打到了天空

  - 直接采样Skylight的球谐函数



对于Trace击中的情况做图示，如下图：

![image-20240717204353390](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240717204353390.png)

![image-20240717204402688](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240717204402688.png)

以此类推，第三次就可以拿到Probe的3-Bounce Info + 2-Bounce Info，逐次进行增加从而实现无限弹射。

## Features & Optimizations in Voxelization

### 只更新新区域

我们只更新移动前后两个AABB的非相交处

### 动态物体

​	除了移动前后两个AABB的非相交区域处要更新，即新出现的区域要更新，还要考虑原来区域是否有可以移动的动态物体。若存在动态物体，则动态物体包围盒内也要重新做体素化。

## Math Tips

> Here are some math tips for you to understand the code better.

### The Fibonacci sphere algorithm

​	When the probe ray traces to the surface of an object, we can obtain its radiance. Therefore, uniform sampling on the probe is a very important issue. Let's simplify our problem: how evenly distribute n points on a sphere?

​	The Fibonacci sphere algorithm(https://arxiv.org/pdf/0912.4540.pdf) is what we need.

### Wang Hash

![image-20240619144356202](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240619144356202.png)

### Octahedral mapping

![image-20240717213358664](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240717213358664.png)



## 思考：一些可能的改进

### ReSTIR + DDGI

​	DDGI Resampling论文里有提到可以用ReSTIR指导重要性采样，不过个人觉得意义不大，原因在于原论文那么做还是为了Specular项的这种高频信息，希望通过ReSTIR加速了采样结果的收敛的速度。然而在Diffuse中本身信息就是低频的，采样不足也不太影响结果，另外即使有采样不足的情况直接做一个低通平滑滤波还更直接一些。

## Reference

**DDGI related:**

- Zander Majercik, Jean-Philippe Guertin, Derek Nowrouzezahrai, and Morgan McGuire, Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields, *Journal of Computer Graphics Techniques (JCGT)*, vol. 8, no. 2, 1-30, 2019
  Available online [http://jcgt.org/published/0008/02/01/](https://jcgt.org/published/0008/02/01/)
- Zander Majercik, Adam Marrs, Josef Spjut, and Morgan McGuire, Scaling Probe-Based Real-Time Dynamic Global Illumination for Production, *Journal of Computer Graphics Techniques (JCGT)*, vol. 10, no. 2, 1-29, 2021
  Available online [http://jcgt.org/published/0010/02/01/](https://jcgt.org/published/0010/02/01/)
- Blog ["Dynamic Diffuse Global Illumination"](https://morgan3d.github.io/articles/2019-04-01-ddgi/) written by  [Morgan McGuire](https://casual-effects.com/).
- [RTX Global Illumination (RTXGI)](https://github.com/NVIDIAGameWorks/RTXGI-DDGI) from [NVIDIAGameWorks](https://github.com/NVIDIAGameWorks)
- GDC2019: [Ray-Traced Irradiance Fields (Presented by NVIDIA)](https://www.gdcvault.com/play/1026182/)
- GDC2019: [Scalable Real-Time Global Illumination for Large Scenes](https://www.gdcvault.com/play/1026469/Scalable-Real-Time-Global-Illumination)
- GTC China: RTXGI 在游戏剑侠情缘网络版三家园系统中的应用 https://www.nvidia.cn/on-demand/session/gtccn2020-cns20991/

**Voxelization related:**

- [The Basics of GPU Voxelization](https://developer.nvidia.com/content/basics-gpu-voxelization) by NVDIA
- Elmar Eisemann and Xavier Décoret. 2008. Single-pass GPU solid voxelization for real-time applications. In Proceedings of Graphics Interface 2008 (GI '08). Canadian Information Processing Society, CAN, 73–80.

**Sparse Distance Field:**

- SIGGRAPH 2022 Advances in Real-Time Rendering in Games:  [Lumen: Real-time Global Illumination in Unreal Engine 5](https://advances.realtimerendering.com/s2022/index.html#Lumen)

**Others:**

- The Fibonacci sphere algorithm.  https://arxiv.org/pdf/0912.4540.pdf
- Mark Jarzynski and Marc Olano, Hash Functions for GPU Rendering, *Journal of Computer Graphics Techniques (JCGT)*, vol. 9, no. 3, 21-38, 2020 Available online [http://jcgt.org/published/0009/03/02/](https://jcgt.org/published/0009/03/02/)
- Gamma Correct https://learnopengl-cn.github.io/05%20Advanced%20Lighting/02%20Gamma%20Correction/