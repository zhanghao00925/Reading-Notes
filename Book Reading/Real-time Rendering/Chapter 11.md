# Global Illumination

## The Rendering Equation

在实时渲染中，通常只使用局部光照模型。只需要可见点的表面数据就可以计算光照——而这正是gpu最有效地提供的。原语被独立地处理和栅格化，然后被丢弃。在执行在点b的计算时，不能访问点a的照明计算结果。透明度、反射和阴影是全局照明算法的例子。他们使用来自其他物体的信息，而不是被照亮的物体。这些效果大大增加了渲染图像的真实感，并提供线索，帮助观众理解空间关系。同时，它们的模拟也很复杂，可能需要预计算或呈现多个通道来计算一些中间信息。

[Fig 11.2]

[Tab 11.1]

全局照明研究的重点是计算沿着这些路径光传输的方法。当将其应用于实时渲染时，我们常常愿意牺牲一些质量或正确性来进行有效的评估。最常见的两种策略是简化和预先计算。例如，我们可以假设所有的光线在到达眼睛之前都是漫反射的，这种简化可以很好地适用于某些环境。我们还可以离线预先计算一些关于对象间效果的信息，例如生成纹理来记录表面上的照明水平，然后实时地只执行依赖于这些存储值的基本计算。本章将举例说明如何使用这些策略来实现实时的各种全局照明效果。

## General Global Illumination

求解完整渲染方程的算法可以生成令人惊叹的、照片般逼真的图像(图11.3)。但是，对于实时应用程序来说，这些方法的计算开销太大了。那么，为什么要讨论它们呢?

第一个原因是，在静态或部分静态的场景中，这样的算法可以作为预处理运行，存储结果以供渲染期间使用。例如，这是游戏中常见的方法，我们将讨论这类系统的不同方面。

第二个原因是全局光照算法是建立在严格的理论基础上的。它们是直接从渲染方程推导出来的，它们所做的任何近似都会被仔细地分析。在设计实时解决方案时，可以并且应该使用类似的推理方法。即使我们走了捷径，我们也应该知道结果是什么，正确的方法是什么。随着图形硬件变得越来越强大，我们将能够做出更少的妥协，并创建更接近正确物理结果的实时呈现图像。

求解绘制方程的两种常用方法是有限元法和蒙特卡罗法。Radiosity是基于第一种方法的算法;各种形式的射线追踪使用第二种方法。

### Radiosity

关于这个算法已经有很多书了[76,275,1642]，但是其基本思想相对简单。光在环境中反射。你打开一盏灯，照明很快达到平衡。在这种稳定状态下，每个表面都可以被看作是一个光源。基本的辐射算法做了一个简化的假设，即所有的间接光都来自漫反射表面。这个前提不适用于有抛光大理石地板或墙上有大镜子的地方，但对于许多建筑设置来说，这是一个合理的近似。辐射可以遵循一个有效的无限数量的漫反射。使用本章开始介绍的符号，它的轻传输集是LD E。

假设每个表面都由若干个小块组成。对于每个较小的区域，它计算单个的平均辐射值，因此这些小块需要足够小，以捕获所有的照明细节(例如阴影边缘)。但是，它们不需要一对一地匹配底层曲面三角形，甚至不需要在大小上一致。

我们可以推导出第i块的辐射

【11.4】

B i为patch i的辐射度，为辐射度，即补丁我发出的热辐射,ρss是subsurface反照率(9.3节)。只有光源的发射才是非零的。F ij是patch i和j之间的形状因子，形状因子定义为

【11.5】

其中A i是patch i的面积，V (i, j)是点i和点j之间的可见性函数，

【fig 11.4】

radiosity算法的一个重要部分是准确地确定场景中成对patch之间的形状因子。

根据计算得到的形式因子，将所有patch的方程(方程11.4)合并成一个单一的线性系统。然后对系统进行求解，得到每个patch的辐射值。随着补丁数量的增加，由于计算复杂性高，减少这样一个矩阵的成本是相当大的。

由于该算法可伸缩性差且有其他限制，因此很少使用经典的光线来生成照明解决方案。然而，这种预先计算形式因子并在运行时使用它们进行某种形式的光传播的思想在现代实时全球照明系统中仍然很流行。

### Ray Tracing

路径跟踪的唯一缺点是实现高视觉保真度所需的计算复杂性。对于电影质量的图像，可能需要追踪数十亿条路径。已经提出了许多方法来对抗这种效应，而不需要跟踪其他路径。一种流行的技术是重要性抽样。其原理是，通过向大多数光线的照射方向发射更多光线，可以大大减少方差。

## Ambient Occlusion

我们将以最简单的，但在视觉上仍然令人信服的解决方案开始我们的实时替代探索，并逐步建立更复杂的效果贯穿整个章节。

一种基本的全局照明效果是环境光遮蔽(AO)。这种技术是由工业光魔公司(Industrial Light & Magic)的Landis[974]在21世纪初开发的，目的是改善电影《珍珠港》(Pearl Harbor)中电脑生成飞机上的环境照明质量。尽管这种效应的物理基础包含了相当多的简化，但其结果却令人惊讶地似是而非。这种方法在光线缺乏方向性变化且不能显示物体细节的情况下，提供了关于形状的廉价线索。

### Ambient Occlusion Theory

环境光遮蔽的理论背景可以直接从反射方程推导出来。为了简单起见，我们将首先关注Lambertian表面。从这些表面发出的L o照度与表面辐照度e成正比。辐照度是入射照度的余弦加权积分。一般情况下，它取决于表面位置p和表面法向n。同样，为了简单起见，我们假设入射光为常数，对于所有入射方向L, L i (L) = L A，从而得到计算辐照度的公式:

【11.6】

在恒定均匀照度的假设下，辐照度(以及输出的辐照度)不依赖于表面位置或法线，而是在物体上保持恒定。这导致了扁平的外观。

方程11.6没有考虑任何可见性。有些方向可能会被物体的其他部分或场景中的其他物体挡住。这些方向将有不同的入射亮度，而不是亮度。为简单起见，我们将假设来自阻塞方向的传入辐射为零。这忽略了所有可能从场景中其他物体反射的光，并最终从这些被阻挡的方向到达点p，但它极大地简化了推理。因此，我们得到了由Cook和Torrance[285,286]首先提出的方程:

【11.7】

【Fig 11.7】

能见度函数的归一化余弦加权积分称为环境遮挡:

【11.8】

【11.9】

除了k A, Landis[974]还计算了一个平均的未闭合方向，称为弯曲法线。这个方向向量被计算为无阻挡光方向的余弦加权平均值:

【11.10】

【Fig 11.8】

### Visibility and Obscurance

需要仔细定义用于计算环境遮挡因子k A(等于11.8)的可见性函数v(l)。

不幸的是，对于封闭的几何图形来说，可见性函数方法是失败的。想象一个场景，其中有一个封闭的房间，里面有各种各样的物品。所有的表面的k值都是0，因为所有来自表面的射线都会撞击到某物。经验方法试图重现环境遮挡的外观，但不一定要模拟物理可见性，通常在这类场景中效果更好。

一些方法受到Miller的accessibility shading可达性阴影概念的启发[1211]，该概念模拟了表面的角落和缝隙如何捕捉污垢或腐蚀。

茹科夫等。[1970]引入了obscurance的概念,修改环境闭塞的计算用距离映射函数ρ(l)代替可见度函数v(左)。

【11.11】

与v (l),只有两个有效值,1没有交集和0 - tersectionρ(l)是一个连续函数基于射线相交前表面传播的距离。

【Fig 11.9】

尽管试图证明它的物理基础上，掩盖是不正确的物理。然而，它经常给出貌似合理的结果，符合观众的期望。缺点之一是需要手动设置d max的值以获得令人满意的结果。这种妥协经常出现在计算机图形学中，其中一种技术没有直接的物理基础，但“具有感知上的说服力”。目标通常是一个可信的图像，所以这样的技术是可以使用的。也就是说，基于理论的方法的一些优点是它们可以自动工作，并且可以通过对现实世界如何工作的推理进一步改进。

### Accounting for Interreflections

尽管环境光遮蔽产生的结果在视觉上是令人信服的，但它们比全全局照明模拟产生的结果要暗。

【Fig 11.10】

环境遮挡和全全局照明之间的差异的一个重要来源是互反射。公式11.8假设被阻挡方向上的辐射为零，而在现实中，相互反射会从这些方向引入非零辐射。这个差异可以通过增加ka的值来解决。或者obscurance distance mapping function。

以更精确的方式跟踪相互反射是昂贵的，因为它需要解决递归问题。

Stewart和Langer[1699]提出了一种不贵但精确得惊人的近似互反射的方法。这是基于在漫射照明下的Lambertian场景的观察，从给定位置可见的表面位置往往具有相似的亮度。通过假设遮挡方向的辐射强度L i等于当前阴影点的输出辐射强度L o，分解递推，可以得到一个解析表达式:

【11.12】

【11.13】

这个方程将趋向于使环境光遮蔽因子变亮，使它在视觉上更接近全局照明解决方案的结果，包括互反射。效果是高度依赖于ρss的值。基本的近似假设表面颜色在阴影点附近是相同的，从而产生类似color bleeding的效果。

### Precomputed Ambient Occlusion

环境遮挡因子的计算可能会很耗时，而且通常在渲染之前离线完成。预先计算任何与光线有关的信息(包括环境光阻)的过程通常称为烘焙。

计算环境遮挡最常用的方法是蒙特卡罗方法。光线投射并检查与场景的交点，对方程11.8进行数值计算。例如，假设我们随机选取N个方向l均匀分布在正常N周围的半球上，并在这些方向上追踪射线。基于相交结果，我们对可见性函数v进行了评估，环境遮挡可以计算为

【11.14】

环境光阻预计算可以在CPU上进行，也可以在GPU上进行。大多数商业上可用的建模和渲染软件包提供了一个预先计算环境光遮蔽的选项。

遮挡数据对于对象上的每个点都是唯一的。它们通常存储在纹理、卷或网格顶点中。不同存储方法的特点和问题是相似的，不管存储的是什么类型的信号。同样的方法可以用于存储环境光遮蔽、方向光遮蔽或预先计算的照明，如11.5.4节所述。

预先计算的数据也可以用来模拟物体相互之间的环境遮挡效应。Kontkanen和Laine[924,925]将一个对象对其周围环境的环境遮挡效应存储在一个立方体映射中，称为环境遮挡场。他们用二次多项式的倒数来模拟环境光遮蔽值随距离物体的变化。它的系数存储在一个立方体映射中，以模拟遮挡的方向变化。在运行时，利用被遮挡对象的距离和相对位置获取合适的系数，重建遮挡值。

无论我们选择哪种方法来存储环境遮挡值，我们都需要知道我们是在处理一个连续信号。当我们从空间中的一个特定点发射射线时，我们进行采样，当我们在着色之前从这些结果中插入一个值时，我们重建。所有来自信号处理领域的工具都可以用来提高采样-重建过程的质量。

来自育碧的《刺客信条》[1692]和《孤岛惊雷》[1154]系列也使用了一种预先计算的环境光遮蔽来增强其间接照明解决方案。它们从自顶向下的视图呈现世界，并处理生成的深度图来计算大规模的遮挡。根据相邻深度样本的分布情况，采用各种启发式方法对该值进行估计。通过将世界空间的位置投射到纹理空间，生成的世界空间AO地图应用于所有对象。他们称这个方法为World AO。

### Dynamic Computation of Ambient Occlusion

对于静态场景，可以预先计算环境遮挡因子ka和弯曲法向n弯曲。然而，对于物体移动或改变形状的场景，可以通过动态计算这些因素来获得更好的结果。这样做的方法可以分为在对象空间中操作的方法和在屏幕空间中操作的方法。

离线计算环境光遮蔽的方法通常包括从每个表面点向场景投射大量的光线，数十到数百，并检查交叉。这是一个昂贵的操作，实时方法关注的是近似或避免大量计算的方法。

Bunnell[210]通过将曲面建模为放置在网格顶点上的圆盘形元素的集合，计算环境遮挡因子ka和弯曲法线n。之所以选择磁盘，是因为可以通过分析计算一个磁盘与另一个磁盘之间的遮挡，从而避免了投射光线的需要。但是还有一些问题详细信息要参考【文献】。

计算每对元素之间的遮挡是一个O(n 2)阶操作，除了最简单的场景外，这对于其他所有场景都太昂贵了。通过使用简化的曲面表示方法，可以降低成本。Bunnell构造了一个分层的元素树，其中每个节点都是一个磁盘，表示树中低于它的磁盘的聚合。当执行磁盘间遮挡计算时，更高级别的节点用于更远处的表面。这将计算减少到O(n log n)阶，这更加合理。

Evans[444]描述了一种基于符号距离场(SDF)的动态环境遮挡逼近方法。莱特[1910]进一步扩展了在环境光遮蔽中使用符号距离场的方法。Wright执行锥跟踪，而不是使用特定的启发式方法来生成遮挡值。

【Fig 11.12】

Crassin等人[305]在场景的体素表示上下文中描述了类似的方法。他们使用一个稀疏体素八叉树(章节13.10)来存储场景的体素化。他们的计算环境光遮蔽的算法是渲染全局照明效果的更一般方法的一个特例(章节11.5.7)。

Ren等人[1482]将咬合几何近似为一个球体集合(图11.13)。用球谐函数表示被单球遮挡的表面点的可见性函数。

【Fig 11.13】

其他方法具体参考【文献】。

### Screen-Space Methods

对象空间方法的开销与场景复杂度成正比。然而，一些关于遮挡的信息完全可以从屏幕空间数据中推断出来，比如深度和法线。这种方法的成本是恒定的，与场景的细节无关，而只与渲染所用的分辨率有关。

Crytek开发了一种用于Crysis的动态屏幕空间环境闭塞(SSAO)方法[1227]。他们使用z- buffer作为唯一的输入，以全屏方式计算环境光阻。每个像素的环境遮挡因子k A是通过测试一组点来估计的，这些点分布在像素位置周围的一个球面上，对z-buffer进行测试。k A的值是z缓冲区中对应值前面的样本个数的函数。通过的样本数量越少，k A的值就越低。参见图11.14。样本的权值随着距离像素的距离的增加而减小，类似于模糊系数[1970]。注意，由于样本没有使用(n·l) +因子加权，因此产生的环境遮挡是不正确的。而不是只考虑那些在地表以上半球的样本，所有的都被计算和考虑在内。这种简化意味着表面以下的样本在它们不应该被计数的时候被计数。这样做会导致平面变暗，边缘比周围更亮。尽管如此，结果往往是赏心悦目的。参见图11.15。

【Fig 11.14】

【Fig 11.15】

这两种方法的极端简单性很快被工业界和学术界注意到，并产生了大量后续工作。许多方法，如Filion等人[471]在《星际争霸2》中使用的方法和McGuire等人[1174]使用的可伸缩环境模糊法，都使用特定的启发式方法来生成遮挡因子。这些方法具有良好的性能特点，并公开了一些参数，可以手工调整，以达到预期的艺术效果。

其他方法的目的是提供更有原则的方法来计算遮挡。Loos和Sloan[1072]注意到Crytek的方法可以解释为蒙特卡罗积分。他们将计算值称为体积模糊，并将其定义为

【11.15】

其中X是一个三维的,球面点附近,ρ是distance-mapping函数,类似于,方程11.11 d的距离函数,和o (X)是入住率函数,等于零如果X不占领,否则。他们指出,ρ(d)功能没有影响最终的视觉质量,因此使用一个常数函数。在这个假设下，体积模糊是点附近的占用函数的积分。Crytek的方法是对三维邻域随机采样，对样本内的邻域进行评价。Loos和Sloan通过随机采样一个像素的屏幕空间邻域来计算x维的数值积分。

Szirmay-Kalos等人[1733]提出了另一种使用非主要信息的屏幕空间方法，称为volumetric ambient occlusion容量环境闭塞。Bavoil等[119]提出了一种不同的方法来估计局部能见度。他们的灵感来自Max[1145]的地平线绘制技术。他们的方法被称为水平-基于环境遮挡(HBAO)，假设z-buffer中的数据代表一个连续的高度场。

[11.16]

利用linear falloff可以改进成

【11.17】

Jimenez等人[835]也采用了水平方法，他们将这种方法称为地面真值环境光遮蔽(ground-truth ambient occlusion, GTAO)。

【11.18】

【Fig 11.17】

水平面方法中最昂贵的部分是沿着屏幕间距线采样深度缓冲区，以确定水平角。Timonen[1771]提出了一种专门针对改进这一步的性能特性的方法。

深度缓冲并不能很好地表现场景，因为只有距离最近的物体才会被记录在给定的方向上，而我们并不知道它后面发生了什么。许多方法使用各种启发式方法来尝试推断一些关于可见对象厚度的信息。这些近似值在许多情况下是足够的，并且眼睛是宽恕不准确的。虽然有些方法使用多层深度来缓解这个问题，但是由于与呈现引擎的复杂集成和高运行时成本，它们从未得到更广泛的普及。

屏幕空间方法依赖于对z缓冲的重复采样，以形成给定点周围的几何形状的一些简化模型。实验表明，要达到高的视觉质量，需要多达几百个样本。然而，要想用于交互式渲染，最多不超过10到20个样本，甚至更少。Jimenez等人[835]报告称，为了满足60 FPS游戏的性能预算，他们只能使用每个像素一个样本!为了弥合理论与实践之间的差距，屏幕空间方法通常采用某种形式的空间抖动。在最常见的形式中，每个屏幕像素使用稍微不同的随机样本集，这些样本集按径向方向旋转或移动。在AO计算的主要阶段之后，执行全屏过滤。联合双侧滤波(第12.1.1节)用于避免跨面不连续性的滤波，并保留锐边。它利用现有的深度或法线信息进行再严格过滤，只使用属于同一曲面的样本。其中一些方法采用随机变化的采样模式和实验选择的滤波核;其他的使用固定大小的屏幕空间模式(例如，4×4像素)的重复样本集，以及一个限制在该邻域的过滤器。

随着时间的推移，环境光阻计算也经常被超采样[835,1660,1916]。这一过程通常是通过应用不同的采样模式帧并对遮挡因子进行指数平均来完成的。使用上一帧的z-buffer、摄像机转换和动态对象的运动信息，将前一帧的数据重新投影到当前视图。然后将其与当前帧结果混合。

### Shading with Ambient Occlusion

再次考虑反射率方程:

【11.19】

可以变形为

【11.21】

【11.22】

其中

【11.23】

这种形式为我们提供了一个新的视角。方程11.22中的积分可以被认为是对入射亮度L i应用一个方向滤波核K。滤波器K以一种复杂的方式在空间和方向上变化，但它有两个重要的特性。首先，由于点积的存在，它最多只能覆盖点p法线周围的半球。第二，由于分母上的归一化因子，它对整个半球的积分等于1。

执行着色,我们需要计算两个函数的乘积的积分,这一事件光辉L i和过滤函数K在某些情况下,有可能以一个简化的方式描述过滤器和计算这双产品积分以相当低的成本,例如,当L我和K表示使用球形哈尔-莫尼(10.3.2节)。

另一种处理这个方程复杂性的方法是用一个具有类似性质的更简单的滤波器来近似这个滤波器。最常见的选择是归一化余弦核H:

【11.24】

【11.25】

当没有任何东西阻挡射入的光线时，这种近似是准确的。它还覆盖了与我们正在逼近的过滤器相同的角度范围。它完全忽略了可见性，但是环境遮挡k A项仍然存在于方程11.22中，所以在阴影表面上会有一些与可见性相关的暗化。

这意味着，在最简单的形式下，带有环境遮挡的阴影可以通过计算辐照度并乘以环境遮挡值来实现。辐照度可以来自任何来源。例如，它可以从辐照度环境图(章节10.6)中采样。

这个公式也让我们了解到为什么环境光遮蔽对于点状或小面积光源是一个很差的能见度近似值。它们只在表面上形成一个小的立体角——在点灯的情况下是无穷小的——可见性函数对照明积分的值有重要的影响。它几乎以一种二元的方式控制光的贡献。，它要么启用它，要么完全禁用它。忽略可见性，就像我们在公式11.25中做的那样，是一个重要的近似，并且通常不会产生预期的结果。阴影缺乏清晰度，不显示任何预期的方向性，也就是说，似乎不是由特定的灯光产生的。环境光遮蔽对于这种光的可见性建模不是一个好的选择。应该使用其他方法，如阴影映射。然而，值得注意的是，有时小的局部光被用来模拟间接照明。在这种情况下，用环境遮挡值调制它们的贡献是合理的。 ？？？

到目前为止，我们假设我们是在为一个朗伯曲面着色。当处理更复杂、非常数的BRDF时，这个函数不能从积分中提取出来，就像我们在方程11.20中做的那样。对于高光材料，K不仅取决于可见度和法线，还取决于观察方向。典型的微面BRDF的一个叶在整个域中显著改变。用一个单一的、预先确定的形状来近似它是太粗糙而不能产生可信的结果。这就是为什么在漫反射的BRDFs中使用环境光遮蔽是最合理的。其他方法，将在接下来的章节中讨论，更适合更复杂的材料模型。

对于带有环境地图的阴影，Pharr[1412]提出了一种使用GPU的纹理滤波硬件动态执行滤波的替代方案。过滤器K的形状是动态确定的。它的中心是弯曲法线的方向，它的大小取决于ka的值。

## Directional Occlusion

尽管单独使用环境光遮蔽可以极大地提高图像的视觉质量，但它是一个非常简化的模型。当处理大的区域光源时，它给出了一个很差的可见度近似值，更不用说小的或准时的光源了。它也不能正确处理光滑的BRDFs或更复杂的照明设置。

【Fig 11.18】

我们将重点介绍对整个球形或半球形视觉进行编码的方法。，用来描述哪些方向挡住了入射光的方法。虽然这一信息可以用于阴影点灯，但它不是它的主要目的。针对这些特定类型的光的方法——在第7章中广泛讨论——能够获得更好的质量，因为它们只需要为光源的单一位置或方向编码可见性。这里描述的解决方案主要用于为大面积照明或环境照明提供遮挡，其中生成的阴影是柔和的，由近似可见性引起的伪影不明显。此外，这些方法也可以用于在常规阴影技术不可行的情况下提供遮挡，如凹凸贴图细节的自阴影和阴影贴图分辨率不足的超大场景的阴影。

### Precomputed Directional Occlusion

Max[1145]引入了水平映射的概念来描述高场表面的自遮挡。在地平填图中，对于地面上的每一个点，地平的高度角是确定的一些方位角方向，例如，八:北，东北，东，东南，在周围。

这些技术比水平图的存储要求更低，但当未包含的方向集合不像椭圆或圆形时，可能会导致不正确的阴影。

【Fig 11.19】

### Dynamic Computation of Directional Occlusion

许多用于产生环境遮挡的方法也可以用来产生方向性能见度信息。Ren等人[1482]的球谐指数，以及Sloan等人[1655]的屏幕空间变体，都以球谐向量的形式产生可见性。如果使用一个以上的SH波段，这些方法本身提供方向信息。使用更多的波段允许编码的可视性与更精确。

锥追踪方法，如Crassin等人[305]和Wright[1910]的方法，为每条轨迹提供一个遮挡值。由于质量原因，甚至使用多个跟踪来执行环境遮挡估计，因此可用信息已经具有方向性。如果需要在特定方向上的能见度，我们可以追踪更少的视锥细胞。

### Shading with Directional Occlusion

有这么多不同的编码方向遮挡的方法，我们不能提供一个单一的处方来执行阴影。解决方案将取决于我们想要达到的特殊效果。

再次考虑反射系数方程

[11.26]

根据第九章，处理点光源时：

【11.27】

我们可以对区域灯进行类似的推理。在这种情况下，li在任何地方都等于零，除了在被光遮蔽的立体角内，它等于光源发出的辐射。我们称它为L，并假设它在光的立体角上是恒定的。我们可以代替集成在整个球体,Ω,集成在光的立体角,Ω李:

【11.28】

考虑lambert模型

【11.29】

【11.20】

对于环境照明，我们不能限制其集成度，因为照明来自四面八方。我们需要找到一种方法，从方程11。26计算出完整的积分。让我们先考虑兰伯提的BRDF:

【11.30】

可以用球谐方法做：

【11.31】

【11.32】

（TBD）

## Diffuse Global Illumination

接下来的部分涵盖了各种方式模拟不仅遮挡，而且在实时全光反射。它们可以粗略地分为两种算法，即假设光线在到达眼睛之前，会从漫反射面或镜面反射。相应的光路径可以分别写为L(D|S)∗DE或L(D|S)∗SE，许多方法对早期反弹的类型施加了一些约束。第一组的解决方案假设入射光线在阴影点上方平滑地改变，或者完全忽略改变。第二组算法假设在事件方向上有很高的变化率。他们依赖的事实，照明将被访问在一个相对小的固体角度。由于存在这些差异极大的约束，因此分开处理这两组是有益的。我们在这一节介绍漫反射全局照明的方法，下一节介绍高光，最后一节介绍统一的方法。

### Surface Prelighting

辐射和路径跟踪都是为离线使用而设计的。虽然已经有人在实时设置中使用了它们，但其结果仍太不成熟，不能用于生产。目前最常见的做法是使用它们来预先计算与光照相关的信息。这个昂贵的离线过程会提前运行，其结果会被存储起来，稍后在显示过程中使用，以提供高质量的照明。正如在第11.3.4节中提到的，用这种方法对静态场景进行预计算被称为烘焙。

这种做法有一定的限制。如果我们提前执行光照计算，我们就不能在运行时改变场景设置。所有的场景、灯光和材质都需要保持不变。我们不能改变一天的时间，也不能在墙上吹洞。在很多情况下，这种限制是可以接受的。架构可视化可以假设用户只在虚拟环境中走动。游戏对玩家的行为也有限制。在这种应用中，我们可以将几何图形分为静态和动态对象。在预计算过程中使用了静态对象，它们与光照充分交互。静态的墙壁投下阴影，静态的红地毯反射红光。动态对象只作为接收者。它们不会遮挡光线，也不会产生间接照明效果。在这样的场景中，动态几何图形通常被限制在相对较小的范围内，因此它对其余照明的影响可以被忽略或用其他技术建模，以最小的质量损失。例如，动态几何可以使用屏幕空间方法来生成遮挡。一组典型的动态对象包括字符、装饰几何和车辆。

可以预先计算的照明信息的最简单形式是辐照度。对于平坦的，兰伯的表面，以及表面颜色，它充分描述了材料对光线的反应。因为光源的效果是独立于其他光源的，所以动态光可以添加到预先计算的辐照度之上(图11.22)。

【Fig 11.22】

除了在限制最严格的硬件平台上，现在很少使用预先计算的辐照度。因为，根据定义，辐照度是计算给定法线方向，我们不能使用法线映射来提供高频细节。这也意味着只能预先计算平面的辐照度。如果我们需要在动态几何上使用烘焙照明，我们需要其他方法来存储它。这些限制促使人们寻找使用方向组件存储预计算照明的方法。

### Directional Surface Prelighting

一个方法还是使用球谐

球谐和h基的一个问题是它们会出现振子现象(第10.6.1节)。虽然预滤可以缓和这种效果，它也平滑的照明，这可能不总是可取的。在更严格的情况下，例如在低端平台或在渲染虚拟现实时，这种开销可能会令人望而却步。

成本是简单的替代品仍然流行的原因。半衰期2使用自定义的半球基础(第10.3.3节)，存储三个颜色值，为每个样本总共9个系数。环境/高亮/方向(AHD)基础(10.3.3节)也是一个流行的选择，尽管它很简单。它已被用于游戏，如使命召唤[809,998]系列和最后的我们[806]。参见图11.23。

【11.23】

光谱的另一端是为高视觉质量而设计的方法。Neubelt和Pettineo[1268]在游戏顺序为1886(图11.24)中使用纹理映射存储球形高斯函数的系数。代替辐照度，他们存储入射辐照度，它被投影到一组在切线框架中定义的高斯叶(章节10.3.2)。根据特定场景灯光的复杂程度，他们使用5到9个叶。为了产生扩散响应，将球面高斯与沿表面定向的余弦波瓣进行卷积。通过将高斯与镜面BRDF波瓣进行卷积，这种表示也足够精确，可以提供低光泽的镜面效应。Pettineo详细描述了整个系统[1408]。他还提供了一个应用程序的源代码，该应用程序能够烘烤和渲染不同的照明表示。

【11.24】

### Precomputed Transfer

虽然预先计算的照明可以看起来令人惊叹，它也是固有的静态。任何几何形状或照明的改变都可能使整个解决方案无效。就像在现实世界中一样，拉开窗帘(场景中局部的几何体变化)可能会让整个房间充满光(全局的照明变化)。人们花费了大量的研究工作来寻找允许某些类型的变化的解决方案。

如果我们假设场景的几何形状没有改变，只有灯光，我们可以预先计算灯光如何与模型交互。在一定程度上可以预先分析相互反射或地下散射等物间效应，并将结果存储起来，而不需要对实际的辐射值进行操作。接收入射光线并将其转换成对整个场景的亮度分布的描述的功能称为传递函数。预先计算的解决方案称为预先计算传输或预先计算辐射传输(PRT)方法。

与完全离线烘烤照明不同，这些技术确实有明显的运行时成本。在屏幕上显示场景时，我们需要计算特定照明设置的亮度值。为了做到这一点，直接光线的实际数量被“注入”到系统中，然后传递函数被应用到整个场景中。一些方法假设这种直接照明来自环境地图。其他的方案允许照明设置是任意的，并以灵活的方式改变。

【Fig 11.25】

预先计算辐射传输的概念是由Sloan等人引入到图形中[1651]。他们用球谐函数来描述它，但这种方法不必使用SH，基本思想很简单。这使我们能够快速计算整个房间内的全部光线。

最初的PRT论文由Sloan等人[1651]使用相同的推理，但在无限远的照明环境中使用球面谐波表示。它们不是存储场景对监控屏幕的反应，而是通过球谐基函数定义的分布来存储场景对周围光线的反应。通过这样做一些数量的SH波段，他们可以渲染一个场景照明由一个任意的照明环境。他们将照明投射到球形谐波上，将每个结果系数乘以各自的标准化“单位”贡献，然后把它们加在一起，就像我们对监视器所做的那样。

如果我们使用三阶SH作为源和接收器，我们需要为场景中的每个点存储一个9×9的矩阵，这些数据仅用于单色传输。如果我们想要颜色，我们需要三个这样的母单位——每个点的内存数量惊人。

斯隆等人在一年后解决了这个问题[1652]。不是直接存储传输向量或矩阵，而是使用主成分分析(PCA)技术对它们的整个集合进行分析。Sloan等人使用这种方法将转移矩阵的维数从625维(25×25转移矩阵)降至256维。虽然这对于典型的实时应用程序来说还是太高了，但是后来的许多轻传输算法已经采用了PCA作为压缩数据的一种方式。

这种类型的降维本质上是有损的。在极少数情况下，数据会形成一个完美的子空间，但大多数情况下它是近似的，因此将数据投影到它上会导致一些退化。为了提高质量，Sloan等人将转移矩阵集划分为簇，并分别对每个簇执行主成分分析。该过程还包括一个优化步骤，以确保集群边界上没有不连续性。一种允许物体有限变形的扩展也被提出，称为局部变形预计算辐射传输(LDPRT)[1653]。

原始的PRT方法假设无限远的周围照明。虽然这个模型可以很好地模拟室外场景的灯光，但对于室内环境来说，它的限制太大了。然而，正如我们前面提到的，这个概念完全不确定照明的初始来源。Kristensen等人[941]描述了一种方法，该方法计算了一组分散在整个场景中的光的PRT。这对应于拥有大量的“源”基函数。这些光接下来被合并成簇，并且接收几何体被分割成区域，每个区域都受到不同的光子集的影响。这个过程导致对传输数据的显著压缩。在运行时，由任意放置的灯光产生的光照通过对预先计算的集合中最近灯光的数据进行插值来近似。Gilabert和Stefanov[533]在游戏Far Cry 3中使用这种方法来产生间接光照。这种方法的基本形式只能处理点光源。虽然它可以扩展到支持其他类型，但成本会随着每种光的自由度呈指数级增长。

根据这些原则工作的最流行的系统是Geomerics的Enlighten(图11.26)。

【Fig 11.26】

### Storage Methods

无论我们是要使用完全的预计算照明，还是要预先计算传输信息并允许照明的一些变化，结果数据都必须以某种形式存储。对gpu友好的格式是必须的。

光线地图是存储预先计算的光线的最常见方式之一。这些是存储预计算信息的纹理。虽然有时会使用诸如辐照度映射之类的术语来表示存储的特定类型的数据，但术语光映射用于综合描述所有这些。在运行时，使用GPU的内置纹理机制。值通常经过bilinearly过滤，这对于某些表示可能不是完全正确的。例如，当使用AHD表示时，经过插值的D(方向)分量将不再是单位长度，因此需要重新规整。使用插值也意味着A(环境值)和H(高亮值)不完全是我们在采样点直接计算它们的样子。也就是说，即使表示是非线性的，结果通常看起来是可以接受的。

在大多数情况下，光线贴图不使用mipmapping，这通常是不需要的，因为光线贴图的分辨率比典型的反照率贴图或法线贴图小。

为了在纹理中存储光照，对象需要提供一个唯一的参数。当把一个漫反射的颜色纹理映射到一个模型上时，对于网格的不同部分使用相同的纹理区域通常是很好的，特别是如果一个模型的纹理是一般的重复模式。复用光贴图是困难的。光照对网格上的每个点都是独特的，所以每个三角形都需要在光照贴图上占据自己独特的区域。创建参数化的过程从将网格分割成更小的块开始。这可以使用一些启发式自动完成，也可以在创作工具中手工完成。通常情况下，已经出现在其他纹理映射中的分割会被使用。接下来，对每个块进行独立参数化，确保其各部分在纹理空间中不重叠[1057,1617]。在纹理空间中产生的元素称为图表或外壳。最后，所有图表被打包到一个公共纹理中(图11.27)。必须注意，不仅要确保图表不重叠，而且它们的过滤足迹必须保持分开。当渲染一个给定图表(双线性过滤访问四个相邻的图表)时，所有可以访问的文本都应该标记为使用，这样其他图表就不会与它们重叠。否则，图表之间可能会出现流血，其中一个图表的光照可能会在另一个图表上可见。虽然对于光线映射系统来说，为光线映射图之间的间隔提供用户控制的“排水沟”量是相当常见的，但是这种分离是不必要的。

【11.28】

避免出血是mipmapping很少用于光映射的另一个原因。

将图表最优地打包到纹理中是一个NP-complete问题，这意味着没有已知的算法可以生成具有多项式复杂度的理想解。由于实时应用程序在一个纹理中可能有成百上千个图表，所有现实世界的解决方案都使用微调的启发式和仔细优化的代码来快速生成打包[183,233,1036]。如果光映射稍后被块压缩(章节6.2.6)，为了提高压缩质量，可能会在封隔器中添加额外的约束，以确保单个块只包含相似的值。

光贴图的一个常见问题是接缝(图11.29)。因为网格被分割成图表，而且每一个都是独立参数化的，所以不可能确保沿着分割边缘的光线在两边完全相同。这表现为视觉上的不连续性。如果网格是手工分割的，这个问题可以通过在不直接可见的区域分割来避免。但是，这样做是一个费力的过程，并且不能在自动生成参数化时应用。Iwanicki[806]在最终的光贴图上执行一个后期处理，沿着分裂的边缘修改纹理，以最小化两边插值值之间的差异。Liu和Ferguson等人[1058]通过平等约束沿边缘执行插值值匹配，并求解最能保持平滑性的texel值。另一种方法是在创建参数化和包装图表时考虑这种约束。Ray等人[1467]展示了如何使用保持网格的参数化来创建不受seam工件影响的光映射。

【11.29】

预先计算的光照也可以存储在网格的顶点上。缺点是光线的质量取决于网格镶嵌的精细程度。因为这个决定通常是在创作的早期阶段做出的，所以很难确保网格上有足够的顶点来在所有预期的光照情况下看起来很好。此外，镶嵌可能是昂贵的。如果网格被精细地镶嵌，照明信号将被过采样。如果使用了存储光照的定向方法，整个表示需要在GPU的顶点之间进行插值，并传递到像素着色器舞台来执行光照计算。在顶点和像素着色器之间传递如此多的参数是相当少见的，并且生成了现代gpu没有优化的工作负载，这会导致效率低下和性能下降。由于所有这些原因，在顶点上存储预先计算的光照很少被使用。

即使有关入射辐射的信息需要在表面上(除了在做体积渲染时，在第14章中讨论)，我们可以预先计算和体积存储它。这样做，照明可以在空间的任意点被查询，为在预计算阶段没有出现在场景中的物体提供照明。注意，然而，这些对象将不会正确地反射或遮挡照明。

Greger等人[594]提出了辐照度体积，它代表五维(三空间、两方向)辐照度函数，并对辐照度环境图进行稀疏的局部采样。即在空间中有一个三维网格，每个网格点处是一个辐照度环境图。动态对象从最近的映射中插值辐照度值。Greger等人使用两级自适应网格进行空间采样，但也可以使用其他体数据结构，如八叉树[1304,1305]。

容纳照明的体积结构不必是规则的。一个流行的选择是将它存储在一个不规则的点云中，然后将这些点连接起来形成Delaunay四面体(图11.30)。一旦确定了合适的四面体，存储在四角的光线就会被调整，使用已经可用的质心坐标。该操作不需要GPU加速，但它只需要4个插值值，而不是网格上的三线性插值需要8个值。

【Fig 11.31】

照明被预先计算和存储的位置可以手动放置[134,316]或自动放置[809,1812]。它们通常被称为照明探针，或光探针，因为它们探测(或取样)照明信号。该术语不应与“光探头”(第10.4.2节)相混淆，后者是记录在环境地图中的远距离照明。

从一个四面体网格中采样的光线质量高度依赖于该网格的结构，而不仅仅是探针的整体密度。如果它们分布不均匀，所得到的网格可以包含细长的、产生视觉伪影的四面体。如果用手放置探测器，问题可以很容易地得到纠正，但这仍然是一个手动的过程。四面体的结构与场景的几何结构无关，因此，如果处理不当，光线将被插入到墙壁上，并产生失真，就像辐照量一样。

对静态和动态几何图形使用不同的照明存储方法是一种常见的做法。例如，静态网格可以使用光照贴图，而动态对象可以从体积结构中获得光照信息。虽然这种方案很流行，但它会造成不同类型的测量法之间的外观不一致。其中一些差异可以通过正则化来消除，在正则化中照明信息在各个表示中平均。

当烘烤照明时，需要小心计算其值只有在它们是真正有效的。网格常常是不完美的。一些顶点可能被放置在几何体内部，或者网格的某些部分可能会自相交。如果在这些有缺陷的位置上计算入射光，结果将是不正确的。它们会导致不必要的暗色或不正确的无阴影照明流血。Kontkanen和Laine[926]以及Iwanicki和Sloan[809]讨论了可以用来丢弃无效样本的不同启发式方法。

### Dynamic Diffuse Global Illumination

虽然预先计算的照明可以产生令人印象深刻的效果，但它的主要优点也是它的主要缺点——它需要预先计算。这样的脱机进程可能会很长。对于典型的游戏关卡来说，照明烘焙需要花费很多小时是很常见的。因为灯光计算需要很长时间，艺术家通常被迫同时在多个层次上工作，以避免在等待烘烤完成时停机。反过来，这通常会导致渲染资源的过度负载，并导致烘烤时间更长。这种循环会严重影响生产力并导致挫败感。在某些情况下，甚至不可能预先计算光照，因为几何图形在运行时改变或由用户在某种程度上创建。

在全动态环境中模拟全局光照的最早方法之一是基于“即时辐射”[879]。尽管它的名字，这种方法与radiosity算法几乎没有共同之处。在它里面，光线是从光源向外投射的。对于光线击中的每个位置，都放置一盏灯，代表该表面元素的间接照明。这些光源称为虚拟点光源(VPLs)。在很多情况下，一次反弹就足以创造可信的结果。这是一种离线的方法，但它启发了Dachsbacher和Stamminger[321]的一种名为反射阴影地图(RSM)的方法。

与普通的阴影贴图相似(7.4节)，反射阴影贴图是从光的角度渲染的。除了深度之外，它们还储存了关于可见表面的其他信息，如反照率、法线和直接光照(通量)。当执行最后的阴影时，RSM的texels被当作点光源来提供间接照明的单反射。

【11.32】

为了获得高质量的结果并在光移动过程中保持时间稳定性，需要创建大量的间接光。如果创建的太少，它们往往会在RSM重新生成时迅速改变它们的位置，并导致闪烁的伪影。另一方面，从性能的角度来看，使用过多的间接光源是一个挑战。

不同的方法被提出来解决缺乏间接遮挡。详细参考【文献】。

### Light Propagation Volumes

辐射传输理论是模拟电磁辐射在介质中传播的一般方法。它解释了散射、发射和吸收。尽管实时图形力图展示所有这些效果，但除了最简单的情况外，用于这些模拟的方法成本太高，无法直接应用于渲染。然而，一些在现场使用的技术已经被证明在实时图形中是有用的。

光传播体积(LPV)是由Kaplanyan[854]提出的，灵感来自辐射传输中的离散纵坐标法。每个cell都有一个通过它的辐射方向分布。他使用二阶球面谐波来处理这些数据。

【Fig 11.33】

详情请参考文献。

### Voxel-Based Methods

基于体素的方法由Crassin[304]提出，体素锥跟踪全局光照(VXGI)也基于体素化的场景表示。几何图形本身以稀疏体素八叉树的形式存储，见13.10节。关键概念是，这个结构提供了一个类似于mipmap的场景表示，这样就可以快速测试空间的遮挡情况。体素也包含了关于它们所代表的几何体反射的光量的信息。它以定向的形式存储，因为辐射在六个主要的方向上被反射。使用反射阴影贴图，直接光线首先被注入到八叉树的最低层。然后在层次结构中向上传播。

【Fig 11.34】

### Screen-Space Methods

就像屏幕空间环境光遮蔽(第11.3.6节)，一些漫反射的全局光照效果可以仅使用存储在屏幕位置的表面值来模拟[1499]。这些方法不像SSAO那样流行，主要是因为有限的可用数据所产生的人工现象更为明显。

### Other Methods

## Specular Global Illumination

前面介绍的方法主要是为了模拟漫射全局照明。对于有光泽的材料，镜面瓣比用于漫射照明的余弦瓣更紧。如果我们想要展示一种非常闪亮的材料，一种带有薄镜面波瓣的材料，我们需要一种能够提供高频细节的辐射表示。另外，这些条件也意味着对反射率方程的评估只需要有限的实心角度的光照入射，而不像兰伯氏BRDF反射整个半球的光照。

### Localized Environment Maps

目前所讨论的方法还不足以令人信服地渲染抛光材料。对于这些技术，辐射场太粗糙，无法精确编码入射辐射的细节，这使得反射看起来暗淡。如果在同一材质上使用分析光，产生的结果也与高光不一致。一种解决方案是使用更球形的高斯函数或更高阶的SH函数来获得我们需要的细节。

在实时设置中为全局照明提供高光组件的最流行的解决方案是局部环境地图。

【Fig 11.36】

高光表面的位置离环境图的中心越远，得到的结果与实际情况的差异就越大。Brennan[194]和Bjorke[155]提出了解决这一问题的一种方法。他们假设光线来自一个有限大小的球体，半径由用户定义，而不是将入射光线看作来自一个无限远的周围球体。当查找入射辐射时，方向不是直接用于索引环境地图，而是作为来自评估表面位置并与这个球体相交的射线。接下来，计算一个新的方向，从环境地图的中心到交叉口位置。这个向量作为查找方向。参见图11.37。该程序具有在空间中“固定”环境地图的效果。这样做通常被称为视差校正。

【Fig 11.37】

这种技术在游戏中非常流行。当使用光泽材料时，阴影点和与代理形状的交点之间的距离可以用来决定使用预过滤环境地图的哪个级别(图11.38)。这样做可以模拟当我们离开阴影点时BRDF波瓣的增长足迹。

当多个探测覆盖相同的区域时，就可以建立关于如何组合它们的直观规则。例如，探测可以有一个用户设置的优先级参数，使得具有较高值的探测优先于具有较低值的探测，或者它们可以平滑地相互融合。

【Fig 11.38】

不幸的是，该方法的简单化本质导致了各种工件。访问环境地图的表面位置不会有这些物体完全相同的视图，因此纹理的存储结果不是完全正确的。

代理也会导致(有时是严重的)光泄漏。通常，查找会从环境地图的明亮区域返回值，因为简化的光线投射错过了可能导致遮挡的局部几何图形。这个问题有时可以通过使用定向遮挡方法得到缓解(第11.4节)。另一个缓解这一问题的流行策略是使用预先计算的漫射光，它通常以更高的分辨率存储。环境贴图中的值首先被渲染位置的平均漫射光所分割。这样做可以有效地从环境地图中去除平滑的、漫反射的部分，只留下高频部分。当着色执行时，反射乘以在阴影位置的漫射光[384,999]。这样做可以部分缓解反射探头空间精度的不足。

### Dynamic Update of Environment Maps

使用本地化反射探测器要求每个环境映射都需要被渲染和过滤。这项工作通常是离线完成的，但有些情况下可能需要在运行时完成。如果是一款开放世界游戏，每天的时间都在变化，或者世界几何图形是动态生成的，离线处理所有这些地图可能会花费很长时间并影响生产力。

在实践中，一些游戏在运行时渲染反射探测。这种类型的系统需要仔细调优，以免严重影响性能。除了一些琐碎的情况外，不可能在每一帧中都重新渲染所有可见探针，这些假设允许我们在加载时呈现几个探针，而在进入视图时逐步呈现其余的探针，一次呈现一个。即使当我们确实想要在反射探测中渲染动态几何图形时，我们几乎肯定能够以较低的帧率更新探测。我们可以定义渲染反射探针所需的帧时间，并在每一帧中更新固定数量的反射探针。根据每个探测器到摄像机的距离、自上次更新以来的时间和类似的因素来决定更新顺序。在时间预算特别小的情况下，我们甚至可以将单个环境映射的渲染分割为多个帧。例如，我们可以在每一帧中只渲染立方体映射的单个面。

### Voxel-Based Methods

在大多数性能受限的场景中，本地化环境映射是一种很好的解决方案。然而，它们的质量往往不能令人满意。当每帧有更多的可用时间时，可以使用更精细的方法。

体素锥跟踪——在稀疏八叉树[307]和级联版本[1190](11.5.7节)——也可以用于镜面组件。

在离线卷积时，通常使用高质量滤波。这种过滤需要对输入纹理进行多次采样，这在高帧速率下是不可能实现的。科尔伯特和Křivanek[279]开发了一种方法达到相应的过滤质量相对较低的样本数量(64个),使用重要性抽样。

### Planar Reflections

如果反射表面的数量有限，而且它们是平面的，我们可以使用常规的GPU渲染管道来创建从这些表面反射的场景图像。些图像不仅能提供准确的镜面反射，而且还能通过对每幅图像进行一些额外处理来呈现似是而非的光泽效果.

[fig 11.39]

### Screen-Space Methods

就像环境遮挡和漫射全局照明，一些镜面效果可以单独在屏幕空间中计算。这样做比在漫反射情况下稍微精确一些，因为镜面瓣的锐度。关于辐射的信息只需要从反射视点周围有限的立体角来获取，而不是从整个半球来，所以屏幕数据更有可能包含它。这种方法首先由Sousa等人提出[1678]，同时也被其他开发人员发现。这类方法称为屏幕空间反射(SSR)。

给定被着色点的位置、视图向量和法线，我们可以沿着沿着法线反射的视图向量追踪光线，测试与深度缓冲的交点。该测试是通过沿着射线迭代移动、将位置投射到屏幕空间并从该位置检索z缓冲区深度来完成的。如果光线上的点比深度缓冲所代表的几何图形离摄像机更远，这意味着光线在几何图形中，并且检测到命中。然后可以从颜色缓冲中读取相应的值，以获得从跟踪方向入射的亮度值。在有限的距离内，二叉搜索可用于准确定位交叉位置。

McGuire和Mara[1179]注意到，由于透视投影，在均匀的世界空间间隔中步进会造成屏幕空间中沿光线的采样点分布不均匀。靠近相机的部分光线采样不足，因此可能会错过一些撞击事件。距离较远的像素被过采样，因此相同深度的缓冲区像素被多次读取，产生不必要的内存流量和冗余计算。他们建议用数字差分分析仪(DDA)代替在屏幕空间中进行射线行进，这是一种用于光栅线的方法。

屏幕空间反射的基本形式只跟踪一条光线，只能提供镜面反射。然而，完美的镜面是相当罕见的。在现代的、基于物理的渲染管道中，更经常需要glossy反射，SSR也可以用于渲染这些。

Uludag[1798]描述了一种使用分层深度缓冲区(秒19.7.2)加速跟踪的优化。首先，创建层次结构。深度缓冲区被逐步向下采样，每步在每个方向上以2倍的倍数向下采样。较高层次上的像素存储较低层次上的四个对应像素之间的最小深度值。接下来，通过层次结构执行跟踪。如果在一个给定的步骤中光线没有击中存储在单元格中的几何图形，它将被推进到单元格的边界，在下一步使用一个低分辨率的缓冲区。如果射线在当前单元中遇到命中，它将被推进到命中位置，并在下一步使用更高分辨率的缓冲区。当对最高分辨率缓冲区的命中被注册时，跟踪终止(图11.41)。

[Fig 11.41]

屏幕空间反射是一个非常好的工具，可以提供一组特定的效果，比如在平面上对附近物体的局部反射。他们实质上提高了实时镜面照明的质量，但他们没有提供一个完整的解决方案。本章中描述的不同方法通常相互叠加，以交付完整而健壮的系统。屏幕空间反射作为第一层。如果它不能提供准确的结果，局部反射探针被用作备用。如果在给定区域中没有应用任何探测，则使用全局默认探测[1812]。这种类型的设置提供了一种一致和稳健的方式来获得可信的间接镜面贡献，这对于一个可信的外观是特别重要的。

## Unified Approaches

使用路径跟踪来绘制质量可接受的图像所需要的计算量远远超过了即使是快速cpu的能力，所以使用gpu代替。它们的极速和计算单元的灵活性使它们很适合这项任务。实时路径跟踪的应用包括架构遍历和电影渲染的预可视化。这些用例可以接受较低和变化的帧速率。技术，如渐进细化(第13.2节)，可用于提高图像质量时，相机是静止的。高端系统可以使用多个gpu。

相反，游戏需要渲染最终质量的帧，并且需要在预算的时间内始终如一地做到这一点。GPU可能还需要执行除渲染本身之外的任务。例如，像粒子模拟这样的系统经常被转移到GPU来释放一些CPU处理能力。所有这些元素结合在一起使得路径跟踪在今天的渲染游戏中变得不切实际。