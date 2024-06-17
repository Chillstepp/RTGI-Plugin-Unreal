# RTGI-Plugin-Unreal

![image-20240615164451874](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240615164451874.png)

## Introduction

> RTGI Plugin: DDGI based on Voxelization, Global SDF and multiple fantastic tricks.

​		This method mainly based on DDGI. We use Cascade-Global-SDF and Voxelization to replace hardware ray tracing. We also did a lot to reduce overhead.



## Features & Optimizations

- Infinite Scroll
- Cascaded probes
- Relocate probe origin when view direction changed (For better view frustum coverage).
- Progressive Frame Update
- SRGB Blend
- Ray Tracing Budget
- Classify
- Temporal Filter
- GPU-Driven
- Probe Relocation
- Clipmap and AABB Scroll are used to reduce Voxelization overhead
- Easy to tune parameters

### Infinite Scroll

It's cost a lost time to recompute all the probe data when the camera moved. As shown in the figure below, we can just update those probes which in **blue area** obviously.

![image-20240617150515182](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240617150515182.png)

```c++
(probeCoords + ProbesCounts + ProbesScrollOffsets[cascadeIndex].xyz) % ProbesCounts
```



### Cascaded probes

- We use 4 cascade probe volumes. The four cascade probe volumes have same probe counts.The only difference between them is the **probe spacing** (The distance of  two adjacent probes).
- With 4 cascade probe volumes, we can apply probes to cover lager scene. 
  - Our the largest cascade distance scale is 10.0f. `const float cascadesDistanceScales[] = { 1.0f, 2.0f, 5.0f, 10.0f };`
  - Probe Spacing in first cascade is `200*200*200 ` by default.
  - Probe Count in one cascade is `20*20*10` by default.
  - So the largest probe volume will cover `400m * 400m * 200m`.
  - You can tune `Probe Spacing`, `Probe Count`, `cascadesDistanceScales[]` by your demands.



### Classify

Classify just likes a state machine. There are four states of probe: 

```c++
#define DDGI_PROBE_STATE_INACTIVE 0
#define DDGI_PROBE_STATE_ACTIVATED 1
#define DDGI_PROBE_STATE_ACTIVE 2
#define DDGI_PROBE_STATE_SLEEP 3 // Stop Tracing Rays, but can be sampled
```

- DDGI_PROBE_STATE_INACTIVE
  - Probe is too far from geometry. 

### Progressive Frame Update

- We update each cascade probe by `uint64 cascadeFrequencies[] = { 2, 3, 5, 7 };`, which means the `i`th cascade probe will update every `cascadeFrequencies[i]` frame.
- Our method only updates one cascade per frame. This means that when the frame count can be divided by two or more `cascadeFrequencies`, we will only update the smallest cascade. 
- You can open `r.rtgi.AlwaysUpdateProbe 1` ,which will update all cascade every frame.

### Relocate probe origin when view direction changed (For better view frustum coverage)

​	We shift probe origin towards to the view direction for better view frustum coverage when place probes.This feature can be opened by `r.rtgi.RelocateProbeVolumeOriginWhenViewDirectionChanged`.

![image-20240617132615340](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240617132615340.png)



There are more probes in frustum after `r.rtgi.RelocateProbeVolumeOriginWhenViewDirectionChanged`  feature opened.

| Close                                                        | Open                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image-20240617133408601](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240617133408601.png) | ![image-20240617133432739](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240617133432739.png) |

### Probe Relocation



### Infine Bounce



## Overview







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

**Voxelization related:**

- [The Basics of GPU Voxelization](https://developer.nvidia.com/content/basics-gpu-voxelization) by NVDIA
- Elmar Eisemann and Xavier Décoret. 2008. Single-pass GPU solid voxelization for real-time applications. In Proceedings of Graphics Interface 2008 (GI '08). Canadian Information Processing Society, CAN, 73–80.

Sparse Distance Field:

- SIGGRAPH 2022 Advances in Real-Time Rendering in Games:  [Lumen: Real-time Global Illumination in Unreal Engine 5](https://advances.realtimerendering.com/s2022/index.html#Lumen)

**Others:**
