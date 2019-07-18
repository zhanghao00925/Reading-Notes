# GAN DISSECTION: VISUALIZING AND UNDERSTANDING GENERATIVE ADVERSARIAL NETWORKS

> David Bau1,2, Jun-Yan Zhu1, Hendrik Strobelt2,3, Bolei Zhou4, Joshua B. Tenenbaum1, William T. Freeman1, Antonio Torralba1,2
1 Massachusetts Institute of Technology, 2 MIT-IBM Watson AI Lab, 3 IBM Research, 4The Chinese University of Hong Kong

[arixiv 1811.10597](https://arxiv.org/abs/1811.10597)

P.S. 文章里的bar上划线标的感觉有点不统一。

## 简介

现在GANs以及其变种，已经能达到很好的效果。然而，GANs的内部到底是如何表示我们视觉世界的，什么造成了GANs结果中的artifacts，什么样的结果会影响GANs的学习，还没有好的答案。

作者在本文中提出了一个分析的框架，来可视化GAN的各个部分，用来理解GANs。本文中作者研究GANs的内部表示。

Figure 1

作者提出了一个一般化的方法来在不同的抽象层次上可视化和理解GANs。作者首先定位一组interpretable units，可能和object concepts有关。这些units的featuremaps和某一类的semantic segentation非常接近。然后，在网络中对一组units进行干预，这样可以造成一类的objects消失或出现。作者使用standard causality metric标准因果关系度量来量化这些units的causal effect因果效应。最后，作者研究了这些causal object units和background的contextual relationship。作者研究了在一张新的图片中，那些位置可以插入object concepts，以及这种干预是如何和图像的其他objects相互作用的。

## 相关工作

Generative Adversarial Networks.

Visualizing deep neural networks

Explanining the decisions of deep neural networks.

## 方法

作者的目标是分析在GAN的生成器$G: z \rightarrow x$z的internal representation中objects比如树，如何encoded的。作者用representation描述一个特定的layer的输出tensor $r$，$r=h(z),x=f(r)=f(h(z))=G(z)$。

因为，$r$有生成图像$x$所需的所有必要信息，所以其一定包含了推断图像中任意可见类$c$的信息。作者研究这些信息在$r$中是如何encode的，作者希望知道希望理解$r$是否以某种方式显式表示了$c$，可以在$P$处的$r$分解成两个部分。

$$
r_{U,P}=(r_{U,P},r_{\bar{U,P}})
$$

在$P$处生成object $c$主要取决于$r_{U,P}$，而对$r_{\bar{U,P}}$不敏感。作者将featuremap的每一个通道称为一个unit $U$，$\mathbb{U,P}$分别表示整个units集合，和在$r$中的featuremap pixels。作者分两个阶段研究$r$的结构。

+  Dissection(解剖)：从大类开始，通过度量r中的每一个unit和每一个类$c$的一致性来判断在$r$中有显式表示的类。
+  Intervention（介入）: 对于在dissection阶段找出的类，作者判断causal sets of units（有因果关系的units），通过强制sets of units激活与否来度量units和objects classes之间的causal effects。

Figure 2

### CHARACTERIZING UNITS BY DISSECTION

$r_{u,\mathbb{P}}$是unit $u$的单通道$h \times w$的featuremap，通常比图像小。作者希望知道这个unit是否encode了一个semantic class，例如tree。受到image classification的启发，作者先获得一个每个类的semantic segmentation $s_c(x)$。然后，通过intersection-over-union（IoU）来度量unit $u$中的thresholded featuremap和segmenttation之间的空间一致性:

$$
\mathrm{IoU}_{u,c} \equiv \frac{\mathbb{E}_z|(r_{u,\mathbb{P}}^\uparrow > t_{u,c} ) \land s_c(x)|}{\mathbb{E}_z|(r_{u,\mathbb{P}}^\uparrow > t_{u,c} ) \lor s_c(x)|} \text{, where } t_{u,c}=\argmax_t \frac{\mathrm{I}(r_{u,\mathbb{P}}^\uparrow > t_{u,c} ; s_c(x)}{\mathrm{H}(r_{u,\mathbb{P}}^\uparrow > t_{u,c},s_c(x)}  
$$

threshold $t_{u,c}$的选择是通过最大化information quality ratio $\mathrm{I/H}$得到的。（P.S论文中这个公式有涉及到另一篇Paper）

使用$\mathrm{IoU}$在每一个unit上对相关的concepts（也就是class）进行排序，然后每个unit都标记上最相关的concept。

Figure 3

一旦定位了一组units和一个object class匹配，第二个问题，就是哪些unit触发了绘制这些objects。作者说，一个unit和output object相关并不意味着它cause（产生了这个）output。

### MEASURING CAUSAL RELATIONSHIPS USING INTERVENTION

通过强制强制在$r$中的一组units $U$激活与否观察$c$是否生成。作者通过强制$r_{U,P}=0$来消融这些units。通过$r_{U,P}=k$来强制插入units，其中$k$的值是针对每个class不同的常量。

+ Original iamge：$x = G(z) \equiv f(r) \equiv f(r_{U,P}, r_{\bar{U,P}})$
+ Image with $U$ ablated: $x_a = f(0, r_{\bar{U,P}})$
+ Image with $U$ inserted: $x_i = f(k, r_{\bar{U,P}})$

通过比较$x_a$和$x_i$可以度量这个因果关系。作者收到前人工作的启发定义了一个对unit的average causal effect（ACE）：

$$
\delta_{U \rightarrow c} \equiv \mathbb{E}_{z,P}[s_c(x_i)] - \mathbb{E}_{z,P}[s_c(x_a)]
$$

**Finding sets of units with high ACE**

对于一个有$d$个units的$r$直接搜索集合$U$开销太大。作者优化一个continuous intervention $\alpha \in [0,1]^d$，$\alpha_u$指示了一个unit的intervention degree（介入程度）。作者最大化$\delta_{\alpha \rightarrow c}$

+ Image with $U$ ablated: $x_a' = f((1-\alpha)\odot r_{\mathbb{U},P}, r_{\mathbb{U},\bar{P}})$
+ Image with $U$ inserted: $x_i' = f(\alpha \odot k + (1-\alpha)\odot r_{\mathbb{U},P}, r_{\mathbb{U},\bar{P}})$
+ Objective: $\delta_{\alpha \rightarrow c} = \mathbb{E}_{z,P}[s_c(x_i')] - \mathbb{E}_{z,P}[s_c(x_a')]$

$r_{\mathbb{U},P}$是在$P$处的all-channel featuremap。作者优化目标函数时还带了L2的正则项，目的是最小化casual unit集合的大小。

Figure 4

## Result

有一些有意思的结果，关于每一层的结果和什么相关（虽然比如靠近Input的是整体的结构，deep的部分是material但是可视化看起来很好）。

然后作者通过这些得到的unit和区域的关系可以修复artifacts，但是这里还是要人来看什么地方是artifacts（😰，花10分钟从512个unit中定位出20个产生artifacts的units），然后把这些unit的输出消融掉，就可以提高一点结果的质量（看起来感觉也一般般）

Figure 8a

但是这些物体也不是都可以去除，一些物体例如椅子就很难完全消除。（效果不稳定）

Figure 9

## 其他

作者说他的方法可以用在Debug GANs等很多场景，局限在于，他的方法没有分析discriminator的部分。也没有比较不同GANs的质量的能力。作者希望之后可以将方法用于VAE等其他生成模型。

这样看，后面Siggraph的文章应该就是基于这个知识，然后利用修改不同功能的unit的value来修改图片，但是在Siggraph的文章中还加入了一下image prior，提高一些质量。