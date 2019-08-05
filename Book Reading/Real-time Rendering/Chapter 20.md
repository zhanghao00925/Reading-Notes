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

As a start on improving performance, we could determine the screen bounds of a light volume and use them to draw a screen-space quadrilateral that covers a smaller part of the image.