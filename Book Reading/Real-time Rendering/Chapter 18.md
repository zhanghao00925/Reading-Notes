# Chapter 19 Acceleration Algorithms

在实时绘制中，有至少四个性能目标：
+ 每秒更多帧
+ 更高的分辨率和采样率
+ 更真实的材质和光照
+ 增加几何复杂度

本章主要介绍加速计算机图形绘制的算法，特别是绘制大量的几何。许多这类算法的核心是基于空间数据结构。基于空间数据结构，介绍剔除算法。LOD技术降低绘制剩下的物体的复杂度，在本章结束部分，介绍绘制大型模型的系统，包括virtual texturing, streaming, transcoding 和 terrain rendering。

## Spatial Data Structures

一个用来组织几何体的n维空间。用来加速查询几何实体是否重叠。这些查询在很多操作中都要用到。将查询从O(n)提高到O(logn)，但是构建的代价比较大。

常见的空间数据结构有BVH，BSP trees。

### BVH

