# Chapter 20

When one or more elements become expensive do we need to use more involved techniques to rein in costs. Here we concentrate on techniques for reducing costs when evaluting materials and lights. 

For many of these methods, there is an additional processing cost, with the hope being that this expense is made up by the savings obtained. Others trade off between bandwidth and computation, often shifting the bottleneck.

As with all such schemes, which is best depends on your hardware, scene structure, and many other factors.

Evaluating a pixel shader for a material can be expensive. This cost can be reduced by various shader level of detail simplifcation techniques. When there are several light sources affecting a surface, two different strategies can be used. One is to build a shader that supports multiple light sources, so that only a single pass is needed. Another is `multi-pass shading`, where we create a simple one-light pixel shader a light and evaluate it, adding each result to the framebuffer. So, for three lights, we would draw the primitive three times, changing the light for each evaluation. This second method might be more efficient overall than a single pass system, because each shader used is simpler and faster. If a renderer has many different types of lights, a one-pass pixel shader must include them all and test for whether each is used, making for a complex shader.

If we can efficiently determine that a surface does not contribute to the final image, then we can save the time spent shading it. One technique performs a `z-prepass`, where the opaque geometry is rendered and only z-depths are written. The geometry is then rendered again fully shaded, and the z-buffer from the first pass culls away all framents that are not visible. This type of pass is an attempt to decouple the process of finding what geometry is visible from the operation of subsequently shading that geometry. 

## Deferred Shading

The idea behind `deferred shading` is to perform all visibility testing and surface property evaluations before performing any material lighting computations. For interactive rendering, deferred shading specifically means that all material parameters associated with the visible objects are generated and stored by an initial geometry pass, then lights are applied to these stored surface values using a post-process. Geometry pass establishes all geometry and material information for the pixel, so the objects are no longer needed.

Note that overdraw can happen in this initial pass, the difference being that the shader's execution is considerably less -- tranferring values to bufers -- than that of evaluating the effect of a set of lights on the material.

The buffers used to store surface properties are commonly called `G-buffers`, also occasionally called `deep buffers`. Each G-buffer is a separate render target. Systems have gone as high as eight. Having more targets uses more bandwidth, which increase the chande that this buffer is the bottleneck.

After the pass creating the G-buffers, a separate process is used to compute the effect of illumination. One method is to apply each light one by one, using the G-buffers to compute its effect. This process is about the most inefficient way to use G-buffers, since every stored pixel is accessed for every light, similar to how basic forward rendering applies all lights to all surface fragments. Such an approach can end up being slower than forward shading, due to the extra cost of writing and reading the G-buffers.

As a start on improving performance, we could determine the screen bounds of a light volume and use them to draw a screen-space quadrilateral that covers a smaller part of the image. In this way, pixel processing is reduced, often significantly. We can also use the third screen dimension, z-depth. By drawing a rough sphere mesh encompassing the volume, we can trim the sphere's area of effect further still.

Long shaderes with dynamic branches often run considerably more slowly, so a large number of smaller shaders can be more efficient, but also require more work to generate and manage. Since all shading functions are down in a single pass with forward shading, it is more likely that the shader will need to change when the next object is rendered, leading to inefficiency from swapping shaders.

The deferred shading method of rendering allows a strong separation between lighting and material definition. Each shader is focused on parameter extraction of lighting, but not both. Shorter shaders run faster, both due to length and the ability to optimize them. The number of registers used in a shader determines occupancy. This decoupling of lighting and material also simplifies shader system management.

With each light handled fully in a single pass, deferred shading permits having only one shadow map in memory at a time. However, this advantage disappear with the more complex light assignment schemes, as lights are evaluated in groups.

Basic deferred shading supports just a single material shader with a fixed set of parameters, which constrains what material models can be portrayed. One way to support differet material descriptions is to store a material ID or mask per pixel in some given field. The shader can then perform different computations based on the G-buffer contents. This approach could also modify what is stored in the G-buffers, based on this ID or mask value.

Basic deferred shading has some other drawbacks. G-buffer video memory requirement can be significant, as can the related bandwidth costs in repeatedly accessing these buffers. We can mitigate these costs by storing lower-precision values or compressing the data. 

Two important technical limitations of deferred shading involve **transparency** and **antialiasing**.

Transparency is not supported in a basic deferred shading system, since we can store only one surface per pixel. One solution is to use forward rendering for transparent objects after the opaque surfaces are rendered with deferred shading. For early deferred systems this meant that all lights in a scene had to be applied to each transparent object, a costly process, or other sximplifications had to be performed. while it is possible to now store lists of transparent surfaces for pixels and use a pure deferred approach, the norm is to mix deferred and forward shading as desired for transparency and other effects.

Deferred shading could store all $N$ samples per element in the G-buffers to perform antialiasing, but the increases in memory cost, fill rate, and computation make this approach expensive. To overcome this limitation, Shishkovtsov uses an edge detection method for approximating edge coverage computations(?). Other morphological post-processing methods for antialiasing can also be used, as well as temporal antialiasing. Several deferred MSAA avoid computing the shade for every sample by detecting which pixels or tiles have edges in them. 

Even with these limitations, deferred shading is a practical rendering method used in commercial programs. It naturally separates geometry from shading, and lighting from materials, meaning that each element can be optimized on its own. One area of particular interest is decal rendering, which has implications for any rendering pipeline.

## Decal Rendering

A decal is some design element, such as a picutre or other texture, applied on top of a surface. Decal are often seen in video games in such forms as tire marks, bullet holes, or player tags sprayed onto surface. Decals are used in other applications for applying logos, annotations or other content. For terrain systems or cities, for example, decals can allow artists to avoid obvious repetition by layering on detailed textures, or by recombining various patterns in different ways.

A decal can blend with the underlying material in a variety of ways. It might modify the underlying color but not the bump map. Alternately, it might replace just the bump mapping, such as an embossed logo does. It could define a different material entirely. The variations have implications for how forward and degerred shading systems store and process decals. 

To begin, the decal must be mapped to the surface, like any other texture. Since multiple texture coordinates can be stored at each vertex, it is possible to bind a few decals to a single surface. This approach is limited, since the number of values that can be saved per vertex is relatively low. Each decal needs its own set of texture coordinates. A large number of small decals applied to a surface would mean saving these texture coordinates at every vertex, even though each decal affects only a few triangles in the mesh.

To render decals attached to a mesh:

One approach is to have the pixel shader sample every decal and blend one atop the next. This complicates the shader, and if the number of decals varies over time, frequent recompilation or other measures may be required.

Another approach that keeps the shader independent from the decal system is to render the mesh again for each decal, layering and blending each pass over the previous one. If a decal spans just a few triangles, a separate, shorter index buffer can be created to render just this decal's sub-mesh. 

One other decal system, modifying this texture provides a simple "set it and forget it" solution. This baked solution avoids shader complexity and wasted overdraw, but at the cost of texture management and memory use.

Rendering the decals separately is the norm, as different resolutions can then be applied to the same surface, and the base texture can be reused and repeated without needing additional modified copies in memory.

A popular solution for static or rigid objects is to treat the decal as a texture orthographically projected through a limited volume.

Deferred shading excels at rendering such decals. Instead of needing to illuminate and shade each decal, as with standard forward shading, the decal's effect can be applied to the G-buffers, avoiding the sahding overdraw that occurs with forward shading. 

Several problems with decals in a deferred setting:

Blending is limited to what operations are available during the merge stage in the pipeline. Another concern is fringing along silhouette edges of decals, caused by gradient errors due to using screen-space information projected back into world space.

Decals can be used for dynamic elements 
