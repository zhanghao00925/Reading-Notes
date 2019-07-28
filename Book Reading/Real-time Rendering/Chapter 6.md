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

This unwrapping process is one facet of a larger field of study, **mesh parameterization**.

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

Another type of corresponder is a matrix transformation, which can be applied in the vertex or pixel shader. This enables to translating, rotating, scaling, shearing, or projecting the texture on the surface.

> The order of transforms for textures must be the reverse of the order one would expect.

Another class of corresponder functions controls the way an image is applied.

+ In OpenGL, this type of corresponder function is called the `wrapping mode`.
+ In DirectX, it is called the `texture addressing mode`

Common corresponder functions of this type are

+ wrap, repeat, or tile : is often the default.
+ mirror : this provides some continuity along the edges of the texture.
+ clamp or clamp to edge : This function is useful for avoiding accidentally taking samples from the opposite edge of a texture when bilinear interpolation happens near a texture's edge.
+ border or clamp to border : This function can be  good for rendering decals onto single-color surfaces, for example, as the edge of the texture will blend smoothly with the border color.

These corresponder functions can be assigned differently ofr each texture axis.

A common solution to avoid such `periodicity` problem is to combine the texture values with another, no-tiled, texture. Multiple textures are combianed based on terrain type, altitude, slope, and other factors. Another option to avoid periodicity is to use shader programs to implement specialized corresponder functions that randomly recombine texture patterns or tiles.

The advantage of being able to specify $(u,v)$ values in a range of $[0,1]$ is that image textures with different resolutions can be swapped in without having to change the values stored at the vertices of the model.

### Texture Values

Image texturing constitutes the vast majority of texture use in real-time work, but procedural functions can also be used. In the case of procedural texturing, the process of obtaining a texture value from a texture-space location does not involve a memory lookup, but rather the computation of a function.

The values returned from the texture are optionally transformed before use.

> Shading normals stored in a color texture.

## Image Texturing

For the rest of this chapter, the image texture will be referred to simply as the `texture`. In additon, when we refer to a pixel's `cell`, we mean the screen grid cell surrounding that pixel.

In this section we particularly focus on methods to rapidly sample and filter textured images.

The question of what the float point coordinates of the center of a pixel are : `truncating` and `rounding`.

One term worth explaining at this point is `dependent texture read`, which has two definitions. Today such reads can have an impact on performance, depending on the number of pixels being computed in a batch, among other factors.

The texture image size used in GPUs is usually $2^m \times 2^n$ texel, where $m$ and $n$ are non-negative integers. These are referred to as `power-of-two` (POT) textures. Modern GPUs can handle `non-power-of-two` (NPOT) textures of arbitrary size, which allows a generated image to be treated as a texture. However, some older mobile GPUs may not support mipmapping for NPOT texture. Graphics accelerators have different upper limits on texture size.

What happens if the projected square covers ten times as many pixels as the original image contains (called `magnification`), or ifthe projected square covers only a small part of the screen (`minification`)? The answer is that it depends on what kind of sampling and filtering methods you decide to use for these two separate cases.

However, the desired result is to prevent aliasing in the final rendered image, which in theory requires sampling and filtering the final pixel colors. The distinction here is between filtering the inputs to the shading equation, or filtering its output. As long as the inputs and output are linearly, then filtering the individual texture values is equivalent ot filtering the final colors.

However, many shader input values stored in textures. Standard texture filtering methods may not work well for these textures, resulting in aliasing.

###  Magnification

The most common filtering techniques for magnification are `nearest neighbor` and `bilinear interpolation`. There is also cubic convolution. This enables much higher magnification quality.

+ Nearest neighbor : `pixelation` effect occurs, resulting a blocky appearance. While the quality of this method is sometimes poor, it requires only one texel to be fetched per pixel.
+ bilinear interpolation : The result is blurrier, and much of the jaggedness from using the nearest neighbor method has disappeared.

A common solution to the blurriness that accompanies magnification is to use `deail textures`. These are textures that represent fine surface details. Such detail is overlaid onto the magnified texture as a separate texture, at a different scale. The high-frequency repetitive pattern of the detail texture, combined with the low-frequency magnified texture, has a visual effect similar to the use of a single high-resolution texture.

Using bilinear interpolation gives varying grayscale samples across the texture. By remapping the texture looks more like a checkerboard again, while also giving some blend between texels. Using a higher-resolution texture would have a similar effect.

If bicubic filter are considered too expensive, Quilez proposes a simple technique using a smooth curve to interpolate in between a set of $2 \times 2$ texel.

Too commonly used curves:

+ smoothstep curve : $s(x)=x^2(3-2x)$
+ quintic curve : $q(x)=x^3(6x^2-15x+10)$

### Minification

To et a correct color value for each pixel, you should integrate the effect of the texels nfluencing the pixel.

Several different methods are used on GPUs. 

One method is to use the nearest neighbor, which works exactly as the corresponding magnification filter does. This filter may cause severe aliasing problems. Toward the horizon, artifacts appear because only one of the many texels influencing a pixel is chosen to represent the surface. Such artifacts are even more noticeable as the surface moves with respect to the viewer, and are one manifestation of what is called temporal aliasing.

Better solutions are possible. The signal frequency of a texturre depneds upon how closely spaced its texels are on the screen. Due to the Nyquist limit, we need to make sure that the texture's signal frequency is no greater than half the sample frequency. To more fully address this problem, various texture minification algorithm have been developed.

The basic idea behind all texture antialiasing algorithms is the same: to preprocess the texture and create data structures that will help compute a quick approximation of the effect of a set of texels on a pixel.

For real-time work, these algorithms have the characteristic of using a fixed amount of time and resources for execution. In this way, a fixed number of samples are taken per pixel and combined to compute the effect of a (potentially huge) number of texels.

#### Mipmapping

The most popular method of antialiasing for textures is alled `mipmapping`. `Mip` stands for `multum in parvo`, Latin for "many things in a small place", a good name for a process in which the original texture is filtered down repeatedly into smaller images.

When the mipmapping minimization filter is used, the original texture is augmented with a set of smaller versions of the texture before the actual rendering takes place. The texture (at level zero) is downsampled to a quarter of the original area, with each new texel value often computed as the average of four neighbor texels in the original texture. The new, level-one texture is sometimes called a `subtexture` of the original texture. The reduction is performed recursively until one or both of the dimensions of the texture equals one texel. The set of images as a whole is often called a `mipmap chain`.

Two important elements in forming high-quality mipmaps:

+ Good filtering
+ Gamma correction

The common way to form a mipmap level is to take each $2 \times 2$ set of texels and average them to get the mip texel value. The filter used is then a box filter, one of the worst filters possible. As it has the effect of blurring low frequencies unnecessarily, while keeping some high frequencies that cause aliasing. It is better to use a Gaussian, Lanczos, Kaiser, or similar filter.

For textures encoded in a nonlinear space (such as most color textures), ignoring gamma correction when filtering will modify the perceived brightness of the mipmap levels. As you get farther away from the object and the uncorrected mipmaps get used, the object can look darker overall, and contrast and details can also be affected. For this reason, it is important to convert such textures from sRGB to linear space, perform all mipmap filtering in that space, and convert the final results back into sRGB color space for storage.

Some textures have a fundamentally nonlinear relationship to the final shaded color. Although this poses a problem for filtering in general, mipmap generation is particularly sensitive to this issue, since many hundred or thousands of pixels are being filtered. Specialized mipmap generation methods are often neeeded for the best results.

The basic process of accessing this structure while texturing is straightforward. A screen pixel encloses an area on the texture itself. Using the pixel's cell boundaries is not strictly correct, but is used here to simplify the presentation. Texels outside of the cell an influence the pixel's color. The goal is to determine roughly how much of the texture influences the pixel. There are two common measures used to compute $d$ (which OpenGL calls $\lambda$, and which is also known as the texture level of detail). 

+ One is to use the longer edge of the quadrilateral formed by the pixel's cell to approximate the pixel's coverage.
+ Another is to use as a measure the largest absolute value of the four differentials $\partial u / \partial x, \partial v / \partial x, \partial u / \partial y, \partial v / \partial y$. Each differential is a measure of amount of change in the texture corrdinate with respect to a screen axis. These gradient values are available to pixel shader programs using Shader Model 3.0 or newer. Since they are based on the differences between values in adjacent pixels, they are not accessible in sections of the pixel shader affected by dynamic flow control.

The intent of computing the coordinate $d$ is to determine where to sample along the mipmap's pyramid axis. The goal is a pixel-to-texel ratio of at least $1 : 1$ to achieve the Nyquist rate. The important principle here is that as the pixel cell comes to include more texel and $d$ increases, a smaller, blurrier version of the texture is accessed. The value $d$ is analogous to a texture level, but instead of an integer value, $d$ has the fractional value of the distance between levels. The texture level above and the level below $d$ location is sampled. This entire process is called `trilinear interpolation` and is performed per pixel.

One user control on the d-coordinate is the `level of detail bias` (LOD bias). This is a value added to $d$, and so it affects the relative perceived sharpness of a texture. A good LOD bias for any given texture will vary with the image type and with the way it is used.

The benefit of mipmapping is that, instead of trying to sum all the texels that affect a pixel individually, precombined sets of texel are accessed and interpolated. This process takes a fix amount of time.

However, mipmapping has several flaws. A major one is `overblurring`.

#### Summed-Area Table

Another method to avoid overblurring is the `summed-area table` (SAT). To use this method, one first creates an array that is the size of the texture but contains more bits of precision for the color stored. At each location in this array, one must compute and store the sum of all the corresponding texture's texel in the rectangle formed by this location and texel $(0,0)$. During texturing, the pixel cell's projection onto the texture is bound by a rectangle. The summed-area table is then accessed to determine the average color of this rectangle, which is passed back as the texture's color for the pixel.

$$
c=\frac{s[x_{ur},y_{ur}]-s[x_{ur},y_{ll}]-s[x_{ll},y_{ur}]+s[x_{ll},y_{ll}]}{(x_{ur}-x_{ll})(y_{ur}-y_{ll})}
$$

The summed-area table is an example of what are called `anisotropic filtering` algorithms. Such algorithms retrieve texel values over areas that are not square.

Summed area tables, which give higher quality at a reasonable overall memory cost, can be implemented on mordern GPUs. Improved filtering can be critical to the quality of advanced rendering techniques.

#### Unconstrained Anisotropic Filtering

For current graphics hardware, the most common method to further improve texture filtering is to reuse existing mipmap hardware. 

The basic idea is that the pixel cell is back-projected, this quadrilateral (quad) on the texture is then sampled several times, and the samples are combined. Instead of using a single mipmap sample to approximate this quad's coverage, this algorithm uses several squares to cover the quad. The shorter side of the quad can be used to determine $d$ (unlike in mipmapping, where the longer side is often used); this makes the average area smaller (and so less blurred) for each mipmap sample. The quad's longer side is used to create a `line of anisotropy` parallel to the longer side and through the middle of the quad. When the amount of anisotropy is between $1:1$ and $2:1$, two samples are taken along ths line. At higher ratios of anisotropy, more samples are taken along the axis.

This scheme allows the line of anisotropy to run in any direction, and so does not have the limitations of summed-area tables. It also requires no more texture memory than mipmaps do, since it uses the mipmap algorithm to do its sampling.

### Volume Textures

A direct extension of image textures is three-dimensional image data that is accessed by $(u,v,w)$. A related idea is to represent volumetric lights in this form. The illumination on a point on a surface is found by finding the value for its location inside this volume, combined with a directions for the light.

Most GPUs support mipmapping for volume textures. Since filtering inside a single mipmap level of a volume texture involves trilinear interpolation, filtering between mipmap levels requires `quadrilinear interpolation`. Since this involves averaging the results from 16 texels, precision problems may result, which can be solved by using a higher-precision volume texture.

Although volume textures have significantly higher storage requirements and are more expensive to filter, they do have some unique advantages. The complex process of finding a good tewo-dimensional parameterizaton for the three-dimensional mesh can be skipped, since three-dimensional location can be used directly as texture coordinates. This avoids the distortion and seam problems that commonly occur with the two-dimensional parameterizations.

Using volume texture for surface texturing is extremely inefficient, since the vast majority of samples are not used.

### Cube Maps

Another type of texture is the `cube texture` or `cube map`, which has six square textures, each of which is associated with one face of a cube. A cube map is accessed with a three-component texture coordinate vector that specifies the direction of a ray pointing from the center of the cubde outward.

Cube maps are useful for representing values wich are a function of direction; they are most commonly used for environment mapping.

### Texture Representation

There are several ways to improve performance when handling many textures in an application. This section focus on texture atlases, texture arrays, and bindless textures, all of which aim to avoid the costs of changing textures while rendering.

To be able to batch up as much work as possible for the GPU, it is generally preferred to change state as little as possible. To that end, one may put several images into a single larger texture, called a `texture atlas`. Care also needs to be taken with mipmnap generation and access, since the upper levels of the mipmap may encompass several separate, unrelated shapes.

One difficulty with using an atlas is wrapping/repeat and mirror modes, which will not properly affect a subtexture but only the texture as a whole. Another problem can occur when generating mipmaps for an atlas, where one subtexture can bleed into another. However, this can be avoided by generating the mipmap hierarchy for each subtexture separately before placing them into a large texture atlas and using power-of-two resolutions for the subtextures.

A simpler solution to these issues is to use an API construction called `texture arrays`, which completely avoids any problems with mipmapping and repeat modes. All subtextures in a texture array need to have the same dimensions, format, mipmap hierarchy, and MSAA settings. Like a texture atlas , setpup is only done once for a texture array, and then any array element can be accessed using an index in the shader. This can be $5 \times$ faster than binding each subtexture.

A feature that can also help avoid state change costs is API support for `bindless textures`. Without bindless textures, a texture is bound to a specific texture unit the API. One problem is the upper limit on the number of texture units, which complicates matters for the programmer. The driver makes sure that the texture is resident on the GPU side. With bindless textures, there is no upper bound on the number of textures, because each texture is associated by just a 64-bit pointer, sometimes called a `handle`, to its data structure. These handles can be accessed in many different ways, e.g., throung uniforms, through varying data, from other textures, or from a shader storage buffer object(SSBO). The application needs to ensure that the textures are resident on the GPU side. Bindless textures avoid any type of binding cost in the driver, which makes rendering faster.

### Texture Compression

One solution that directly attacks memory and bandwidth problems and caching concerns is fixed-rate `texture compression`.