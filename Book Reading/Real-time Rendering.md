# Chapter 6 Texturing

Texturing is a process that takes a surface and modifies its appearance at each location using some image, function, or other data source. 

+ `Parallax mapping` uses a texture to appear to deform a flat surface when rendering it.
+ `Parallax occlusion mapping` casts rays against a heightfield texture for improved realism.
+ `Displacement mapping` truly displaces the surface by modifying triangle heights forming the model.

## The Texturing Pipeling

Texturing is a technique for efficiently modeling variations in a surface's material and finish. Texturing works by modifying the values used in the shading equation. The way these values are changed is normally based on the position on the surface.

Texturing can be described by a generalized texture pipeline.

1. A location in space is the starting point for the texturing process. This location can be in world space, but is more often in the model's frame of reference.
2. Using Kershaw's terminology, this point in space then has a projector function applied to it to obtain a set of numbers, called `texture coordinates`.
3. Before values be used to access the texture, one or more `corresponder` functions can be used to transform the texture coordinates to texture space. These texture-space location are used to obtain values from the texture.
4. `Texture coordinates` is used for accessing the texture, this process is called `mapping`, which leads to the phrase `texture mapping`.
5. The retrieved values are then potentially transformed yet again by a value transform function, and finally these new values are used to modify some property of the surface.

The reason for the complexity of the pipeline is that each step provides the user with a useful control. It should be noted that not all steps need to be activated at all times.

### The Projector Function

The first step in the texture process is obtaining the surface's location and projecting it into texture coordinate space, usually two-dimensional space.Functions commonly used in modeling programs include spherical, cylindrical, and planar projections.

Other inputs can be used to a projector function.

> the surface normal can be used to choose which of six planar projection directions is used for the surface.

Other projector functions are not projections at all, but are an implicit part of surface creation and tessellation.

Non-interactive renderers often call these projector functions as part of the rendering process itself. A single projector function may suffice for the whole model, but often the artist has to use tools to subdivide the model and apply various projector functions separately.

In real-time work, projector functions are usually applied at the modeling stage, and the results of the projection are stored at the vertices. Sometimes it is advantageous to apply the projection function in the vertex or pixel shader. Doing so can increase precision, and helps enable various effects, including animation.

> Some rendering methods, such as `enviornment mapping`, have specialized projector funcgtions of their own that are evaluated per pixel.

+ Spherical projection casts points onto an imaginary sphere centered around some point.
+ Cylindrical projection is useful for objects that have a natural axis, such as surfaces of revolution. Distortion occurs when surfaces are near-perpendicular to the cylinder's axis.
+ Planar projection is useful for applying decals.

Artist often must manualy decompose the model into near-panar pieces. There are tools that help minimize distortion by unwrapping the mesh, or creating a near-optimal set of planar projections, or that otherwise aid this process. The goal is to have each polygon be given a fairer share of a texture's area, while also maintaining as much mesh connectivity as possible.

> Connectivity is important in that sampling artifacts can appear along edges where separate parts of a texture meet.

**This unwrapping process is one facet of a larger field of study, `mesh parameterization`.**

The texture coordinate space is not always a two-dimensional plane; sometimes it is a three-dimensional volume $(u,v,w)$, with $w$ being depth along the projection direction. Other systems use up to four coordinates, often designated $(s,t,r,q)$, $q$ is used as the fourth value in ahomogeneous coordinate.

Another important type of texture coordinate space is directional, where each pointin the space is accessed by an input direction. The most common type of texture using a directional parameterization is the `cube map`.

One-dimensional texture images and functions have their uses.

+ On a terrain model the coloration can be determined by altitude.
+ Lines can also be textured.
+ Such textures are also useful for converting from one value to another, i.e., as a lookup table.

However the coordinate values are applied, the idea is the same: These texture coordinates are interpolated across the surface and used to retrieve texture values.

### The corresponder Function

Corresponder functions convert texture coordinates to texture-space locations. They provide flexibility in applying textures to surfaces.

> One example is to use the API to select a portion of an existing texture for display.



# Chapter 20

When one or more elements become expensive do we need to use more involved techniques to rein in costs. Here we concentrate on techniques for reducing costs when evaluting materials and lights.For many of these methods, there is an additional processing cost, with the hope being that this expense is made up by the savings obtained. Others trade off between bandwidth and computation, often shifting the bottleneck.

