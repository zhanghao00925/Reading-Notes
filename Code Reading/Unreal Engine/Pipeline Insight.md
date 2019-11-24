# Unreal Pipeline insight

## Render Thread

### Call stack

#### `FRendererModule::BeginRenderingViewFamily`

First get render scene by 

``` cpp
FScene* const Scene = ViewFamily->Scene->GetRenderScene();
```

> What is `ViewFamily` \
> `ViewFamily` is an instance of class `FSceneViewFamily`, contains `TArray<const FSceneView*> Views` which make up the family. And render target and some other information will be used in rendering.

> What is `Views` in `ViewFamily` \
> Views are an Array of `FSceneView`. And `FSceneView` contains matrix, clear color and so on. Each view will be used to project actors from scene space to 2D screen region.

> What is `ViewFamily->Scene` \
> `ViewFamily->Scene` is an pointer of `FSceneInterface` which is an interface to the private scene manager implementation of a scene. You can use this interface to add primitives, lights, decals and so on. As far as I am concerned everything in level is managed by this interface.

> What is `FScene` \
> FScene implement `FSceneInterface`. To another word, you can used FScene intance to managed the scene.

After get scene pointer, if it is not null.

``` cpp
World = Scene->GetWorld();
if (World)
{
    //guarantee that all render proxies are up to date before kicking off a BeginRenderViewFamily.
    World->SendAllEndOfFrameUpdates();
}
```
`SendAllEndOfFrameUpdates()` would send all render updates to the rendering thread. I will talk more about it when I read source about components, cause it calls `Component->OnEndOfFrameUpdateDuringTick()`.

> What is `World` \
> `World` is an pointer of `UWorld`, which contains actors and levels. For more information about UE4's level and world, you can read [InsideUE4](https://zhuanlan.zhihu.com/p/22924838).

``` cpp
ENQUEUE_RENDER_COMMAND(UpdateDeferredCachedUniformExpressions)(
		[](FRHICommandList& RHICmdList)
		{
			FMaterialRenderProxy::UpdateDeferredCachedUniformExpressions();
		});
        
ENQUEUE_RENDER_COMMAND(UpdateFastVRamConfig)(
    [](FRHICommandList& RHICmdList)
{
    GFastVRamConfig.Update();
});
```

(To be done)

Then

``` cpp
// Flush the canvas first.
Canvas->Flush_GameThread();
```

> In `Flush_GameThread()`, sends a message to the rendering thread to draw the batched elements, it will set viewport.

> What is `batched elements` \
> (To be done)

> What is `Canvas` \
> `Canvas` is an instance of `FCanvas` which encapsulates the canvas state.

``` cpp
if (Scene)
{
    // We allow caching of per-frame, per-scene data
    Scene->IncrementFrameNumber();
    ViewFamily->FrameNumber = Scene->GetFrameNumber();
}
else
{
    // this is passes to the render thread, better access that than GFrameNumberRenderThread
    ViewFamily->FrameNumber = GFrameNumber;
}
```

Increment frame number.

Then

``` cpp
for (int ViewExt = 0; ViewExt < ViewFamily->ViewExtensions.Num(); ViewExt++)
{
    ViewFamily->ViewExtensions[ViewExt]->BeginRenderViewFamily(*ViewFamily);
}
```

Used in VR and AR devices.

Then

``` cpp
// Set the world's "needs full lighting rebuild" flag if the scene has any uncached static lighting interactions.
if(World)
{
    // Note: reading NumUncachedStaticLightingInteractions on the game thread here which is written to by the rendering thread
    // This is reliable because the RT uses interlocked mechanisms to update it
    World->SetMapNeedsLightingFullyRebuilt(Scene->NumUncachedStaticLightingInteractions, Scene->NumUnbuiltReflectionCaptures);
}
```

Nothing special here.

Then

``` cpp
// Construct the scene renderer.  This copies the view family attributes into its own structures.
FSceneRenderer* SceneRenderer = FSceneRenderer::CreateSceneRenderer(ViewFamily, Canvas->GetHitProxyConsumer());
```

In `FSceneRenderer::CreateSceneRenderer()` create Renderer according to the FeatureLevel:

``` cpp
EShadingPath ShadingPath = InViewFamily->Scene->GetShadingPath();
FSceneRenderer* SceneRenderer = nullptr;

if (ShadingPath == EShadingPath::Deferred)
{
    SceneRenderer = new FDeferredShadingSceneRenderer(InViewFamily, HitProxyConsumer);
}
else 
{
    check(ShadingPath == EShadingPath::Mobile);
    SceneRenderer = new FMobileSceneRenderer(InViewFamily, HitProxyConsumer);
}

return SceneRenderer;
```

In contructor, `StencilLODDitherCVar` is set.

> What is `FHitProxyConsumer` \
> Manage `HHitProxy`, which is used to collect data about what was clicked on in the viewport.

Then

``` cpp
if (!SceneRenderer->ViewFamily.EngineShowFlags.HitProxies)
{
    USceneCaptureComponent::UpdateDeferredCaptures(Scene);
}
```

In `UpdateDeferredCaptures()`, it first find all `SceneCapturesToUpdate`, then for each Component in this array, call `Component->UpdateSceneCaptureContents(Scene)`, in this functuion, simply call `Scene->UpdateSceneCaptureContents(this)`(More detail to be done).

Then

``` cpp
// We need to execute the pre-render view extensions before we do any view dependent work.
// Anything between here and FDrawSceneCommand will add to HMD view latency
ENQUEUE_RENDER_COMMAND(FViewExtensionPreDrawCommand)(
    [SceneRenderer](FRHICommandListImmediate& RHICmdList)
    {
        ViewExtensionPreRender_RenderThread(RHICmdList, SceneRenderer);
    });
```
As mentioned before, this function is used in VR/AR. More detail to be done.

Then

``` cpp
if (!SceneRenderer->ViewFamily.EngineShowFlags.HitProxies)
{
    for (int32 ReflectionIndex = 0; ReflectionIndex < SceneRenderer->Scene->PlanarReflections_GameThread.Num(); ReflectionIndex++)
    {
        UPlanarReflectionComponent* ReflectionComponent = SceneRenderer->Scene->PlanarReflections_GameThread[ReflectionIndex];
        SceneRenderer->Scene->UpdatePlanarReflectionContents(ReflectionComponent, *SceneRenderer);
    }
}
```

``` cpp
SceneRenderer->ViewFamily.DisplayInternalsData.Setup(World);
```

More detail to be done.

Then

``` cpp
ENQUEUE_RENDER_COMMAND(FDrawSceneCommand)(
    [SceneRenderer](FRHICommandListImmediate& RHICmdList)
    {
        RenderViewFamily_RenderThread(RHICmdList, SceneRenderer);
        FlushPendingDeleteRHIResources_RenderThread();
    });
```

call most important function `RenderViewFamily_RenderThread`

#### `RenderViewFamily_RenderThread`

[Reference](https://docs.unrealengine.com/en-US/Programming/Rendering/Overview/index.html)

In `RenderViewFamily_RenderThread`, it calls `SceneRenderer->Render(RHICmdList)` which actually render the scene.

#### `FDeferredShadingSceneRenderer::Render`

In `Render(FRHICommandListImmediate& RHICmdList)`, first `PrepareViewRectsForRendering();`.

Then

``` cpp
if (Scene->SunLight && Scene->HasAtmosphericFog())
{
    // Only one atmospheric light at one time.
    Scene->GetAtmosphericFogSceneInfo()->PrepareSunLightProxy(*Scene->SunLight);
}
```

In `PrepareSunLightProxy(*Scene->SunLight)` prepare information for further calculation. In this function's implement, it gives a reference paper about [atmosperic fog rendering](https://media.contentapi.ea.com/content/dam/eacom/frostbite/files/s2016-pbs-frostbite-sky-clouds-new.pdf).

Then

``` cpp
#if WITH_MGPU
	const FRHIGPUMask RenderTargetGPUMask = (GNumExplicitGPUsForRendering > 1 && ViewFamily.RenderTarget) ? ViewFamily.RenderTarget->GetGPUMask(RHICmdList) : FRHIGPUMask::GPU0();
	ComputeViewGPUMasks(RenderTargetGPUMask);
#endif // WITH_MGPU
```

Skip multi gpu rendering for now.

Then 

``` cpp
FSceneRenderTargets& SceneContext = FSceneRenderTargets::Get(RHICmdList);

//make sure all the targets we're going to use will be safely writable.
GRenderTargetPool.TransitionTargetsWritable(RHICmdList);

// this way we make sure the SceneColor format is the correct one and not the one from the end of frame before
SceneContext.ReleaseSceneColor();
```

`FSceneRenderTargets::Get(RHICmdList)` simply return `TGlobalResource<FSceneRenderTargets> SceneRenderTargetsSingleton`

> What is `FSceneRenderTargets` \
> as far as i am concerned, it is an encapsulates of GBuffer including some other data.

Then test if Decal feature supported.

``` cpp
const bool bDBuffer = !ViewFamily.EngineShowFlags.ShaderComplexity && ViewFamily.EngineShowFlags.Decals && IsUsingDBuffers(ShaderPlatform);
```

``` cpp
WaitOcclusionTests(RHICmdList);
```

> Guess : wait last frame's occlusion test.

Then do some initialize jobs about texture and render target.

``` cpp
// Initialize global system textures (pass-through if already initialized).
GSystemTextures.InitializeTextures(RHICmdList, FeatureLevel);

// Allocate the maximum scene render target space for the current view family.
SceneContext.Allocate(RHICmdList, this);
```

After get some showflags, 

``` cpp
// Find the visible primitives.
RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

bool bDoInitViewAftersPrepass = false;
{
    SCOPED_GPU_STAT(RHICmdList, VisibilityCommands);
    bDoInitViewAftersPrepass = InitViews(RHICmdList, BasePassDepthStencilAccess, ILCTaskData, UpdateViewCustomDataEvents);
}
```

Initializes primitive visibility for the views through various culling methods, sets up dynamic shadows that are visible this frame, intersects shadow frustums with the world if necessary (for whole scene shadows or preshadows).

In `InitViews` function,

``` cpp
PreVisibilityFrameSetup(RHICmdList);

RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

FViewVisibleCommandsPerView ViewCommandsPerView;
ViewCommandsPerView.SetNum(Views.Num());

ComputeViewVisibility(RHICmdList, BasePassDepthStencilAccess, ViewCommandsPerView, DynamicIndexBufferForInitViews, DynamicVertexBufferForInitViews, DynamicReadBufferForInitViews);

RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

// This has to happen before Scene->IndirectLightingCache.UpdateCache, since primitives in View.IndirectShadowPrimitives need ILC updates
CreateIndirectCapsuleShadows();
RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

PostVisibilityFrameSetup(ILCTaskData);
RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
```

this part do visibility culling.

And also setup `VolumetricFogGrid` and `LightPropagationVolume` in `SetupVolumetricFog();` and `OnStartRender(RHICmdList);`

We skip ray tracing part so far.

