Our approach relies on a simplifying assumption. We split light paths, treating view-dependent and view-independent light differently. Separating paths has a long history in graphics [Chen et al. 1991; Slusallek et al. 1998; Wallace et al. 1987], allowing significant acceleration of illumination computations. We focus on view-dependent paths, while for view-independent paths we use traditional light maps.

While precomputing all light paths can enable interactive rendering of realistic lighting, reprojecting this data into novel views raises three main challenges. First, the dense angular sampling needed to capture view-dependent effects can impose high memory requirements, since glossiness and complex geometry imply the need for denser sample rates. Second, reprojecting glossy probe samples into a novel view can be challenging and costly. This is because complex reflector and reflected geometry/materials make it hard to find the best samples in the probes for a given novel view. Third, directly reprojecting paths can cause sharpening at glossy reflections of occlusion boundaries, as parallax changes between the probe position and the novel view can sharpen the precomputed blur.

Our approach has three main contributions:
An adaptive light probe parameterization to increase resolution depending on scene geometry and material properties, reducing the overall memory footprint.

An algorithm using efficient reflection estimation and on-the-fly search to gather view-dependent texels from probes, providing high-quality interactive rendering of glossy paths.

To avoid sharpening at glossy occlusion boundaries, we introduce a new approach that splits the convolution effect of the BRDF into two steps. First, we render the probes using materials with lower roughness in precomputation, and second, during rendering we apply efficient, adaptive-footprint bilateral filtering reproducing the original material roughness.

We address **three challenges** of probe-based glossy rendering: first, **reducing memory footprint** of the probes; second, efficiently and accurately reprojecting glossy path information to the **novel view** and third, **avoiding sharpening** at glossy reflection occlusion boundaries. Our method is outlined in Fig. 2

For the first challenge, we maximize the amount of information stored where glossy surfaces are visible by computing a **parameterization** for each probe with more resolution assigned to shinier surfaces and objects with higher geometric complexity (Fig. 2b). We generate this parameterization using *quasi-harmonic maps* [Zayer et al. 2005]. We also precompute scene geometric curvature information which is used at runtime for gathering (Fig. 2e). A visible geometry map is also generated along with a map containing the reflected positions visible in each direction of a probe (Fig. 2c).

For the second challenge, we efficiently render accurate viewdependent paths by introducing a **hierarchical gathering** approach. We first perform trilinear interpolation between probes.We compute the perfect mirror reflected position visible at each point of the novel view, and reproject it into each selected probe while taking specular motion into account (Fig. 2f).We base our approach on specular path perturbation [Chen and Arvo 2000b], but generalize it to arbitrary geometry using curvature approximation. This estimate provides an initialization for a search process performed in probe space to gather the probe texels best corresponding to the – possibly glossy – reflection. We accumulate these points from the probes and blend them according to the material properties at the reflector surface. The gather process is critical to the success of our approach, since it renders our method robust to inaccuracies in the reprojection process and finds the best available data in the probe.

The third challenge occurs because naively reprojecting glossy reflections from the probes can create a *sharpening effect* at occlusion boundaries in the reflections. To overcome this issue, **we introduce a new approach that separates the convolution effect of BRDFs into two steps** (Fig. 2g). We first reduce the roughness of materials in the probe precomputation, and apply bilateral filtering in screen space during rendering. Importantly, we estimate a footprint for the screen space filter that closely reproduces the overall glossiness of the original materials

## PROBE GENERATION AND STORAGE

We first describe the per-probe data and how we compute it. We then present our adaptive parameterization that concentrates probe texels in important regions, reducing memory usage at a given image quality.We also describe additional geometric information needed as part of real-time rendering.




