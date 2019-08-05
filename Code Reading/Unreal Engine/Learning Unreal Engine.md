# 光源

## 光源类型

+ 方向光
+ 点光源
+ 聚光灯

> `Source Radius` and `Source Length` can affect hightlight's shape.  
> `Min roughness` can affect hightlight's blur.  
> `Radius` can affect soft shadow.  

+ 天空光

> 远处的入射光信息涵盖了“Sky Distance Threshold”设置的距离以外的所有入射光来源，包括远景、天空盒、云雾等。它的实现方式是先将远处的入射光信息渲染到Cube Map中，然后使用这张Cube Map来计算光照。

## 实时性

+ Static

> 运行时不能被移动或修改，其所有光照信息，包括：直接光照、间接光照、阴影都是预计算存储在Lightmap中。此外，光源的Radius会影响阴影的软硬程度，Radius越大阴影越软。

+ Stationary

> 在运行时不能被移动，但可以修改光照强度和颜色。其直接光照是实时计算，间接光照是预计算好的，阴影对于动态物体采用Shadow Map，对于静态物体则预计算好。

+ Movable

> 在运行时可以被移动以及修改，但是它仅仅包含直接光照和阴影，并不包含间接光照。此外，为了减少实时阴影计算的时间开销，Unreal 4引擎对于这种类型光源的Shadow Map进行了缓存。当光源静止不动时，场景中静态物体的Shadow Map被保存下来下一帧继续使用。

## 渲染管线

+ Deferred Rendering

# 全局光照

Unreal 4中的全局光照系统叫做`Lightmass`。对于静态和动态的物体，Unreal 4采用了与Unity类似的解决方案：静态物体使用Lightmap记录物体表面光照信息，动态物体使用点云记录空间中光照信息，在实时计算时根据物体位置进行插值计算。

## Lightmass Importance Volume

在Unreal 4引擎中，为了提高重要区域全局光的采样率和光照计算质量，引入了Lightmass Importance Volume。该区域表示了场景中重要的区域，例如：玩家可以寻路到达的地方等。在该区域内，Lightmass会在计算全局光照时增加光子（Photon）的数量，从而加大这些区域的间接光照采样率，提高了全局光照质量。而在这些区域以外，则使用少量光子，加快计算速度。

## Indirect Lighting Cache

对于动态物体，Unreal 4引擎与Unity引擎采用了相似的全局光照算法。在Unreal 4引擎中，可以通过在空间中分布了点云，对间接光照进行采样计算并存储在Indirect Lighting Cache中。然后根据动态物体当前的位置插值计算间接光照。

## Summary

关于全局光照，Unreal 4引擎和Unity引擎采用的方法比较类似。在渲染预计算光照时，都是对静态物体采用Light Map，对动态物体采用点云（光照探针）。
操作上的不同之处在于Unreal 4引擎需要设置Importance Volume，然后Indirect Lighting Sample会自动分布。

# 阴影

Unreal 4引擎中的阴影渲染主要包含两部分：

+ 预计算阴影
+ 实时计算阴影

## 预计算阴影

Unreal 4引擎的预计算阴影渲染不仅支持不透明物体阴影渲染，而且支持半透明物体阴影渲染。

## 实时阴影

Unreal 4引擎的实时阴影计算支持：

+ Shadow Map

> 对于Directional光源，Unreal 4引擎与Unity引擎同样支持Cascade Shadow Map。并且，通过设置“Dynamic Shadow Distance Stationary Light”可以支持将远处的物体阴影计算逐步用静态阴影替代，从而减少性能开销。

+ Screen Space Ambient Occlusion

# 反射效果

Unreal 4引擎的材质系统在渲染时提供了反射效果，并通过`Metallic`和`Roughness`参数进行控制。

Unreal 4引擎也提供了Reflection Capture Actor抓取场景的Cube Map以提供反射功能。\

****

## 材质编辑