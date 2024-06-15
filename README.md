# RTGI-Plugin-Unreal

![image-20240615164451874](https://raw.githubusercontent.com/Chillstepp/MyPicBed/master/master/image-20240615164451874.png)

## Introduction

> RTGI Plugin: DDGI based on Voxelization, Global SDF and multiple fantastic tricks.

​	This method mainly based on DDGI. We use Cascade-Global-SDF and Voxelization to replace hardware ray tracing. We also did a lot to reduce overhead.



## Features

- Support Infinite Scroll.
- SRGB Blend
- Ray Tracing Budget
- Classify
- Temporal Filter
- GPU-Driven
- Probe Relocation



## Overview











## Reference

DDGI related:

- Zander Majercik, Jean-Philippe Guertin, Derek Nowrouzezahrai, and Morgan McGuire, Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields, *Journal of Computer Graphics Techniques (JCGT)*, vol. 8, no. 2, 1-30, 2019
  Available online [http://jcgt.org/published/0008/02/01/](https://jcgt.org/published/0008/02/01/)
- Zander Majercik, Adam Marrs, Josef Spjut, and Morgan McGuire, Scaling Probe-Based Real-Time Dynamic Global Illumination for Production, *Journal of Computer Graphics Techniques (JCGT)*, vol. 10, no. 2, 1-29, 2021
  Available online [http://jcgt.org/published/0010/02/01/](https://jcgt.org/published/0010/02/01/)
- Blog ["Dynamic Diffuse Global Illumination"](https://morgan3d.github.io/articles/2019-04-01-ddgi/) written by  [Morgan McGuire](https://casual-effects.com/).
- [RTX Global Illumination (RTXGI)](https://github.com/NVIDIAGameWorks/RTXGI-DDGI) from [NVIDIAGameWorks](https://github.com/NVIDIAGameWorks)
- GDC2019: [Ray-Traced Irradiance Fields (Presented by NVIDIA)](https://www.gdcvault.com/play/1026182/)
- GDC2019: [Scalable Real-Time Global Illumination for Large Scenes](https://www.gdcvault.com/play/1026469/Scalable-Real-Time-Global-Illumination)

Voxelization related:

- [The Basics of GPU Voxelization](https://developer.nvidia.com/content/basics-gpu-voxelization) by NVDIA
- Elmar Eisemann and Xavier Décoret. 2008. Single-pass GPU solid voxelization for real-time applications. In Proceedings of Graphics Interface 2008 (GI '08). Canadian Information Processing Society, CAN, 73–80.

Sparse Distance Field:

- SIGGRAPH 2022 Advances in Real-Time Rendering in Games:  [Lumen: Real-time Global Illumination in Unreal Engine 5](https://advances.realtimerendering.com/s2022/index.html#Lumen)

Others:
