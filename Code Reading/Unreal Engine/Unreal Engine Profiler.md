# Unreal Engine Profiler

## CPU Profiler

### Operator

Console `stat SceneRendering` `stat Game` `stat DumpFrame -ms=0.1`

### Cause of the Problem

+ The render thread needs to process each object (culling, material setup, lighting setup, collision, update cost, etc.). More complex materials result in a higher setup cost.
+ The render thread needs to prepare the GPU commands to set up the state for each draw call (constant buffers, textures, instance properties, shaders) and to do the actual API call. Base pass draw calls are usually more costly than depth only draw calls.
+ DirectX validates some data and passes the information to the graphics card driver.S
+ The driver (e.g. NVIDIA, AMD, Intel, ...) validates further and creates a command buffer for the hardware. Sometimes this part is split in another thread.

### Method to solve the Problem

+ Reducing object count (static/dynamic meshes, mesh particles)
+ Reducing view distance (e.g. on the Scene Capture Actor or per object)
+ Adjusting the view (more zoomed out view, moving objects to not be in the same view)
+ Avoiding SceneCaptureActor (needs to re-render the scene, set fps to be low or update only when needed)
+ Avoiding split screen (Always more CPU bound than single view, needs custom scalability settings or content to be more aggressive)
+ Reducing Elements per draw calls (combine materials accepting more complex pixel shaders or simply have less materials, combine textures to fewer larger textures - only if that reduces material count, use LOD models with fewer elements)
+ Disabling features on the mesh like custom depth or shadow casting
+ Changing light sources to not shadow cast or having a tighter bounding volume (view cone, attenuation radius)

## GPU Profiler

### Operator

Console `ProfileGPU`

Shortcut `Ctrl + Shift + ,`

### Cause of the Problem

+ EarlyZPass: By default we use a partial z pass. DBuffer decals require a full Z Pass. This can be customized with r.EarlyZPass and r.EarlyZPassMovable.
+ Base Pass: When using deferred, simple materials can be bandwidth bound. Actual vertex and pixel shader is defined in the material graph. There is an additional cost for indirect lighting on dynamic objects.
+ Shadow map rendering: Actual vertex and pixel shader is defined in the material graph. The pixel shader is only used for masked or translucent materials.
+ Shadow projection/filtering: Adjust the shader cost with r.ShadowQuality.Disable shadow casting on most lights. Consider static or stationary lights.
+ Occlusion culling: HZB occlusion has a high constant cost but a smaller per object cost. Toggle r.HZBOcclusion to see if you do better without it on.
+ Deferred lighting: This scales with the pixels touched, and is more expensive with light functions, IES profiles, shadow receiving, area lights, and complex shading models.
+ Tiled deferred lighting: Toggle r.TiledDeferredShading to disable GPU lights, or use r.TiledDeferredShading.MinimumCount to define when to use the tiled method or the non-deferred method.
+ Environment reflections: Toggle r.NoTiledReflections to use the non-tiled method which is usually slower unless you have very few probes.
+ Ambient occlusion: Quality can be adjusted, and you can use multiple passes for efficient large effects.
+ Post processing: Some passes are shared, so toggle show flags to see if the effect is worth the performance

### Method to solve the Problem

+ A full EarlyZ pass costs more draw calls and some GPU cost but it avoids pixel processing in the Base Pass and can greatly reduce the cost there.
+ Optimizing the HZB could cause the culling to be more conservative.
+ Enabled shadows can reduce the lighting cost of the light if large parts of the screen are in shadow.

****

## Test

In observer mode's worst case, Roaming state.

Test position : -35190.0 727 515.0

### Adjust light parameters

adjust light parameters, including `AttenuationRadius` and `position`(The light used be hidden in the lamp mesh, so the shadow part is wasted). Now it can show shadow correctly.


+ FPS befor : lower than 10
+ FPS after : about 16

> after adjust light parameters, the bottleneck tranform to car light. As far as i am concerned, the back lights and turn lights don't need to cast shadow, and head lights can use a single light with a light function to make it appear to be two lights.

### Adjust car light parameters

adjust `NPCVehicle` 's light parameters, including change `SetVisibility` to `bAffectsWorld` to lower the light cost.

+ FPS not change much, due to the shadowdepth cost.

As far as i am concerned, the back lights and turn lights don't need to cast shadow, set `cast shadow` to `false`

+ FPS befor : about 16
+ FPS after : about 24

> after ajust car light parameters, the bottleneck is transform to three addition rendertarget(for different sensor).

### Others

There are some console commands that can affect shadow. Details can be seen in [this website](https://forums.unrealengine.com/development-discussion/rendering/5920-shadows-disappear-with-increased-camera-distance).