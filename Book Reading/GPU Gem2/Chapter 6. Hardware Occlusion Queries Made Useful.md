# Chapter 6. Hardware Occlusion Queries Made Useful

## Introduction

This feature makes it possible for an application to ask the 3D API (OpenGL or Direct3D) whether or not any pixels would be drawn if a particular object were rendered. With this feature, applications can check whether or not the bounding boxes of complex objects are visible; if the bounds are occluded, the application can skip drawing those objects. Hardware occlusion queries are appealing to today's games because they work in completely dynamic scenes.

This is due to two main problems related to naive usage of the occlusion query feature: the overhead caused by issuing the occlusion queries themselves (since each query adds an additional draw call), and the latency caused by waiting for the query results.

To achieve this, the algorithm exploits the spatial and temporal coherence of visibility by reusing the results of occlusion queries from the previous frame in order to initiate and schedule the queries in the current frame. This is done by storing the scene in a hierarchical data structure (such as a k-d tree or octree [Cohen-Or et al. 2003]), processing nodes of the hierarchy in a front-to-back order, and interleaving occlusion queries with the rendering of certain previously visible nodes.

## What Is Occlusion Culling?

Online occlusion culling is more general in that it works for fully dynamic scenes. Typical online occlusion-culling techniques work in image space to reduce computation overhead. Even then, however, CPUs are less efficient than, say, GPUs for performing rasterization, and thus CPU-based online occlusion techniques are typically expensive (Cohen-Or et al. 2003).

Fortunately, graphics hardware is very good at rasterization. Recently, an OpenGL extension called NV_occlusion_query, or now ARB_occlusion_query, and Direct3D's occlusion query (D3DQUERYTYPE_OCCLUSION) allow us to query the number of rasterized fragments for any sequence of rendering commands. Testing a single complex object for occlusion works like this (see also Sekulic 2004):

1. Initiate an occlusion query.
2. Turn off writing to the frame and depth buffer, and disable any superfluous state. Modern graphics hardware is thus able to rasterize at a much higher speed (NVIDIA 2004).
3. Render a simple but conservative approximation of the complex object—usually a bounding box: the GPU counts the number of fragments that would actually have passed the depth test.
4. Terminate the occlusion query.
5. Ask for the result of the query (that is, the number of visible pixels of the approximate geometry).
6. If the number of pixels drawn is greater than some threshold (typically zero), render the complex object.

The approximation used in step 3 should be simple so as to speed up the rendering process, but it must cover at least as much screen-space area as the original object, so that the occlusion query does not erroneously classify an object as invisible when it actually is visible. The approximation should thus be much faster to render, and does not modify the frame buffer in any way.

But step 5 involves waiting until the result of the query actually becomes available. Since, for example, Direct3D allows a graphics driver to buffer up to three frames of rendering commands, waiting for a query results in potentially large delays. In the rest of this chapter, we refer to steps 1 through 4 as "issuing an occlusion query for an object."

## Hierarchical Stop-and-Wait Method

### The Naive Algorithm, or Why Use Hierarchies at All?

let's take a look at the naive occlusion query algorithm first:

1. Sort objects front to back.
2. For each object
   1. Issue occlusion query for the object (steps 1–4 from previous section).
   2. Stop and wait for result of query.
   3.  If number of visible pixels is greater than 0, render the object.

there might be tens of thousands of objects in the scene, most of which are hidden by nearby buildings. If each of these objects is not very complex, then issuing and waiting for queries for all of them is more expensive than just rendering them.

### Hierarchies to the Rescue!

The big advantage of using hierarchies for occlusion queries is that we can now test interior nodes, which contain much more geometry than the individual objects.

With hierarchies, though, interior nodes group a larger number of draw calls, which are all saved if the node is occluded using a single query. So we see that in some cases, a hierarchy is what makes it possible to gain anything at all by using occlusion queries.

### Hierarchical Algorithm

The naive hierarchical occlusion-culling algorithm works like this—it is specified for a node within the hierarchy and initially called for the root node:

1. Issue occlusion query for the node.
2. Stop and wait for the result of the query.
3. If the node is visible:
    1. If it is an interior node:
        1. Sort the children in front-to-back order.
        1. Call the algorithm recursively for all children.
    1. If it is a leaf node, render the objects contained in the node.

This algorithm can potentially be much faster than the basic naive algorithm, but it has two significant drawbacks.

### Problem 1: Stalls

The query has to be sent to the GPU. There it sits in a command queue until all previous rendering commands have been issued (and modern GPUs can queue up more than one frame's worth of rendering commands!). Then the bounding box associated with the query must be rasterized, and finally the result of the query has to travel back over the bus to the driver.During all this time, the CPU sits idle, and we have caused a CPU stall. But that's not all. While the CPU sits idle, it doesn't feed the GPU any new rendering commands.Now when the result of the occlusion query arrives, and the CPU has finally figured out what to draw and which commands to feed the GPU next, the command buffer of the GPU has emptied and it has become idle for some time as well. We call this GPU starvation. Obviously, we are not making good use of our resources.

Figure 6-1 CPU Stalls and GPU Starvation Caused by Occlusion Queries

### Problem 2: Query Overhead

We wanted to reduce the overhead for the occlusion queries by grouping objects together. Unfortunately, this approach increases the number of queries (especially if many objects are visible): in addition to the queries for the original objects, we have to add queries for their groupings. So we have improved the good case (many objects are occluded), but the bad case (many objects are visible) is even slower than before.

The number of queries is not the only problem. In the worst case, we might end up filling the entire screen tens of times just for rasterizing the bounding geometry of interior nodes.

## Coherent Hierarchical Culling

###  Idea 1: Being Smart and Guessing
