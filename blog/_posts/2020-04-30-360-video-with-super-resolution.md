---
title: 基于超分辨率的360视频的传输
# image: /assets/img/blog/...
description: >
  这是一篇infocomm2020的文章(streaming 360-degree videos with super-resolution)，主要是利用超分辨率(super-resolution)的技术来优化360视频的传输
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [PAPER]
---

360视频可为用户提供身临其境的体验，但与普通视频相比，传输需要更多带宽。最新的360视频流系统使用**viewport-prediction(视口预测)**来减少带宽需求——预测并仅传输用户将要观看的视频内容。但是预测容易出错，导致用户体验质量（QoE）变差。文章设计了一个可以减少带宽需求，同时提高视频质量的360视频传输系统——PARSEC。PARSEC主要的思想就是权衡网络和客户端的计算能力，使用客户端的计算能力来缓解网络传输的压力。PARSEC使用基于超分辨率的方法，在保证视频质量的同时，减少视频的传输带宽。PARSEC解决了与为360度视频流使用超分辨率相关的一系列挑战：大型深度学习模型，缓慢的计算速度以及增强视频质量的变化。为此，PARSEC在较短的视频片段上训练小型微模型，然后将传统的视频编码与超分辨率技术结合起来以克服挑战。我们通过FCC发布的宽带网络traces和4G/LTE网络traces对真实WiFi网络上的PARSEC进行评估。PARSEC在降低带宽需求的同时，大大超过了最新的360视频流系统。

# 主要内容
文章主要想解决的问题就是360视频的传输问题。由于360视频是一个全景的视频，同样的分辨率相比于普通的视频，360视频的数据量要更大。目前比较先进的方法是进行视口预测，只传输用户将要看到的区域。由于预测的精度问题，经常会出现tiles缺失的问题。另外的解决办法就是观看区域传高码率，然后其他区域传低码率。这些非视口区域的视频也会竞争带宽。<br>
怎么解决这个问题呢，文章想到的是使用超分辨率的技术，在客户端重构高分辨率的视频来减少传输的数据量。这里存在的问题是使用深度神经网络进行360视频效率很低(每秒能重构的帧数)，而且网络也会很大(竞争带宽)，以及如何决策哪些tiles是需要进行重构的。<br>
PARSEC主要分为两部分，一部分解决超分网络效率低下的问题，另一部分是tiles的动态决策。

# streaming 360 videos using super-resolution
这一部分的话，作者首先做了一个实验，来验证*训练的视频越长，清晰度越高，模型就越大而且每秒重构的帧数就越小*这个推论。360视频的一个特征就是每一个视频segment都由许多个tiles组成，这样就可以对单独一个tile进行重构。<br>
所以超分这部分就是每一个segment都对应一个网络，然后使用当前segment的tiles来训练这个超分网络，该网络是和segment一起传输到客户端。
![]({{site.data.strings.blog_url}}sr-arch.png)

上图是超分网络的结构，首先从像素中提取视频的高维度特征，然后中间的卷积层用来学习原始视频中缺失的细节，最后deconvolution层将低分辨率视频映射到高分辨率视频。网络的层数是取决于视频的长度和视频重建的目标质量(训练过程中，会手动调整网络的层数，使得网络可以重建成不同分辨率的视频)。训练过程中使用**PSNR**作为损失函数。

# neural-aware ABR algorithm

![]({{site.data.strings.blog_url}}parsec-arch.png)

这一部分，主要任务就是：<br>
1. 决定直接从服务器抓取的tiles
2. 决策被抓取的tiles的码率
3. 决定重构的tiles<br>

ABR算法除了需要视频的的信息(包括segments，tiles等信息)，还需要视口预测的概率模型，以及可用的网络能力和客户端的计算能力。其中tiles的状态有(fetch, generate, miss)。<br>

这部分是很假的，并不是所谓的"neural-aware":

> The client makes the decision about which tiles to fetch or generate using an ABR algorithm that is ‘neural-aware,’ i.e., understands  the tradeoffs between the two methods.

## viewport prediction
视口预测主要就是根据：
1. 离线的视频数据分析
2. 用户的头部追踪traces<br>

来预测接下来的用户行为。这里主要的方法是使用[论文](https://dl.acm.org/doi/pdf/10.1145/3083165.3083180)。

## QoE模型
QoE模型是根据视频播放过程中的视口区域的视频质量来衡量的。<br>
对segments质量的打分(期望))$$E\left(Q\right)$$：<br>
$$E\left(Q\right) = \sum_{i=1}^{N}p_{i}\left(q_{i,D}r_{i,D}+q_{i,G}r_{i,G}\right)$$

<br>
对tiles丢失的打分(期望))$$E\left(M\right)$$:<br>
$$E\left(M\right) = \sum_{i=1}^{N}p_{i}r_{i,M}$$

<br>
对tiles间差异的打分$$V_{s}$$:<br>
$$V_{s} = \sum_{i=1}^{N}StdDev\left[p_{i}\left(q_{i,D}r_{i,D}+q_{i,G}r_{i,G}\right)\right]$$
<br>

对segments间差异的打分$$V_{t}$$:<br>
$$V_{t} = \left| E\left(Q\right) - Q^{t-1}\right|$$
<br>

QoE模型：<br>
$$
max \quad QoE = E\left(Q\right) - \beta E\left(M\right) - \xi \left(V_{s}+V_{t}\right)
$$<br>
$$
\text {s.t.} \quad q_{i,D} \ge r{i,D}
$$<br>
&emsp;&emsp;$$
\quad r_{i,D}+r_{i,G}+r_{i,M} = 1, \quad \forall i = 1,\cdots,N
$$<br>
&emsp;&emsp;$$
\quad \sum_{i=1}^{N}d\left(q_{i,D}\right)r_{i,D} + \delta \le P_{t} 
$$<br>
&emsp;&emsp;$$
\quad \sum_{i=1}^{N}d\left(q_{i,G}\right)r_{i,G} + \delta \le P_{t}
$$

## 贪心算法
使用贪心算法求解(对于每一个segment来说)：<br>
1. 初始化所有的tiles为的质量为0，表示既没有被下载，也没有被重构。
2. 根据视口预测概率模型挑选最有可能被观看的tile，提升它的视频质量(可以是直接下载的，也可以是重构的)
3. 检查提升当前tile的视频质量或者提升剩下的tiles中概率最高的tile的质量，看能不能提升QoE

这样就可以保证每一次选择都是最优的...

# 总结
文章的思路就是窄带高清的思路，借用超分网络来重构视频，使得系统可以尽可能的减少数据的传输量。通过考虑客户端的网络承载能力和计算能力，基于viewport预测的概率模型，贪心地计算出每一个tiles的目标码率(可以是下载的，也可以是重建的)。文章主要的贡献就是：<br>
1. 对每一个segments，使用tiles来训练超分网络
2. 建立一个QoE模型，使用贪心算法求解得到每一个segments中对应的下载tiles子集和重构tiles子集(其中包括每一个tiles分辨率)

算法的结果其实是很依赖于viewport的预测的，对于viewport概率高的tile，会导致网络资源或计算资源像少数的几个tiles倾斜，导致视频整体的质量不高(视频的平滑度)，特别是作者还划分了200个tiles。

<br>
感兴趣的可以[阅读原文](https://www3.cs.stonybrook.edu/~arbhattachar/assets/pdf/infocom20b.pdf)