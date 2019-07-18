# Semantic Photo Manipulation with a Generative Image Prior

## 简介

利用GANs来修改照片还是一个比较由挑战性的工作，有两个难点：

1. GANs很难精确地重现一个输入图片。
2. 在修改之后，新合成的像素通常和原始图片不相匹配。

因此，在本文中，作者通过让从GANs中学到的图像的先验分布适应于一个单独的图片的图像统计（？）。作者的方法能够准确地重建输入图像，同时合成新的内容保持和输入图像外观的一致性。

Deep generative models的作用是提供一个隐式的语义表示，在这个情境下可以直接进行修改，然后当语义修改了之后，可以保留图片的真实感。在将生产模型用于真实图片绘制的道路上，有两个技术挑战：
1. 很难找到一个隐式的编码$z$可以通过深度生成模型重现给定的图像$x$，重建的结果大致上捕捉了输入图像的视觉内容，但是细节上的差距非常明显。
2. 在修改之后，生成的像素通常和真实图像中的内容不相容，使得将生成的内容拼接到原始图像中非常由挑战性。

作者的Key idea是学习一个针对图像的生成网络$G'\approx G$，对于输入图像$x$在修改的区域以外，产生几乎准确的结果$x \approx G'(z)$。

## 相关工作

GAN和Photo Manipulation方面的工作。

## 网络结构

Figure 2

给定一张照片作为输入，首先，通过一个图片生成器重新绘制图片。为了更加准确地重建输入图片，作者的方法不仅优化Latent表示，而且自适应生成器。作者的方法根据每次的修改更新Latent表示，根据修改之后的表示，绘制最后的结果。看上去更加真实。

为了解决真实感问题，作者提出了使用一个image-specific的生成器$G'$它能自适应于一张特定的图片。首先，这个生成模型能够和输入图片产生几乎一样的结果；其次，这个模型$G'$需要和$G$非常接近，使得它们可以共用一个底层的语义表示。通过上面两个目标，$G'$可以在语义修改的同时保留原始图片的视觉细节。

在一个成功的修改中，$G'$不需要输入图像$x$在它的至于中，作者发现，直呼要让修改的区域$z_e=edit(z)$外可以重现绘制出来即可。因此作者定义了一个user stroke binary mask $mask_e$:

$$
\mathcal{L}_{match}=\lVert (G'(z_e)-x)\odot(1-mask_e)\rVert_1
$$

其中$G'$和$G$要有类似的Latent structure。否则修改操作的结果就会不太好。

P.S 这里的Edit操作如何进行没有细讲，使用的方法在作者之前做的一片Paper（GAN Dissection: Visualizing and Understanding Generative Adversarial Networks https://arxiv.org/abs/1811.10597）内。

### preserving semantic representation

为了保证$G'$和原始的$G$有相似的Lantent space structure，在构建$G'$时保留前面的部分层，只在决定fine-grained细粒度的细节的层进行扰动。

为了让$G$的semantic structure不变，作者将网络分成了两个部分，high-level layers $G_H$包含第$1$到第$h$层，fine-grained layers $G_F$包含第$h+1$到$n$层，使得$G(z)\equiv G_F(G_H(z))$。

Figure 3

在构建$G'$时，只需要调整$G_F$的部分。

$$
z_h \equiv G_H(z) \equiv g_h(g_{h-1}(\dots g_1(z)\dots)) \\
G_F(z_h) \equiv g_n(g_{n-1}(\dots g_{h+1}(z_h)\dots))
$$

$h$的选择时根据实验得出的，作者发现$h=n-5$的时候效果比较好。

为了获得好的重建图像$x$的效果需要更新$G_F$部分，但是直接更新权重会造成过拟合，生成器会对于$z$的小的变化过于敏感，出现一些artifacts。

因此，作者训练了一个小网络$R$来产生一些perturbations（扰动）$\delta_i$，对$G_F$每一层的输出乘上一个$1+\delta_i$。每一个$\delta_i$有和$i$层的featuremap相同的channels和dimensions。

$$
G'_F(z_h) \equiv g_n((1+\delta_{n-1})\odot g_{n-1}(\dots ((1+\delta_{h+1})\odot g_{h+1}(z_h)\dots))) \\
G'(z) \equiv G'_F(G_H(z))
$$

perturbation network R 不需要输入，训练开始时随机初始化。为了避免过拟合，作者还添加了一个L2的正则项

### Overall optimization

下面是$G'$的整体上的优化的部分。为了学到这个$G'$，首先固定编辑了的semantic表示$z_e$，和没有编辑的像素$x \odot mask_e$，以及预先学到的$G$

然后根据下面的loss学习perturbation network R：

$$
\mathcal{L} = \mathcal{L}_match + \lambda_{reg}\mathcal{L}_{reg}
$$

作者用standard adam solver设置学习率为0.1，优化1000步。在一个GPU上进行优化需要大概30秒，能够达到的平均PSNR（峰值信噪比）为30.6。

作者说，自己对于每一个图片都产生一个自适应的网络是来自之前deep image prior和deep internal learning方面的启发。这些方法常用于inpainting和super-resolution。区别在于，作者使用GAN来身材图像，以及作者的目标是将修改的语义部分和原始图片混合。

### 其他

其他还有一些是UI和交互上的细节的设计，和网络关系不大。

## 总结

作者说，自己的方法的还是受限于GANs的本身特点，图像质量受到深度的限制。同时Latent space的选择也比较困难，对latent space的操作可能会对场景中一些无关的内容进行修改，在增删物体的部分，也会有不能干净地修改的问题。